## ZooKeeper简介

<https://www.bilibili.com/video/av48456809>

### 1.Zookeeper概述

- Zookeeper 是一个开源的分布式协调服务框架 ，主要用来解决分布式集群中应用系统的一致性问题和数据管理问题
- Zookeeper 本质上是一个**分布式文件系统**, 适合存放**小文件**，也可以理解为一个数据库
- 使用一个和**文件树**结构相似的数据模型
- Naming Service、配置管理、Leader Election、服务发现、同步、Group Service、Barrier、分布式队列、两阶段提交
- 提供分布式锁服务，统一命名服务，状态同步服务，集群管理，分布式应用配置项的管理，分布式消息队列，分布式协调
- 特性
  - 全局数据一致性
  - 可靠性：如果消息被其中一台服务器接受，那么将被所有的服务器接受
  - 顺序性：消息的发布具有顺序性
  - 数据更新原子性：一次数据更新要么成功（半数以上节点成功），要么失败
  - 实时性：保证客户端在一定时间内获得服务端的响应

### 2.Zookeeper集群搭建

1. 集群规划

   | 服务器ip        | 主机名 | myid |
   | --------------- | ------ | ---- |
   | 192.168.200.100 | node01 | 1    |
   | 192.168.200.110 | node02 | 2    |
   | 192.168.200.120 | node03 | 3    |

2. 下载zk安装包并解压

   ```
   tar -zxvf apach-zookeeper-3.5.6.tar.gz
   ```

3. 修改配置文件

   ```reStructuredText
   cp zoo_sample.cfg zoo.cfg
   mkdir /home/liurunze/lrz_software/zookeeper/apach-zookeeper-3.5.6/zkdatas
   ```

   修改zoo.cfg

   ```reStructuredText
   dataDir=/home/liurunze/lrz_software/zookeeper/apach-zookeeper-3.5.6/zkdatas
   # 保留多少个快照
   autopurge.snapRetainCount=3
   # 日志多少小时清理一次
   autopurge.purgeInterval=1
   # 集群中服务器地址
   server.1=node01:2888:3888
   server.2=node02:2888:3888
   server.3=node03:2888:3888
   ```

4. 添加myid

   ```reStructuredText
   echo 1 > zkdatas/myid
   ```

5. 将zk文件发送至其他机器

   ```reStructuredText
   scp -r /home/liurunze/lrz_software/zookeeper/apach-zookeeper-3.5.6 node02:/home/liurunze/lrz_software/zookeeper/
   scp -r /home/liurunze/lrz_software/zookeeper/apach-zookeeper-3.5.6 node03:/home/liurunze/lrz_software/zookeeper/
   ```

6. 修改这些机器上的myid

   ```reStructuredText
   echo 2 > myid
   ```

7. 三台机器一起启动服务

   ```reStructuredText
   sh bin/zkServer.sh start
   ```

8. 启动后查看状态

   ```reStructuredText
   sh bin/zkServer.sh status
   ```

### 3. Zookeeper shell

1. 进入zk客户端

   进入192.168.200.100:2181机子上的zk客户端：

   ```shell
   sh zkCli.sh -s 192.168.200.100:2181
   ```

2. 创建节点

   create -s或-e path data acl

   -s表示序列化节点，-e表示临时节点，acl用来进行权限控制

   ```reStructuredText
   create -s /test hello  #表示在/test下创建一个名为hello的带有序列化的永久节点
   ```

3. 读取节点

   - ls path：读取path节点下有哪些节点
   - get path：读取该节点的值和属性
   - ls2 path：读取path节点下有哪些节点和该节点的属性

4. 更新节点

   set path data [version]：修改path节点的值为data

5. 删除节点

   delete path [version]：删除path节点

   rmr path：递归删除path节点以及其子节点
   
6. 给节点做限制
   
   setquota -n|-b val path：-n设置path节点的子节点的最大数量；-b设置path节点数据的最大长度
   
7. history

   列出历史记录

   


### 4. Zookeeper 数据模型

是一种类似文件树的结构模型，每一个节点都是一个Znode

Znode包括三个**属性**：

- status

  - dataVersion：数据版本号，从0开始，每次对节点进行set操作，dataversion += 1

  - cversion：子节点版本号
  - aclVersion
  - cZxid：Znode创建的事务id
  - mZxid：Znode被修改的事务id
  - ctime：节点创建时的时间戳
  - mtime：节点最后一次更新的时间戳
  - ephemeralOwner：如果该节点是临时节点，则值表示为sessionid

- data

- children

Znode包括两个**类型**：

节点类型在创建是规定，不能更改

- 临时节点：会话结束，临时节点删除，临时节点没有子节点
- 永久节点

### 5.Zookeeper集群通信

#### 5.1 Zab协议-广播

![](..\img\Zookeeper\Zab协议-广播.png)

#### 5.2 Zab协议-恢复模式

- 当leader宕机或丢失大多数Follower后，进入恢复模式

- 当新leader被选举出来，且大多数Follower完成与leader状态同步后，结束恢复模式

- 恢复模式的过程

  1. 发现集群中被commit的proposal的最大zxid，找到新leader

  2. 建立新的epoch（zxid的高32位），保证之前的leader无法commit新的Proposal
  3. 新leader拥有版本最新的数据，并将这些数据与其他follower同步

#### 5.3 一致性保证

- 顺序一致性

  客户端发送的更新操作会被顺序执行

- 原子性

  更新要么成功要么失败

- 单一系统镜像

  一个客户端只会看到一个view，无论连接到zk集群的哪个机器上

- 可靠性

  一旦更新被应用，该更新会被持久化

  如果客户端得到更新成功的状态码，则该更新一定生效

  被客户端更新成功的结果，不会回滚

- 实时性

  保证客户端在几十秒内看到的是最新的视图（最终一致性）

#### 5.4 notice

- 只保证同一客户端看到的是单一镜像，由于leader只确保一半以上的ack收到就commit，所以其他未接收Proposal的版本会低。如果保证不同客户端看到同一镜像，可调用sync，保证客户端连接的follower与leader完全同步后，再读数据。
- zookeeper读性能比写性能好
- zookeeper集群最好是奇数
- 如果有n个机器宕机，该集群必须还有n+1个活着的机器，不然没法选leader

### 6. Zookeeper Watch

是一种服务端-服务端和服务端-客户端的通讯机制，类似于发布订阅模式。

watch的**特点**：

- 一次性触发：监听只触发一次
- 事件封装：watchedEvent中包含三个属性：通知状态，事件类型和节点路径
- 异步发送
- 先注册再触发

**通知状态和事件类型：**

| KeeperState        | EventType        | 触发条件                      | 说明                                    |
| ------------------ | ---------------- | ----------------------------- | --------------------------------------- |
|                    | None             | 连接成功                      |                                         |
| SyncConnected（0） | NodeCreated      | Znode被创建                   | 此时处于连接状态                        |
| SyncConnected（0） | NodeDeleted      | Znode被删除                   | 此时处于连接状态                        |
| SyncConnected（0） | NodeDataChanged  | Znode数据被改变               | 此时处于连接状态                        |
| SyncConnected（0） | NodeChildChanged | Znode的子Znode<br/>数据被改变 | 此时处于连接状态                        |
| Disconnected（0）  | None             | 客户端和服务端<br/>断开连接   | 此时客户端和服务器处于<br/>断开连接状态 |
| Expired（-112）    | None             | 会话超时                      | 会收到一个<br/>SessionExpiredException  |
| AuthFailed（4）    | None             | 权限验证失败                  | 会收到一个<br/>AuthFailedException      |



### 7. Zookeeper 客户端API

- 导包

  ```xml
  <dependency>
      <groupId>org.apache.zookeeper</groupId>
      <artifactId>zookeeper</artifactId>
      <!-- 和zk版本保持一致 -->
      <version>3.4.9</version>
  </dependency>
  ```

- 实践

  ```java
  public static void main(String[] args) {
  
      ZooKeeper zkClient = null;
      try {
  
          // 创建zk客户端对象，并设置监听
          zkClient = new ZooKeeper("192.168.200.100:2181,192.168.200.110:2181",
                                   30000, x -> {
                                       System.out.println("事件状态："+x.getState());
                                       System.out.println("事件类型："+x.getType());
                                       System.out.println("事件路径："+x.getPath());
                                   });
  
          // 创建节点
          zkClient.create("/helloZK", "创建节点".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
  
          // 查看节点值，并设置监听
          byte[] data = zkClient.getData("/hello", true, null);
          System.out.println(new String(data));
  
          // 改变节点值
          zkClient.setData("/helloZK", "hello".getBytes(), -1);
  
      } catch (Exception e){
          e.printStackTrace();
      } finally {
          try {
              zkClient.close();
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  
  }
  ```

  

### 8. Zookeeper 选举机制

> zk选举机制默认算法是 FastLeaderElection，简单的说是，**通票数大于半数**

#### 8.1 概念

- 服务器id（myid）

  myid值越大，权重越大

- 选举状态

  - LOOKING，观望中，参与选举
  - FOLLOWING，跟随者状态
  - LEADING，领导者状态
  - OBSERVING，观察者状态，不能参与投票

- 数据id（ZXID）

  数据版本越新，权重越大

- 逻辑时钟

  参与选举的次数越多，逻辑时钟越大，权重越大

#### 8.2 全新集群选举

- 都先给自己一票
- myid大的获得所有票数
- 一旦获取票数超过集群的半数，选举结束
- 其他之后进来的直接为follower

#### 8.3 非全新集群选举

1. 逻辑时钟小的选举结果被忽略，重新投票
2. 统一逻辑时钟后，zxid大的胜出
3. zxid相同的情况下，服务器id（myid）大的胜出

### 9. Zookeeper 应用

#### 9.1 数据发布与订阅（配置中心）

- 统一管理的数据不能过大
- 统一管理的数据一般是配置信息（如地址、账号密码）等数据
- 订阅者首次去获取这些数据并设置监听
- 监听到数据变化后，做出相应的处理，并再次设置监听

#### 9.2 命名服务

> 在分布式系统中，通过使用命名服务，客户端应用能够根据名字来获取资源或服务的地址，提供者等信息。
>
> 通过zk创建节点的API，可以创建一个*全局唯一的path*，这个path就可以作为一个名称
>
> 分布式服务框架Dubbo，使用zk的命名服务，来维护全局的服务地址列表

#### 9.3 分布式锁

独占锁原理：

1. 如果想操作一个文件，需要先在指定目录创建节点（该节点是临时的，非序列化的）
2. 谁创建节点，谁就可以操作文件
3. 操作完成，断开连接，临时节点消失
4. 其他应用监听这个节点是否存在，如果不存在，则可以通过在指定目录下创建节点，来获得对文件操作的锁

时序性原理：

1. 在指定目录下创建临时的，序列化的节点
2. 可根据节点的序列化大小来判断时序