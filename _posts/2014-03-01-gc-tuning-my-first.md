---
layout: post
title: "GC Tuning Action"
description: ""
category: "j2ee"
tags: []
---
　　人生第一次的JVM GC调优，并且是在生产环境动刀，着实有点兴奋，感谢Hugh.Yang的帮助。今天运维同事报障说服务器 jvm old gen 内存占用升高，已经到1.3G，第一反应觉得产生问题的情况有2种可能：一是young gen不足，新生的对象直接在old gen创建；还有一种是在young gen的对象没有被hold住，经过很少的垃圾回收次数就被升级到old gen，但这两种情况的本质是一样的，那就是young gen内存不足。下面是运维同事截的图：  
　　![报障图](/assets/img/gc-tuning-my-first/1.png "报障图")   
　　首先检查tomcat的java\_opts配置，-Xms=512M -Xmx=2048m，按照jvm server端NewRatio=2的默认值，分配到young gen的内存最大只有2048/(2+1)=682m，这样内存确实有点小，最后给运维同事建议临时调大jvm内存到3G，暂时解决young gen内存不足的问题，并打印GC日志到文件做分析，以便找到最优的方案，最后参数配置：-Xms=512M -Xmx=3072m -verbose:gc -XX:+PrintGCDetails  -XX:+PrintGCTimeStamps -Xloggc:/xx/xx/logs/tomcat_gc.log。   
　　![682M的young gen](/assets/img/gc-tuning-my-first/3.png "682M的young gen")   

　　按照以上的JVM参数配置，年轻代的内存最大为1024M，老年代最大内存为2048M,可是，FULL GC还是保持在15分钟左右触发一次，而且老年代的内存到350M的时候就触发FULL GC，以下是截图：   
　　![350M触发FULL GC](/assets/img/gc-tuning-my-first/4.png "350M触发FULL GC")   
　　就当我们无解的时候，Hugh.Yang说-Xms -Xmx 推荐设成一样的，避免运行过程中JVM heap size shrink或expand，谢谢Hugh.Yang帮我们解决了这个问题，修改参数后，重启服务，观察gc log，log信息截图如下：   
　　![最后的GC logs](/assets/img/gc-tuning-my-first/5.png "最后的GC logs")   

　　很不错，这样的调整在１天的时间内都没有再发生FULL GC。但优化才刚开始，因为从最后的GC logs（上图）中，我们可以看到，minor gc的回收过于频繁，大约３s左右１次，而且，虽然young gen的垃圾大部分被回收，但old gen的内存一直在上升，这说明FULL GC迟早会来，且有可能回收会很耗时，因为目前的分配的内存是３G。Hugh.Yang建议先观察日志，等发生至少一次FULL GC再做调整，so，目前在等待中。。。  

－－－－－－－－－－－－－－分割线20130315   
　　上面最后提到minor gc会每3s就执行1次，年轻代是1G，刚开始并没有感觉到异常，后来和另一个每日PV更大的系统做比较，PV大的系统年轻代是600M，也是在3s左右发生一次minor gc，所以，该系统的内存分配确实有点快。   
　　分析过程，使用apache ab压测，jvisual profiler 内存分配情况：   
　　![堆栈](/assets/img/gc-tuning-my-first/8.jpg "堆栈")   
　　![stack tree](/assets/img/gc-tuning-my-first/9.jpg "stack tree")   
　　getMobileViews方法上榜，该方法每次从DB或mem获取5000－6000条数据，当response完成，这5000条数据便会成为垃圾，并发量越大产生垃圾越多。我们做了稍微的优化，思路大概是在本地(tomcat)增加一个计数器，mem也增加一个版本计算器，当本地和远程的计数器不一致的情况下，才从DB或mem获取数据，这样暂时是一个“治标不治本”的解决办法，后期会从减小数据量方面入手。   
　　下面是改前和改后的代码，留存一下。   
　　![优化前代码](/assets/img/gc-tuning-my-first/10.jpg "优化前代码")   
　　![优化后代码](/assets/img/gc-tuning-my-first/13.jpg "优化后代码")
