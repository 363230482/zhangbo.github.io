#搭建MongoDB ReplicaSet步骤
由于我这里是单机搭建，所以用配置文件的方式来配置。
我们先创建好对应的日志和data目录。
需要注意如果是单机搭建，需要修改logpath,dbpath及端口信息等。
`mongod1.cnf`:
```text
systemLog:
   destination: file
   path: "/home/zhangbo/apps/mongodb/log/mongod3.log" #指定log文件
   logAppend: true
storage:
   dbPath: "/home/zhangbo/apps/mongodb/data3" #指定data目录
   journal:
      enabled: true
processManagement:
   fork: true
net:
   bindIp: 127.0.0.1
   port: 27047 #指定IP和端口
setParameter:
   enableLocalhostAuthBypass: false
replication:
   replSetName: "rsTest" #副本集名称，整个副本集必须一致
```
然后就可以启动各个mongd实例，我这里依次启动了3个实例。
```text
./bin/mongod --config mongo1.cnf
./bin/mongod --config mongo2.cnf
./bin/mongod --config mongo3.cnf
```
启动之后，我们可以用`ps -ef |grep mongo`查看是否启动成功。
然后，我们使用`mongo`客户端，连接上任一启动好的实例,由于此处是单机，需要指定端口：
```text
./bin/mongo --port 27027
```
连接上之后，我们就开始副本集的初始化操作,注意外面的`_id`和配置文件里面的`replSetName`保持一致：
```text
rs.initiate( {
   _id : "rsTest",
   members: [
      { _id: 0, host: "127.0.0.1:27027" },
      { _id: 1, host: "127.0.0.1:27037" },
      { _id: 2, host: "127.0.0.1:27047" }
   ]
})
```
上述操作返回`OK`后，我们可以使用`rs.conf()`指令查看当前副本集的配置。
也可以通过`rs.status()`查看当前副本集的详细状态。
此时，如果我们登录到Primary节点，会显示`rsTest:PRIMARY>`,其他节点则显示`rsTest:SECONDARY>`。

到此，整个副本集就搭建完成了。我们可以在Primary节点上写入测试数据：
```text
use test #创建test数据库
db.user.insert({"_id":1,"name":"test"}) #插入数据
db.user.find() #查询数据
```
然后我们在Secondary节点上试试查询：
```text
use test #切换到test数据库
db.user.find() #查询数据，默认会报错`not master and slaveOk=false`

上面Secondary节点的查询和写入默认要报错，因为Secondary默认是不能读和写的，那我们可以开启Sedondary节点的读操作：
```text
rs.slaveOk() #开启读
db.user.find() #查询，此时应该可以看到查询出了数据
```
由于Secondary节点是不能写入数据的，所以如果你在Secondary节点写入数据的话，应该会看到`"not master"`的报错信息。