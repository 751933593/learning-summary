### Sqoop

#### 1.简介

[Apache Sqoop](http://sqoop.apache.org)是一种用于在Apache Hadoop和关系数据库等结构化数据存储之间高效传输大数据的工具。

Sqoop在2012年3月成功从孵化器毕业，现在是顶级Apache项目：更多信息

最新的稳定版本是1.4.7。最新的Sqoop2是1.99.7。请注意，1.99.7不兼容1.4.7和功能不完整，它不打算用于生产部署。

#### 2.下载安装

下载地址：http://www.apache.org/dyn/closer.lua/sqoop/1.4.7

文档地址：http://sqoop.apache.org/docs/1.4.7/index.html

#### 3.案例

- 修改配置文件

  在sqoop-env.sh中配置Sqoop与Hadoop中其他框架的连接
  
- 添加mysql的jdbc包

- 连接mysql

  ```shell
  bin/sqoop list-databases --connect jdbc:mysql://hadoop102:3306/ --username root passowrd 123456
  ```

##### 3.1 导入

- 将mysql中数据导入到hdfs中

  ```powershell
  # 将整个表中的数据导入到hdfs中
  bin sqoop import \
  --connect jdbc:mysql://hadoop102:3306/databasename \
  --username root \
  --password 123456 \
  --table staff \
  --target-dir /staff \
  --delete-taget-dir \
  --num-mappers 1 \
  --fields-terminated-by "\t"
  # 将sql查询出的数据导入到hdfs中
  -- query "SELECT * FROM x WHERE a='foo' AND \$CONDITIONS"
  # 提取表的指定列
  -- columns id,name
  ## 指定条件
  -- where "id=1"
  ```

- 将mysql中数据导入到hive中

  ```powershell
  # 先将数据从mysql导入到hdfs（user/用户名/表名）中，然后从hdfs导入到hive中，并删除原hdfs中的数据
  bin/sqoop import \
  --connect jdbc:mysql://hadoop102:3306/databasename \
  --username root \
  --password 123456 \
  --table staff \
  --hive-import \
  --field-terminated-by "\t" \
  --hive-overwrite \
  --hive-table staff_hive
  ```

- 将mysql数据导入到hbase

  ```powershell
  bin/sqoop import \
  --connect jdbc:mysql://hadoop102:3306/databasename \
  --username root \
  --password 123456 \
  --table staff \
  --column-family "info" \
  --hbase-create-table \
  --hbase-row-key "id" \
  --hbase-talbe "hbase_company" \
  --num-mappers 1 \
  --split-by id
  ```


##### 3.2 导出

- 将数据从hdfs导出到mysql

  ```powershell
  bin/sqoop export \
  --connect jdbc:mysql://hadoop102:3306/databasename \
  --username root \
  --password 123456 \
  --table staff \
  --export-dir /staff \
  --num-mappers 1 \
  --import-fieldsfields-terminated-by "\t"
  ```

##### 3.3 sqoop脚本

1. 创建一个脚本文件

   ```powershell
   touch ./job/aa.opt
   ```

2. 编写脚本

   ```powershell
   vi ./job/aa.opt
   export 
   --connect 
   jdbc:mysql://hadoop102:3306/databasename 
   --username 
   root 
   --password 
   123456 
   --table 
   staff 
   --export-dir 
   /staff 
   --num-mappers 
   1 
   --import-fieldsfields-terminated-by "\t"
   ```

3. 执行脚本

   ```powershell
   bin/sqoop --options-file ./job/aa.opt
   ```

   

