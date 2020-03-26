### Flume

#### 1.定义

 Flume是一种分布式的、可靠的、可用的服务，用于有效地收集、聚合和移动大量的日志数据。它具有简单灵活的基于流数据流的体系结构。它具有鲁棒性和容错性，具有可调的可靠性机制和许多故障转移和恢复机制。它使用一个简单的可扩展数据模型，允许在线分析应用程序。  

#### 2.基础架构

![](img\Flume\Flume组成架构图.jpg)

##### 2.1 Agent

Agent是一个JVM进程，它以事件的形式将数据由源头传至目的

Agent主要有三个组件组成：source、channel、sink

##### 2.2 Source

Source是负责接受数据到Agent的组件。Source可以接受各种类型、源头的数据，包括**avro**、thrift、**exec**、jms、**spolling dir**、**tail dir**、**netcat**、sequence generator、syslog、http、legacy等。

##### 2.3 Sink

Sink不断的轮询Channel中的事件且批量的移除他们，并将这些事件批量写入到存储或索引系统、或者被发送到另一个Flume Agent。Sink组件的目的地包括**HDFS**、**Logger**、**avro**、thrift、ipc、**file**、**HBase**、solr等。

##### 2.4 Channel

Channel是位于Source和Sink之间的缓冲区。可以使Source和Sink速率不同。Channel是线程安全的，可以同时处理几个Source的写入操作和几个Sink的读取操作。

- Memory Channel是内存中的队列
- File Channel将所有事件写入磁盘
- Kafka Channel

##### 2.5 Event

传输单元，数据以Event的形式在Flume中传输。Event由Header和Body组成，Header用来存放event的一些属性，以key-value形式存储；Body用来存放数据，以字节数组存储。

#### 3.安装部署

1. 下载

   archive.apache.org/dist/flume

2. 解压

3. 修改配置文件

   flume-env.sh

   ```powershell
   # 配置JAVA_HOME
   export JAVA_HOME=/opt/module/jdk1.8
   ```

#### 4.简单案例

##### 4.1 官方案例

使用netcat工具向本机某个端口发送数据，flume使用netcat source接收，memory channel缓冲，Logger sink将数据日志打印到控制台查看。

1. 安装netcat

   ```powershell
   yum install -y nc
   # 使用netcat开启服务端
   nc -lk 44444
   # 使用netcat开启客户端
   nc hostname 44444
   ```

2. 写一个flume的配置文件 netcat-flume-logger.conf

   ```properties
   # example.conf: A single-node Flume configuration
   
   # Name the components on this agent
   a1.sources = r1
   a1.sinks = k1
   a1.channels = c1
   
   # Describe/configure the source
   a1.sources.r1.type = netcat
   a1.sources.r1.bind = localhost
   a1.sources.r1.port = 44444
   
   # Describe the sink
   a1.sinks.k1.type = logger
   
   # Use a channel which buffers events in memory
   a1.channels.c1.type = memory
   a1.channels.c1.capacity = 1000
   a1.channels.c1.transactionCapacity = 100
   
   # Bind the source and sink to the channel
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel = c1
   ```

3. 启动一个agent

   ```powershell
   $ bin/flume-ng agent \
   --conf conf/ \
   --conf-file netcat-flume-logger.conf \
   --name a1 \
   -Dflume.root.logger=INFO,console
   ```

4. 启动netcat客户端，连接agent的netcat服务端，并向agent发送数据

   ```shell
   netcat localhost 44444
   ```

##### 4.2 实时监控单个追加文件

监控hive的日志（/opt/module/hive/logs/hive.log），并上传至HDFS中

1. 写agent的配置文件 exec-flume-hdfs.conf

   ```properties
   # Name the components on this agent
   a1.sources = r1
   a1.sinks = k1
   a1.channels = c1
   
   # Describe/configure the source
   a1.sources.r1.type = exec
   a1.sources.r1.command = tail -F /opt/module/hive/logs/hive.log
   
   # Describe the sink
   a1.sinks.k1.type = hdfs
   a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H
   a1.sinks.k1.hdfs.filePrefix = logs-
   a1.sinks.k1.hdfs.round = true
   a1.sinks.k1.hdfs.roundValue = 1
   a1.sinks.k1.hdfs.roundUnit = hour
   
   # Use a channel which buffers events in memory
   a1.channels.c1.type = memory
   a1.channels.c1.capacity = 1000
   a1.channels.c1.transactionCapacity = 100
   
   # Bind the source and sink to the channel
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel = c1
   ```

2. 添加flume依赖HDFS的jar

   ```reStructuredText
   commons-configuration-1.6.jar
   commons-io-2.4.jar
   hadoop-auth-2.7.2.jar
   hadoop-common-2.7.2.jar
   hadoop-hdfs-2.7.2.jar
   htrace-core-3.1.0-incubating.jar
   ```


##### 4.3 实时监控目录下多个新文件

写agent的配置文件 spolling-flume-hdfs.conf

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /opt/module/flume/upload_dir

# Describe the sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H
a1.sinks.k1.hdfs.filePrefix = logs-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
a1.sinks.k1.hdfs.roundUnit = hour

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

##### 4.4 实时监控目录下多个追加文件

写agent的配置文件 tailDir-flume-hdfs.conf

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = TAILDIR
a1.sources.r1.positionFile = /var/log/flume/taildir_position.json
a1.sources.r1.filegroups = f1 f2
a1.sources.r1.filegroups.f1 = /var/log/test1/example.log
a1.sources.r1.filegroups.f2 = /var/log/test2/.*log.*

# Describe the sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H
a1.sinks.k1.hdfs.filePrefix = logs-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 1
a1.sinks.k1.hdfs.roundUnit = hour

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

#### 5.事务

![](img\Flume\Flume事务.jpg)

#### 6.Flume Agent内部原理

![](img\Flume\FlumeAgent内部原理.jpg)

- FailoverSinkProcessor 故障转移，即设置一个active和多个standby，如果active出了问题，则走standby

#### 7.拓扑结构

##### 7.1 简单串联

![](img\Flume\FlumeAgent串联拓扑图.jpg)

##### 7.2 复制和多路复用

![](img\Flume\FlumeAgent多路复用拓扑图.jpg)

##### 7.3 负载均衡和故障转移

![](img\Flume\FlumeAgent负载均衡拓扑图.jpg)

##### 7.4 聚合

![](img\Flume\FlumeAgent聚合拓扑图.jpg)

#### 8.自定义Interceptor

- 引入依赖

```xml
<dependency>
    <groupId>org.apache.flume</groupId>
    <artifactId>flume-ng-core</artifactId>
    <version>1.7.0</version>
</dependency>
```

- 自定义拦截器

  ```java
  public class MyFlumeInterceptor import Interceptor{
      public void initialize(){}
      public Event intercept(Event event){}
      public List<Event> intercept(List<Event> events){}
      public void close(){}
  }
  ```

- 打包放入Flume中

- 编写配置文件

  ```properties
  # channel interceptor
  a1.sources.r1.interceptors = i1 i2
  a1.sources.r1.interceptors.i1.type = org.apache.flume.interceptor.HostInterceptor$Builder
  a1.sources.r1.interceptors.i1.preserveExisting = false
  a1.sources.r1.interceptors.i1.hostHeader = hostname
  a1.sources.r1.interceptors.i2.type = org.apache.flume.interceptor.TimestampInterceptor$Builder
  # channel seleter
  a1.sources.r1.selector.type = multiplexing
  a1.sources.r1.selector.header = state
  a1.sources.r1.selector.mapping.CZ = c1
  a1.sources.r1.selector.mapping.US = c2 c3
  a1.sources.r1.selector.default = c4
  ```

#### 9.自定义Source

```java
public class MySource extends AbstractSource implements Configurable, PollableSource {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation, convert to another type, ...)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external client
  }

  @Override
  public void stop () {
    // Disconnect from external client and do any additional cleanup
    // (e.g. releasing resources or nulling-out field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    try {
      // This try clause includes whatever Channel/Event operations you want to do

      // Receive new data
      Event e = getSomeData();

      // Store the Event into this Source's associated Channel(s)
      getChannelProcessor().processEvent(e);

      status = Status.READY;
    } catch (Throwable t) {
      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    } finally {
      txn.close();
    }
    return status;
  }
}
```

#### 10.自定义Sink

```java
public class MySink extends AbstractSink implements Configurable {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external repository (e.g. HDFS) that
    // this Sink will forward Events to ..
  }

  @Override
  public void stop () {
    // Disconnect from the external respository and do any
    // additional cleanup (e.g. releasing resources or nulling-out
    // field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    // Start transaction
    Channel ch = getChannel();
    Transaction txn = ch.getTransaction();
    txn.begin();
    try {
      // This try clause includes whatever Channel operations you want to do

      Event event = ch.take();

      // Send the Event to the external repository.
      // storeSomeData(e);

      txn.commit();
      status = Status.READY;
    } catch (Throwable t) {
      txn.rollback();

      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    }
    return status;
  }
}
```

#### 11.数据流监控

##### 11.1 Ganglia的安装与部署

##### 11.2 操作Flume测试监控

#### 12.面试题

- Flume数据传输监控
- Source、Sink、Channel用的哪些
- Channel Selector
- 参数调优
- 事务机制
- 采集数据会丢失吗

