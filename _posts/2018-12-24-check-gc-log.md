---
title: Check GC Log
categories:
- 
tags:
---

# gdb
# string 

给CMS GC搭配-XX:+PrintGCDetails和-XX:+PrintGCTimeStamps可以打印出很多关于GC的信息.理解GC日志信息可以帮助日常的性能调优以达到最优的性能。

下面就来看下gc日志的详情:

39.910: [GC 39.910: [ParNew: 261760K->0K(261952K), 0.2314667 secs] 262017K->26386K(1048384K), 0.2318679 secs]
上面是年轻代(ParNew)回收信息,年轻代总大小261952Kb,垃圾回收后使用量从261970Kb到0Kb,整个回收过程花费0.2318679秒。

40.146: [GC [1 CMS-initial-mark: 26386K(786432K)] 26404K(1048384K), 0.0074495 secs]
现在开始的是使用CMS收集器对老年代做的回收,老年代的容量是786432Kb，占用量是26386Kb时触发了老年代的收集。这是CMS过程的初始标记阶段,这个阶段会停止所有应用线程(stop-the-world)，该阶段单线程执行,主要做两件事情:

标记GC根节点可以直接引用的对象；
遍历新生代对象,标记可达的老年代对象。
40.154: [CMS-concurrent-mark-start]
这是并发标记过程的开始，在初始标记阶段停止的线程在这个阶段会重新启动,这个阶段应用线程和GC线程并行执行,多个GC线程并发标记活跃的对象。

40.683: [CMS-concurrent-mark: 0.521/0.529 secs]
并发标记阶段总共花费了0.521秒。因为该阶段是并发执行的,运行期间可能会在老年代产生新的垃圾，或者老年代的对象被新生代对象重新引用,这些对象都是要重新标记的，有以下几种场景:

新生代对象晋升到老年代
新生代新创建对象引用了原先老年代未被标记的对象
直接在老年代分配的对象
老年代对象的引用关系发生变化
为了提高重新标记的效率,该阶段会把上述对象所存在的card标记为dirty，后续只需要扫描dirty card对应的对象。
40.683: [CMS-concurrent-preclean-start]
预清理阶段的开始,这个阶段是并发执行的,可以通过参数-XX:-CMSPrecleaningEnabled关闭；这个阶段主要是处理前面介绍的4种常见，减少重新标记的开销,因为CMS设计的目的是减少应用的停顿时间,而重新标记是stop-the-word的,所以这一阶段减少重新标记的工作。

40.701: [CMS-concurrent-preclean: 0.017/0.018 secs]
并发预清理阶段花费0.017秒CPU时间,0.018秒系统时间。

40.704: [GC40.704: [Rescan (parallel) , 0.1790103 secs]40.883: [weak refs processing, 0.0100966 secs] [1 CMS-remark: 26386K(786432K)] 52644K(1048384K), 0.1897792 secs]
这个阶段主要处理并发标记清除阶段新产生的垃圾对象,这个阶段是Stop-the-world.主要做一下几件事情:

遍历新生代对象，重新标记
根据GCRoots,重新标记
遍历老年代的Dirty card重新标记，这部分对象大部分在预清理阶段已经处理过。
40.894: [CMS-concurrent-sweep-start]
清除死亡的(没有标记的)对象，这个阶段jvm应用线程并发执行。

41.020: [CMS-concurrent-sweep: 0.126/0.126 secs]
清除阶段会0.126秒.

41.020: [CMS-concurrent-reset-start]
Reset阶段

41.147: [CMS-concurrent-reset: 0.127/0.127 secs]
这个阶段,CMS的数据结构重新初始化,为下一次垃圾回收做准备。

上面这些是一个正常的CMS周期,不同版本的JVM可能会有细微差别,大体还是差不多,接下来我们来看一下其他的场景:

197.976: [GC 197.976: [ParNew: 260872K->260872K(261952K), 0.0000688 secs]197.976: [CMS197.981: [CMS-concurrent-sweep: 0.516/0.531 secs]
(concurrent mode failure): 402978K->248977K(786432K), 2.3728734 secs] 663850K->248977K(1048384K), 2.3733725 secs]
可以看出开始了新生代的收集,但是没有执行因为这时候又发生了CMS GC，CMS老年代又没有足够的空间来容纳新生代的对象，这样就出现了concurrent mode failure这样的字样,这个时候会放弃CMS GC周期转而执行一次Full GC。

concurrent mode failure可以通过增加老年代的空间或者减少老年代GC的阈值来解决,可以把GC参数CMSInitiatingOccupancyFraction设置小一点,默认是92,这个参数只在第一次收集时才生效,后续jvm会自动调整,你要同时设置UseCMSInitiatingOccupancyOnly=true这个参数才一直生效,CMSInitiatingOccupancyFraction这个参数要选的比较合适才行，太小了会导致CMS GC太频繁,太大了就可能出现并发模式失败,有时候你会发现老年代明明有足够的空间还是出现了promotion failures，这是因为CMS GC产生了很多内存碎片,没有连续的内存区域来存储新生代的对象。CMS GC还有一种原因是晋升担保,cms收集器会计算新生代的对象是否能够成功晋升到老年代,如果晋升担保失败也是会触发一次CMS GC周期，最新版本的CMS晋升担保会根据新生代晋升到老年代对象的历史数据来决定这个担保的值。

283.736: [Full GC 283.736: [ParNew: 261599K->261599K(261952K), 0.0000615 secs] 826554K->826554K(1048384K), 0.0003259 secs]
Full GC。

283.736: [Full GC 283.736: [ParNew: 261599K->261599K(261952K), 0.0000288 secs]
新生代GC除了因为在新生代分配内存失败外,还可能是因为GC Locker导致,当JNI中的native方法需要访问JVM中的String或者数组时,需要调用GetStringCritical函数，到数据复制完成期间，必须保证原始数据不被修改，为了防止发生GC期间回收该字符串对象,如果期间发生了YoungGC会被标记成disabled,放弃这次GC，等线程执行完临界区的代码之后，执行ReleaseStringCritical方法,会弥补一次新生代GC，通过GC日志查看到的GC Cause就是GC Locker

2803.125: [GC 2803.125: [ParNew: 408832K->0K(409216K), 0.5371950 secs] 611130K->206985K(1048192K) icms_dc=4 , 0.5373720 secs]
2824.209: [GC 2824.209: [ParNew: 408832K->0K(409216K), 0.6755540 secs] 615806K->211897K(1048192K) icms_dc=4 , 0.6757740 secs]
CMS也可以运行在增量模式,通过jvm参数-XX:+CMSIncrementalMode开启增量模式;增量模式下,CMS收集器并不是一直占用执行权限,而是把CMS周期分成很多小的周期,每个小周期执行完后又把执行权限还给应用线程.

这两个阶段分别花费了737和675毫秒,在这两个周期中间,ICMS被标记出来,icms_dc=4表示4%的时间花在CMS的增量模式上，那增量模式共花费的时间可以这样计算:
0.04 * (2824.209 - 2803.125 - 0.537) = 821 ms

从JDK1.5开始,CMS多了一个阶段,叫做concurrent-abortable-preclean，可中断的预清理就当在并发预清理和重新标记中间,这个阶段也是用来减少重新标记花费的时长,这个阶段通过jvm参数CMSScheduleRemarkEdenSizeThreshold设置阈值,默认为2M，即新生代使用率超过2M才开启concurrent-abortable-preclean，这个阶段主要循环做2件事情:

处理from和to区的对象,标记可达的老年代对象
和并发预清理一样，处理被标记为dirty card区的对象
这个逻辑不会一直循环下去，打断这个循环的条件有三个：

通过参数CMSMaxAbortablePrecleanLoops设置最多循环的次数，默认是0,没有循环次数限制;
执行这个逻辑的时间达到了CMSMaxAbortablePrecleanTime参数设置的阈值，默认是5s。
新生代Eden区的内存使用率达到了CMSScheduleRemarkEdenPenetration参数设置的阈值，默认50%，会退出循环。
如果在循环退出之前，发生了一次YGC，对于后面的Remark阶段来说，大大减轻了扫描年轻代的负担。

7688.150: [CMS-concurrent-preclean-start]
7688.186: [CMS-concurrent-preclean: 0.034/0.035 secs]
7688.186: [CMS-concurrent-abortable-preclean-start]
7688.465: [GC 7688.465: [ParNew: 1040940K->1464K(1044544K), 0.0165840 secs] 1343593K->304365K(2093120K), 0.0167509 secs]
7690.093: [CMS-concurrent-abortable-preclean: 1.012/1.907 secs]
7690.095: [GC[YG occupancy: 522484 K (1044544 K)]7690.095: [Rescan (parallel) , 0.3665541 secs]7690.462: [weak refs processing, 0.0003850 secs] [1 CMS-remark: 302901K(1048576K)] 825385K(2093120K), 0.3670690 secs]
上面的日志信息，在并发预清理之后开启了可中断的预清理阶段,当新生代的使用率到达522484K(50%阈值)时,预清理被中断随后执行重新标记阶段。
