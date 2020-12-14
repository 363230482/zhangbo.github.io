参数官方文档：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html

常用参数：
```shell script
# 调整heap大小,OOM时导出heap dump供排查,打印GC日志
-Xms4g -Xmx4g -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.dump -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+PrintGCApplicationStoppedTime
```

========== 调整内存 ==========
-Xms 20M 堆内存的最小值
-Xmx 200M 堆内存的最大值
-Xmn 50M 新生代内存大小
-XX:MaxNewSize=256m
-XX:InitialHeapSize=1073741824 
-XX:MaxHeapSize=1073741824
-XX:NewRatio=2 新生代比例=老年代 / 新生代
-XX:SurvivorRatio=8 新生代幸存者区比例 = eden / from = eden / to
-XX:PretenureSizeThreshold=102400 单位byte，新对象超过这个值的，直接在老年代分配内存，只对Serial和ParNew有效
-Xss 1M 设置虚拟机栈的大小
-Xoss 1M 设置本地方法栈的大小，在Hotspot无效（Hotspot并不区分虚拟机栈和本地方法栈）
-XX:PermSize 128M 方法区的初始值
-XX:MaxPermSize 1024M 方法区的最大值
-XX:MaxMetaspaceSize=512M 设置jdk1.8中元数据区最大内存
-XX:MaxDirectMemorySize=1024M 本机的直接内存，默认为堆的最大值（-Xmx）
-XX:+HanlderPromotionFailure 设置老年代担保分配，只对JDK1.6u24之前的版本有效
-XX:G1HeapRegionSize=2M	G1回收器的Region大小，1-32M之间，为2的指数，默认会根据堆的大小计算

========== GC ==========
-XX:+UseSerialGC 新生代，老年代都使用串行收集，新生代复制算法，老年代标记压缩算法，独占（STW）
-XX:+UseParNewGC 新生代使用ParNew GC，此时老年代默认为Serial GC，为Serial GC的简单并行版本,STW
-XX:+UseParallelGC 新生代使用Parallel GC，老年代默认为Serial GC，更注重吞吐量的GC，复制算法，STW
-XX:+UseParallelOldGC 老年代使用ParallelOld GC，新生代默认使用Parallel GC，标记压缩算法,STW
-XX:+UseConcMarkSweepGC 老年代使用CMS，新生代默认使用ParNew GC，标记清除算法。流程为：初始标记(STW)--并发标记--预清理--重新标记(STW)--并发清理--并发重置
-XX:UseG1GC 使用G1 GC，分区算法，新生代，老年代可以不是连续的。regoin之间可以看为复制算法，整个堆可以看做整理算法，流程为：初始标记(STW)--跟区域扫描--并发标记--重新标记(STW)--独占清理(STW)--并发清理

-XX:-CMSPrecleaningEnabled 关闭CMS的预清理阶段，预清理为正式清理做检查和准备，还会尝试控制GC的停顿时间
-XX:ParallelGCThreads 8 设置使用ParNew，G1时的并行线程数量，当CPU<8时，默认为CPU数量，CPU>8时，默认为 3+（5*CPU）/ 8
-XX:MaxGCPauseMillis 500 设置使用Parallel，ParallelOld，G1时，GC暂停的最大毫秒数
-XX:GCTimeRatio 99 设置使用Parallel GC时，垃圾收集时间占总时间的比例，为1-99的整数。默认为99，此时系统将不花费超过1/(1+99)的时间，即1%的时间。
-XX:UseAdaptiveSizePolicy 开启GC自适应策略，在使用Parallel GC时使用。当这个参数打开之后，就不需要手工指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX：SurvivorRatio）、晋升老年代对象年龄（-XX：PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量
-XX:ConcGCThreads 2 CMS并发收集线程数，特指在并发时和用户线程并发的线程，默认为(ParallelGCThreads + 3) / 4。ParallelGCThreads表示GC并行时使用的线程数，如果新生代为ParNew，则为新生代GC的线程数量
-XX:ParallelCMSThreads 2 CMS并发收集线程数，特指在并发时和用户线程并发的线程
-XX:CMSInitiatingOccupancyFraction 68 指老年代空间占用为68%时，执行一次CMS
-XX:+UseCMSCompactAtFullCollection 指在CMS GC完成时进行一次内存碎片整理
-XX:CMSFullGCsBeforeCompaction 5 指在完成5次CMS GC后进行一次碎片整理
-XX:+CMSClassUnloadingEnabled 如果条件允许，使用CMS策略回收永久代的class数据
-XX:+CMSScavengeBeforeRemark	在CMS重新标记（STW）之前，强制发起一次MinorGC，减少Remark阶段需要扫描的对象，缩短Remark阶段的停顿时间。
-XX:InitiatingHeapOccupancyPercent 45 值整个堆使用率达到45%时触发并发标记周期的执行，一旦设置，中途不能修改
-XX:GCPauseIntervalMillis 设置G1 GC的停顿间隔时间 

-XX:+DisableExplicitGC 禁用System.gc()，System.gc()会使用传统的Full GC回收，如设置了并发GC，默认不会使用并发GC
-XXL:+ExplicitGCInvokesConcurrent 在System.gc()时，使用设置的收集器回收。如设置了CMS或G1等，则会使用并发GC
-XX:-ScavengeBeforeFullGC 关闭Full GC之前的一次新生代GC。在FullGC时，如果使用并行收集器，则默认会执行一次新生代回收，避免将所有回收工作交给一次FullGC而导致较长的停顿；串行收集器则不会导致新生代回收。
-XX:+PrintGC 打印垃圾回收日志
-XX:+PrintGCDetails 打印垃圾回收日志
-XX:+PrintGCCause
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式，从JVM启动到现在的时间差）
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-Xloggc:gc.log 打印GC日志
-Xnoclassgc 关闭class的垃圾回收
-verbose:gc 打印GC信息
-XX:+PrintHeapAtGC GC时打印堆的详细情况
-XX:+PrintGCApplicationStoppedTime 打印GC停顿时间
-XX:PrintFLSStatistics=2 打印碎片统计信息，FreeListSpace，值不为0即打印

-XX:-UseGCLogFileRetation	GC日志文件重用，在GC日志达到设置的大小时，会覆盖之前的GC日志，最好关闭
-XX:GCLogFileSize=8K		GC日志文件大小
-XX:NumberOfGCLogFiles=5	GC日志文件数量

========== 对象分配 ==========
-XX:+NeverTenure 对象永远不会晋升到老年代.当我们确定不需要老年代时，可以这样设置。这样设置风险很大,并且会浪费至少一半的堆内存
-XX:+AlwaysTenure 表示没有幸存区,所有对象在第一次GC时，会晋升到老年代。
-XX:InitialTenuringThreshold=10 设定老年代阀值的初始值
-XX:MaxTenuringThreshold=15 设置新生代对象晋升到老年代的年龄阈值，默认为15
-XX:TargetSurvivorRatio=50 设置Survivor区的使用率，默认为50%，如果在GC后Survivor仍然超过设定的值，则可能使新生代对象提前晋升到老年代
-XX:PretenureSizeThreshold=1000 单位字节，如果新分配对象大小超过该值，直接分配到老年代，只针对串行收集器和ParNew有效，默认为0，即不指定大小，由运行时决定
-XX:+UseTLAB 使用TLAB，线程私有分配缓冲，在eden区为每个线程预分配一块内存，加快对象分配
-XX:TLABRefillWasteFraction=64 表示允许TLAB中浪费的比例，默认为64，即1/64的空间为refill_waste
-XX:TLABSize=102400 指定TLAB的大小，单位字节
-XX:-ResizeTLAB 禁止自动调整TLAB的大小，默认会自动调整
-XX:+PrintTLAB 打印TLAB的使用情况

========== 执行情况 ==========
-XX:-BackgoundCompilation 禁止后台编译
-XX:+DoEscapeAnalysis 开启逃逸分析，在server模式有效
-Xint 解释执行，一般情况下运行速度会低10倍甚至更多
-Xcomp 编译执行，在第一次执行时把所有的字节码编译为本地代码，会损失部分性能，因失去了多次执行时的分支预测编译
-Xmixed 混合模式执行，默认为混合模式
-XX:NativeMemoryTracking={summary|detail} 直接内存跟踪，会导致JVM执行性能下降5-10%
-XX:+PrintTenuringDistribution 指定JVM在每次新生代GC时，输出幸存区中对象的年龄分布

========== 类加载 ==========
-verbose:class 打印class的加载信息
-XX:+TraceClassLoading 查看类的加载信息
-XX:+TraceClassUnloading 查看类的卸载信息，需在FastDebug版的虚拟机中才支持

========== 内存溢出OOM ==========
-XX:+HeapDumpOnOutOfMemoryError 让虚拟机在出现内存溢出异常时，dump出当前的堆转储快照信息
-XX:HeapDumpPath=/path/heap.dump 设置dump文件路径
-XX::OnOutOfMemoryError=/path/xxx.sh 在内存溢出时执行指定的脚本，可以发送邮件，重启应用等

========== 打印JVM参数 ==========
-XX:+PrintVMOptions 在虚拟机启动时打印传递给虚拟机的显示参数
-XX:+PrintCommandLineFlags 在JVM启动时打印所有的参数，包含显示指定的参数及JVM默认的隐式设置的参数
-XX:+PrintFlagsFinal JVM启动时打印所有系统参数的值，包含JVM及当前操作系统的参数
-XX:+PrintInlining 打印方法内联相关信息
-XX:+PrintCompilation 打印即时编译相关信息
-XX:+PrintFlagsInitial	打印虚拟机初始化参数

========== 锁 ==========
-XX:+UseBiasedLocking 开启偏向锁
-XX:BiasedLockingStartupDelay 在JVM启动后立即开启偏向锁，默认在JVM启动4秒后开启，一般JVM在启动时采用
-XX:BiasedLockingBulkRebiaseThreshold=20 偏向锁撤销数，当超过这个数时，JVM宣布偏向锁失效，默认为20
-XX:BiasedLockingBulkRevokeThreshold=40 偏向锁撤销数，当超过这个数时，JVM认为该类对象已不适合偏向锁，并在之后的加锁过程中直接设置为轻量级锁，默认为40
-XX:+PrintBiasedLockingStatistics 在即时编译C1级别时打印各类锁的个数
-XX:+PrintPreciseBiasedLockingStatistics 在即时编译C2级别时打印各类锁的个数
-XX:TieredStopAtLevel=1 限制JVM仅使用C1级别的即时编译

-XX:+UseCondCardMark MinorGC时，会采用CardTable表示老年代对象指向新生代的引用，此标志会提升CardTable更新效率
-XX:+UnlockCommercialFeatures 解锁商业特性
-XX:+FlightRecorder 使用飞行器记录JVM运行情况

jcmd pid VM.unlock_commercial_features 命令行解锁商业特性
jcmd pid JFR.start duration=120s filename=/app/app.jfr 导出一定时间内的JVM运行情况，然后通过JMC（Java Mission Control）查看运行详情。