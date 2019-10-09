---
title: IO模型
categories:
 - Linux
tags:
 - IO模型
 - Redis
 - Nginx
---

# Unix5种IO模型

Unix和Linux有5中I/O模型

![IO模型](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/io-model.png?raw=true)

类比你管理一个国际化景区，有中餐厅/英餐厅/意大利餐厅/西班牙餐厅等等，这些餐厅共用一个厨房。

- 阻塞式IO ：你呆在中餐厅盯着，等中国人来了（等待数据），收钱写菜单（将数据从内核复制到用户空间），再去厨房炒菜并端上来。这样每个餐厅都需要雇一个人在那里候着，浪费人力。
- 非阻塞式IO ： 在餐厅等着不如去打扫景区卫生，每隔5分钟去中餐厅看看有没有人来，有人的话收钱写菜单（将数据从内核复制到用户空间），再去厨房炒菜并端上来。节约了清洁工，很多时间浪费在扫地点到餐厅的路上。
- IO多路复用 ： 景区装了个大屏幕可以看到每个餐厅的录像，你在监控室一直盯着屏幕上哪个餐厅有人（等待数据可选择阻塞），再去对应的餐厅收钱写菜单（将数据从内核复制到用户空间），再去厨房炒菜并端到对应餐厅。
- 信号驱动 ： 景区在每个餐厅门口装了警报器，如果有人进来就能发出语音播报哪个餐厅有人来了，你放下扫把跑去对应的餐厅收钱写菜单（将数据从内核复制到用户空间），再去厨房炒菜并端到对应餐厅。
- 异步IO ： 景区不仅在每个餐厅门口装了警报器，还装了二维码支付点餐器，有人来了并支付和点餐后，你随身携带的机器自动语音播报通知你哪个餐厅点了啥，你放下扫把去厨房炒菜并端到对应餐厅。

前面4种都是同步IO，因为都要收钱写菜单。


## io多路复用select/poll/epoll

io多路复用有select/poll/epoll三种，接着上面的景区装了个大屏幕可以看到每个餐厅的录像，你在监控室一直盯着屏幕上哪个餐厅有人（等待数据可选择阻塞），再去对应的餐厅收钱写菜单（将数据从内核复制到用户空间），再去厨房炒菜并端到对应餐厅。

![io多路复用](https://github.com/funson86/funson86.github.io/blob/master/_posts/image/io-multi.png?raw=true)

- select ： 大屏幕只能分成1024(x86)或者2048(x64)格，每次都要从第一个格子看到最后，眼睛会看花
- poll ： 大屏幕可以分成无限多的格，每次都要从第一个格子看到最后，眼睛也会看花
- epoll ： 大屏幕可以分成无限多的格，监控会把有人的餐厅监控格子放到最前面，瞬间找到哪个餐厅

## Redis的IO多路复用

ae.c 文件中

```
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
```

如果有epoll，则使用epoll，如果有kqueue，则用kqueue，否则默认使用select方法。select 函数是作为 POSIX 标准中的系统调用，在不同版本的操作系统上都会实现，所以将其作为保底方案。

## Nginx的IO多路复用源码

在auto/modules文件中

```
if [ $EVENT_POLL = YES ]; then
    have=NGX_HAVE_POLL . auto/have
    CORE_SRCS="$CORE_SRCS $POLL_SRCS"
    EVENT_MODULES="$EVENT_MODULES $POLL_MODULE"
fi

if [ $NGX_TEST_BUILD_EPOLL = YES ]; then
    have=NGX_HAVE_EPOLL . auto/have
    have=NGX_HAVE_EVENTFD . auto/have
    have=NGX_TEST_BUILD_EPOLL . auto/have
    EVENT_MODULES="$EVENT_MODULES $EPOLL_MODULE"
    CORE_SRCS="$CORE_SRCS $EPOLL_SRCS"
fi
```


# 参考
- https://www.cnblogs.com/diegodu/p/6823855.html