---
title: Nginx & PHP-FPM 原理和调优
categories:
 - 源码分析 
tags:
 - nginx
 - php-fpm
---

## NGINX 和 PHP-FPM交互模型 

php-fpm目前也是php标配的组件，nginx和php-fpm使用Fastcgi模式进行交互。我们对比一下apache和php交互的mod_php模式


### MOD_PHP模式

apahce将php作为一个子模块来运行，在编译的时候加入php5_module，当web访问php文件时，apache会调用php5_module来解析php代码。中间通过sapi来传递通信数据。 apache -> httpd -> php5_module -> sapi -> php

![MOD_PHP模式](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/mod_php.png?raw=true)

每次apache获取一次请求，会产生一个进程通过sapi来完成请求，用户过多，并发数多服务器扛不住。

### MOD_FASTCGI模式

- cgi(Common Gateway Interface)是外部程序和Web服务器之间的标准接口，但是每次web请求都会启动cgi和退出cgi，每次都要解析php.ini/重新载入全部扩展/并重初始化全部数据结等初始化过程以及清理过程。
- Fastcgi 事先常驻，把解析php.ini/重新载入全部扩展/并重初始化全部数据结等初始化过程先完成，当有连接过来时，生成或选择子进程处理连接。返回后继续处理其他的连接
  
![MOD_FASTCGI模式](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/mod_fastcgi.png?raw=true)

当有连接来的时候，生成或选择子进程处理连接，处理后可以继续处理其他请求，这样并发数可以


## NGINX

Nginx后台进程包括一个master进程和多个worker进程，master进程主要用来管理worker进程
- 监控worker进程状态，当worker进程退出时，自动重新启动worker进程
- 接收外部信号，向各worker进程发送信号 比如kill -HUP pid重启nginx，master进程重新加载配置文件，再启动新的worker进程，并告诉老的worker进程退出（处理完手头未完成的请求）。

在master进程先建立好需要listen的socket，再fork出子进程，所有的worker子进程在新连接来的时候都可读（子进程在调用accept后会堵塞睡眠，当第一个客户端连接到来后，所有子进程都会唤醒，但只会有一个子进程accept调用成功，其他子进程返回失败，代码中的处理往往在accept返回失败后，继续调用accept，这就是惊群），在读之前先去抢accept_mutex，抢到的那个进程注册listenfd读事件，在读事件里调用accept接受该连接。

当一个worker进程在accept这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完整的请求就是这样的了。我们可以看到，一个请求，完全由worker进程来处理，而且只在一个worker进程中处理。

### 异步非阻塞

事件通常有三种类型，信号、网络事件、定时器。

一个请求分为：1.建立连接 2.接收数据 3.处理数据 4.发送数据

- 阻塞 读写事件没有准备好，就在等，进入内核等待，操作系统把cpu让给别人用了，增加进程数，即为apache模型
- 非阻塞 读写事件没有准备好，马上返回EAGAIN，去看其他的，过会儿再来检查事件是否准备好
- 异步非阻塞 （I/O多路复用（multiplexing）的本质是通过一种机制（系统内核缓冲I/O数据），让单个进程可以监视多个文件描述符，一旦某个描述符就绪（一般是读就绪或写就绪），能够通知程序进行相应的读写操作）系统调用就是像select/poll/epoll/kqueue这样的系统调用。它们提供了一种机制，可以同时监听多个读写事件，调用他们是阻塞的，设置超时时间，当有事件准备好了，就返回去处理。如果没有就返回EAGAIN，我们将它再次加入到epoll里面

补充：UNIX支持同步IO和异步IO，同步IO又分为 阻塞IO 非阻塞IO IO多路复用 信号驱动IO

定时器。由于epoll_wait等函数在调用的时候是可以设置一个超时时间的，所以nginx借助这个超时时间来实现定时器。nginx里面的定时器事件是放在一颗维护定时器的红黑树里面，每次在进入epoll_wait前，先从该红黑树里面拿到所有定时器事件的最小时间，在计算出epoll_wait的超时时间后进入epoll_wait。所以，当没有事件产生，也没有中断信号时，epoll_wait会超时，也就是说，定时器事件到了。这时，nginx会检查所有的超时事件，将他们的状态设置为超时，然后再去处理网络事件。由此可以看出，当我们写nginx代码时，在处理网络事件的回调函数时，通常做的第一个事情就是判断超时，然后再去处理网络事件。

我们可以用一段伪代码来总结一下nginx的事件处理模型：
```
while (true) {
    for t in run_tasks:
        t.handler();
    update_time(&now);
    timeout = ETERNITY;
    for t in wait_tasks: /* sorted already */
        if (t.time <= now) {
            t.timeout_handler();
        } else {
            timeout = t.time - now;
            break;
        }
    nevents = poll_function(events, timeout);
    for i in nevents:
        task t;
        if (events[i].type == READ) {
            t.handler = read_handler;
        } else { /* events[i].type == WRITE */
            t.handler = write_handler;
        }
        run_tasks_add(t);
}
```

### 调优

1. 一般来说nginx 配置文件中对优化比较有作用的为以下几项：

- worker_processes 8; worker进程一般和cpu核数一致，比如2CPU每个4核，那么设置为8，这样避免同一个cpu进程间上下文切换
- worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000; 将8 个进程分配到8 个cpu
- worker_rlimit_nofile 65535;  当一个nginx 进程打开的最多文件描述符数目(每个连接要占用一个文件描述符)，最好和ulimit -n 中保持一致。需要设置linux系统文件描述符的数量大于等于worker_rlimit_nofile，否则最多是linux系统中的，通过sysctl -a | grep fs.file 查看系统文件描述符数量
- use epoll;
- worker_connections 65535; 每个进程允许的最多连接数， 理论上每台nginx 服务器的最大连接数为worker_processes*worker_connections。
- keepalive_timeout 60; 超时时间
- client_header_buffer_size 4k; 客户端请求头部的缓冲区大小，一般不会超过1k，使用系统分页命令getconf PAGESIZE 取得，一般为4k。
- open_file_cache max=65535 inactive=60s; 这个将为打开文件指定缓存，默认是没有启用的，max 指定缓存数量，建议和打开文件数一致，inactive 是指经过多长时间文件没被请求后删除缓存。
- open_file_cache_valid 80s; 多久检查文件缓存的有效信息
- open_file_cache_min_uses 1; open_file_cache 指令中的inactive 参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive 时间内一次没被使用，它将被移除。


2. 关于内核参数的优化：

vi /etc/sysctl.conf

- net.ipv4.tcp_max_tw_buckets = 6000  timewait 的数量，默认是180000。
- net.ipv4.ip_local_port_range = 1024 65000 允许系统打开的端口范围。
- net.ipv4.tcp_tw_recycle = 1 启用timewait 快速回收。
- net.ipv4.tcp_tw_reuse = 1 开启重用。允许将TIME-WAIT sockets 重新用于新的TCP 连接。
- net.ipv4.tcp_syncookies = 1 开启SYN Cookies，当出现SYN 等待队列溢出时，启用cookies 来处理。
- net.core.somaxconn = 262144 web 应用中listen 函数的backlog 默认会给我们内核参数的net.core.somaxconn 限制到128，而nginx 定义的NGX_LISTEN_BACKLOG 默认为511，所以有必要调整这个值。
- net.core.netdev_max_backlog = 262144 每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。
- net.ipv4.tcp_max_orphans = 262144 系统中最多有多少个TCP 套接字不被关联到任何一个用户文件句柄上。如果超过这个数字，孤儿连接将即刻被复位并打印出警告信息。这个限制仅仅是为了防止简单的DoS 攻击，不能过分依靠它或者人为地减小这个值，更应该增加这个值(如果增加了内存之后)。
- net.ipv4.tcp_max_syn_backlog = 262144 记录的那些尚未收到客户端确认信息的连接请求的最大值。对于有128M 内存的系统而言，缺省值是1024，小内存的系统则是128。
- net.ipv4.tcp_timestamps = 0 时间戳可以避免序列号的卷绕。一个1Gbps 的链路肯定会遇到以前用过的序列号。时间戳能够让内核接受这种“异常”的数据包。这里需要将其关掉。
- net.ipv4.tcp_synack_retries = 1 为了打开对端的连接，内核需要发送一个SYN 并附带一个回应前面一个SYN 的ACK。也就是所谓三次握手中的第二次握手。这个设置决定了内核放弃连接之前发送SYN+ACK 包的数量。
- net.ipv4.tcp_syn_retries = 1 在内核放弃建立连接之前发送SYN 包的数量。
- net.ipv4.tcp_fin_timeout = 1 如 果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2 状态的时间。对端可以出错并永远不关闭连接，甚至意外当机。缺省值是60 秒。2.2 内核的通常值是180 秒，3你可以按这个设置，但要记住的是，即使你的机器是一个轻载的WEB 服务器，也有因为大量的死套接字而内存溢出的风险，FIN- WAIT-2 的危险性比FIN-WAIT-1 要小，因为它最多只能吃掉1.5K 内存，但是它们的生存期长些。
- net.ipv4.tcp_keepalive_time = 30 当keepalive 起用的时候，TCP 发送keepalive 消息的频度。缺省是2 小时。

使配置立即生效可使用如下命令：
/sbin/sysctl -p


## PHP-fpm

php-fpm支持三种运行模式，分别为static、ondemand、dynamic，默认为dynamic 。 

- static : 静态模式，启动时分配固定的worker进程。 
- ondemand: 按需分配，当收到用户请求时fork worker进程。 
- dynamic: 动态模式，启动时分配固定的进程。伴随着请求数增加，在设定的浮动范围调整worker进程。


### 运行原理
php-fpm采用master/worker架构设计，与nginx设计风格有点类似。 master进程负责CGI、PHP公共环境的初始化及事件监听操作。worker进程负责请求的处理功能。

master进程

master进程工作流程分为4个阶段

1. cgi初始化阶段：分别调用fcgi_init()和 sapi_startup()函数，注册进程信号以及初始化sapi_globals全局变量。 
2. php环境初始化阶段：由cgi_sapi_module.startup 触发。实际调用php_cgi_startup函数，而php_cgi_startup内部又调用php_module_startup执行。 php_module_startup主要功能：a).加载和解析php配置；b).加载php模块并记入函数符号表(function_table)；c).加载zend扩展 ; d).设置禁用函数和类库配置；e).注册回收内存方法； 
3. php-fpm初始化阶段：执行fpm_init()函数。负责解析php-fpm.conf文件配置，获取进程相关参数（允许进程打开的最大文件数等）,初始化进程池及事件模型等操作。 
4. php-fpm运行阶段：执行fpm_run() 函数，运行后主进程发生阻塞。该阶段分为两部分：fork子进程 和 循环事件。fork子进程部分交由fpm_children_create_initial函数处理（ 注：ondemand模式在fpm_pctl_on_socket_accept函数创建）。循环事件部分通过fpm_event_loop函数处理，其内部是一个死循环，负责事件的收集工作。

worker进程

worker进程分为 接收客户端请求、处理请求、请求结束三个阶段。 

- 接收客户端请求：执行fcgi_accept_request函数，其内部通过调用accept 函数获取客户端请求。注意到accept之前有一个请求锁的操作，这么设计是为了避免请求出现“惊群”的现象。当然，这是一个可选的选项，可以取消该功能。 

```
//请求锁
FCGI_LOCK(req->listen_socket);
req->fd = accept(listen_socket, (struct sockaddr *)&sa, &len);
//释放锁
FCGI_UNLOCK(req->listen_socket);
```

处理请求阶段：首先，分别调用fpm_request_info、php_request_startup获取请求内容及注册全局变量($_GET、$_POST、$_SERVER、$_ENV、$_FILES)；然后根据请求信息调用php_fopen_primary_script访问脚本文件；最后交给php_execute_script执行。php_execute_script内部调用zend_execute_scripts方法将脚本交给zend引擎处理。 

请求结束阶段：执行php_request_shutdown函数。此时 回调register_shutdown_function注册的函数及__destruct()方法，发送响应内容、释放内存等操作。


### 调优

系统层面

- 少安装PHP模块, 费内存
- 调高linux内核打开文件数量

php-fpm层面

- pm = dynamic; dynamic表示php-fpm进程数是动态的，最开始是pm.start_servers指定的数量，如果请求较多，则会自动增加，保证空闲的进程数不小于pm.min_spare_servers，如果进程数较多，也会进行相应清理，保证多余的进程数不多于pm.max_spare_servers   static表示php-fpm进程数是静态的, 进程数自始至终都是pm.max_children指定的数量，不再增加或减少
- pm.max_children = 300; 静态方式下开启的php-fpm进程数量
- pm.start_servers = 20; 动态方式下的起始php-fpm进程数量
- pm.min_spare_servers = 5; 动态方式下的最小php-fpm进程数量
- pm.max_spare_servers = 35; 动态方式下的最大php-fpm进程数量  一个进程大概需要消耗30M 8G内存一般设置100  1G内存最好选静态5个
- request_terminate_timeout = 30; 最大执行时间, 在php.ini中也可以进行配置(max_execution_time)
- request_slowlog_timeout = 2; 开启慢日志
- slowlog = log/$pool.log.slow; 慢日志路径
- rlimit_files = 1024; 增加php-fpm打开文件描述符的限制




php-fpm 关闭：kill -INT`cat /usr/local/php/var/run/php-fpm.pid`

php-fpm 重启：kill -USR2`cat /usr/local/php/var/run/php-fpm.pid`



## 参考 

- https://blog.csdn.net/resilient/article/details/82420863
- http://tengine.taobao.org/book/chapter_02.html
- https://www.cnblogs.com/yueminghai/p/8657861.html
- https://www.cnblogs.com/cocoliu/p/8566193.html

