# tomcat线程池和jdk提供的通用线程池在执行流程上有一点差异
tomcat自身重写了一个`org.apache.tomcat.util.threads.ThreadPoolExecutor`, 继承了jdk的`java.util.concurrent.ThreadPoolExecutor`  
自定义了`org.apache.tomcat.util.threads.TaskQueue`, 继承`java.util.concurrent.LinkedBlockingQueue`

corePoolSize默认10，maxPoolSize默认200，workQueue容量默认Integer.Max_value

## 总体执行流程
```
new ThreadPoolExecutor -> prestartAllCoreThreads -> execute -> SubmittedCount<pool size(pool size最小为core size，因为core size已全部restart)所以进入work queue -> 唤醒core thread执行 -> 
直到当前活跃任务>core size时 -> new max thread -> SubmittedCount等于max size时 -> 加入work queue -> 等待空闲线程执行 ->
执行拒绝策略，默认抛RejectedExecutionException -> 进入catch，queue.offer(timeout)（默认0ms）尝试重新加入队列 -> 加入失败，抛异常
```

```
// 初始化线程池
org.apache.tomcat.util.net.AbstractEndpoint#createExecutor
// 执行
org.apache.tomcat.util.net.AbstractEndpoint#processSocket
org.apache.tomcat.util.threads.ThreadPoolExecutor#execute(java.lang.Runnable)
org.apache.tomcat.util.threads.ThreadPoolExecutor#execute(java.lang.Runnable, long, java.util.concurrent.TimeUnit)
```

## 1.线程池实例化后，会马上初始化core thread
```
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
    super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, new RejectHandler());
    // 线程池实例化后，会马上初始化core thread
    prestartAllCoreThreads();
}

public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true))
        ++n;
    return n;
}
```

## 2.使用自定义的work queue
主要体现在offer接收task时
```
public boolean offer(Runnable o) {
  //we can't do any checks
    if (parent==null) return super.offer(o);
    //we are maxed out on threads, simply queue the object
    // 当前获取线程数==最大线程数，则加入队列阻塞等待空闲线程执行
    if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
    //we have idle threads, just add it to the queue
    // 当前已提交的任务<=活跃线程数，则加入队列阻塞等待空闲线程执行
    if (parent.getSubmittedCount()<=(parent.getPoolSize())) return super.offer(o);
    //if we have less threads than maximum force creation of a new thread
    // 返回false，则在jdk的thread pool executor中会创建max pool thread
    if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
    //if we reached here, we need to add it to the queue
    return super.offer(o);
}
```

## 3.线程池在收到RejectExecutionException时的处理
```
public void execute(Runnable command, long timeout, TimeUnit unit) {
    // 记录当前已提交的任务
    submittedCount.incrementAndGet();
    try {
        // 执行任务,由于core pool size的线程已预先全部开启，所以前面core个task实际上会先进work queue，然后在唤醒core thread执行
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        if (super.getQueue() instanceof TaskQueue) {
            final TaskQueue queue = (TaskQueue)super.getQueue();
            try {
                // 最大等待timeout时间,再次加入队列，失败后在抛异常，默认等待0ms
                if (!queue.force(command, timeout, unit)) {
                    // 已提交任务 -1
                    submittedCount.decrementAndGet();
                    throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
                }
            } catch (InterruptedException x) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(x);
            }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }
    }
}
```