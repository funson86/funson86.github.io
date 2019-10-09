---
title: memcache和redis比较
categories:
 - 源码分析
tags:
 - memcache
 - redis
---

# Memcache

memcached是一款服务器缓存软件，基于libvent开发，使用的多线程模型，主线程listenaccept，工作线程处理消息。多线程非阻塞IO复用模型

## 网络模型

1. 创建主线程和若干个工作线程（和nginx/php-fpm比较类似）；
2. 主线程监听、接受连接；
3. 然后将连接信息分发（求余）到工作线程，每个工作线程有一个conn_queue处理队列；
4. 工作线程从conn_queue中取连接，处理用户的命令请求并返回数据；

[memcache网络模型](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/memcache-network.jpg)

[memcache网络模型](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/memcache-network1.jpg)

多线程模型可以发挥多核作用，但是引入了cache coherency和锁的问题，比如，Memcached最常用的stats 命令，实际Memcached所有操作都要对这个全局变量加锁，进行计数等工作，带来了性能损耗。


# Redis

## 网络模型

Redis采用单线程模型，通过IO多路复用来监听多个连接，非阻塞IO，同时单线程串行执行避免了不必要的锁的开销。

但是Redis也提供了一些简单的计算功能，比如排序、聚合等，对于这些操作，单线程模型实际会严重影响整体吞吐量，CPU计算过程中，整个IO调度都是被阻塞住的。 无法利用CPU多核

# 区别


## 内存管理

Memcached使用预分配的内存池的方式，使用slab和大小不同的chunk来管理内存，Item根据大小选择合适的chunk存储，内存池的方式可以省去申请/释放内存的开销，并且能减小内存碎片产生，但这种方式也会带来一定程度上的空间浪费，并且在内存仍然有很大空间时，新的数据也可能会被剔除，原因可以参考Timyang的文章：http://timyang.net/data/Memcached-lru-evictions/

Redis使用现场申请内存的方式来存储数据，并且很少使用free-list等方式来优化内存分配，会在一定程度上存在内存碎片，Redis跟据存储命令参数，会把带过期时间的数据单独存放在一起，并把它们称为临时数据，非临时数据是永远不会被剔除的，即便物理内存不够，导致swap也不会剔除任何非临时数据(但会尝试剔除部分临时数据)，这点上Redis更适合作为存储而不是cache。


## 数据一致性

Memcached提供了cas命令，可以保证多个并发访问操作同一份数据的一致性问题。 

Redis没有提供cas 命令，并不能保证这点，不过Redis提供了事务的功能，可以保证一串命令的原子性，中间不会被任何操作打断。

### 存储方式

Memcached基本只支持简单的key-value存储，不支持枚举，只能在内存中不支持持久化、不支持复制等功能，单个value至支持1MB。

Redis除key/value之外，还支持list,set,sorted set,hash等众多数据结构，提供了KEYS进行枚举操作，但不能在线上使用，如果需要枚举线上数据，Redis提供了工具可以直接扫描其dump文件，枚举出所有数据，Redis还同时提供了持久化（RDB和AOF）和复制等功能。单个value支持1G，支持事务，支持pub/sub消息订阅机制

# 参考
- https://www.cnblogs.com/phper12580/p/8027113.html