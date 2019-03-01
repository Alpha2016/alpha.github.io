---
layout:     post
title:      理解多线程概念及Master-Worker模式
subtitle:   理解多线程概念及Master-Worker模式
date:       2019-03-02
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - 多线程
    - master-worker 模式
    - worker 线程
---

&ensp;&ensp;Master-Worker模式是常用的并行设计模式。核心思想是，系统由两个角色组成，Master和Worker，Master负责接收和分配任务，Worker负责处理子任务。任务处理过程中，Master还负责监督任务进展和Worker的健康状态；Master将接收Client提交的任务，并将任务的进展汇总反馈给Client。有时我们希望即使在线程函数完成时也保持工作线程保持活动状态，以便我们可以重新提交不同的作业（通常在线程池中）。这种类似于工作者状态，而不是常规的线程，执行完任务就被销毁，worker 线程意味着没有任务则处于活动（待命）状态。<br />

&ensp;&ensp;Master-Worker是一种将串行任务并行化的方案，模式满足于可以将大任务划分为小任务的场景，是一种分而治之的设计理念。通过多线程或者多进程多机器的模式，可以将小任务处理分发给更多的CPU处理，降低单个CPU的计算量，通过并发/并行提高任务的完成速度，提高系统的性能。 同时，如果有需要，Master进程不需要等待所有子任务都完成计算，就可以根据已有的部分结果集计算最终结果集。<br />

如图：
 ![Master Worker 模式](https://alpha2016.github.io/img/2019-03-02-master-worker-model.jpg "Master Worker 模式")


工作线程回收需要满足三个条件：<br />
1)  参数allowCoreThreadTimeOut为true<br />
2)  该线程在keepAliveTime时间内获取不到任务，即空闲这么长时间<br />
3)  当前线程池大小 > 核心线程池大小corePoolSize。<br />

延伸阅读：

1. [CSDN 文章 多线程之Master-Worker工作模式学习](https://blog.csdn.net/a347911/article/details/53421102 "多线程之Master-Worker工作模式学习")
2. [掘金文章 理解JAVA线程池](https://juejin.im/entry/58fada5d570c350058d3aaad "深入理解JAVA线程池") 
3. [sf文章 理解和使用线程池](https://segmentfault.com/a/1190000015808897 "理解和使用线程池") 
