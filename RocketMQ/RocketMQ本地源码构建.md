# 源码构建流程

## git clone 
下载源码到本地，导入IDEA，然后`mvn compile`不报错
```
git clone https://github.com/apache/rocketmq.git
```

## 启动 Namesrv
我们直接找到`org.apache.rocketmq.namesrv.NamesrvStartup`,main方法启动就可以了。启动报错
```text
Please set the ROCKETMQ_HOME variable in your environment to match the location of the RocketMQ installation
```
根据错误提示，需要我们设置环境变量`ROCKETMQ_HOME`。但怎么设置呢？不过我们在源码搜索一下`variable in your environment`，看到
```java
// org.apache.rocketmq.namesrv.NamesrvStartup#createNamesrvController
if (null == namesrvConfig.getRocketmqHome()) {
    System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
    System.exit(-2);
}
```
那我们在看下`org.apache.rocketmq.common.namesrv.NamesrvConfig#rocketmqHome`,发现源码如下：
```java
// org.apache.rocketmq.common.namesrv.NamesrvConfig
// ROCKETMQ_HOME_PROPERTY = "rocketmq.home.dir";
// ROCKETMQ_HOME_ENV = "ROCKETMQ_HOME";
private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
```
看到这里，我们为了方便，可以直接在源码`NamesrvStartup`的main方法第一行中。你也可以设置环境变量`ROCKETMQ_HOME`,或者在启动时接入命令行参数`-Drocketmq.home.dir=/path`
```
public static void main(String[] args) {
    System.setProperty(MixAll.ROCKETMQ_HOME_PROPERTY, "D:\\IdeaProjects\\github\\rocketmq");
    main0(args);
}
```
然后在启动，结果还是报错，报错信息如下：
```text
Caused by: java.io.FileNotFoundException: D:\IdeaProjects\github\rocketmq\conf\logback_namesrv.xml (系统找不到指定的文件。)
```
提示在我们配置的`ROCKETMQ_HOME`目录下，没有找到`conf\logback_namesrv.xml`配置文件，那我们在项目中搜索`logback_namesrv.xml`文件，
找到在`distribution\conf`目录下存在很多配置，那我们可以直接使用这些配置文件就可以了。我们直接把`distribution\conf`目录copy到我们配置的`ROCKETMQ_HOME`目录下就可以了。
然后在来启动一下，控制台打印`The Name Server boot success. serializeType=JSON`,说明NameSrv终于启动成功了。
下面我们来启动一下Broker

## 启动 Broker
我们还是惯例找到`org.apache.rocketmq.broker.BrokerStartup`,直接启动试试，此时控制台报异常
```text
Please set the ROCKETMQ_HOME variable in your environment to match the location of the RocketMQ installationDisconnected from the target VM, address: '127.0.0.1:51797', transport: 'socket'
```
是不是很熟悉，不用多说，直接把Namesrv中添加的那一行代码copy过来，然后我们知道broker是需要连接namrsrv的，所以我们在`BrokerStartup`的main方法第一行添加如下两行代码,同样，你也可以设置命令行参数来达到同样的效果
```
public static void main(String[] args) {
    System.setProperty(MixAll.ROCKETMQ_HOME_PROPERTY, "D:\\IdeaProjects\\github\\rocketmq");
    System.setProperty(MixAll.NAMESRV_ADDR_PROPERTY, "127.0.0.1:9876");
    start(createBrokerController(args));
}
```
然后在启动试试，结果控制台奇迹般的打印如下，表示启动成功了，好吧。。。
```text
The broker[DESKTOP-TL22HJH, 10.22.13.146:10911] boot success. serializeType=JSON
```
到这里，虽然表示启动成功了，那我们再来验证一下是否可以成功发送和接收消息呢？

## producer send message
先来一个Producer的Demo。
```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

public class ProducerTest {
    public static void main(String[] args) throws Exception {
        String namesrvAddr = "localhost:9876";
        String group = "test_group";
        String topic = "test_hello_rocketmq";
        // 构建Producer实例
        DefaultMQProducer producer = new DefaultMQProducer();
        producer.setNamesrvAddr(namesrvAddr);
        producer.setProducerGroup(group);
        // 启动producer
        producer.start();
        // 发送消息
        SendResult result = producer.send(new Message(topic, "hello rocketmq".getBytes()));
        System.out.println(result.getSendStatus());
        // 关闭producer
        producer.shutdown();
    }
}
```
至此，控制台打印`SEND_OK`,我们的producer发送消息也OK了，接下来，我们在来试试Consumer消费消息。

## consumer receive message
直接上Consumer的代码。
```java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerOrderly;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;

import java.util.List;
import java.util.concurrent.TimeUnit;

public class ConsumerTest {

    public static void main(String[] args) throws Exception {
        String namesrvAddr = "localhost:9876";
        String group = "test_consumer_group";
        String topic = "test_hello_rocketmq";
        // 初始化consumer
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer();
        consumer.setNamesrvAddr(namesrvAddr);
        consumer.setConsumerGroup(group);
        // 订阅topic
        consumer.subscribe(topic, (String) null);
        // 设置消费的位置，由于producer已经发送了消息，所以我们设置从第一个开始消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        // 添加消息监听器
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                msgs.forEach(msg -> {
                    System.out.println(new String(msg.getBody()));
                });
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        // 启动consumer
        consumer.start();
        // 由于是异步消费，所以不能立即关闭，防止消息还未消费到
        TimeUnit.SECONDS.sleep(2);
        consumer.shutdown();
    }
}
```
启动消费者，能够成功消费到消息，控制台打印`hello rocketmq`，至此，源码本地搭建的调试流程就结束了，