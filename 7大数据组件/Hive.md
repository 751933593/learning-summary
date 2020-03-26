### Hive

#### 1.Hive的基本概念

##### 1.1 什么是Hive

- 由Facebook开源用于解决海量结构化日志的数据统计
- 是基于Hadoop的一个数据仓库工具，可以将结构化数据转化成一张表，并提供类SQL的查询功能
- 本质是将HQL转化成MapReduce程序

##### 1.2 优缺点

- 优点
  - 操作接口采用类SQL的语法，避免写MR程序，提高开发效率
  - 支持用户自定义函数
  - 计算框架默认是MR，可以换成Spark
- 缺点
  - 迭代算法无法表达（Spark Sream解决）
  - 数据挖掘方面不擅长（Spark ML解决）
  - 自动生成MR作业，不够智能化
  - 调优比较困难，粒度较粗

##### 1.3 架构原理

![](img\Hive\Hive架构原理.jpg)

##### 1.4 与数据库对比

- 查询语言

  类似

- 存储位置

  Hive：数据在HDFS中

  数据库：数据在块设备或本地文件系统

- 数据更新

  Hive：不建议对数据改写

- 索引

  Hive：没有索引，并行访问数据，对于大量数据的访问仍占有优势

  数据库：有索引，对于少量特定条件的数据访问效率高

- 执行

  Hive：通过MR（或其他）来实现

  数据库：通常有自己的执行引擎

- 可扩展性

#### 2.Hive的安装

1. 解压hive安装包

2. 修改配置文件hive-env.sh

   ```powershell
   # 配置HADOOP_HOME的路径
   export HADOOP_HOME=/opt/module/hadoop-2.7.2
   # 配置HIVE_CONF_DIR路径
   export HIVE_CONF_DIR=/opt/module/hive/conf
   ```

3. 进入hive操作

   ```powershell
   # 进入hive
   bin/hive
   # 创建表，以/t作为分隔符
   hive> create table student(id int,name string) row fromat delimited fields terminated by '\t';
   # 插入信息
   hive> insert into student values(1,"zhangsan");
   # 查询数据
   hive> select count(*) from student;
   # 将本地数据加载到hadoop的hive中
   hive> load data local inpath '/opt/module/data/stu.txt' into table sutdent;
   ```

4. 使用mysql存储hive元数据信息

   - hive下的lib包必须要mysql的驱动包

   - 修改配置文件hive-site.xml

     ```xml
     <configuration>
         <property>
             <name>javax.jdo.option.ConnectionURL</name>
             <value>jdbc:mysql://hadoop102:3306/metastore?			createDatabaseIfNotExist=true</value>
         </property>
         <property>
             <name>java.jdo.option.ConnectionDriverName</name>
             <value>com.mysql.jdbc.Driver</value>
         </property>
         <property>
             <name>javax.jdo.option.ConnectionUserName</name>
             <value>root</value>
         </property>
         <property>
             <name>javax.jdo.option.ConnectionPassword</name>
             <value>123456</value>
         </property>
     </configuration>  
     ```

   5. 使用命令执行hive

      ```powershell
      # 参数-e：执行后面的hql
      bin/hive -e "select * from student"
      # 参数-f：执行文件中的hql
      bin/bive -f ./student.hql
      ```

   6. 其他常见配置

      ```xml
      <property>
          <name>hive.cli.print.header</name>
          <value>true</value>
      </property>
      <property>
          <name>hive.cli.print.current.db</name>
          <value>true</value>
      </property>
      ```

      

#### 3.Hive的数据类型

- 基本数据类型

    | Hive数据类型 | Java数据类型 |
    | :----------: | :----------: |
    |   tinyint    |     byte     |
    |   smalint    |    short     |
    |     int      |     int      |
    |    bigint    |     long     |
    |   boolean    |   boolean    |
    |    float     |    float     |
    |    double    |    double    |
    |    string    |    string    |
    |  timestamp   |   时间类型   |
    |    binary    |   字节数组   |

- 集合数据类型

  - struct
  - map
  - array

  **举例：**

  ```reStructuredText
  zhangsan, xiaoming_xiaohong, xiaozhang1:12_xiaozhang2:15, jingan_shanghai
  lisi, xiaohei_xiaoming, xiaoli1:10_xiaoli2:18, xuhui_shanghai
  ```

  ```sql
  create table test(
  	name string,
  	friends array<string>,
  	children map<string,int>,
  	address struct<area:string, city:string>
  )
  row format delimited fields terminated by ','
  collection items terminated by '_'
  map keys terminated by ':'
  lines terminated by '\n';
  ```

- 类型转化

  强转： select cast('1' as int)

#### 4.DDL数据定义

- 创建数据库

  create database 数据库的名称 (loaction 数据库的路径)

- 查询数据库

  show databases (like 'db_*')

- 查询数据库详情

  desc database 数据库的名称

- 创建表

  ```sql
  create [external] table [if not exists] table_name
  [(col_name data_type [comment col_comment], ...)]
  [comment table_comment]
  [partitioned by (col_name data_type [comment col_comment], ...)]
  [clustered by (col_name, ...)
   [sorted by (col_name [asc|desc], ...)]] into num_buckets buckets]
  [row format row_format]
  [stored as file_format]
  [location hdfs_path]
  ```

- 查询表（详情）

  show tables (like *)

  desc formatted 表名

- 管理表（内部表）与外部表

  创建表时加external即为创建外部表

  内部表删除时数据也删除，一般临时表常用

  外部表删除时，只删除hive存在mysql中的元数据信息，HDFS中的数据未删除

- 分区表

  数据存储路径：/usr/hive/warehourse/数据库名/表名/分区名/数据

  根据某列的内容进行分区（例如，一天一个分区），当使用where按天查询时，会做**谓词下推**优化，可以只查询对应分区的内容，提高效率。
  
  ```sql
  -- 可以在创建表时指定分区
  create table table_name (id int,name string) partitioned by (mouth string)
  -- 将数据加载到分区表中
  load data local inpath '/opt/module/datas/dept.txt' into table default.table_name partition(mouth='2019-12')
  -- 查询分区表中的数据
  select * from table_name where mouth='2019-12'
  -- 新增分区
  alter table table_name add partition(mouth='2019-11') partition(mouth='2019-12');
  -- 删除分区
  alter table table_name drop partition(mouth='2019-11'),partition(mouth='2019-12');
  -- 查看分区表有多少个分区
  show partition table_name;
  -- 查看分区表结构
  desc formatted table_name;
  ```
  
  - 可以有多级分区，一般到二级分区为止
  - 数据上传到HDFS后，由于hive表没有映射分区，读取不到数据，可以由以下方式解决：
    1. msck repair table table_name
    2. alter table table_name add partittion(mouth='2019-12',day='05')
  
- 修改表

  - 重命名表

    ```sql
    alter table table_name1 rename to table_name2
    ```

  - 更新单个列

    ```sql
    alter table table_name change [column] col_oldname col_newname col_type [comment col_comment] [first|after col_name]
    ```

  - 增加多个列或替换所有列

    ```sql
    alter table table_name add|replace columns (col_name col_type [comment col_comment], ...)
    ```

#### 5.DML数据操作

##### 5.1 数据导入

- 使用Load向表中加载数据

  ```sql
  load data [local] inpath '/opt/module/datas/aa.txt' [overwirte] into table table_name [partition(partiton_col1=val1, ...)];
  ```

- 通过查询语句向表中导入数据

  ```sql
  insert overwrite|into table table_name partition(mounth='2019-12') select id,name from table_name
  -- 根据多个表查询结果插入数据
  from table_name
  insert overwrite table table_name partition(mouth='2019-12')
  select id,name where mouth='2019-10'
  insert overwrite table table_name partition(mouth='2019-11')
  select id,name where mouth='2019-10';
  ```

- 创建表时插入数据

  ```sql
  create table table_name1 as select * from table_name2;
  ```

- 创建表时通过location指定数据加载路径

  ```sql
  create table if not exist table_name(id int,name string) r
  row format delimited fields terminated by '\t'
  location '/user/hive/warehouse/XXX';
  ```

- import数据到指定的hive表中

  *注意：先用export导出数据后，再用import导入数据*，必须是空表，且该表格式和数据对应

  improt table student partition(month='2019-12') from '/user/hive/warehourse/export/student'

##### 5.2 数据导出

- 使用insert导出

  ```sql
  insert overwrite|into [local] directory '本地路径或者hdfs路径' 
  row format fields terminated by '\t'
  select * from table_name;
  ```

- Hadoop命令

  ```powershell
  dfs -get hdfs路径 本地路径
  ```

- hive shell 命令

  ```powershell
  bin/hive -e 'select * from default.table_name' > '本地路径'
  ```

- export 导出到HDFS

  ```sql
  export table table_name to 'hdfs路径'
  ```

- 使用sqoop导出到关系型数据库

##### 5.3 清空表数据

#### 6.查询

 https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select 

- 排序

  - 全局排序 order by，只使用一个reducer

  - 分区区内排序 sort by 一般结合distribute by使用，且设置reducer的个数为多个

    ```sql
    -- 按照班级分区，并每个班级按照年龄排序
    select * from student distribute by class_no sort by age;
    ```

  - cluster by ：当分区字段和排序字段相同时，使用cluster by

- 分桶表的查询

  - 创建分桶表

    ```sql
    create table stu_bucket(id int, name string)
    clustered by(id) into 4 buckets
    row format delimited fields terminated by '\t';
    ```

  - 设置属性

    set hive.enforce.bucketing=true;

    set mapreduce.job.reduces=-1;

  - 向表中插入数据

    insert into table  stu_bucket select * from stu

  - 分桶抽样查询

    ```sql
    -- 分桶抽样tablesample(bucket x out of y)，x表示从第几个桶开始抽，y表示每次间隔y，共抽(桶的总个数/y)次
    select * from stu_bucket tablesample(bucket 1 out of 4 on id)
    ```

#### 7.函数

##### 7.1 常见函数

- 空值赋值：nvl
- 时间函数：data_format 、data_add、datadiff
- 字符串替换：regep_replace
- case when XXX then XXX else XXX end、if(XXX,XXX,XXX)

##### 7.2 行列转换

- 行转列 ：concat('hello','-','word')、concat_ws('-','hello','world')、collect_set(col)

- 列转行：explode(col)，将hive中的array或map拆分成多行

  ```sql
  -- name	 category
  -- 《国产凌凌漆》  ["动作","喜剧","剧情"]
  -- 《三傻大闹宝莱坞》  ["喜剧","爱情","歌舞","剧情"]
  select movie, category_name from movie_info 
  lateral view -- 侧写
  explode(category) table_tmp as category_name
  -- name	 category_name
  -- 《国产凌凌漆》  动作
  -- 《国产凌凌漆》  喜剧
  -- 《国产凌凌漆》  剧情
  -- 《三傻大闹宝莱坞》  喜剧
  -- 《三傻大闹宝莱坞》  爱情
  -- 《三傻大闹宝莱坞》  歌舞
  -- 《三傻大闹宝莱坞》  剧情
  ```

##### 7.3 窗口函数

窗口函数 over

指定函数工作的数据窗口大小，这个窗口大小可能会随着行的变化而变化

current row：当前行

n preceding ：往前n行

n following：往后n行

unbounded：无边界；unbounded preceding 往前无数行

```sql
-- 其中，distribute by... sort by...和partition by... order by...等价
select name,orderdate,cost, sum(cost) over(distribute by name sort by orderdata rows between unbounded preceding and current row) from order_info
```

lag(col,n)：往前第n行的数据

```sql
select name,orderdate,cost, lag(orderdata,1,'1970-01-01') over(distribute by name sort by orderdata) from order_info
```

lead(col,n)：往后第n行的数据

ntile(n)：把有序分区中的行，分发到指定数据的组中，各个组有编号。对于每一行，ntile返回此行所属组的编号

```sql
-- 查询按时间排序后的前20%的信息
select name,orderdate,cost from (
    select name,orderdate,cost, 
    ntile(5) over(order by orderdata) as groupnum from order_info
)where groupnum=1;
```

排名函数：Rank

- rank()

  例：1、1、3

- dense_rank()

  例：1、1、2

- row_number()

  例：1、2、3

```sql
select name,orderdate,cost, rank() over(partition by name order by cost desc) from order_info
```

##### 7.4 查看系统函数

查看系统的函数

```sql
-- 查看所有函数
show functions;
-- 查看某个函数
desc function 函数名
-- 查看某个函数的详情
desc fuction extended 函数名
```

##### 7.5 自定义函数

用户自定义函数分为三种：

- UDF（user-defined-function）
- UDAF（user-defined aggregation function）
- UDTF（user-defined table-generating function）

**UDF实现步骤：**

0. maven依赖

   ```xml
   <dependency>
   	<groupId>org.apache.hive</groupId>
       <artifactId>hive-exec</artifactId>
       <version>1.2.1</version>
   </dependency>
   ```

1. 继承org.apache.hadoop.hive.ql.exec.UDF 并实现 evaluate函数（支持重载）

   ```java
   import org.apache.hadoop.hive.ql.exec.UDF
       
   public class MyUDF extends UDF{
       public int evaluate(int data, int num){
           return data+num;
       }
   }
   ```

2. 添加jar，并指定jar路径

   ```sql
   hive> add jar linux_jar_path
   ```

3. 创建function

   ```sql
   -- class_name（类的全路径） 必须是字符串
   hive> create [temporary] function [dbname.]function_name as class_name;
   ```

4. 删除函数

   ```sql
   hive> drop [temporary] function [if exists] [dbname.]function_name;
   ```

**notice：**UDF必须要有返回类型，可以返回null，但返回类型不能是void

**UDTF实现步骤：**

0. 将字符串以分隔符分开，并炸裂显示。

1. 继承org.apache.hadoop.hive.ql.udf.generic.GenericUDTF 并实现抽象方法

   ```java
   public class MyUDTF extends GenericUDTF{
       public StructObjectInspector initialize(StructObjectInspector argOIs) 
           throws 	UDFArgumentException {
           // 定义输出数据的列名集合，这个列名可以被我们的HQL中定义列的别名所覆盖
           List<String> fieldNames = new ArrayList<>();
           fieldNames.add("word");
           // 定义输出数据的类型
           List<ObjectInspectorFactory> fieldOIs = new ArrayList()<>;
           fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
           
           return ObjectInspectorFactory
               .getStandardStructObjectInspector(fieldNames,fieldOIs);
       }
       
       public void process(Object[] args){
           String result = args[0].toString().split(args[1].toString());
           for(String word : result){
               forward(new ArrayList(){{
                   add(word);
               }})
           }
       }
   }
   ```

2. 打包，添加jar，定义函数，执行

#### 8.压缩和存储

Hadoop是不支持Snappy压缩的，所以我们需要将Snappy压缩功能写进去后，重新编译Hadoop源码，使其支持Snappy压缩。

##### 8.1 Hadoop源码支持Snappy压缩

- 资源准备
- jar包安装
- 编译源码

##### 8.2 Hadoop压缩配置（可不开启）

##### 8.3 开启Map输出阶段压缩

##### 8.4 开启Reduce输出阶段压缩

##### 8.5 文件存储格式

- 列式存储和行式存储

  textFile和sequenceFile是行式存储，orc和Parquet是列式存储

- TextFile格式

- Orc格式

  分段列存储，默认1w行分一段，每段按照列存储

- Parquet格式

- 主流文件存储格式对比

##### 8.6 存储和压缩结合

#### 9.优化

##### 9.1 Fetch抓取

有些hql不需要MR计算，在hive-default.xml中配置：

hive.fetch.task.conversion = more

##### 9.2 本地模式

在一定条件下，hive查询只用本地模式，不需要集群计算

hive.exec.mode.loacl.auto = true

hive.exec.mode.local.auto.inputbytes.max=134217728

hive.exec.mode.local.auto.input.files.max=4

##### 9.3 表的优化

- 小表join大表

  hive.auto.convert.join = true

- 大表join大表

  过滤掉空id

  如果不能过滤掉空id，给空id一个随机值，防止空id过多造成数据倾斜

- Mapjoin

  缓存小表

  hive.auto.convert.join = true

  hive.mapjoin.smalltable.filesize=25000000   # 大小表的分界值

- group by

  在map阶段执行完成后，同一个key发给一个reducer，当某个key过多时，会产生数据倾斜
  
  map端预聚合：hive.map.aggr=true
  
  在map端进行聚合操作的条目数目：hive.groupby.mapaggr.checkinterval=100000
  
  有数据倾斜时自动负载均衡：skewindata=true
  
- count(distinct)

  使用group by进行去重，将数据放入不同的reducer中执行，虽然变慢，但是数据量大时不会造成内存溢出

- 笛卡尔积

  避免使用笛卡尔积，join是不加on条件或是无效的on条件

- 动态分区

  根据数据中的某个字段进行分区

  开启动态分区功能：hive.exec.dynamic.partition=true

  设置为非严格模式（动态分区的模式默认为strict，表示必须指定一个分区字段为静态分区，nonstatic表示允许所有的分区字段都可以使用动态分区）：hive.exec.dynamic.partition.mode=nonstrict

  在所有的MR节点上，最多可以创建多个分区：hive.exec.max.dynamic.partitions=1000

  在每个MR节点上，最大分区数：hive.exec.max.dynamic.partitions.pernode=100

  整个MRjob中，最大可以创建多少个文件：hive.exec.max.created.files=1000000

  当有空分区生成时，是否抛异常：hive.error.on.empty.partition=false

- 分区分桶


##### 9.4 MR优化

- 合理设置map数

  - 通常情况下，作业会通过input的目录产生一个或多个map任务

    主要决定因素有：input文件的总个数、input文件大小、集群设置的块大小

  - 是不是map越多越好？

    不是。如果一个任务中有很多小文件，每个文件会被当做一个块，用一个map任务来完成，一个map任务启动和初始化的时间，远远大于处理逻辑业务的时间。造成资源浪费。

  - 是不是保证每个map处理接近128M的文件块最好？

    不一定。当一个128M的文件字段少，行数多，处理逻辑业务复杂，还是使用多个map任务分别处理好。

- 小文件合并

- 复杂文件增加map数

- 设置reduce数

  - 设置reduce个数

    mapreduce.job.reduces = 10

  - 动态设置reduce个数

    mapreduce.job.reduces = -1

    每个reduce处理的数据量：hive.exec.reducers.bytes.per.reducer=256000000

    每个任务最大的reduce数：hive.exec.reducers.max=1009

    计算reduce数的公式：reduce数=min(每个任务最大的reduce数, 总数据量/每个reduce处理的数据量)

  - reduce数是不是越多越好？

    不是。1. reduce过多，开启和初始化浪费时间与资源 2. reduce后产生的文件数又多又小

##### 9.5 并行执行

map任务和reduce任务并行执行，节省时间，但占用资源（内存）大

开启并行执行：hive.exec.parallel=true

一个任务最大并行数：hive.exec.parallel.thread.number=16

##### 9.6 严格模式

设置严格模式：hive.mapred.mode=strict

严格模式下以下操作不被允许：

- Cartesian Product 笛卡尔积
- No partition being picked up for a query 查询分区表时没有指定分区
- Comparing bigints and strings
- Comparing bigints and doubles
- Orderby without limit

##### 9.7 JVM重用

场景：小文件或task数多的情况，执行时间短时使用jvm重用

Hadoop的默认配置通常是使用派生jvm来执行map和reduce任务的。这是jvm启动过程可能会造成很大的开销，尤其是一个job包含几千个task任务的情况。jvm重用可以使jvm实例在同一个job重复使用N次，N可以在mapred-site.xml中配置，通常在10-20之间。

mapreduce.job.jvm.numtasks=10

缺点是某个job由于其中的某个task执行时间过长，导致一直占用jvm，而其他的job无法使用。

##### 9.8 推测执行

将一个task，再启动一个备份task执行，谁先执行完成，就使用谁的

##### 9.9 压缩

##### 9.10 执行计划（Explain）

explain 执行的SQL

可以帮助我们分析这个SQL的执行过程

