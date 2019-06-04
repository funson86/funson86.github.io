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

- cgi(Common Gateway Interface)是外部程序和Web服务器之间的标准接口，但是每次web请求都会启动cgi和退出cgi，每次都要解析php.ini的初始化过程以及清理过程。
- Fastcgi 事先常驻，把解析php.ini等初始化过程先完成，当有连接过来时，生成或选择子进程处理连接。返回后继续处理其他的连接
  
![MOD_FASTCGI模式](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/mod_fastcgi.png?raw=true)
