### Oozie

#### 1.简介

Oozie是一个管理Apache Hadoop作业的工作流调度系统.

Oozie工作流程是行动的定向周期图（DAG）。

Oozie协调员作业是由时间（频率）和数据可用性触发的经常性Oozie工作流作业。

Oozie集成了Hadoop堆栈的其余部分，支持几种类型的Hadoop作业（如Java map-reduce、Streamingmap-reduce、Pig、Hive、Sqoop和Distcp）以及系统特定的作业（如Java程序和shell脚本）。

Oozie是一个可扩展、可靠和可扩展的系统。

#### 2.模块功能

##### 2.1 模块

- Workflow

  顺序执行流程节点，支持fork（分支多个节点），join（合并节点）

- Coordinator

  定时触发workflow

- Bundle Job

  绑定多个任务

##### 2.2 常用节点

- 控制流节点（control flow node）

  一般定义在工作流开始或结束的位置，如start、end、kill等。以及提供工作流的执行路径机制，如decision、fork、join等。

- 动作节点（action node）

  负责具体的动作，如拷贝文件，执行脚本等

#### 3.Oozie的部署

##### 3.1 部署Hadoop（cdh版）

https://www.bilibili.com/video/av35357845?p=2

- 修改Hadoop配置
- 启动Hadoop集群

#### 3.2 部署Oozie

https://www.bilibili.com/video/av35357845?p=3

- 解压Oozie

- 在Oozie根目录下解压oozie-hadooplibs-4.0.0-cdh5.3.6.tar.gz

  解压完成后，在根目录下会出现hadooplib/

- 在Oozie根目录下创建libext/

- 拷贝依赖的jar

- 将ext-2.2.zip拷贝到libext/下

- 修改Oozie的配置文件

- 在Mysql中创建Oozie的数据库

- 初始化Oozie

- Oozie的启动与关闭

  bin/oozied.sh start

  bin/oozied.sh stop

- 访问Oozie的web页面

  http://hadoop102:11000/oozie