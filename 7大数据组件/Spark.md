### Spark

#### 1.MR在Hadoop1.0时的缺陷

- 每次计算都是独立的。获取数据-计算-存储数据。不适合数据挖掘和机器学习这样的迭代计算。
- 计算性能慢。基于文件存储，连续计算时存在IO。
- 与Hadoop紧密耦合在一起
- JobTracker -> TaskTracker

#### 2.在Hadoop2.x中出现了yarn

- 由于MR存在缺陷，而计算框架与Hadoop紧密耦合，所以我们需要将资源和任务分隔开

- ApplicationMaster将RM与Driver解耦，Container将NM与Task解耦

  ![](img\spark\yarn基本结构.png)

#### 3.Spark的特点

- 快。计算的结果是存在内存中的。

- 易用。支持Java、Python

- 基本结构

  ![](img\spark\spark基本结构.jpg)

#### 4.Local模式

##### 4.1 概述 

Local模式就是运行在一台机器上的模式，用于练手和测试

- local[K]：指定K个线程，每个线程模拟一个Worker
- local[*]：电脑几核就启动几个线程

##### 4.2 安装与使用

1. 下载：spark.apache.org

   本次下载版本：spark-2.4.4-bin-hadoop2.7.tgz

2. 官方求pi案例

   ```powershell
   bin/spark-submit \
   --class org.apache.spark.examples.SparkPi \
   --executor-memory 1G \
   --total-executor-cores 2 \
   ./examples/jars/spark-examples_2.11-2.4.4.jar \
   100
   ```

3. 启动

   ```powershell
   bin/spark-shell
   ```

4. 控制台实现WordCount

   - 创建文件夹，里面的文件放入单词

       ```powershell
       mkdir input
       echo 'hello world' >> input/1.txt
       echo 'hello scala' >> input/1.txt
       echo 'hello spark' >> input/2.txt
       ```

   - 进入spark控制台执行命令

     ```powershell
     sc.textFile("input").flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_).collect
     ```

     textFile：读取文件夹中的文件
     
     flatMap：将文件中的一行分割成一个个的单词
     
     map：将单词映射成元组
     
     reduceByKey：按照key进行聚合，相加
     
     collect：将数据传送到Driver中
   
5. 接口实现WordCount

   - idea中安装插件，下载ScalaSDK
   
   - 安装windows环境中的Hadoop：https://blog.csdn.net/qq_32575047/article/details/81941201 
   
   - pom.xml
   
     ```xml
     <dependency>
         <groupId>org.apache.spark</groupId>
         <artifactId>spark-core_2.11</artifactId>
         <version>2.4.4</version>
     </dependency>
     ```
   
   - 代码，在项目根目录下创建input，里面放单词文件
   
     ```scala
     import org.apache.spark.rdd.RDD
     import org.apache.spark.{SparkConf, SparkContext}
     
     object WordCount {
     
         def main(args: Array[String]): Unit = {
     
             // local模式
             // 创建SparkConf对象
             val conf = new SparkConf().setMaster("local[*]").setAppName("WordCount")
     
             // 创建spark上下文对象
             val sc = new SparkContext(conf)
     
             // 将文件夹下的文件一行一行的读取
             val lines: RDD[String] = sc.textFile("input")
     
             // 将数据分解成单词集合
             val words: RDD[String] = lines.flatMap(_.split(" "))
     
             // 将每个单词变成(key,1)形式的元组
             val wordsTuples: RDD[(String, Int)] = words.map((_,1))
     
             // 将元组按照key聚合，值相加
             val wordsReduce: RDD[(String, Int)] = wordsTuples.reduceByKey(_+_)
     
             val tuples: Array[(String, Int)] = wordsReduce.collect()
     
             tuples.foreach(println)
         }
     }
     ```
   
   - 打包和发布
   
     ```xml
     <build>
             <finalName>WordCount</finalName>
             <plugins>
                 <plugin>
                     <groupId>net.alchim31.maven</groupId>
                     <artifactId>scala-maven-plugin</artifactId>
                     <version>3.2.2</version>
                     <executions>
                         <execution>
                             <goals>
                                 <goal>compile</goal>
                                 <goal>testCompile</goal>
                             </goals>
                         </execution>
                     </executions>
                 </plugin>
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-assembly-plugin</artifactId>
                     <version>3.0.0</version>
                     <configuration>
                         <archive>
                             <manifest>
                                 <mainClass>WordCount</mainClass>
                             </manifest>
                         </archive>
                     </configuration>
                     <executions>
                         <execution>
                             <id>make-assembly</id>
                             <phase>package</phase>
                             <goals>
                                 <goal>single</goal>
                             </goals>
                         </execution>
                     </executions>
                 </plugin>
             </plugins>
         </build>
     ```
   
     启动jar包
   
     ```powershell
     bin/spark-submit \
     --class com.capture.spark.WordCount \
     ./WordCount-jar-with-dependencies.jar
     ```
   
     **注意：如果是Yarn模式，则会在HDFS中寻找input文件**

#### 5.Yarn模式

##### 5.1 概述

spark客户端直接连接yarn，不需要额外的构建spark集群。有yarn-client和yarn-cluster两种模式，主要区别在于：Driver程序运行节点。

- yarn-client：Driver程序运行在客户端，适用于交互、调试、希望立即看到输出
- yarn-cluster：Driver程序运行在由RM启动的APPMaster，适用于生成环境

##### 5.2 安装与使用

1. 修改Hadoop配置文件yarn-site.xml，添加以下内容：

   ```xml
   <!-- 是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
   <property>
   	<name>yarn.nodemanager.pmem-check-enable</name>
       <value>false</value>
   </property>
   <!-- 是否启动一个线程检查每个任务正使用的虚拟内存，如果任务超出分配值，则直接将其杀掉，默认是false -->
   <property>
   	<name>yarn.nodemanager.vmem-check-enabled</name>
       <value>false</value>
   </property>
   ```

2. 修改spark配置文件spark-env.sh，添加如下配置：

   ```powershell
   # yarn配置文件的路径
   YARN_CONF_DIR=/home/liurunze/lrz_software/spark-tools/hadoop/etc/hadoop
   ```

3. 将以上两个配置文件分发一下

4. 执行PI测试程序，需要提前启动yarn集群和HDFS集群

   ```powershell
   bin/spark-submit \
   --class org.apache.spark.examples.SparkPi \
   --master yarn \
   --deploy-mode client \
   ./examples/jars/spark-examples_2.11-2.4.4.jar \
   100
   ```

##### 5.3 Yarn查看Spark日志

1. 修改spark的配置文件spark-defaults.conf

   ```properties
   spark.yarn.historyServer.address=hadoop102:18080
   spark.history.ui.port=18080
   ```

2. 重启spark历史服务

   ```powershell
   sbin/stop-history-server.sh
   sbin/start-history-server.sh
   ```

3. 提交任务到yarn执行

   ```powershell
   bin/spark-submit \
   --class org.apache.spark.examples.SparkPi \
   --master yarn \
   --deploy-mode client \
   ./examples/jars/spark-examples_2.11-2.4.4.jar \
   100
   ```

4. 在Hadoop的web页面查看日志

5. 启动命令

   ```powershell
   bin/spark-shell --master yarn
   ```

##### 5.4 Spark在Yarn上执行流程

![](img\spark\spark在yarn上执行流程.jpg)

#### 6.Standalone模式

##### 6.1 概述

构建一个有Master+Slave构成的Spark集群

![](img\spark\standalone模式.jpg)

##### 6.2 安装与使用

1. 修改spark的配置文件slaves

   ```reStructuredText
   node01
   node02
   node03
   ```

2. 修改spark的配置文件spark-env.sh

   ```properties
   SPARK_MASTER_HOST=node01
   SPARK_MASTER_PORT=7077
   ```

3. 分发spark包

   ```powershell
   xsync spark/
   ```

4. 启动

   ```powershell
   sbin/start-all.sh
   util.sh
   
   # 启动时如果报错：JAVA_HOME not set，在sbin/spark-config.sh中加入：
   export JAVA_HOME=xxx
   ```

5. 求PI测试

   ```shell
   bin/spark-submit \
   --class org.apache.spark.examples.SparkPi \
   --master spark://node01:7077 \
   --executor-memory 1G \
   --total-executor-cores 2 \
   ./examples/jars/spark-examples_2.11-2.4.4.jar \
   100
   ```


#### 7.RDD

##### 7.1 概述

RDD（resilient distributed dataset）弹性分布式数据集，是spark中最基本的计算逻辑抽象，它代表一个不可变、可分区、里面的元素可并行计算的集合。


- A list of partitions

  有分区，可并行，可扩展

- A function for computing each split

  有函数逻辑来计算分区中的数据

- A list of dependencies on other RDDs

  与其他RDD形成依赖

-  Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)

  分区数据支持key-value

- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

  有列表存储每个分区的优先位置

##### 7.2 创建RDD

创建方式有三种：

- 从集合中创建

  ```scala
  val listRDD: RDD[Int] = sc.makeRDD(List(1,2,3,4))
  val arrayRDD: RDD[Int] =sc.parallelize(Array(1,4))
  ```

- 从外部存储中创建

  ```scala
  val fileRDD: RDD[String] = sc.textFile("input")
  // textFile中默认去最小分区数，但是依赖于Hadoop的分片规则:
  // 当文件有5个数，而分区设置为2，则实际会有3个分区
  ```

  

- 从其他RDD创建

##### 7.3 处理单个RDD

1. **map**

   ```scala
   sc.makeRDD(1 to 10)
                   .map(_*2)
                   .collect()
                   .foreach(println)
   ```

2. **mapPartitions**

   - 减少了计算传输的次数
   - 增大了传输计算的内容，由于每次计算复杂，数据量大，可能会造成OOM

   ```scala
   sc.makeRDD(1 to 10)
                   .mapPartitions({
                       //对分区中的每个数据x2
                       partDatas => partDatas.map(_*2)
                   })
                   .collect()
                   .foreach(println)
   ```

3. **mapPartitionsWithIndex**

   ```scala
   sc.makeRDD(1 to 10,2)
                   .mapPartitionsWithIndex{
                       case(num,partDatas) => {
                           partDatas.map((_,"part_no:"+num))
                       }
                   }
                   .collect()
                   .foreach(println)
   ```

4. **flatMap: 将集合中的数据平铺出来**

   ```scala
   // 将[{1,2},{3,4}]变为1 2 3 4
   sc.makeRDD(Array(List(1,2),List(3,4)))
                   .flatMap(datas => datas)
                   .collect()
                   .foreach(println)
   ```

5. **glom: 将每个分区里的输入放到数组里**

   ```scala
   sc.makeRDD(1 to 16,4)
                   .glom()
                   .collect()
                   .foreach(datas => {
                       println(datas.mkString(","))
                   })
   ```

6. **group: 按照传入函数结果进行分组**

   ```scala
   sc.makeRDD(1 to 16)
   				// 按照余数为0，1，2分组
                   .groupBy(i => i%3)
                   .collect()
                   .foreach(println)
   /* 
   打印结果为：[Int,Iterable[Int]]
   (0,CompactBuffer(3, 6, 9, 12, 15))
   (1,CompactBuffer(1, 4, 7, 10, 13, 16))
   (2,CompactBuffer(2, 5, 8, 11, 14))
   */
   ```

7. **filter**

   ```scala
   sc.makeRDD(1 to 16)
   				// 过滤出能被3整除的数
                   .filter(i => i%3==0)
                   .collect()
                   .foreach(println)
   ```

8. **sample: 采样**

   ```scala
   sc.makeRDD(1 to 16)
   				//第一个参数：取后是否放回
                   .sample(false,0.7,1)
                   .collect()
                   .foreach(println)
   ```

9. **distinct: 去重**

   ```scala
   sc.makeRDD(List(3,3,2,5,3,2,3,6))
   				// 去重后将数据放入2个分区
                   .distinct(2)
                   .collect()
                   .foreach(println)
   ```

10. **coalesce: 缩减分区（合并分区）**

    ```scala
    sc.makeRDD(1 to 16, 4)
    				// 将4个分区缩减为2个
                    .coalesce(2)
                    .collect()
                    .foreach(println)
    ```

11. **repartition: 重新分区**

    ```scala
    sc.makeRDD(1 to 16, 4)
                    .repartition(2)
                    .collect()
                    .foreach(println)
    ```

12. **sortBy**

    ```scala
    sc.makeRDD(1 to 16, 4)
    				// false：降序
                    .sortBy(x => x, false)
                    .collect()
                    .foreach(println)
    ```

##### 7.4 处理多个RDD

1. **union: 并集**

   ```scala
   sc.makeRDD(1 to 16, 4)
                   .union(sc.makeRDD(10 to 20, 4))
                   .collect()
                   .foreach(println)
   ```

2. **subtract: 差集**

   ```scala
   sc.makeRDD(1 to 16, 4)
                   .subtract(sc.makeRDD(10 to 20, 4))
                   .collect()
                   .foreach(println)
   ```

3. **intersection: 交集**

   ```scala
   sc.makeRDD(1 to 16, 4)
                   .intersection(sc.makeRDD(10 to 20, 4))
                   .collect()
                   .foreach(println)
   ```

4. **cartesian: 笛卡尔积**

   ```scala
   sc.makeRDD(1 to 16, 4)
                   .cartesian(sc.makeRDD(10 to 20, 4))
                   .collect()
                   .foreach(println)
   ```

5. **zip:组合（分区数相等，每一个分区的个数相等）**

   ```scala
   sc.makeRDD(List(1,2,3), 4)
                   .zip(sc.makeRDD(Array("a","b","c"), 4))
                   .collect()
                   .foreach(println)
   ```

##### 7.5 处理key-value形式的RDD

1. **partitionBy: 自定义分区**

   ```scala
   sc.makeRDD(List((1,"a"),(2,"b"),(3,"c"),(4,"d")), 4)
                   .partitionBy(new MyPartitioner(4))
                   .mapPartitionsWithIndex({
                       case (num,datas) => {
                           datas.map(("分区号："+num,_))
                       }
                   })
                   .collect()
                   .foreach(println)
   
   class MyPartitioner (partitions: Int) extends Partitioner {
       override def numPartitions: Int = {
           partitions
       }
       override def getPartition(key: Any): Int = {
           //都放入0号分区中
           0
       }
   }
   ```

2. **groupByKey**

   ```scala
   sc.makeRDD(List((1,"a"),(1,"b"),(4,"c"),(4,"d")), 4)
   				// key相同的放在一组
                   .groupByKey()
                   .collect()
                   .foreach(println)
   ```

3. **reduceByKey（与groupByKey性能更好一些，因为在shuffle之前，有一个combine）**

   ```scala
   sc.makeRDD(List((1,"a"),(1,"b"),(4,"c"),(4,"d")), 4)
   				// key相同的放在一组
                   .reduceByKey(_+_)
                   .collect()
                   .foreach(println)
   ```

4. **aggregateByKey: 分区内运算和分区间运算不相同**

   ```scala
   sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
   				// 初始值为0，分区内相同key区最大值，分区间相同key的值相加
                   .aggregateByKey(0)(math.max(_,_),_+_)
                   .collect()
                   .foreach(println)
   ```

5. **foldByKey: 分区内运算和分区间运算相同**

   ```scala
   sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
                   .foldByKey(0)(_+_)
                   .collect()
                   .foreach(println)
   ```
   
6. **combineByKey: 改变初始值的结构 **

   ```scala
   sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
                   .combineByKey(
                       // 将初始值设为(_,1)
                       (_,1),
                       // 分区内计算：(a(3,1),2)=>(a(3+2),1+1)
                       (acc:(Int,Int),v)=>(acc._1+v,acc._2+1),
                       // 分区间计算：(c(10,2),c(8,1))=>(c(10+8,2+1))
                       (acc1:(Int,Int),acc2:(Int,Int))=>(acc1._1+acc2._1,acc1._2+acc2._2)
                   )
                   .collect()
                   .foreach(println)
   ```

7. **sortByKey**

   ```scala
   sc.makeRDD(List((3,"a"),(2,"a"),(4,"c"),(3,"b"),(6,"c"),(8,"c")), 2)
   				// 根据key正序排序
                   .sortByKey(true)
                   .collect()
                   .foreach(println)
   ```

8. **mapValues: 对value做处理**

   ```scala
   sc.makeRDD(List((3,"a"),(2,"a"),(4,"c"),(3,"b"),(6,"c"),(8,"c")), 2)
                   .mapValues(_+"-end")
                   .collect()
                   .foreach(println)
   ```

9. **join: key相同的连接起来（内连接）**

   ```scala
   sc.makeRDD(List((3,"a"),(2,"a"),(4,"c"),(3,"b"),(6,"c"),(8,"c")), 2)
           .join(sc.makeRDD(List((3,"b"),(2,"b"),(4,"d"),(3,"e"),(6,"s"),(8,"f")), 2))
           .collect()
           .foreach(println)
   ```

10. **cogroup: key相同的连接起来（外连接）**

    ```scala
    sc.makeRDD(List((3,"a"),(2,"a"),(4,"c"),(3,"b"),(6,"c"),(8,"c")), 2)
                    .cogroup(sc.makeRDD(List((3,"b"),(2,"b"),(4,"d"),(3,"e"),(6,"s")), 2))
                    .collect()
                    .foreach(println)
    ```

##### 7.6 Action

1. **reduce: 先分区内聚合，再分区间聚合**

   ```scala
   println( sc.makeRDD(1 to 5, 2).reduce(_+_) )
   ```

2. **collect: 返回数组的形式**

3. **count: 返回RDD内集合中数据的个数（Long）**

4. **first: 返回RDD内集合中第一个数据**

5. **take: 返回RDD内集合中前N个数据**

   ```scala
   sc.makeRDD(1 to 5, 2)
                   .take(3)
   				.foreach(println)
   ```
   
6. **takeOrdered: 返回拍讯后RDD内集合中前N个数据**

7. **aggregate: 分区内和分区间计算不同（分区间计算也要计算初始值）**

   ```scala
   println( sc.makeRDD(1 to 5, 2).aggregate(0)(_+_,_+_) )
   ```
   
8. **fold: 分区内和分区间计算相同（分区间计算也要计算初始值）**

   ```scala
   println( sc.makeRDD(1 to 5, 2).fold(0)(_+_) )
   ```
   
9. **saveAsTextFile、saveAsSequenceFile、saveAsObjectFile**

   ```scala
   sc.makeRDD(1 to 5, 2).saveAsObjectFile("output1")
   sc.makeRDD(1 to 5, 2).saveAsObjectFile("output2")
   sc.makeRDD(1 to 5, 2).saveAsObjectFile("output3")
   ```
   
10. **countByKey**

    ```scala
    sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
                       .countByKey().foreach(println)
    ```

11. **foreach**

    ```scala
    sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
                       .foreach(println)
    ```

##### 7.7 RDD依赖关系

```scala
sc.makeRDD(List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8)), 2)
                   .map((_,1))
					.reduceByKey(_+_)
					// 查看rdd的依赖关系
					.toDebugString()
```

RDD任务划分为：Application、Job、Stage、Task

- Application：初始化一个SparkContext即生成一个Application
- Job：一个Action算子就是一个Job
- Stage：遇到一个shuffle依赖则划分一个Stage
- Task：数据分发到多少个分区（Executor），就有多少个任务

##### 7.8 RDD的缓存

checkpoint：将数据持久化缓存到检查点（HDFS中）

```scala

        val conf = new SparkConf().setMaster("local[*]").setAppName("WordCount")

        val sc = new SparkContext(conf)
		// 设置checkpoint目录
        sc.setCheckpointDir("check")

        val reduce = sc.makeRDD(List(1,2,3,4),4)
                .map((_,1))
                .reduceByKey(_+_)
        //  将reduce持久化      
		reduce.checkpoint()
		reduce.foreach(println)
		println(reduce.toDebugString)
		
		sc.stop()
```

##### 7.9 RDD的读取与存储

- HDFS
- MySQL
- HBASE

#### 8.Spark的三大数据结构

RDD、广播变量、累加器

- RDD：弹性分布式数据集

- 广播变量：分布式只读共享变量

  ```scala
  		// 根据key合并List((1,1),(2,2),(3,3))与List((1,"a"),(2,"b"),(3,"c"))
          val list = List((1,1),(2,2),(3,3))
          // 声明广播变量
          val boradcast: Broadcast[List[(Int, Int)]] = sc.broadcast(list)
          sc.makeRDD(List((1,"a"),(2,"b"),(3,"c")))
                  .map{
                      case (key, value) => {
                          var v2: Any = null
                          for (elem <- boradcast.value) {
                              if (key == elem._1){
                                  v2 = elem._2
                              }
                          }
                          (key, (value, v2))
                      }
                  }
                  .foreach(println)
  ```

- 累加器：分布式只写共享变量

  ```scala
  // 创建一个累加器
  val accumulator: LongAccumulator = sc.longAccumulator
  
  sc.makeRDD(List(1,2,3,4),4)
  	.foreach(i => accumulator.add(i))
  
  println(accumulator.value)
  ```

  自定义累加器：

  ```scala
  		// 创建一个累加器
          val accumulator = new WordAccumulator()
  
          // 向sc中注册累加器
          sc.register(accumulator)
  
          sc.makeRDD(List("hello","word","spark","scala"),4)
                  .foreach(word => accumulator.add(word))
  
          println(accumulator.value)
  
  class WordAccumulator extends AccumulatorV2[String,ArrayList[String]]{
  
      val list = new ArrayList[String]()
  
      // 是否是初始状态
      override def isZero: Boolean = {
          list.isEmpty
      }
  
      override def copy(): AccumulatorV2[String, ArrayList[String]] = {
          new WordAccumulator()
      }
  
      override def reset(): Unit = {
          list.clear()
      }
  
      override def add(v: String): Unit = {
          if (v.contains("a")) {
              list.add(v)
          }
      }
  
      // 与另外一个累加器合并
      override def merge(other: AccumulatorV2[String, ArrayList[String]]): Unit = {
          list.addAll(other.value)
      }
  
      // 获取累加器的结果
      override def value: ArrayList[String] = list
  }
  ```

#### 9.Spark SQL

##### 9.1 Spark SQL 的两种数据结构

- DataFrame
- DataSet

##### 9.2 DataFrame

- 创建session视图

    ```shell
    scala>val dataframe = spark.read.json("1.json")
    scala>dataframe.show
    scala>dataframe.createTempView("student")
    scala>spark.sql(select * from student).show
    ```

- 创建全局视图

  ```shell
  scala>val dataframe = spark.read.json("1.json")
  scala>dataframe.show
  scala>dataframe.createGlobalTempView("student")
  scala>spark.sql(select * from global_temp.student).show
  ```

- RDD转换为DF

  RDD、DF、DS 之间转换需要导入隐式转换规则：import spark.implicits._

  rdd.toDF("列名1", "列名2")

  如果rdd中是对象集，可以直接使用rdd.toDF

- DF转换为RDD

  df.rdd
  
- DF转换成DS

    df.as[类型]

##### 9.3 DataSet

- 创建DS

  ```powershell
  # 创建样例类
  scala> case Person(name:String, age: Long)
  # 创建DS
  scala> val ds = Seq(Person("Sam",12)).toDS()
  scala> ds.show
  ```

- RDD转换成DS

  rdd.toDS

- DS转换成RDD

  ds.rdd

- DS转化成DF

  ds.toDF

##### 9.4 使用IDEA开发SparkSQL程序

- 导入maven依赖

  ```xml
  <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_2.11</artifactId>
      <version>2.4.4</version>
  </dependency>
  ```

- RDD、DF、DS

  ```scala
  val conf = new SparkConf().setMaster("local[*]").setAppName("Spark01_RDD")
  
  // 创建spark上下文对象
  val spark: SparkSession = SparkSession.builder().config(conf).getOrCreate()
  
  // 创建RDD
  val rdd: RDD[(Int, String, Int)] = spark.sparkContext.makeRDD(List((1, "zhangsan", 				10), (2, "lisi", 20), (3, "wangwu", 30)))
  
  // 导入隐式转换
  import spark.implicits._
  
  // 将RDD转换为DF
  val df: DataFrame = rdd.toDF("id", "name", "age")
  
  // 将DF转换为DS
  val ds: Dataset[Person] = df.as[Person]
  
  // 将DS转换为DF
  val df1: DataFrame = ds.toDF()
  
  // 将DF转换为RDD
  val rdd1: RDD[Row] = df1.rdd
  
  rdd1.foreach(row => println(row.getString(1)))
  
  spark.stop
  
  case class Person(id:Int, name:String, age:Int)
  ```

- udf（用户自定义函数）

  ```scala
  // 创建RDD
  val rdd: RDD[(Int, String, Int)] = spark.sparkContext.makeRDD(List((1, "zhangsan", 				10), (2, "lisi", 20), (3, "wangwu", 30)))
  
  // 导入隐式转换
  import spark.implicits._
  
  val ds: DataFrame = rdd.toDF("id", "name", "age")
  
  ds.createOrReplaceTempView("person")
  
  // 用户自定义函数
  spark.udf.register("addName",(x:String)=>"name: "+x)
  
  spark.sql("select id,addName(name),age from person").show()
  ```

- 用户自定义聚合函数（弱类型）

  ```scala
  def main(args: Array[String]): Unit = {
  
      val conf = new SparkConf().setMaster("local[*]").setAppName("Spark01_RDD")
  
      // 创建spark上下文对象
      val spark: SparkSession = SparkSession.builder().config(conf).getOrCreate()
  
      // 创建RDD
      val rdd: RDD[(Int, String, Int)] = spark.sparkContext.makeRDD(List((1, "zhangsan", 				10), (2, "lisi", 20), (3, "wangwu", 30)))
  
      // 导入隐式转换
      import spark.implicits._
  
      val ds: DataFrame = rdd.toDF("id", "name", "age")
  
      ds.createOrReplaceTempView("person")
  
      // 用户自定义聚合函数
      val myAgeAvg = new MyAgeAvg()
      spark.udf.register("ageAvg",myAgeAvg)
  
      spark.sql("select ageAvg(age) from person").show()
  }
  
  // 自定义求年龄平均值
  class MyAgeAvg extends UserDefinedAggregateFunction{
  
      // 输入类型
      override def inputSchema: StructType = {
          new StructType().add("age",IntegerType)
      }
  
      // 缓冲区参数类型
      override def bufferSchema: StructType = {
          new StructType().add("sum",IntegerType).add("count",IntegerType)
      }
  
      // 返回类型
      override def dataType: DataType = DoubleType
  
      // 函数是否稳定
      override def deterministic: Boolean = true
  
      // 初始化缓冲区的参数
      override def initialize(buffer: MutableAggregationBuffer): Unit = {
          buffer(0)=0
          buffer(1)=0
      }
  
      // 新数据来时，缓冲区数据更新
      override def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
          buffer(0)=buffer.getInt(0)+input.getInt(0)
          buffer(1)=buffer.getInt(1)+1
      }
  
      // 每个分区的缓冲区数据合并
      override def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
          buffer1(0)=buffer1.getInt(0)+buffer2.getInt(0)
          buffer1(1)=buffer1.getInt(1)+buffer2.getInt(1)
      }
  
      override def evaluate(buffer: Row): Any = {
          buffer.getInt(0).toDouble / buffer.getInt(1)
      }
  }
  ```

- 用户自定义聚合函数（强类型）

  ```scala
  def main(args: Array[String]): Unit = {
  
      val conf = new SparkConf().setMaster("local[*]").setAppName("Spark01_RDD")
  
      // 创建spark上下文对象
      val spark: SparkSession = SparkSession.builder().config(conf).getOrCreate()
  
      // 创建RDD
      val rdd: RDD[(Int, String, Int)] = spark.sparkContext.makeRDD(List((1, "zhangsan", 				10), (2, "lisi", 20), (3, "wangwu", 30)))
  
      // 导入隐式转换
      import spark.implicits._
      val df: DataFrame = rdd.toDF("id", "name", "age")
      val ds: Dataset[Person] = df.as[Person]
  
      // 用户自定义聚合函数
      val myAgeAvgStrong = new MyAgeAvgStrong
      // 将聚合函数转化为查询列
      val ageColnum: TypedColumn[Person, Double]=myAgeAvgStrong.toColumn.name("avgAge")
  
      ds.select(ageColnum).show()
  }
  
  case class Person(id:Int, name:String, age:Int)
  case class AgeAvgBuffer(var sum:Int, var count:Int)
  
  class MyAgeAvgStrong extends Aggregator[Person,AgeAvgBuffer,Double]{
      // 缓冲区初始化
      override def zero: AgeAvgBuffer = {
          AgeAvgBuffer(0,0)
      }
      // 缓冲区内聚合计算
      override def reduce(b: AgeAvgBuffer, a: Person): AgeAvgBuffer = {
          b.sum = b.sum+a.age
          b.count = b.count+1
          b
      }
      // 不同分区的缓冲区合并
      override def merge(b1: AgeAvgBuffer, b2: AgeAvgBuffer): AgeAvgBuffer = {
          b1.count = b1.count+b2.count
          b1.sum = b1.sum+b2.sum
          b1
      }
  
      override def finish(reduction: AgeAvgBuffer): Double = {
          reduction.sum.toDouble / reduction.count
      }
  
      override def bufferEncoder: Encoder[AgeAvgBuffer] = Encoders.product
  
      override def outputEncoder: Encoder[Double] = Encoders.scalaDouble
  }
  ```

##### 9.5 Spark SQL 的读取与存储

- 读取存储文件

  ```shell
  # 默认读取parquet文件
  spark.read.format("json").load("路径")
  spark.read.json("路径")
  # 存储 模式有：error（存在就报错，默认），append，overwrite，ignore（存在就忽略）
  spark.write.mode("append")format("json").save("路径")
  ```

- 读取关系型数据库

  ```powershell
  # 首先需要在spark的启动lib包中放入jdbc的连接包
  # 方式一
  val df = spark.read.format("jdbc")
  	.option("url","jdbc:mysql://linux1:3306/rdd")
  	.option("dbtable","user")
  	.option("user","root")
  	.option("password","123456")
  	.load()
  # 方式二
  val connectionProperties = new java.util.Properties()
  connectionProperties.put("user","root")
  connectionProperties.put("password","123456")
  val df1 = spark.read
  	.jdbc("jdbc:mysql://linux1:3306/rdd","user",connectionProperties)
  ```

- 和HIVE连接

  将HIVE中的hive-site.xml放入（或软连接）spark的conf文件夹中

#### 10.Spark Streaming

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.4.4</version>
</dependency>
```

![](img\spark\SparkStreaming架构图.png)

- 计算WordCount

  1. 在Linux上安装netcat，使用netcat发送数据

     ```powershell
     yum install -y nc
     ```

  2. 启动netcat

     ```powershell
     nc -lk 9999
     ```

  3. 编写代码

     ```scala
     def main(args: Array[String]): Unit = {
     
         val conf = new SparkConf().setMaster("local[*]").setAppName("SparkStreaming")
     
         // 创建SparkStreaming上下文对象，传入采集周期
         val streamingContext = new StreamingContext(conf, Seconds(5))
     
         // 从端口中取数据
         val dataStream: ReceiverInputDStream[String] = 				streamingContext.socketTextStream("192.168.200.120", 9999)
     
         // wordCount
         dataStream.flatMap(_.split(" "))
                     .map((_,1))
                     .reduceByKey(_+_)
                     .print()
     
         // 启动采集器
         streamingContext.start()
         // Driver等待采集器停止
         streamingContext.awaitTermination()
     }
     ```

  4. 发送数据

##### 10.1 数据源

- 文件

  采集文件夹下的文件

  ```scala
  // 采集test文件夹下的数据
  streamingContext.textFileStream("test")
  ```

- 内存

- 自定义数据源

  ```scala
  // 自定义接收器
          val dataStream: ReceiverInputDStream[String] = streamingContext.receiverStream(new MyReceiver("192.168.200.120",9999))
  
  class MyReceiver(host:String, port:Int) extends Receiver[String](StorageLevel.DISK_ONLY){
  
      private var socket: Socket = _
  
      def receive(): Unit = {
  
          val reader = new BufferedReader(new InputStreamReader(socket.getInputStream(),"utf-8"))
          var line:String = null
          while((line = reader.readLine())!=null) {
              if (line.equalsIgnoreCase("end")){
                  return
              }
              store(line)
          }
          if (!isStopped()) {
              restart("Socket data stream had no more data")
          } else {
              println("Stopped receiving")
          }
      }
  
      override def onStart(): Unit = {
          socket = new Socket(host, port)
          // Start the thread that receives data over a connection
          new Thread("Socket Receiver") {
              setDaemon(true)
              override def run() { receive() }
          }.start()
      }
  
      override def onStop(): Unit = {
          // in case restart thread close it twice
          synchronized {
              if (socket != null) {
                  socket.close()
                  socket = null
              }
          }
      }
  }
  ```

- Kafka数据源

  ```xml
  <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
      <version>2.4.4</version>
  </dependency>
  
  <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-clients</artifactId>
      <version>0.11.0.2</version>
  </dependency>
  ```

  ```scala
  // 从kafka中采集数据,topic:word;groupid:consumer001;partition:3
  val kafkaDStream: ReceiverInputDStream[(String, String)] = KafkaUtils.createStream(streamingContext, "192.168.200.120:2181", "consumer001", Map("word" -> 3))
  
  ```

##### 10.2 将无状态数据变为有状态

```scala
val conf = new SparkConf().setMaster("local[*]").setAppName("SparkStreaming")

// 创建SparkStreaming上下文对象
val streamingContext = new StreamingContext(conf, Seconds(5))

// 设置检查点
streamingContext.sparkContext.setCheckpointDir("checkpoint")

// 从kafka中采集数据,topic:word;groupid:consumer001;partition:3
val kafkaDStream: ReceiverInputDStream[(String, String)] = KafkaUtils.createStream(streamingContext, "192.168.200.120:2181", "consumer001", Map("word" -> 3))

kafkaDStream.flatMap(t=>t._2.split(" "))
.map((_,1))
// 根据key将数据的状态保存起来，放入检查点中
.updateStateByKey{
case (seq, buffer) =>{
var sum = buffer.getOrElse(0)+seq.sum
Option(sum)
}
}
.print()

// 启动采集器
streamingContext.start()
// Driver等待采集器停止
streamingContext.awaitTermination()
```

##### 10.3 window：滑动窗口

- scala 中的滑动窗口

```scala
val ints = List(1, 2, 3, 4, 5, 6)
// 设置滑动窗口的大小为3，步长为2
val iterator: Iterator[List[Int]] = ints.sliding(3, 2)
for (elem <- iterator) {
println(elem.mkString(","))
}
```

- spark中的滑动窗口

```scala
// 设置窗口大小为15秒，步长为5秒
val wDStream: DStream[(String, String)] = kafkaDStream.window(Seconds(15), Seconds(5))
```

##### 10.4 transform

```scala
// 这里的代码在Driver中执行，程序启动执行1次
kafkaDStream.transform{
    case rdd =>{
        // 这里的代码在Driver中执行，执行的次数与采集周期有关
        rdd.map{
            case x =>{
                // 这里的代码在Executor中执行，执行的次数与Executor的个数有关
                (x,1)
            }
        }
    }
}
```

##### 10.5 join

可以将两个同的StreamingContext中的数据做关联

##### 10.6 foreachRDD

查询一个DStream中的所有RDD

#### 11.Spark Core

##### 11.1 Yarn的部署流程

##### 11.2 Spark的框架组件与通信

- Driver

  Spark中的驱动节点，用于执行Spark任务中的main方法，负责实际代码的执行。Driver在Spark作业执行时主要负责：

  1. 将用户的程序转化为作业（job）
  2. 在Executor之间调度任务（task）
  3. 跟踪Executor的执行情况
  4. 通过UI展示查询运行情况

-  Executor

  Executor节点负责运行具体任务，任务之间彼此独立。Executor有两个核心功能：

  1. 负责运行任务（task），并将结果返回给Driver
  2. 通过自身的块管理器（Block Manager）为用户程序中要求缓存的RDD提供内存式存储。RDD是直接缓存在Executor进程内的，因此任务可以在运行时充分利用缓存数据加速运算

##### 11.3 Spark 任务调度机制

![](img\spark\Spark任务调度图.jpg)





