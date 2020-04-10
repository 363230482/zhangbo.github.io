#G1GC过程及常见参数
官方文档：[https://docs.oracle.com/en/java/javase/13/gctuning/garbage-first-garbage-collector.html]  
>垃圾优先收集器简介  
Garbage-First（G1）垃圾收集器的目标是具有大量内存的多处理器计算机。它尝试以极高的可能性满足垃圾收集暂停时间目标，同时几乎不需要配置即可实现高吞吐量。G1的目标是使用当前的目标应用程序和环境在延迟和吞吐量之间达到最佳平衡，其特点包括：  
>
>堆大小最大为数十GB或更大，其中超过50％的Java堆占用实时数据。  
对象分配和升级的速率可能会随时间而显着变化。  
堆中有大量碎片。  
可预测的暂停时间目标目标不超过几百毫秒，避免了长时间的垃圾收集暂停。  

G1GC使用`Snapshot-At-The-Beginning (SATB)`的算法，这种算法会导致部分内存不能及时的释放，但它可以提供更好的低延迟。

+ 1.年轻代回收(Young-only phase)：  
这个阶段会开始几轮普通的年轻代回收,此阶段会回收年轻代。
然后在老年代区域达到回收阈值时，G1会开始并发年轻代收集，以代替普通年轻代回收。
年轻代并发回收又分为3个阶段：  
  > 1.(Concurrent Start)：这个阶段除了普通的年轻代回收之外，还会并发的执行老年代的标记。  
   2.(Remark)：会STW。执行全局标记和类的卸载，会标记region的使用情况。  
   3.(CleanUp)：会STW。此阶段和Remark阶段，G1将计算能够回收的区域的成本，以确定是否开始后续的回收阶段。
+ 2.空间回收(Space Reclamation)：  
此阶段包含多个混合回收，来回收年轻代和老年代的空间，直到G1觉得回收更多的老年代的花费不值得时，停止此阶段，重新开始仅年轻代的回收。  
+ 3.作为备份，在G1GC时，如果可能发生OOM，则会进行全堆STW的Full GC。  

默认情况下，巨大对象(Humongous Objects)只会在CleanUp阶段和Full GC时被回收。
但基本类型的数组，G1会尝试在数组不可达时进行回收。可通过参数关闭这种早期回收。
Humongous Objects在整个生命周期中不会被移动，它所占据的region中的剩余空间也不会被其他对象使用。

G1GC相关参数，参数中所有的数值需根据实际情况设置，这里只是为了说明：
```text
-XX:+UseG1GC #开启G1GC
-XX:G1HeapRegionSize=2m #region大小，1-32m之间，为2的次幂
-XX:MaxGCPauseTimeMillis=200 #设置GC暂停目标时间，默认200
-XX:GCPauseTimeInterval=200 #最大暂停时间间隔的目标。默认情况下，G1不设置任何目标，允许G1在极端情况下连续执行垃圾收集
-XX:PauseTimeIntervalMillis=200 #
-XX:InitialHeapSize=4g #堆的初始大小
-XX:MaxHeapSize=4g #堆的最大值
-XX:MinHeapFreeRatio=40 #最小可用内存百分比
-XX:MaxHeapFreeRatio=80 #最大可用内存百分比
-XX:G1NewSizePercent=5 #新生代最小比例，G1会自动调整新生代的大小，以达到设定的GC暂停时间，默认5%
-XX:G1MaxNewSizePercent=60 #新生代最大比例，如和最小一样，则将禁用暂停时间的控制，默认60%
-XX:NewSize=1g #新生代最小值
-XX:MaxNewSize=1g #新生代最大值，同上，如果和最小值一样，则禁用GC暂停时间的控制
-XX:G1MixedGCCountTarget=8 #回收的预期region数量
-XX:G1HeapWastePercent=5 #集合中允许的未回收空间设置为候选百分比。如果收集集合候选中的可用空间低于该间隔，则G1停止空间回收阶段。默认5%
-XX:G1MixedGCLiveThresholdPercent=85 #在空间回收阶段，不会收集活动对象占用率高于此百分比的老年代区域。默认85%
-XX:G1PeriodicGCInterval=5000 #周期GC的最小毫秒数，为了避免应用不活跃，导致长时间没有垃圾回收
-XX:G1PeriodicGCSystemLoadThreshold=80 #如果系统调用getloadavg()的最近一分钟的系统负载大于该值，则不触发G1PeriodicGCInterval设置的周期GC
-XX:InitiatingHeapOccupancyPercent=45 #老年代占用比例，达到该比例后，会结束普通年轻代回收，进入`Space Reclamation`。默认会开启自适应，此值仅用于第一次混合回收，后续将使用自适应算出的比例。
-XX:-G1UseAdaptiveIHOP #针对上一个参数，JVM默认时自适应策略决定老年代占用比例，通过此参数，可禁用自适应策略。
-XX:G1HeapReservePercent=30 #堆预留空间，剩余空间小于此值时，会开启混合回收。此值也是G1UseAdaptiveIHOP自适应IHOP的参考选项
-XX:G1EagerReclaimHumongousObjects #关闭巨大对象的早期回收
-XX:ParallelGCThreads=4 #暂停时并发GC的最大线程数，默认CPU核心>=8时，为核心数；大于8时为5/8.
-XX:ConcGCThreads=2 #并发GC最大线程数，默认为ParallelGCThreads/4.
-XX:-G1EnableStringDeduplication #禁用重复字符串数据删除。默认禁用
```