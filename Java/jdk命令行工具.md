工具官方文档：[https://docs.oracle.com/javase/8/docs/technotes/tools/]

## jps
jps： JVM Process Status Tool，JVM进程监视工具，列出正在运行的虚拟机进程
命令格式： jps [option] [hostid]
option：
-q 只输出LVMID(Local VM Identifier)，省略主类的名称
-m 输出虚拟机启动时传递给主类main()的参数
-l 输出主类的全面，如果进程是jar，则输出jar所在的路径
-v 输出虚拟机启动时JVM的参数

## jstat
jstat： JVM statistics Monitoring Tool，JVM统计信息监控工具，监控各种运行状态信息
命令格式：jstat [option vmid [interval [s|ms] [count]]]
interval和count代表查询间隔时间和次数，省略默认为1次
option：
-class 监视类装载，卸载数量，总空间及类装载所耗费的时间
-gc 监视Java堆，包括Eden，两个Survivor，老年代，永久代等的容量，已用空间，GC时间合计等信息
-gccapacity 与-gc基本相同，主要关注堆中各个区域用到的最大，最小空间
-gcutil 与-gc基本相同，主要关注堆中各个区域已使用的百分比
-gccause 与-gcutil一致，但会额外输出上一次GC的原因
-gcnew 监视新生代的GC情况
-gcnewcapacity 与-gcnew基本相同，关注新生代的最大，最小空间
-gcold
-gcoldcapacity
-gcpermcapacity
-compiler 输出被JIT编译过的方法，耗时等信息
-printcompilation 输出已经被JIT编译成功的方法

## jinfo
jinfo：Configuration Info for Java，Java配置信息工具，实时查看和调整虚拟机参数
命令格式： jinfo [option] pid
option：
-sysprops 输出虚拟机进程的System.getProperties()的结果
-flag 查看或修改部分虚拟机参数，如：
-flag [+|-]name 查看/启用/禁用属性name 
-flag name=value 动态设置属性

## jmap
jmap：Memory Map for Java，Java内存映射工具，用于生成堆转储快照
命令格式：jmap [option] vmid
option：
-dump 生成Java堆转储快照，格式为-dump:[live,]format=b,file=filename 其中live,表示是否只dump存活的对象
-finalizeinfo 显示在F-Queue队列中等待Finalizer线程执行finalize()的对象，只在Linux/Solaris平台下有效
-heap 显示Java堆的详细信息，如使用的垃圾回收器，参数配置，分代状况等，只在Linux/Solaris下有效
-histo 显示堆中对象统计信息，包括类，实例对象，合计数量
-permstat 以ClassLoader为统计口径，显示永久代内存状况，只在Linux/Solaris下有效
-F 当虚拟机对-dump选项没有响应时，使用此选项强制生成dump快照，只在Linux/Solaris下有效

## jhat
jhat：JVM Heap Analysis Tool，堆转储快照分析工具，分析jmap生成的堆转储快照
命令格式：jhat [-port 7000] xxx.heapdump 
该命令会创建一个Web服务，用来在浏览器中查看分析的dump文件

## jstack
jstack：Stack Trace for Java，Java堆栈跟踪工具，用于生成虚拟机当前时刻的线程快照
命令格式：jstack [option] vmid
option：
-F 当正常输出不被响应时，强制输出线程堆栈快照
-l 除了线程堆栈外，显示关于锁的附加信息
-m 如果调用到本地方法的话，可以显示C/C++的堆栈

## jcmd
jcmd：多功能命令行
命令格式：jcmd -l 列出jvm的pid
jcmd pid help 列出所支持的命令，pid也可以用main方法的权限定名来代替
