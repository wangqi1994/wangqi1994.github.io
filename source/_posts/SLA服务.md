---
title: "SLA服务可用性4个9是什么意思？怎么达到？"
catalog: true
toc_nav_num: true
date: 2020-05-18 12:00:00
subtitle: "SLA服务可用性4个9是什么意思？怎么达到？"
article_type: 0 # 0 原创，1 转载
tags:
- [拓展]

categories:
- [拓展]

updateDate: 2020-05-18 12:00:00
top: 0  #置顶
---

SLA服务可用性4个9是什么意思？怎么达到？


SLA：服务等级协议（简称：SLA，全称：service level agreement）。是在一定开销下为保障服务的性能和可用性，服务提供商与用户间定义的一种双方认可的协定。通常这个开销是驱动提供服务质量的主要因素。

SLA的定义来源百度，这到底是什么意思呢？

我们平常经常看到互联网公司喊口号，我们今年一定要做到3个9、4个9，即99.9%、99.99%，甚至还有5个9，即99.999%。

这么多9代表什么意思呢？

首先，SLA的概念，对互联网公司来说就是网站服务可用性的一个保证。9越多代表全年服务可用时间越长服务更可靠，停机时间越短，反之亦然。

这么多9是怎么计算的呢？

全年拿365天做计算吧，看看几个9要停机多久时间做能才能达到！

1年 = 365天 = 8760小时

99.9 = 8760 * 0.1% = 8760 * 0.001 = 8.76小时

99.99 = 8760 * 0.0001 = 0.876小时 = 0.876 * 60 = 52.6分钟

99.999 = 8760 * 0.00001 = 0.0876小时 = 0.0876 * 60 = 5.26分钟

从以上看来，全年停机5.26分钟才能做到99.999%，即5个9。依此类推，要达到6个9及更多9，可说是非常难了吧。

怎么做到更多的9

每个公司对几个9的定义都不一样，互联网公司至少都是99.99吧。像一些政府网站，如社保公积金等，经常故障服务不可用，能做到99.9就不错了。

如果我们提供的服务可用性越低，意味着造成的损失也越大，别的不说，如果是特别重要的时刻，或许就在某一分钟，你可能就会因服务不可用而丢掉一笔大的订单，这都是始料未及的。所以，只要尽可能的提升SLA可用性才能最大化的提高企业生产力。

要做到更多的9，就要不断的监控自己的服务，服务挂掉能及时恢复服务。就像开车出远门，首先得检查轮胎，同时还得准备一个备胎一样的道理。

https://blog.csdn.net/youanyyou/article/details/79406515