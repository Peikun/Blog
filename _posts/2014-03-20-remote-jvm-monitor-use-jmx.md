---
layout: post
title: "remote jvm monitor use jmx"
description: ""
category: "j2ee"
tags: []
---
　　如果需要使用visualvm对远程JVM进行监控，需开启JMX服务，tomcat提供了这样的支持。  
 
```java
-Djava.rmi.server.hostname=127.0.0.1 \
-Dcom.sun.management.jmxremote.port=8060 \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false
```

　　使用JDK7，visualvm还无法对远程JVM进行profiler，JDK8已经支持。   

　　介绍JMX不错的文章：   
　　[JMX-维基百科](http://zh.wikipedia.org/wiki/JMX "JMX-维基百科")   
　　[Java SE 6 新特性: JMX 与系统管理](http://www.ibm.com/developerworks/cn/java/j-lo-jse63/ "Java SE 6 新特性: JMX 与系统管理")
