---
layout: post
title: "Apache HBase的适用场景"
date: 2015-03-08 08:03
comments: true
categories: [Translation, Big Data]
published: false
---

原文：http://blog.cloudera.com/blog/2011/04/hbase-dos-and-donts/

最近我在[洛杉矶Hadoop用户组](http://www.meetup.com/LA-HUG/)做了一次关于[HBase适用场景](http://www.meetup.com/LA-HUG/pages/Video_from_April_13th_HBASE_DO%27S_and_DON%27TS/)的分享。在场的听众水平都很高，给到了我很多值得深思的反馈。主办方是来自Shopzilla的Jody，我非常感谢他能给我一个在60多位Hadoop使用者面前演讲的机会。可能一些朋友没有机会来洛杉矶参加这次会议，我将分享中的主要内容做了一个整理。如果你没有时间阅读全文，以下是一些摘要：

* HBase很棒，但不是关系型数据库或HDFS的替代者；
* 配置得当才能运行良好；
* 监控，监控，监控，重要的事情要说三遍。

Cloudera是HBase的铁杆粉丝。我们热爱这项技术，热爱这个社区，发现它能适用于非常多的应用场景。HBase如今已经有很多[成功案例](#use-cases)，所以很多公司也在考虑如何将其应用到自己的架构中。我做这次分享以及写这篇文章的动因就是希望能列举出HBase的适用场景，并提醒各位哪些场景是不适用的，以及如何做好HBase的部署。

<!-- more -->

## 何时使用HBase



## <a id="use-cases"></a>使用案例

* Apache HBase: [Powered By HBase Wiki](http://wiki.apache.org/hadoop/Hbase/PoweredBy)
* Mozilla: [Moving Socorro to HBase](http://blog.mozilla.com/webdev/2010/07/26/moving-socorro-to-hbase/)
* Facebook: [Facebook’s New Real-Time Messaging System: HBase](http://highscalability.com/blog/2010/11/16/facebooks-new-real-time-messaging-system-hbase-to-store-135.html)
* StumbleUpon: [HBase at StumbleUpon](http://www.stumbleupon.com/devblog/hbase_at_stumbleupon/)
