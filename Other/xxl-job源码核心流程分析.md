#服务端：
## add job
com.xxl.job.admin.controller.JobInfoController#add 仅校验并保存数据库
## start job
com.xxl.job.admin.controller.JobInfoController#start 修改状态为1 running

## trigger job
com.xxl.job.admin.core.thread.JobScheduleHelper.scheduleThread
com.xxl.job.admin.core.thread.JobScheduleHelper.ringThread

###1.scheduleThread  查询job  
默认每5秒扫描一次数据库待执行的任务，如果扫描到有需要执行的job，则sleep 1秒，没有则sleep 5秒。  

1.1 每次先使用`select * from xxl_job_lock where lock_name = 'schedule_lock' for update`获取独占锁；  
1.2 查询TriggerNextTime在5秒内待执行的job，每次最大查询默认6000条。  
1.3 判断TriggerNextTime
>1.3.1 如果当前时间 > TriggerNextTime + 5秒，则更新TriggerNextTime；  
>1.3.2 如果当前时间 > TriggerNextTime，则立即触发job，并更新TriggerNextTime，并执行1.3.3；  
>1.3.3 以上都不满足，则加入timeRing：Map<待触发job的秒数, List<JobId>>。

###2.ringThread 触发job
每秒执行一次，将TimeRing中的当前秒数的List<JobId>remove，并执行`com.xxl.job.admin.core.thread.JobTriggerPoolHelper#trigger`
异步触发job。  
如果在一次循环触发job的时候，由于任务过多或者其他原因，导致单次触发时间超过2秒及以上，可能会造成任务丢失。

#客户端执行器
##启动流程
Spring项目通过配置com.xxl.job.core.executor.impl.XxlJobSpringExecutor bean的方式启动客户端  

### 注册本地的JobHandler
com.xxl.job.core.executor.impl.XxlJobSpringExecutor#initJobHandlerMethodRepository  
然后通过`com.xxl.job.core.executor.XxlJobExecutor#registJobHandler`注册JobHandler。   

### 启动EmbedServer
com.xxl.job.core.executor.XxlJobExecutor#start  
com.xxl.job.core.executor.XxlJobExecutor#initEmbedServer  
客户端通过netty启动了一个`com.xxl.job.core.server.EmbedServer`的HTTP服务，端口为`xxl.job.executor.port`，接收服务端的任务调度请求。  
com.xxl.job.core.server.EmbedServer#start  

### 注册executor到服务端
根据配置的远程服务器地址：xxl.job.admin.addresses，多个用,分割  
com.xxl.job.core.thread.ExecutorRegistryThread#start  
com.xxl.job.core.server.EmbedServer#startRegistry  
com.xxl.job.core.biz.client.AdminBizClient#registry  

##执行job
com.xxl.job.core.server.EmbedServer.EmbedHttpServerHandler#process  
com.xxl.job.core.biz.impl.ExecutorBizImpl#run  
com.xxl.job.core.thread.JobThread#run  


