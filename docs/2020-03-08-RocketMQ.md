# RocketMQ

[TOC]

## 1. 安装

64bit OS Linux/Uinx/Mac

64bit JDK1.8+

Maven 3.2.x

Git

4G+ free disk for Broker server

### 1.1 下载

http://rocketmq.apache.org/

```
配置主机
127.0.0.1       rocketmq-nameserver1
127.0.0.1       rocketmq-master1
```

### 1.2 创建文件目录存放日志等

```shell
mkdir logs
mkdir store
cd store/
mkdir commitlog
mkdir consumequeue
mkdir index
```

logs存储rocketmq日志目录、store存储数据文件目录、commitlog存储消息信息

consumequeue、index存储消息的索引数据

```
conf目录配置文件说明

2m-2s-async：双主双从异步
2m-2s-sync：双主双从同步
2m-noslave：双主

先搭建但节点，进入任意一个配置即可
进入2m-2s-async目录，修改第一个配置文件broker-a.properties
vi broker-a.properties
```

将如下配置覆盖掉

```properties
# 所属集群名字
brokerClusterName=rocketmq-cluster
# broker名字 注意此处不同的配置文件填写不一样
brokerName=broker-a
# 0表示Master，>0表示slave
brokerId=0
# nameServer地址，分号分隔
nameSrvAddr=rocketmq-nameserver1:9876
# 在发送消息时 自动创建服务器不存在的Topic，默认创建的队列数
defaultTopicQueueNums=4
# 是否允许Broker自动创建Topic，建议线下开启 线上关闭
autoCreateTopicEnable=false
# 是否允许Broker自动创建订阅组，建议线下开启 线上关闭
autoCreateSubscriptionGroup=false
# Broker对外服务的监听端口
listenPort=10911
# 删除文件时间点 默认凌晨4点
deleteWhen=04
# 文件保留时间 默认48小时
fileReservedTime=120
# commitlog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
# ConsumeQueue每个文件默认存30w条 针具业务情况调整
mapedFileSizeConsumeQueue=300000
# destoryMapedFileIntervalForcibly=120000
# redeleteHangedFileInterval=120000
# 检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
# 存储路径
storePathRootDir=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store
# commitLog存储路径
storePathCommitLog=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store/commitlog
# 消费队列存储路径
storePathConsumeQueue=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store/consumequeue
# 消费索引存储路径
storePathIndex=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store/index
# checkpoint文件存储路径
storeCheckpoint=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store/checkpoint
# abort文件存储路径
abortFile=/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/store/abort
# 限制的消息大小
maxMessageSize=65536
# flushCommitLogLeastPages=4
# flushConsumeQueueLeastPages=2
# flushCommitLogThoroughInterval=10000
# flushConsumeQueueThoroughInterval=60000
# Broker的角色
# - ASYNC_MASTER 异步复制Master
# - SYNC_MASTER 同步双写Master
# - SLAVE
brokerRole=ASYNC_MASTER
# 刷盘方式
# - ASYNC_FLUSH 异步刷盘
# - SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
# checkTransactionMessageEnable=false
# 发消息线程池数量
# sendMessageTreadPoolNums=128
# 拉消息线程池数量
# pullMessageTreadPoolNums=128
```

进入conf目录 替换所有xml中的${user.home}，保证日志目录正确 下面指令全部替换

```shell
sed -i 's#${user.home}#/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release#g' *.xml
```

sed -i起批量替换作用

```shell
mac下使用
sed -i '' 's#${user.home}#/Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release#g' *.xml
```

### 1.3 内存配置

rocketmq对内存的要求比较高，最少1G，如果内存太少 会影响运行效率和执行性能。我们需要修改bin目录下的runbroker.sh和runserver.sh

runbroker.sh

```
改前：
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"

改后：
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g"
```

runserver.sh

```
改前：
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"

改后：
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

### 1.4 启动

先启动na mesrv

```shell
nohup sh mqnamesrv &
```

再启动broker

```shell
nohup sh mqbroker -c /Users/stefan/software/mq/rocketmq-all-4.6.1-bin-release/conf/2m-2s-async/broker-a.properties > /dev/null 2>&1 &
```

停掉broker

```
sh mqshutdown broker
```

停掉namesrv

```
sh mqshutdown namesrv
```

再输入jps查看进程

```shell
MacBook-Pro:bin stefan$ jps
62387 BrokerStartup
62275 NamesrvStartup
99058 Launcher
62413 Jps
98927 
MacBook-Pro:bin stefan$ 
```

### 1.5 控制台安装

https://github.com/apache/rocketmq-externals

## 2. 详细介绍

### 2.1 概念

* producer：消息生产者，负责产生消息，一般有业务系统负责产生
* consumer：消息消费者，负责消费消息，一般是后台系统负责异步消费
* push consumer：consumer的一种，应用通常向consumer对象注册一个Listener接口，一旦收到消息，consumer对象立刻回调Listener接口方法
* pull consumer：consumer的一种，应用通常主动调用consumer的拉消息方法，从broker拉消息，主动权由应用控制
* producer group：一类producer的集合名称，这类producer通常发送一类消息，且发送逻辑一致
* consumer group：一类consumer的集合名称，这类consumer通常消费一类消息，且消费逻辑一致
* broker：消息中专角色，负责存储消息、转发消息，一般也称为server，在JMS规范中称为provider

### 2.2 消息生产和消费介绍

1. rocketmq可以发送普通消息、顺序消息、事务消息，顺序消息能实现有序消费，事务消息可以解决分布式事务实现数据最终一致性。

2. rocketmq有2种常见的消费模式，分别是DefalutMQPushConsumer和DefaultMQPullConsumer。其实无论是push合适pull，其本质都是拉取消息，只是实现机制不同
3. D efaultMQPushConsumer其实并不是broker主动向sonsumer推送消息，二十consumer向broker发送请求，保持了一种长链接，broker会每5秒检测一次是否有消息，如果有消息则把消息推送给consumer。使用DefaultMQPushConsumer实现消息消费，broker会主动记录消息消费的偏移量
4. DefaultMQPullConsumer是消费方主动去broker拉取数据，一般会在本地使用定时任务实现，使用它获取消息状态方便、负载均衡性能可控，但消息的及时性差，且需要手动记录消息消费的偏移量信息，所以工作中多数推荐使用push模式
5. rocketmq发送的消息默认会存储到4个队列中，当然创建几个队列存储数据可以自己定义

### 2.3 消息发送

消息发送有这么几个流程

```
1. 创建DefaultMQProducer
2. 设置Namesrv地址
3. 开启DefaultMQProducer
4. 创建消息Message
5. 发送消息
6. 关闭DefaultMQProducer
```

1. 引入坐标

   ```xml
   <dependency>
     <groupId>org.apache.rocketmq</groupId>
     <artifactId>rocketmq-client</artifactId>
     <version>${rocketmq-client.version}</version>
   </dependency>
   ```

2. 创建一个Producer类，按照上面步骤实现消息发送

   ```java
   private void Producer() throws Exception {
     String NAMESRV_ADDRESS = "127.0.0.1:9876";
   
     // 创建
     DefaultMQProducer producer = new DefaultMQProducer("Test");
     // 设置
     producer.setNamesrvAddr(NAMESRV_ADDRESS);
     // 开启
     producer.start();
     /**
      * 构建消息
      * topic：主题
      * tags：主要用于消息过滤
      * keys：消息的唯一值
      * body：消息
      */
     Message message = new Message("Topic_Demo",
                                   "Tags",
                                   "Keys_1",                           "hello".getBytes(RemotingHelper.DEFAULT_CHARSET));
   
     // 发送
     SendResult result = producer.send(message);
     System.out.println(result);
   
     // 关闭
     producer.shutdown();
   }
   ```

### 2.4 消息消费

消费消息几个步骤

```
1. 创建DefaultMQPushConsumer
2. 设置namesrv地址
3. 设置subscribe 这里是要读取的主题信息
4. 创建消息监听MessageListener
5. 获取消息信息
6. 返回消息读取状态
```

```java
private void consumer() throws Exception {
  String NAMESRV_ADDRESS = "127.0.0.1:9876";
  DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("Test");
  consumer.setNamesrvAddr(NAMESRV_ADDRESS);
  /**
   * topic：主题
   * subExpression：标签 可以为tags
   */
  consumer.subscribe("Topic_Demo", "*");

  // 创建消息监听MessageListener
  consumer.setMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list,
                                                    ConsumeConcurrentlyContext consumeConcurrentlyContext) {
      try {
        // 迭代消息信息
        for (MessageExt messageExt : list) {
          // 获取主题
          String topic = messageExt.getTopic();
          // 获取标签
          String tags = messageExt.getTags();
          // 获取信息
          byte[] body = messageExt.getBody();
          String bodyStr = new String(body, RemotingHelper.DEFAULT_CHARSET);

          System.out.println("消息:Topic="+topic+"Tags="+tags+"body="+bodyStr);
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
      } catch (UnsupportedEncodingException e) {
        e.getStackTrace();
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
      }
    }
  });
  consumer.start();
}
```

### 2.5 如何顺序消费

因为默认生产4个队列，全局消息是不保证消费顺序的。可以只配置一个队列或者指定队列发送就可以保证顺序消费。下面为指定队列消费例子

生产者

```
只需要改造发送
// 发送
/**
 * 第一个：发送的消息
 * 第二个：选中指定消息队列对象
 * 第三个：指定对应的队列下标
 */
 SendResult result = producer.send(message, new MessageQueueSelector() {
 @Override
 public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
 // 获取队列的下标
 Integer index = (Integer) o;
 // 获取对应下标的队列
 return list.get(index);
 }
 }, 0);
```

消费者

```java
// 创建消息监听MessageListener
consumer.setMessageListener(new MessageListenerOrderly() {
  @Override
  public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
    try {
      // 迭代消息信息
      for (MessageExt messageExt : list) {
        // 获取主题
        String topic = messageExt.getTopic();
        // 获取标签
        String tags = messageExt.getTags();
        // 获取信息
        byte[] body = messageExt.getBody();
        String bodyStr = new String(body, RemotingHelper.DEFAULT_CHARSET);

        System.out.println("消息:Topic="+topic+"Tags="+tags+"body="+bodyStr);
      }
      return ConsumeOrderlyStatus.SUCCESS;
    } catch (UnsupportedEncodingException e) {
      e.getStackTrace();
      return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
    }
  }
});
```

### 2.6 事务消息

在rocketmq4.3.0版本后，开放了事务消息这一特性，对于分布式事务而言，最常说的还是二阶段提交协议

1. 事务消息流程

主要是通过消息的异步处理，可以保证本地事务和消息发送同时成功执行或失败，从而保证数据的最终一致性

```java
// 代码中使用
TransactionMQProducer producer = new xxxx
  
// 指定消息监听对象，用于执行本地事务和消息回查
producer.setTransactionListener(自己实现);
// 添加线程池

TransactionSendResult result = producer.sendMessageInTransaction(message, "hello-transaction");
```

2. 因为mq是数据最终一致性，还需要业务补偿等侵入 建议用seata

### 2.7 广播消息

消费者

```
// 默认消费模式是集群模式 设置为广播模式
consumer.setMessageModel(MessageModel.BROADCASTING);
```

## 3. 集群

### 3.1 集群模式

1. 单个master

   一旦broker重启或宕机，将导致服务不可用

2. 多个master

   一个集群全都是master没有slave

   优点：配置简单，单个master宕机或重启对应用什么影响，在磁盘配置为RAID10时，即使机器宕机不可恢复，消息也不会丢失(异步刷盘丢失少量消息，同步刷盘则是一条都不会都是)，性能最好

   缺点：当单个broker宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息的实时性会受到影响

3. 多master多slave 异步复制

   每个master配置一个slave，有多对的master-slave，HA采用的是异步复制方式，主备有短暂的消息延迟，毫秒级别的(master收到消息之后立刻向应用返回成功标识，同时向slave写入)

   优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受到影响，因为master宕机之后，消费者仍可以从slave消费，此过程对应用透明，不需要人工干预，性能同多个master模式几乎一样

   缺点：master宕机磁盘损坏的情况下，会丢失少量消息

4. 多master多slave 同步双写

   每个master配置一个slave，有多对master-slave。HA采用同步双写模式，主备都写成功，才会向应用返回成功

   优点：数据与服务都无单点，master宕机情况下消息无延迟，服务可用性与数据可用性都非常高

   缺点：性能比异步复制略低大约10%左右，发送单个master的RT会略高，目前主机宕机后，slave不能自动切换为主机，后续会支持自动切换功能

### 3.2 一主一从搭建

同前面配置一样，只需把broker-a-s.properties文件改成上面一样 稍稍改动

```properties
brokerId=1
brokerRole=SLAVE
```

### 3.3 双主双从

同上 配置即可























