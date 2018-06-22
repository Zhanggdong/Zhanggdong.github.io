---
title: Redis面试题整理
date: 2018-05-26 12:12:57
categories: 
- Redis
---

### 1、Redis为什么单线程的，能说说它的原理吗

Redis使用了单线程架构和I/O多路复用模型来实现高性能的内存数据库服务。Redis使用了单线程架构，预防了多线程可能产生的竞争问题，但是也会引入另外的问题。Redis单线程架构导致无法充分利用CPU多核特性，通常的做法是在一台机器上部署多个Redis实例。

那么Redis使用单线程模型，为什么还那么快：

第一，纯内存访问，Redis将所有数据放在内存中，内存的响应时长大约为100纳秒，这是Redis达到每秒万级别访问的重要基础。
第二，非阻塞I/O，Redis使用epoll作为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不在网络I/O上浪费过多的时间。

第三，单线程避免了线程切换和竞态产生的消耗。



### 2、mySQL里有2000w数据，redis中只存20w的数据，如何保证redis中的数据都是热点数据

Redis 内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。redis 提供 6种数据淘汰策略：

* volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
* volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
* volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
* allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
* allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
* no-enviction（驱逐）：禁止驱逐数据



### 3、缓存穿透可以介绍⼀一下么？你认为应该如何解决这个问题

缓存穿透是指用户查询数据，在数据库没有，自然在缓存中也不会有。这样就导致用户查询的时候，在缓存中找不到，每次都要去数据库中查询。

解决思路：

1，如果查询[数据库](http://cpro.baidu.com/cpro/ui/uijs.php?adclass=0&app_id=0&c=news&cf=1001&ch=0&di=128&fv=17&is_app=0&jk=9d81f77002e20c6d&k=%CA%FD%BE%DD%BF%E2&k0=%CA%FD%BE%DD%BF%E2&kdi0=0&luki=7&mcpm=0&n=10&p=baidu&q=smileking_cpr&rb=0&rs=1&seller_id=1&sid=6d0ce20270f7819d&ssp2=1&stid=9&t=tpclicked3_hc&td=1682280&tu=u1682280&u=http%3A%2F%2Fwww.th7.cn%2Fdb%2Fnosql%2F201510%2F136276.shtml&urlid=0)也为空，直接设置一个默认值存放到缓存，这样第二次到缓冲中获取就有值了，而不会继续访问数据库，这种办法最简单粗暴。

2，根据缓存数据Key的规则。例如我们公司是做机顶盒的，缓存数据以Mac为Key，Mac是有规则，如果不符合规则就过滤掉，这样可以过滤一部分查询。在做缓存规划的时候，Key有一定规则的话，可以采取这种办法。这种办法只能缓解一部分的压力，过滤和系统无关的查询，但是无法根治。

3，采用布隆[过滤器](http://cpro.baidu.com/cpro/ui/uijs.php?adclass=0&app_id=0&c=news&cf=1001&ch=0&di=128&fv=17&is_app=0&jk=9d81f77002e20c6d&k=%B9%FD%C2%CB%C6%F7&k0=%B9%FD%C2%CB%C6%F7&kdi0=0&luki=6&mcpm=0&n=10&p=baidu&q=smileking_cpr&rb=0&rs=1&seller_id=1&sid=6d0ce20270f7819d&ssp2=1&stid=9&t=tpclicked3_hc&td=1682280&tu=u1682280&u=http%3A%2F%2Fwww.th7.cn%2Fdb%2Fnosql%2F201510%2F136276.shtml&urlid=0)，将所有可能存在的数据哈希到一个足够大的BitSet中，不存在的数据将会被拦截掉，从而避免了对[底层](http://cpro.baidu.com/cpro/ui/uijs.php?adclass=0&app_id=0&c=news&cf=1001&ch=0&di=128&fv=17&is_app=0&jk=9d81f77002e20c6d&k=%B5%D7%B2%E3&k0=%B5%D7%B2%E3&kdi0=0&luki=2&mcpm=0&n=10&p=baidu&q=smileking_cpr&rb=0&rs=1&seller_id=1&sid=6d0ce20270f7819d&ssp2=1&stid=9&t=tpclicked3_hc&td=1682280&tu=u1682280&u=http%3A%2F%2Fwww.th7.cn%2Fdb%2Fnosql%2F201510%2F136276.shtml&urlid=0)存储系统的查询压力。关于布隆[过滤器](http://cpro.baidu.com/cpro/ui/uijs.php?adclass=0&app_id=0&c=news&cf=1001&ch=0&di=128&fv=17&is_app=0&jk=9d81f77002e20c6d&k=%B9%FD%C2%CB%C6%F7&k0=%B9%FD%C2%CB%C6%F7&kdi0=0&luki=6&mcpm=0&n=10&p=baidu&q=smileking_cpr&rb=0&rs=1&seller_id=1&sid=6d0ce20270f7819d&ssp2=1&stid=9&t=tpclicked3_hc&td=1682280&tu=u1682280&u=http%3A%2F%2Fwww.th7.cn%2Fdb%2Fnosql%2F201510%2F136276.shtml&urlid=0)，详情查看：基于BitSet的布隆过滤器(Bloom Filter) 

大并发的缓存穿透会导致缓存雪崩。



