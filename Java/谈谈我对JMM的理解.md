JMM语言级内存模型 > CPU硬件级内存模型 > 顺序型内存模型  
正确同步，禁止一定的指令重排序 > 不改变单线程的指令重排序 > 不重排序(理论模型，执行效率最低)

首先，在并发编程模型里，线程之间如何通信以及线程之间如何正确的同步。线程通信主要有两种方式：
+ 共享内存，需要程序员显示的同步，Java采用这种，所以提供了synchronized/volatile/final/Lock等
+ 消息传递，由于消息的发送必须在消息的接收之前，所以是隐式的同步  

JMM是对硬件，操作系统等做了一个抽象，他主要分为了主内存和工作内存。主内存是所有线程共享的。工作内存是线程私有的，不会存在同步问题。  

由于各级硬件的操作速度不同，所以线程在读写一个变量时，会先将该变量从主内存copy到自己的工作内存，
就导致了同一个变量存在多个副本集，就可能会导致数据不一致的问题。

所以JMM提供了一系列的手段，来保证原子性/可见性/有序性等。
如``as-if-serial`,`happens-before`,`synchronized`,`volatile`,`final`等。

1.as-if-serial  
比如由于编译器优化和运行时的处理器优化，可能会对没有数据依赖关系的指令进行重排序。
但JMM规定了各级优化的指令重排序，一定要遵守`as-if-serial`原则，即保证优化后的单线程执行结果是不会改变的。

2.happens-before  
>+ 一个线程的start `happens-before`该线程后续的任意操作；
>+ 锁的释放`happens-before`该锁后续的获取；
>+ volatile写`happens-before`后续的volatile读等；
>+ `happens-before`还具有传递性，如A`happens-before`B，B`happens-before`C，那么A一定`happens-before`C。

3.volatile  
`volatile`通过插入内存屏障的方式，来禁止处理器的重排序，保证线程可见性。  
`volatile`读会强制将工作内存的数据失效，去主内存读取最新的数据；
`volatile`写则强制将工作内存的数据刷新到主内存，不利用写缓冲区带来的效率。
`volatile`读会在该指令后方插入`LoadLoad`和`LoadStore`指令，来禁止重排序。
`volatile`写会在该指令前插入`StoreStore`,后面插入`StoreLoad`指令。  
但由于`volatile`不能保证原子性，所以`volatile`仅用于单个变量读写，如果需要原子性，则可以考虑同步操作。  

4.synchronized  
`synchronized`通过对象头的Monitor对象来实现线程独占的方式(排它锁)来处理线程之间的同步。  
上面说的`volatile`读则相当于`synchronized`锁的获取的语义，清空当前线程的工作内存的数据，强制从主内存读取最新的数据；
`volatile`写相当于`synchronized`锁的释放，强制刷新当前线程的工作内存的写缓冲区到主内存。  
但由于`volatile`不能保证原子性，所以复制操作需要依赖`synchronized`，
但同样`synchronized`不能禁止指令重排序，所以在单例模式的双重检查锁中，需要`volatile`来禁止new对象过程的指令重排序。

5.final  
final由于在赋值后就不能在改变它的值，正是由于它的不变性，同样也可以安全的在多个线程之间传递共享。  
但需要注意的是，final修饰的引用类型的变量，仅是其引用的地址不可变，里面的值需要使用者注意是否能更改。