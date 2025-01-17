
**大纲**


**1\.如何优化每秒十万QPS的社交APP的JVM性能(增加S区大小 \+ 优化内存碎片)**


**2\.如何对垂直电商APP后台系统的FGC进行深度优化(定制JVM参数模版)**


**3\.不合理设置JVM参数可能导致频繁FGC(优化反射的软引用被每次YGC回收)**


**4\.线上系统每天数十次FGC导致频繁卡顿的优化(大对象问题)**


**5\.电商大促活动下严重FGC导致系统直接卡死的优化(System.gc()导致)**


**6\.问题汇总**


 


**1\.如何优化每秒十万QPS的社交APP的JVM性能(增加S区大小 \+ 优化内存碎片)**


**(1\)案例背景**


**(2\)高并发查询导致对象快速进入老年代**


**(3\)老年代必然会触发频繁GC**


**(4\)优化前的线上系统JVM参数**


**(5\)频繁Full GC导致的大量内存碎片**


**(6\)这个案例如何进行优化**


 


**(1\)案例背景**


本案例的背景是一个高峰期每秒有十万QPS的社交APP。该APP每日有数百万的日活用户，用户操作最多的功能是浏览某个陌生人的个人页面，流量最大的功能模块是个人主页模块，并且高峰期在晚上。


 


所以会有大量活跃用户在一个集中的时间段内频繁的访问个人主页数据，而这类数据的量还通常很大，要包含很多信息。通常一个个人主页的数据甚至可能有几M，大致可以认为：一次个人主页的查询，就会加载出大概5M的数据。而且一般在高峰期内，一些活跃用户会连续点击他感兴趣的个人主页，比如连续1个小时都在不停的点击。


 


所以该社交APP的高峰期QPS是很高的，假设这个社交APP流量最大的个人主页模块高峰期最多每秒有10万QPS。当然在底层存储中，这些个人主页数据一定是基于缓存来存放的，个人主页模块会基于Redis缓存来查询个人主页数据。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d50681ad1ddf49ba9db5c027983bd068~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=nJD28zut2VC9YG%2FSfigMQMPkfFI%3D)
**(2\)高并发查询导致对象快速进入老年代**


由于每秒并发量太高，导致在高峰期这个系统的新生代Eden区被迅速填满并频繁触发YGC。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/98691981e0ab416b9e87917ed4936fec~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=Kti567bP1IUkJ0ADgMzzA8po5%2BU%3D)
而且每次在YGC时，还有很多请求是没处理完毕的。因为每秒请求太多，所以在触发YGC一瞬间，必然有很多请求没处理完。这就导致每次YGC时，Eden区都会有很多对象需要存活下来。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/df4808c041534de2977df11bb605b7b4~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=nqYcq%2BB%2Fm9o1KM6pBdMN3r8O9WE%3D)
因此在高峰期经常出现YGC后存活对象较多，在S区中放不下的问题。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d2de9bf825024be1a483b7bb9f873ae7~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=UhLAYbuX9q2OkeCQtM273YUwPys%3D)
于是又会导致大量对象快速进入老年代，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/cf9bf9126459459482022581704bdf21~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=ZTmPQHwV%2Bd6Y6%2FiuFxJhrkelaNw%3D)
**(3\)老年代必然会触发频繁GC**


一旦在高并发场景下YGC后存活对象过多，导致对象快速进入老年代，必然会频繁触发老年代GC，对老年代进行垃圾回收。所以上述APP在高峰期会出现主页服务对应的JVM频繁发生老年代GC，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e4b314feb5fc470e8c6f56f2821feb7f~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=xSlVDlqB9rTYFmx7sbh9KgXH8RE%3D)
**(4\)优化前的线上系统JVM参数**


针对上述场景，最核心的优化点是：


一.增加机器，尽量让每台机器承载更少的并发请求，减轻压力


二.给新生代的Survivor区更大内存空间，让每次YGC后的存活对象停留在Survior区，别进入老年代


 


但是这里先不考虑上述优化，在优化前的线上系统中，JVM有两个比较关键的参数如下：



```
 -XX:+UseCMSCompactAtFullCollection
 -XX:CMSFullGCsBeforeCompaction=5
```

由于CMS垃圾回收器采用标记清理算法，所以CMS回收后会造成大量的内存碎片。上述两个参数就指定了：在5次FGC后会触发一次Compaction压缩操作。这个压缩操作会把存活对象放到紧邻在一起，避免出现大量内存碎片。


 


**(5\)频繁Full GC导致的大量内存碎片**



```
 -XX:+UseCMSCompactAtFullCollection
 -XX:CMSFullGCsBeforeCompaction=5
```

上述参数会设置5次FGC后才进行一次压缩操作，以此来解决内存碎片问题，以便可以空出大片连续可用内存空间。


 


所以这就导致这5次FGC过程中，每次FGC后都会产生大量的内存碎片。大量内存碎片又会导致很多问题，其中一个问题就是提高了FGC频率。


 


因为触发老年代GC的一个重要条件：就是YGC后的存活对象无法放入Survivior要放入老年代。如果老年代也没足够的连续可用内存放这些对象，那就必须触发FGC了。


 


所以假设一次FGC过后，老年代中有一部分内存里都是大量的内存碎片。没法放入完整的一些大对象，只有部分内存是连续可用的内存空间。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2729ad7388e84f32b552b48ea284b148~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=gsxia3AC9lcokoo6XZvWW43Z5YQ%3D)
大量对象快速进入老年代会导致老年代的连续可用内存很快就满了，此时很多内存碎片是无法放入更多对象的，于是就会触发一次FGC。


 


比如老年代有2G内存，其中1\.5G是连续可用的，0\.5G是内存碎片。如果老年代中都是连续空闲内存，则对象占用达将近2G时才会触发FGC。但现在对象占用达1\.5G就触发FGC了，剩下0\.5G是没法放入存活对象的。所以就会产生如下的问题：每进行一次FGC，老年代就会产生更多内存碎片，内存碎片越来越多。内存碎片越来越多会导致连续可用内存越来越少，更快触发下次FGC。直到几次FGC后，才会触发一次Compaction操作去整理内存碎片。


 


**(6\)这个案例如何进行优化**


**一.增加新生代和Survivor区大小**


用jstat分析各机器的JVM运行状况，然后判断每次YGC后存活对象大小。然后增加Survivor区的内存，避免对象快速进入老年代。


 


虽然增加了新生代和Survivor区大小，但还是会慢慢有对象进入老年代。毕竟系统负载很高，彻底让对象不进入老年代也很难做到。所以当时增加新生代和Survivor区大小后，每小时还是会有一次FGC。


 


**二.优化CMS内存碎片**


针对CMS内存碎片问题进行优化，在降低FGC频率后，务必设置如下参数：



```
 -XX:+UseCMSCompactAtFullCollection
 -XX:CMSFullGCsBeforeCompaction=0
```

这两个参数的意思是：每次FGC后都整理一下内存碎片。如果进行很多次FGC才整理一下内存碎片：那么每次FGC过后，老年代内存碎片会越来越多，下次FGC会更快到来。也就是如果不及时解决CMS内存碎片问题，就会导致FGC越来越频繁。比如第一次FGC是一小时后，第二次是40分钟后，第三次是20分钟后。


 


**2\.如何对垂直电商APP后台系统的FGC进行深度优化(定制JVM参数模版)**


**(1\)垂直电商业务背景**


**(2\)垂直电商APP的JVM性能问题**


**(3\)公司级别的JVM参数模板**


**(4\)如何优化每次FGC的性能**


**(5\)采用JVM参数模板后的效果**


 


**(1\)垂直电商业务背景**


国内有很多中小型垂直类电商公司的，主要做一些细分领域的电商业务。比如有的APP专门做消费分期类电商业务，在该APP里购物可分期付费。有的APP专门做服装定制、有的APP是做时尚潮流服饰等。


 


某垂直电商APP，注册用户数百万，每日活跃用户也就几十万。每天APP的整体请求量也就几千万，高峰期的QPS也就每秒数百请求。


 


这个APP虽然不大，看起来很普通，且它的后台系统也不会有多大压力。但同样也可能会有JVM相关的性能问题，需要进行一些细致的优化。


 


**(2\)垂直电商APP的JVM性能问题**


类似这样的一个垂直电商APP，它会出现哪些JVM性能问题呢？


 


问题就出在类似这样的一个创业型公司：虽然有少数架构师，但大部分一线工程师可能对JVM都没那么精通。架构师又没那么多精力把控细节的地方，所以直接导致一个很大的问题，就是大部分一线工程师开发完系统后，上线时不对JVM进行参数设置。可能很多时候都是使用一些默认JVM参数，当系统负载逐渐增高时这些默认参数就会有问题。


 


如果不设置\-Xmx、\-Xms之类的堆内存大小：那么启动一个系统，默认会给堆内存几百M、新生代和老年代也是几百M。


 


所以该公司的很多后台系统，基本都是采用默认JVM参数部署启动的。前期没什么问题，但中后期开始，有一定用户量和负载就会出现问题了。


 


默认参数下：新生代内存过小，会导致S区内存过小，同时Eden区也会过小。Eden区过小，就会导致频繁触发YGC。Survivor区过小，就会导致放不下YGC后的存活对象，只能进入老年代。从而导致老年代很快就会放满了，然后频繁触发FGC。


 


所以该公司的垂直电商APP的各个系统通过jstat分析JVM GC后发现：基本上在高峰期的时候，每小时都会发生好几次FGC。


 


一般在正常情况下，FGC都是以天为单位发生的。比如每天发生一次FGC，或者几天发生一次FGC。如果每小时都发生几次FGC，那么就会导致系统每小时都卡顿好几次。


 


所以我们可以在分析系统情况后，给该公司定制一套JVM参数模板。在大部分工程师都对JVM优化不是很精通的情况下：通过推行一个JVM参数模板，可让各系统短时间内就优化好JVM的性能。


 


**(3\)公司级别的JVM参数模板**


其实这个公司级别的或者团队级别的JVM参数模板，是很有用的，因为并不是每位一线开发都精通JVM的核心运行原理和性能优化。


 


所以作为一个团队的Leader，或者是一个中小型公司的架构师，那么必然需要为团队或者公司定制一套基本的JVM参数模板。然后尽量让大部分系统套用这个模板，基本保证JVM性能别太差。避免初中级工程师直接使用默认的JVM参数，比如可能8G内存的机器，JVM堆内存只分配了几百M。


 


下面是为该公司定制出来的、适合这种创业公司的、JVM参数模板：



```
 -Xms4096M -Xmx4096M -Xmn3072M -Xss1M 
 -XX:PermSize=256M -XX:MaxPermSize=256M 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:CMSInitiatingOccupancyFraction=92 
 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0
```

为什么如此定制JVM参数模板呢？


 


**一.首先8G内存的机器，分配4G内存给JVM堆，是比较合理的**


因为可能还有其他进程会使用内存，一般不让JVM堆内存占满机器内存。


 


**二.然后分配3G内存给新生代(因为基本高峰期每次YGC存活几十M对象)**


之所以分配3G是因为能尽量让新生代大一些，从而让S区有300M左右。假设用默认的JVM参数，可能新生代就几百M，Survivor区就几十M。根据对该业务系统的分析，每次垃圾回收过后存活对象可能会有几十M。因为在垃圾回收时可能有部分请求没处理完，此时会有几十M对象存活。在默认参数下很容易触发动态年龄判定规则，让部分对象进入老年代。


 


所以应该给新生代更大内存空间，让Survivor区的空间也更大。这样即使在YGC瞬间有部分请求没处理完毕，有几十M的存活对象。这时在几百M的Survivor空间中也可以轻松放下，而不会进入老年代。


 


基本在这个内存分配下，该公司的大部分后台业务系统都没问题了。不同系统运行的情况略有不同，但基本每次YGC后都会存活几十M对象。所以这个JVM参数模板，都可以适用。


 


只要按上述JVM参数模板分配内存，那么对象进入老年代速度会很慢。该公司的全部系统，配合这个JVM参数模板的重新部署和上线后。通过jstat观察各系统，基本上发现FGC变成了几天才会发生一次。


 


**三.参数模板里加入Compaction相关的参数**


保证每次FGC后都执行一次压缩，避免内存碎片。


 


**(4\)如何优化每次FGC的性能**


下面介绍进行JVM优化时可能会调整的两个参数，这两个参数可以优化FGC的性能，把每次FGC的时间进一步降低一些。


 


**一.\-XX:\+CMSParallelInitialMarkEnabled**


这个参数会在CMS垃圾回收器的"初始标记阶段"开启多线程并发执行，CMS在初始标记阶段，是会进行Stop the World的，这会导致系统停顿。所以该阶段开启多线程并发，可以尽量优化该阶段性能，减少STW时间。


 


**二.\-XX:\+CMSScavengeBeforeRemark**


这个参数会在CMS的重新标记阶段前先尽量执行一次YGC，这样做有什么作用呢？因为CMS的重新标记也是会Stop the World的。所以在重新标记前，先执行一次YGC，就能回收一些新生代的垃圾对象。如果能提前回收一些垃圾对象，在重新标记阶段就可以少扫描一些对象。此时就可以提升CMS的重新标记阶段的性能，减少耗时。


 


所以在JVM参数模板中也加入这两个参数：



```
 -Xms4096M -Xmx4096M -Xmn3072M -Xss1M 
 -XX:PermSize=256M -XX:MaxPermSize=256M 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:CMSInitiatingOccupancyFaction=92 
 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 
 -XX:+CMSParallelInitialMarkEnabled -XX:+CMSScavengeBeforeRemark
```

**(5\)采用JVM参数模板后的效果**


经过使用jstat观察各业务系统的JVM GC情况，发现明显有很大好转。基本上各系统的YGC都是几分钟～十几分钟一次，每次耗时几十毫秒，FGC基本都在几天一次，每次耗时也在几百毫秒。


 


当各个系统的JVM达到上述这个GC情况，就对线上系统没多大影响了。哪怕不太懂JVM优化的开发只要套用这个模板，那么对一些普通系统，都能保证JVM性能不出现大问题，比如频繁YGC和FGC导致频繁卡顿。


 


**3\.不合理设置JVM参数可能导致频繁FGC(优化反射的软引用被每次YGC回收)**


**(1\)案例背景**


**(2\)问题的产生**


**(3\)查看GC日志**


**(4\)查看Metaspace内存占用情况**


**(5\)一个综合性的分析思路**


**(6\)到底是什么类不停地被加载**


**(7\)为什么会频繁加载奇怪的类**


**(8\)JVM创建的奇怪类有什么玄机**


**(9\)为什么JVM创建的奇怪的类会不停地变多**


**(10\)如何解决这个问题**


**(11\)案例总结**


 


**(1\)案例背景**


这个案例是因为新手工程师对JVM优化了解不足，然后不知道从哪里找来一个非常特殊的JVM参数进行了错误设置，从而导致线上系统频繁出现FGC。


 


**(2\)问题的产生**


这个场景的发生过程大致如下：某天团队里一个新手工程师心血来潮，在当天上线一个系统时，自作主张设置了某个JVM参数。设置了完这个JVM参数后，就导致线上频繁接到JVM的FGC告警。大家就很奇怪，于是就开始排查那个系统。


 


**(3\)查看GC日志**


一般公司都会接入类似Zabbix、OpenFalcon或自研的一些监控系统。监控系统一般都做的很好，可以直接接入业务系统。然后在监控系统上看到每台机器的CPU、磁盘、内存、网络的一些负载、JVM内存使用波动折线图、JVM GC发生的频率折线图、甚至业务系统自己上报的某些业务指标情况。而且一般都会针对线上运行的机器和系统设置一些告警，比如可以设置如果发现系统10分钟内发生超过3次FGC，就发送告警。


 


一.一旦发生告警，可以登录到线上机器，查看对应的GC日志，此时发现GC日志中有大量FGC记录。


 


二.那么是什么原因导致FGC呢？在日志里，看到了包含"Metadata GC Threshold"关键字的日志：



```
[Full GC（Metadata GC  Threshold）xxxxx, xxxxx]
```

从这里可知，频繁的FGC就是由Metaspace元数据区(永久代)导致的，这个Metaspace区域一般是放一些加载到JVM的类。


 


三.为什么Metaspace元数据区会频繁被占满而触发FGC？根据FGC定义，FGC是针对年轻代、老年代、永久代进行的整体的GC，所以FGC会进行Metaspace元数据区的垃圾回收。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9085c720e81d40888d0f4a820230956b~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=lzepj7dHjIYzRsGzHqNnpfQiH%2BU%3D)
**(4\)查看Metaspace内存占用情况**


接着需要看一看Metaspace区域的内存占用情况，简单点可以通过jstat来观察。如果有监控系统，监控系统会展示出Metaspace内存占用的波动曲线图，类似如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/5e23cae5f9df466c8023bb33801deeba~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=aQG6F%2FCYcfTmqArdnpL3Bm0bzK4%3D)
看起来Metaspace区的内存呈现一个波动的状态，它总是会先不断增加，达到一个顶点后，就会把Metaspace区占满。然后就触发一次FGC，FGC会回收Metaspace区的垃圾，所以接下来Metaspace区的内存占用又变得很小了。


 


**(5\)一个综合性的分析思路**


很明显，系统在运行过程中，不停地产生新的类。然后这些类被加载到Metaspace区，逐渐把Metaspace区占满，接着触发一次FGC回收掉Metaspace区中的部分类。这个过程不断循环，从而造成Metaspace区反复被占满，导致反复FGC。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bdec39601ec740d5bb4661660557e81b~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=2Jg%2BNwP1RDHjT68dZ78i73iEimA%3D)
**(6\)到底是什么类在不停地被加载**


到底是什么类不停被加载到JVM的Metaspace里？这时可以在JVM启动参数中加入如下两个参数：



```
 -XX:TraceClassLoading -XX:TraceClassUnloading
```

这两个参数会追踪类加载和类卸载的情况，会通过日志打印出JVM中加载了哪些类、卸载了哪些类。


 


加入这两个参数后，就可以看到在日志文件中，输出了一堆日志，日志里面显示类似如下内容：



```
[Loaded sun.reflect.GeneratedSerializationConstructorAccessor from __JVM_Defined_Class]
```

可以看到，JVM在运行期间不停地加载了大量的类到Metaspace区域里，这个类就是GeneratedSerializationConstructorAccessor。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/86a8aad1274f426fbccd90328bb3ac72~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=nB5KVjMeEm0pHdk0JEUDNbMcDbs%3D)
就是因为在JVM运行期间不停地加载这种奇怪的类，然后不停地把Metaspace区域占满，最后才会频繁引发不停地执行FGC。


 


所以频繁FGC不仅仅只会由老年代触发，有时也会因为Metaspace区的类太多而触发。


 


**(7\)为什么会频繁加载奇怪的类**


遇到这种问题，先看这种不停被加载的类到底是什么类。是业务系统自己的类，还是JDK内置的类？


 


如果查阅一些资料就很容易明白，其实这个类GeneratedSerializationConstructorAccessor会在使用Java的反射时被加载。


 


反射代码类似如下：



```
Method method = XXX.class.getDeclaredMethod(xx, xx);
method.invoke(target, params);
```

简单来说，就是首先通过XXX.class获取某个类。然后通过getDeclaredMethod获取该类的方法，这个方法是一个Method对象，通过Method.invoke可以调用该类的方法。


 


在执行这种反射代码时，JVM会在反射调用一定次数后动态生成一些类。这些类就是类似GeneratedSerializationConstructorAccessor的类，这样下次再执行反射时，就可以直接调用这些类的方法，这属于JVM底层的一个优化机制。


 


所以我们可以得出如下结论：如果代码里大量使用了反射，那么JVM就会动态生成一些类并放入到Metaspace区域里。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/5a51ffadf3604c1181f7f693ea8c9a82~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=wR95WUxsrusenrDIReeH3lR2Z7A%3D)
**(8\)JVM创建的奇怪类有什么玄机**


JVM为什么会不停地创建那些奇怪的类然后放入到Metaspace中？因为上面这种JVM创建的类，其Class对象都是SoftReference软引用的。


 


**一.每个类本身也是一个Class对象**


一个Class对象就代表了一个类，同时这个Class对象代表的类，可以派生出来很多实例对象。比如Class Boy就是一个类，它本身是由一个Class类型的对象来表示。但如果Boy boy \= new Boy()，那么就是实例化了这个Boy类的一个对象，这个对象就是一个Boy类型的实例对象。


 


**二.JVM在反射中动态生成的类的Class对象都是SoftReference软引用的**


这里所说的Class对象，是JVM在反射过程中动态生成的类的Class对象，这些Class对象都是SoftReference软引用的。


 


**三.软引用和软引用需要在YGC时回收的公式**


所谓的软引用，正常情况下不会回收，但如果内存比较紧张就会回收。那么SoftReference对象在YGC时要不要回收是怎么进行判断的呢？SoftReference对象需要在YGC时回收的判断公式如下：



```
clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB
```

这个公式的意思是：clock \- timestamp代表了一个软引用对象它有多久没被访问过了，freespace代表JVM中的空闲内存空间，SoftRefLRUPolicyMSPerMB代表：每M空闲内存空间可以允许SoftReference对象存活多久。


 


**四.举个例子**


假设现在JVM创建了一大堆奇怪的类，而且这些类本身的Class对象都是被SoftReference软引用的，然后现在JVM里的内存空间有3000M，SoftRefLRUPolicyMSPerMB默认是1000毫秒。


 


那么就意味着：那些奇怪的被SoftReference软引用的Class对象，可以存活3000 \* 1000 \= 3000秒 \= 50分钟。


 


一般发生GC时，其实JVM内部或多或少总有一些空闲内存的，所以基本上如果不是快要发生OOM内存溢出了，软引用也不会被回收。


 


所以JVM理应会：随着反射代码的执行，动态创建一些奇怪的类，这些类的Class对象都是被SoftReference软引用的。在正常情况下这些Class对象不会被回收，但也不应快速增长。


 


**(9\)为什么JVM创建的奇怪的类会不停地变多**


因为那个新手工程师把SoftRefLRUPolicyMSPerMB参数，直接设置为0。他希望一旦这个参数设置为0，任何软引用对象都可以尽快释放掉。从而尽量释放内存空间出来，这样就可以提高内存利用效率了。但实际上一旦这个参数设置为0后，直接会导致：



```
clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB
```

这个公式的右半边是0，从而导致所有的软引用对象，刚创建出来就可以被YGC回收掉。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/81669b0ed606412c8011e5b0547abaa6~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=vxXQvAkuwqpUomN5%2F5QZ4j513%2Bo%3D)
比如JVM好不容易给动态生成100个奇怪的类，结果因为设置软引用的这个参数为0，就导致在一次YGC时，就回收掉堆里面的这几十个类对象，但是Metaspace中对应的类信息并没有被回收掉。


 


接着JVM在反射代码执行的过程中，还会继续创建这种奇怪的类。在JVM的机制下，会导致Metaspace中这种奇怪类越来越多。


 


下次YGC又会回收掉一些奇怪的类对象，但JVM马上又继续生成这种类，最终导致Metaspace区被占满了。一旦Metaspace区域被占满，就会触发FGC，然后回收掉很多类，接着再次重复上述循环。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c555895106474d18b0d4b6f41e8e4ab8~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=ExcfTkSwAJhrVVdonBJJ6XaXfjg%3D)
为什么软引用的类被快速回收后，会导致JVM不停创建更多的新的类呢？这涉及到底层JDK源码实现，比较复杂，要详细分析JDK底层实现细节。


 


**(10\)如何解决这个问题**


在有大量反射代码的场景下：只要把\-XX:SoftRefLRUPolicyMSPerMB\=0这个参数值设置大一些即可。千万不能设置为0，可以设置个1000、2000、3000、或者5000毫秒，让反射过程中JVM自动创建的一些类的Class对象不要被随便回收。


 


**(11\)案例总结**


因为\-XX:SoftRefLRUPolicyMSPerMB\=0，导致YGC时回收了调用反射时JVM创建的大部分软引用对象(在堆中)，导致下一次调用反射又继续创建类和Class，而Class被放在元空间，从而导致元空间很快满了，于是就触发FGC。


 


多次调用反射时，会因为NativeMethodAccessor的次数影响(默认为15次)，而生成最终的类似于generateXXXAccessorXXX这样的类。generateXXXAccessorXXX类会将反射调用转化为本地调用，提升性能。但如果generateXXXAccessorXXX类的软引用被回收了，就会导致元数据区多次生成相同的类，导致元数据区很快占满，触发FGC。


 


**4\.线上系统每天数十次FGC导致频繁卡顿的优化(大对象问题)**


**(1\)背景**


**(2\)未优化前的JVM性能分析**


**(3\)未优化前的线上JVM参数**


**(4\)根据线上系统的GC情况倒推运行内存模型**


**(5\)老年代里到底为什么会有那么多的对象**


**(6\)定位系统的大对象**


**(7\)针对本案例的JVM和代码优化**


 


**(1\)背景**


一个运行良好的系统，应该几天一次FGC，或者最多一天几次FGC。但有个新系统上线后，发现一天的FGC次数高达数十次，甚至上百次。可见这个新系统在线上的表现非常不好，明显会存在经常性的卡顿。因此要进行一连串的排查、定位、分析和优化，下面介绍整个优化过程。


 


**(2\)未优化前的JVM性能分析**


通过监控平台 \+ jstat工具分析，可以得出该系统优化前的JVM性能表现：


一.机器配置是2核4G


二.JVM堆内存大小是2G


三.系统运行时间是6天


四.系统运行6天内发生的FGC次数和耗时是250次和70多秒


五.系统运行6天内发生的YGC次数和耗时是2\.6万次和1400秒


 


综合分析可知：每天会发生40多次FGC，每小时2次，每次FGC在300毫秒左右；每天会发生4000多次YGC，每分钟发生3次，每次YGC在50毫秒左右。


 


上述数据对任何一个线上系统，都可以用jstat轻松看出来。因为jstat显示出来的FGC和YGC的次数都是系统启动以来的总次数，jstat显示的耗时都是所有GC加起来的总耗时，所以可直接拿到上述数据。


 


所以整体看来，这个系统的性能比较差。每分钟3次YGC，每小时2次FGC，必须要进行优化了。


 


**(3\)未优化前的线上JVM参数**


未优化前的线上JVM参数如下：



```
 -Xms1536M -Xmx1536M -Xmn512M -Xss256K 
 -XX:SurvivorRatio=5 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:CMSInitiatingOccupancyFraction=68 
 -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly 
 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
```

上述参数基本上和我们前面看到的参数没多大不同：一个4G的机器，给JVM堆内存设置1\.5G，其中新生代512M，老年代1G。


 


\-XX:SurvivorRatio设置了5，即Eden : S1 : S2的比例是5 : 1 : 1。所以此时Eden区大致为365M，每个Survivor区域大致为70M。


 


\-XX:CMSInitiatingOccupancyFraction设置了68，所以一旦老年代内存占用达到68%，大概680M时，就会触发一次FGC。


 


此时整个系统的内存模型图如下：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e91cfef3ad284c33bfdca8dfb2a39269~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=r4S1NjJrG5gKJdlCPfpvwxdQIEI%3D)
**(4\)根据线上系统的GC情况倒推运行内存模型**


接下来根据系统的内存模型及GC情况，推导出系统运行时的内存模型。


 


**一.首先可以知道每分钟会发生3次YGC**


这说明系统运行20秒就会让Eden区满了，也就是产生300多M的对象。所以平均下来系统每秒会产生15\~20M的对象，20秒左右就会导致Eden区满，然后触发一次YGC。


 


**二.接着根据每小时2次FGC推断出，每30分钟会触发一次FGC**


\-XX:CMSInitiatingOccupancyFraction\=68表示：当1G的老年代有68%空间(600多M)被占满时，就会触发CMS的GC。再根据每小时2次FGC推断出，每30分钟会触发一次FGC。所以系统每运行30分钟就会导致老年代里有600多M的对象，从而触发CMS垃圾回收器对老年代进行GC。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/b2b79ea54b6547a5a0eac914f53c31c6~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=9BgqZVwYNNNbh1Bg6rAa7muOHH0%3D)
所以根据JVM实际运行情况 \+ JVM参数内存模型，可以得出如下结论：每隔20秒会让365M的Eden区占满，从而触发一次耗时50毫秒的YGC；每隔30分会让680M的老年代占满，从而触发一次耗时300毫秒的FGC。


 


**三.此时其实可以进行猜测**


是不是因为Survivor区域太小了，导致YGC后的存活对象太多放不下，就一直有对象流入老年代，从而导致30分钟后触发FGC。


 


为什么老年代里有那么多对象？一.可能是每次YGC后存活对象较多而S区放不下，或触发动态年龄判断；二.也可能是有很多长时间存活对象，都积累在老年代，始终回收不掉，从而导致老年代很容易达到68%的占比，触发GC。但仅仅是分析而已，还不能轻易下结论。


 


**(5\)老年代里到底为什么会有那么多的对象**


分析到这里，仅仅根据可视化监控和推论是没法往下分析了，因为我们并不知道老年代里到底为什么会有那么多的对象。


 


**一.此时可以用jstat在高峰期观察一下JVM实际运行的情况**


通过jstat的观察可以明确看到，每次YGC过后升入老年代的对象很少。一般来说，每次YGC过后大概会存活几十M对象。由于Survivor区只有70M，所以很容易会触发动态年龄判断规则，导致偶尔一次YGC过后有几十M对象进入老年代。如下图示：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/dcd21707e18e499da350039feaa6233f~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=3O6Gkqot8zPmixJf41VXfBacVes%3D)
因此分析到这里就很奇怪：通过jstat追踪观察，并不是每次YGC后都有几十MB对象进入老年代的。而是偶尔一次YGC才会有几十MB对象进入老年代，是偶尔一次而已。所以正常来说，应该不至于30分钟就导致老年代占用空间达到68%。


 


**二.为什么老年代里会有那么多对象**


通过jstat观察到一个现象：在系统正常运行时，会突然出现五六百M的对象进入老年代。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a5352d728f734a0bb14f7c5dd726a45e~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=bylP1922HXwQNgeH3UstgtMcgxE%3D)
正是因为在系统运行时，突然有几百M对象进入老年代，所以才导致：即使偶尔一次YGC才有几十M对象进入老年代，也平均30分一次FGC。


 


**三.为何系统运行时会突然有几百M对象进入老年代**


原因只能是大对象了。系统运行时，每隔一段时间会突然产生几百M的大对象。这些大对象会直接进入老年代，不会进入新生代的Eden区。然后加上新生代偶尔一次YGC才有几十M对象进入老年代，所以才出现30分钟触发一次FGC。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1f595eb140f7469680046a920eac779d~tplv-obj.image?lk3s=ef143cfe&traceid=20250103233556394EEB5D741561E036B1&x-expires=2147483647&x-signature=4dHdog9Ij%2F9YvlhSPhR32dk%2Fx68%3D)
**(6\)定位系统的大对象**


分析到这里问题就很简单了，接下来只需要通过jstat工具，观察系统什么时候老年代里会突然进入几百M的大对象，然后在这个时候紧接着使用jmap工具导出一份dump内存快照。


 


接着可以使用jhat等可视化工具来分析dump内存快照，通过内存快照的分析，定位出那个几百MB的大对象。可能是几个Map之类的数据结构，大概率是从数据库查出来的大对象。


 


接下来可以地毯式排查这个系统的所有SQL语句，找出可能导致每隔一段时间系统会出现几个上百M大对象的SQL查询，然后进行优化调整。


 


**(7\)针对本案例的JVM和代码优化**


第一步：让开发解决代码中的bug，避免一些极端情况下SQL语句会查出大量数据，从而导致出现大对象。


 


第二步：新生代明显过小，Survivor区空间不够，只有70MB。由于每次YGC后存活几十M对象，容易触发动态年龄判定进入老年代，所以直接调整JVM参数如下：



```
 -Xms1536M -Xmx1536M -Xmn1024M -Xss256K 
 -XX:SurvivorRatio=5 -XX:PermSize=256M -XX:MaxPermSize=256M 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:CMSInitiatingOccupancyFraction=92 
 -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly 
 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
```

直接把新生代空间调整为1G，每个Surivor是150M左右。老年代只留500MB就够了，因为一般不会有对象进入老年代。


 


\-XX:CMSInitiatingOccupancyFraction的参数值调整为92，避免老年代仅占用68%就触发GC，调整为要占用到92%才会触发GC。


 


最后主动设置永久代大小为256MB，因为如果不主动设置永久代，默认的永久代只有几十M。万一系统运行时采用反射，一旦动态加载的类过多，就会频繁触发FGC。


 


这几个步骤优化完毕后，线上系统就表现非常好了，基本上每分钟发生一次YGC，一次在几十毫秒。而FGC几乎就很少，大概几天才会发生一次，一次就耗时几百毫秒而已。


 


**5\.电商大促活动下严重FGC导致系统直接卡死的优化(System.gc()导致)**


有一个新系统上线，平时都还算正常。结果有一次大促活动时，这个系统就直接卡死不动了。这个系统无法处理所有请求，重启这个系统也没有任何效果。


 


这时按照前面的思路去分析JVM的GC问题，首先考虑是不是由于频繁的GC问题导致系统被卡死。


 


**一.jstat发现每秒都有一次FGC**


首先使用jstat去查看系统运行情况，发现奇怪的事情：JVM几乎每秒都执行一次FGC，每次都耗时几百毫秒。


 


**二. 各指标都正常为什么会频繁触发FGC**


继续通过jstat查看JVM各个内存区域的使用量，基本都没什么问题。年轻代对象增长并不快，老年代才占用了不到10%的空间，永久代也就使用了20%左右的空间，各个指标都正常，为什么会频繁触发FGC？


 


**三.是不是系统里出现一行致命代码：System.gc()**


这个System.gc()可不能随便写，System.gc()的每次执行都会指挥JVM去尝试执行一次FGC，System.gc()的每次执行都会回收新生代、老年代、永久代。


 


所以马上找到负责对这个系统进行开发的同学，让他进行排查代码，结果还真的找到了。他使用System.gc()的出发点是好的，他是这么考虑的：代码里经常会加载出大批数据，一旦请求处理完，这些数据就废弃不用。此时这些数据会占据太多内存，于是想用System.gc()触发GC回收它们。


 


结果平时系统运行时，访问量很低，基本不会出问题。但到了大促活动的时候，由于访问量太高。执行System.gc()代码太频繁，导致频繁触发FGC，从而让系统直接卡死。


 


所以针对这个问题：一方面平时写代码时，不要使用System.gc()去随便触发GC。另一方面可以在JVM参数中加入\-XX:\+DisableExplicitGC，这个参数的意思就是禁止显式执行GC，不允许通过代码触发GC。


 


所以推荐将\-XX:\+DisableExplicitGC参数加入到系统的JVM参数中，或者是加入到公司的JVM参数模板中，避免有的开发好心办坏事，导致频繁触发GC。


**6\.问题汇总**


**问题一：**


STW的时候，系统停止，请求发生阻塞。如果老年代STW时间比较长，阻塞了很多请求。等这次老年代垃圾回收完，被阻塞的请求开始处理，又会创建很多对象。于是对JVM造成压力，然后老年代又要GC。所以是不是STW时间久的话，会变相给系统制造更高的并发？


 


答：是的。因为阻塞了很多请求，确实会造成STW恢复后的瞬时处理请求增多。


 


**问题二：**


2核4G或4核8G的Linux机器，能够分配给堆内存的大小最大能有多少？


 


答：一般不能给到最大，操作系统和其他进程都要占用一些。比如4G的机器，给JVM的内存可以是2G\~3G。8G的机器，给JVM的内存可以是4G左右，这样就差不多了。


 


**问题三：**


G1对于大对象的判定规则是超过Region的50%，可以指定超过60%吗？


 


答：G1有参数可以控制大对象，但建议不改变。


 


**问题四：**


有一个有趣的现象：为什么\-XX:CMSFullGCsBeforeCompaction \= 5是大部分公司的设置呢？


 


答：确实有不少博客会推荐设置为5。但要考虑一下，如果通过优化之后，让FGC的频率很低。那么就完全可以让每次FGC后都Compaction一次，FGC慢点而已，但是不至于大量内存碎片导致下一次FGC更快到来。


 


**问题五：**


分析公司的系统，服务器不接受任何请求的情况下：大概16分钟左右1次YGC，1\.5天左右1次FGC。每次YGC有8M对象进入老年代，但服务器每次启动都会有3\-4次FGC，服务器启动时的FGC是否需要优化？


 


答：服务器启动的时候很多内置对象，初始化之类的，这个不需要优化，需要的优化核心是运行期间。


 


**问题六：**


是不是高并发 \+ 慢处理是导致FGC的元凶之一，会带来很多性能问题。


 


答：是的。


 


**问题七：**


G1相对其他回收器有什么劣势，很多地方不用G1是否因为没必要？


 


答：G1未来会成为一个默认的垃圾回收器。好处是只要指定一个垃圾回收停顿时间，G1就自动优化，无需过度优化，坏处是没法精准把控内存分配、使用、垃圾回收，所以有时优先使用ParNew \+ CMS。


 


**问题八：**


由于CMS是扫描老年代的对象，那么在重新标记之前进行一次YGC，YGC回收新生代的对象，对提升重新标记的性能帮助在哪？


 


答：CMS扫描老年代的对象是没错的，但有时新生代和老年代之间的对象有引用关系，就会扫描到新生代里。所以提前YGC可以清理掉一些新生代对象，这可以有助于提升CMS的重新标记阶段的性能。


 


**问题九：**


一.JVM内存超过4G且对系统响应时间敏感的是不是应该采用G1？


二.对高并发、容易产生阻塞的系统，是否考虑减小SurvivorRatio的值？这样可以给S区分配更大的空间，避免短命的对象进入老年代。这样也可能会导致YGC会更频繁些，但YGC很快，所以关系不太大。


 


答：一.超过4G还不至于必须用G1，一般超过16G以上的机器可以考虑用G1。普通的机器都是2C4G或4C8G，这种机器使用CMS\+ParNew没问题的。


二.高并发、大数据量的系统，建议还是根据实际情况去优化各种参数，具体方法参考前面介绍的那种思路。核心系统一定是要使用一整套流程来优化JVM参数的：预估系统的并发量 \-\> 选择合适的服务器配置 \-\> 根据JVM内存模型设定初始参数 \-\> 对系统进行压测 \-\> 观察高峰期对象的增长速率、GC频率、GC后的存活 \-\> 然后根据实际的情况来设定JVM参数 \-\> 最后做好线上JVM监控。


 


**问题十：**


CMSScavengeBeforeRemark参数是希望在CMS重新标记前进行YGC，好处是如果YGC比较有效果则是能有效降低重新标记的时间长度。可以理解为如果大部分新生代对象被回收了，那么作为根的部分就少了，从而提高了CMS重新标记的效率。


 


**问题十一：**


可能新生代的某个GC Root引用了老年代某对象，这个对象就不能清除，所以CMS应该也要扫描年轻代GC Root，所以再进行一次YGC就可以减少扫描的年轻代GC链路。另外G1基于Region收集，通过记忆集记录引用关系来避免全堆扫描。


 


**问题十二：**


由于CMS重新标记阶段需要扫描新生代，所以整个堆中对象数量会影响Remark阶段的耗时，所以Remark之前添加一次可中断的并发预清理。


 


另外为了防止并发预清理阶段等太久都不发生YGC，提供了CMSMaxAbortablePrecleanTime参数，该参数可以设置等待多久没等到YGC就强制进行重新标记，默认是5s。


 


但是最终一劳永逸的办法是：添加参数CMSScavengeBeforeRemark，让Remark之前强制YGC。


 


**问题十三：**


为什么重新标记时提前做一次YGC会提高效率？重新标记不是只针对老年代的对象进行标记的吗？


 


答：老年代扫描时要确认老年代里的存活对象，这时会扫描到新生代。因为有些新生代的对象可能引用了老年代的对象，所以提前做YGC可以把年轻代里一些对象回收掉。从而减少了扫描新生代的时间，可以提升性能。


 


**问题十四：**


CMS垃圾回收不同阶段的处理总结：


初始标记：标记由GC Roots直接关联的对象


并发标记：对老年代所有对象进行追踪，看能否与GC Roots建立关系


最终标记：标记并发标记时引用变动的对象


并发清理：并发清理掉可回收的内存，但是因为用户线程依旧在运行，所以每次FGC都会并发清理不干净，产生浮动垃圾


 


**问题十五：**


JVM问题的排查步骤总结：


步骤一：使用jstat分析机器情况：机器配置、堆内存大小、运行时长、FGC次数时间、YGC次数时间


步骤二：查看具体的JVM参数配置


步骤三：根据JVM参数配置梳理出JVM模型


步骤四：结合jstat查看的GC情况 \+ JVM模型分析


步骤五：通过jhat或MAT查看"jmap dump内存快照"的对象分类情况


步骤六：根据分析的结果再排查具体的问题原因：Bug或者参数设置不合理


步骤七：修复Bug、优化JVM参数


 


 本博客参考[Flowercloud 机场订阅加速](https://flowercloud6.com)。转载请注明出处！
