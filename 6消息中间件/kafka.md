### kafka

#### 1. 消息队列

- 两种数据传输模式
  - 点对点模式（peer to peer）
  
    1. 接收者主动拉取数据
    2. 发送到队列的消息只能被一个接收者接收
    3. 既支持异步的消息传送方式，又支持同步的消息接收方式
  
    4. 缺点：需要实时监控消息队列中的数据
  
  - 发布/订阅模式（一对多）
  
    1. 数据产生后，将消息推送给多个订阅的消费者
    2. 消费数据时，push和pull都可以
    3. 解耦能力比p2p模式更强
    4. 缺点：订阅者多，数据生成速率和接收收速率不一致，产生木桶效应。
  
- 为什么使用消息队列
  - 解耦，将数据生产者和消费者解耦
  - 冗余，生产与消费之间多了一层，数据存放在消息队列中
  - 可恢复，存在消息队列中的数据可以恢复
  - 灵活，可扩展，消峰，缓冲
  - 顺序，先进先出
  - 异步通信
  
- Kafka的设计目标

  - 高吞吐量

    在8核16G的虚拟机，网卡1G的机器上，可支持100万条（每条100B）/秒消息的读写

  - 消息持久化

    所有消息均被持久化到磁盘，无消息丢失，支持消息重放（从头消费）

  - 完全分布式

    producer，broker，consumer支持水平扩展

  - 同时支持在线流处理和离线批处理

#### 2. 初步了解kafka

kafka是LinkedIn公司开发，后交到Apache上维护，目标是为处理实时数据提供一个统一、高通量、低等待的平台。

kafka是分布式消息队列。producer生产数据后按照topic归类放入kafka中，kafka集群有多个kafka实例组成，每个实例被称为broker。

无论是consumer还是kafka，都依赖**zookeeper**集群保存meta信息，原将断点信息保存在内存中，现将断点信息持久化

#### 3. 基本架构图

![](img\kafka\kafka基本架构图.jpg)

- kafka集群是依赖zookeeper的
- borker是kafka集群的单个实例
- 信息是按照topic分类的，每一个topic可以分为不同的partition
- 在同一组的consumer不能同时消费同一个partition
- fowllower只做备份，不做负载；想负载可以把不同partition的leader放入不同的borker中
- 架构优势
  - 把leader放在不同的broker中，提高集群负载
  - 一个组内的消费者可以消费不同分区数据，提高并发

#### 4. 配置文件

- server.properties

  - 配置broker唯一标识

    ```properties
    # The id of the broker. This must be set to a unique integer for each broker.
    broker.id=0
    ```

  - 配置生成日志路径， 这里可能会存放持久化的数据

    ``` properties
    # A comma separated list of directories under which to store log files
    log.dirs=/tmp/kafka-logs
    ```

  - 配置zookeeper集群

    ``` properties
    # Zookeeper connection string (see zookeeper docs for details).
    # This is a comma separated host:port pairs, each corresponding to a zk
    # server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
    # You can also append an optional chroot string to the urls to specify the
    # root directory for all kafka znodes.
    zookeeper.connect=localhost:2181
    ```

#### 5. 操作命令

- 启动kafka broker

  ```shell
  bin/kafka-server-start.sh config/server.properties
  ```

- 创建topic

  ```shell
  bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic topic1 --partitions 3 --replication-factor 1
  ```

- 查看topic

  ```shell
  bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic topic1
  ```

- 启动控制台消费者，并指定消费topic1

  ```shell
  bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic topic1
  ```

- 启动控制台生产者

  ```shell
  bin/kafka-console-producer.sh --broker-list localhost:9092 --topic topic1
  ```

  

#### 6. 工作流程

##### 6.1生产数据

- 写入方式

  producer采用push的方式将消息发布到broker，每条消息都被append到分区（partition）的segment（partition是文件夹，segment是文件，segment文件有两个：index文件记录每条数据的offset值，log文件存储数据）中，属于顺序写磁盘（顺序写磁盘比随机写内存效率高，保障kafka的吞吐率）

- 分区（partition）

  每条消息都被append到分区（partition）中，这些消息的offset在同一区内是有序的

  - 分区的原因

    1. 方便在集群中扩展，每个分区可以通过调整来适应其所在的机器
    2. 提高并发，可以以区为单位消费

  - 分区的方式

    1. 指定某个区，则直接使用这个区
    2. 未指定区，但是指定了key，可以通过key来hash出一个区
    3. 区和key都未指定，使用轮询选出一个区

    ```java
    package org.apache.kafka.clients.producer.internals;
    
    import java.util.List;
    import java.util.Map;
    import java.util.concurrent.ConcurrentHashMap;
    import java.util.concurrent.ConcurrentMap;
    import java.util.concurrent.ThreadLocalRandom;
    import java.util.concurrent.atomic.AtomicInteger;
    
    import org.apache.kafka.clients.producer.Partitioner;
    import org.apache.kafka.common.Cluster;
    import org.apache.kafka.common.PartitionInfo;
    import org.apache.kafka.common.utils.Utils;
    
    /**
     * The default partitioning strategy:
     * <ul>
     * <li>If a partition is specified in the record, use it
     * <li>If no partition is specified but a key is present choose a partition based on a hash of the key
     * <li>If no partition or key is present choose a partition in a round-robin fashion
     */
    public class DefaultPartitioner implements Partitioner {
    
        private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap<>();
    
        public void configure(Map<String, ?> configs) {}
    
        /**
         * Compute the partition for the given record.
         *
         * @param topic The topic name
         * @param key The key to partition on (or null if no key)
         * @param keyBytes serialized key to partition on (or null if no key)
         * @param value The value to partition on or null
         * @param valueBytes serialized value to partition on or null
         * @param cluster The current cluster metadata
         */
        public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
            List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
            int numPartitions = partitions.size();
            if (keyBytes == null) {
                int nextValue = nextValue(topic);
                List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
                if (availablePartitions.size() > 0) {
                    int part = Utils.toPositive(nextValue) % availablePartitions.size();
                    return availablePartitions.get(part).partition();
                } else {
                    // no partitions are available, give a non-available partition
                    return Utils.toPositive(nextValue) % numPartitions;
                }
            } else {
                // hash the keyBytes to choose a partition
                return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
            }
        }
    
        private int nextValue(String topic) {
            AtomicInteger counter = topicCounterMap.get(topic);
            if (null == counter) {
                counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
                AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
                if (currentCounter != null) {
                    counter = currentCounter;
                }
            }
            return counter.getAndIncrement();
        }
    
        public void close() {}
    
    }
    
    ```

- 副本

  - CAP理论

    - Consistency

      一致性有：强一致性、弱一致性、最终一致性

    - Availablity

      在一定时间内做出正确的回复

    - Partition tolerance

      区域间中断，但是区域内部可以正常通信

  - 一致性方案

    - Master-slave

      1. RDBMS的读写分离即为典型的Master-slave方案
      2. 同步复制可保证一致性，但会影像可用性
      3. 异步复制提高可用性，但不保证一致性

    - WNR

      1. 主要用于去中心化（P2P）的分布式系统中。DynameoDB和Cassandra采用此方案
      2. N代表副本数，W代表至少写成功多少个副本，R代表至少读成功多少个副本
      3. 当W+R>N时，读取时至少有一个是最新的数据
      4. 多个写操作的顺序难以保证，可能导致多副本间的写操作顺序不一致，DynameoDB通过向量时钟保证最终一致性

  -  副本如何获取数据
    
     类似于consumer消费数据，follower主动从leader中拉取数据
     
  - ISR

    1. leader会维护一个replication列表，这个表为ISR（in-sync Replica）
    2. 如果一个follower比leader落后太多，或者超过一段时间未向leader发起数据复制请求，则leader将其从ISR中删除
    3. 当ISR中所有的Replica都向leader发送数据复制的请求，则leader可以commit
    
  - Commit策略

    replica.lag.time.max.ms=10000 #10s内follower必须给leader发起复制请求

    replica.lag.max.messages=4000 #leader与follower消息量差异必须小于4000

    min.insync.replicas=1 #ISR中的replica数量最小为1

    request.required.acks=0 #值为-1时启动ISR动态维持一致性和可用性

  - replica恢复

    ![](img\kafka\replica恢复.png)

    BrokerA恢复时，先将数据truncate到m1，然后去leader catch up数据，最后加入到ISR中。

    但由于m3没有提交成功，此时m3数据的提交会retry3次，如果都没有成功，做其他处理（做更长时间的retry或持久化到其他地方），如果其中B选为leader，m3会发送至B

    因此，**kafka保证的数据顺序性，是提交后数据的顺序性**

  - 如何处理ISR中所有的replica都宕机？

    1. 等待ISR中任意一个replica启动成为leader（可用性差，一致性好）
    2. （kafka默认）等待任意一个replica启动成为leader（可用性好，一致性差，因为在ISR中的replica可保证版本最新）

- 写入流程

  生成者push数据时，寻找分区的leader

  ACK应答机制：0、1、all

  - ack = 0

    生产者只管推数据，leader不必应答

  - ack = 1

    leader必须应答，但follower不必应答

  - ack = all或-1

    leader和follower都必须应答，保证数据不丢失
    
    此时使用ISR，动态维持一致性和可用性
  
- 生产者类型（producer.type）

  - sync producer

    低延迟、低吞吐率、无数据丢失

  - aync producer

    高延迟、高吞吐率、可能会有数据丢失

    数据放入queue中，到达一定数量或时间整体发送，如果queue满了，再进来数据会被扔掉

- 官方配置文档

  http://kafka.apache.org/08/documentation.html#producerconfigs

##### 6.2broker保存数据

- **本地存储位置**： topic下的log文件中

- 可以配置文件的删除时间

  - 基于时间：log.retention.hours=100
  - 基于大小：log.retention.bytes=1313131315
  - 读取特定消息的时间复杂度为O(1)，即与文件大小、数量无关。我猜是根据索引找数据的

- **在zookeeper存储位置**：/brokers  /consumer

  ![](img\kafka\zookeeper存储kafka元数据.png)

##### 6.3消费数据

- 高级API

  - 不需要管理offset，系统通过zookeeper自行管理。

  - 不需要管理分区、副本等情况。

  - 由于无法管理消费者的offset，所以同一消费者无法控制从哪里消费，可以换一个其他组的消费者从头消费。

- 低级API

  - 代码复杂，功能强大

- 消费者组

  同一消费者组内的消费者，无法同时消费同一分区
  
- 可以使用Kafka中的Mirror Maker将消息从一个数据中心镜像到另一个数据中心

- consumer rebalance

  1. 记P为存放partition的集合
  2. 记C为消费者组中消费同一topic的消费者集合
  3. N=size(P)/size(C) 向上取整
  4. 解除Ci对原来分配的Partition的消费权
  5. 将第i*N到（i+1）\*N-1个partition分配给Ci

#### 7.kafka API

maven包

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.11.0.2</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka_2.12</artifactId>
    <version>0.11.0.2</version>
</dependency>
<!-- kafka stream -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>0.11.0.2</version>
</dependency>
```



##### 7.1 生产者高级API

1. 生产者配置

```java
	// kafka producer 配置信息
    Properties properties = new Properties();
    // kafka提供服务的端口
    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.160.12:9092");
    // 配置ack应答机制
    properties.put(ProducerConfig.ACKS_CONFIG, "All");
    // 重试次数
    properties.put(ProducerConfig.RETRIES_CONFIG, 0);
    // 每一批的数据量大小
    properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
    // 延时发送数据（毫秒）
    properties.put(ProducerConfig.LINGER_MS_CONFIG, 1);
    // 总共可缓存的数据大小
    properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

	// 配置键值序列化方式
    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");

    // 配置数据按自定义方式放入分区
    properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, "com.capture.kafkapractice.MyPartitioner");
```

2. 生产者将数据按照自定义方式放入topic的partition中

```java
public class MyPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 将数据按照key的hash 返回分区
        return key.hashCode()%3;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> map) {
        //查看或修改配置
    }
}
```

3. 生产者发送数据

```java
    // 创建生产者
	KafkaProducer kafkaProducer = new KafkaProducer(properties);

    for (int i = 0; i < 10; i++) {
        // 普通方式发送数据
        kafkaProducer.send(new ProducerRecord("topic1",String.valueOf(i)));

        // 带回调函数的方式发送数据
        kafkaProducer.send(new ProducerRecord("topic1", String.valueOf(i)), (recordMetadata, e) -> {
            if (e != null) {
                System.out.println("发送失败！");
                e.printStackTrace();
            }
            System.out.println("分区："+recordMetadata.partition());
            System.out.println("offset："+recordMetadata.offset());
        });
    }
	// 关闭生产者
    kafkaProducer.close();
```

##### 7.2消费者高级API

1. 消费者配置

   ```java
       // kafka consumer 配置信息
       Properties properties = new Properties();
       // kafka集群
       properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.160.12:9092");
       // 配置消费者组
       properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
       // 配置是否自动提交
       properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
       // 配置自动提交延时时间   消费过程：获得数据-处理数据-提交offset
       properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
   
       // 键值反序列化方式
       properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
       properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
   
   ```

   不同组如何重复消费？

   ```java
   // 该组从最小的offset开始消费，默认latest
   properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "eariliest");
   ```

   

2. 消费者消费数据
   
   ```java
       KafkaConsumer<String,String> kafkaConsumer = new KafkaConsumer(properties);
   
       // 订阅一个topic
       //kafkaConsumer.subscribe(Collections.singletonList("first-topic"));
       // 订阅多个topic
       kafkaConsumer.subscribe(Arrays.asList("first-topic","second-topic"));
   
       while (true){
           // 每1000ms获取并消费订阅的topic中的数据
           ConsumerRecords<String,String> records = kafkaConsumer.poll(1000);
   
           for (ConsumerRecord record : records) {
               System.out.println("topic: "+record.topic()
                                  +" partition: "+record.partition()
                                  +" offset: "+record.offset());
           }
       }
   ```
	同一组如何重复消费？
   
   ```java
   // 指定某topic的某个分区
   kafkaConsumer.assign(Collections.singletonList(new TopicPartition("first-topic",0)));
   // 设置某topic的某个分区的指针指向哪个offset
   kafkaConsumer.seek(new TopicPartition("first-topic",0), 2);
   ```

##### 7.3消费者低级API：根据topic、partition和offset获取数据

```java
/**
     * 根据topic和partitionId查询partition在集群中的leader
     * @param host
     * @param port
     * @param topic
     * @param partition
     * @return
     */
    public static BrokerEndPoint getLeader(String host, int port, String topic, int partition){

        SimpleConsumer simpleConsumer = new SimpleConsumer(host, port, 100, 102400, "getLeader");

        TopicMetadataResponse metadataResponse = simpleConsumer.send(new TopicMetadataRequest(Collections.singletonList("topic")));

        List<TopicMetadata> topicMetadataList = metadataResponse.topicsMetadata();

        for (TopicMetadata topicMetadata : topicMetadataList) {
            List<PartitionMetadata> partitionMetadataList = topicMetadata.partitionsMetadata();
            for (PartitionMetadata partitionMetadata : partitionMetadataList) {
                if (partitionMetadata.partitionId() == partition) {
                    return partitionMetadata.leader();
                }
            }
        }
        return null;
    }

    /**
     * 从分区leader的offset开始消费（获取）数据
     * @param host
     * @param port
     * @param topic
     * @param partition
     * @param offset
     */
    public static void getData(String host, int port, String topic, int partition, int offset){

        BrokerEndPoint leader = getLeader(host, port, topic, partition);

        if (leader == null) {
            return;
        }

        SimpleConsumer simpleConsumer = new SimpleConsumer(leader.host(), leader.port(), 100, 102400, "getData");

        FetchRequest fetchRequest = new FetchRequestBuilder().addFetch(topic, partition, offset, 1000).build();

        FetchResponse fetchResponse = simpleConsumer.fetch(fetchRequest);

        ByteBufferMessageSet messageAndOffsets = fetchResponse.messageSet(topic, partition);

        for (MessageAndOffset messageAndOffset : messageAndOffsets) {
            long offset1 = messageAndOffset.offset();
            ByteBuffer payload = messageAndOffset.message().payload();
            byte[] bytes = new byte[payload.limit()];
            payload.get(bytes);
            System.out.println(offset1+"--->"+new String(bytes));
        }

    }
```

#### 8.kafka过滤器

1. 自定义过滤器类

    ```java
    public class MyKafkaInterceptor implements ProducerInterceptor<String,String> {

        private int successCount;
        private int failureCount;

        @Override
        public ProducerRecord<String, String> onSend(ProducerRecord<String, String> 			producerRecord) {
            return new ProducerRecord(producerRecord.topic(),
                    producerRecord.partition(),
                    producerRecord.key(),
                    LocalDateTime.now().toEpochSecond(ZoneOffset.of("+8"))+"-								"+producerRecord.value());
        }

        @Override
        public void onAcknowledgement(RecordMetadata recordMetadata, Exception e) {
            if (e == null){
                successCount++;
                return;
            }
            failureCount++;
        }

        @Override
        public void close() {
            System.out.println("生产成功"+successCount+"条");
            System.out.println("生产失败"+failureCount+"条");
        }

        @Override
        public void configure(Map<String, ?> map) {

        }
    }
    ```

2. 将自定义过滤器放入配置中

   ```java
   // 定义过滤器
   properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,new ArrayList<String>(){{
   	add("com.capture.kafkapractice.MyKafkaInterceptor");
   }});
   ```


#### 9.kafka stream

- 特点
  - 高扩展，弹性，容错
  - 轻量级（一个库）
  - 兼容kafka历史版本
  - 实时性 ：  毫秒级延迟，非微批处理，窗口允许乱序数据

- spark和storm都是流式处理框架，为什么要使用kafka stream？

  - kafka stream是流式处理类库，供开发者调用，整个运行方式由开发者控制。

    spark和storm要求开发者按照特定的方式去开发逻辑部分，供框架调用。

  - kafka stream方便嵌入应用程序中，而spark和storm需要部署

  - 大部分流式系统中都部署了kafka作为数据源

  - 使用storm和spark时，需要为框架本身的进程预留资源

  - kafka支持数据持久化，因此kafka提供滚动部署，滚动升级以及重新计算的能力

  - 由于kafka consumer rebalance机制，kafka stream可以动态调整并行

- 代码实践：将topic-first中的数据做流式处理（将字母改为大写），导入到topic-second中

  ```java
  public static void main(String[] args) {
  
          TopologyBuilder topologyBuilder = new TopologyBuilder();
          topologyBuilder.addSource("source","topic-first") //流的数据来源
              	//流的中间处理
                  .addProcessor("processor", () -> new MyStreamProcess(),"source") 
              	//流的数据去处
                  .addSink("sink","topic-second","processor"); 
  
      	// 配置信息
          Properties properties = new Properties();
          properties.put("application.id", "kafka-stream-test");
          properties.put("bootstrap.serves", "192.168.12.12:9092");
  
          KafkaStreams kafkaStreams = new KafkaStreams(topologyBuilder, properties);
          kafkaStreams.start();
      }
  ```

  自定义流式处理：

  ```java
  public class MyStreamProcess implements Processor<byte[],byte[]> {
  
      private ProcessorContext context;
      @Override
      public void init(ProcessorContext processorContext) {
          context = processorContext;
      }
  
      @Override
      public void process(byte[] bytes, byte[] bytes2) {
  
          String s = new String(bytes2);
          // 对value做uppercase操作
          bytes2 = s.toUpperCase().getBytes();
  
          context.forward(bytes, bytes2);
      }
  
      @Override
      public void punctuate(long l) {
  
      }
      @Override
      public void close() {
  
      }
  }
  ```



#### kafka 和 flume的比较

flume：

- 由cloudera公司研发；
- 适合多个生产者；
- 适合消费者不多的情况；每有一个消费，都需要在内存中新生成一份数据供消费者消费
- 适合数据安全性不高的操作；存在file channel和memory channel，一般用memory channel，没有持久化
- 适合于Hadoop生态圈对接

kafka：

- 由linkedin公司研发
- 适合消费者多的情况
- 数据安全性高；有备份机制

**因此我们常用的一种模型是： 线上数据 -> flume ->Kafka -> flume(根据情境增删) -> HDFS**



flume配置文件：配置与kafka关联

[kafka sink]:(http://flume.apache.org/releases/content/1.9.0/FlumeUserGuide.html#kafka-sink)

```properties
a1.sinks.k1.channel = c1
a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
# 数据存入kafka的哪个topic中
a1.sinks.k1.kafka.topic = topic-first
# 配置kafka集群
a1.sinks.k1.kafka.bootstrap.servers = localhost:9092
# How many messages to process in one batch. Larger batches improve throughput while adding latency.
a1.sinks.k1.kafka.flumeBatchSize = 20
# 配置ack机制
a1.sinks.k1.kafka.producer.acks = 1
# 配置延迟提交
a1.sinks.k1.kafka.producer.linger.ms = 1
a1.sinks.k1.kafka.producer.compression.type = snappy
```



