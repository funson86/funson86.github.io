---
title: Nginx & PHP-FPM 原理和调优
categories:
 - 源码分析 
tags:
 - nginx
 - php-fpm
---

@[toc]目录

## NGINX 和 PHP-FPM交互模型 

php-fpm目前也是php标配的组件，nginx和php-fpm使用Fastcgi模式进行交互。我们对比一下apache和php交互的mod_php模式

### MOD_PHP模式



