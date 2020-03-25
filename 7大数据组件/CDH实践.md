### CDH实践

#### 1. 安装包准备

https://archive.cloudera.com/cdh6/6.3.1/parcels/

https://archive.cloudera.com/cm6/6.3.1/repo-as-tarball/

> JDK：
>
> jdk-8u231-linux-x64.tar.gz
>
> MySQL：
>
> mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz
>
> mysql-connector-java-5.1.47.jar
>
> CM：
>
> cm6.3.1-redhat7.tar.gz
>
> Parcel：
>
> CDH-6.3.1-1.cdh6.3.1.p0.1470567-el7.parcel
>
> CDH-6.3.1-1.cdh6.3.1.p0.1470567-el7.parcel.sha1
>
> manifest.json
>

#### 2. 规划

| 节点   | DB    | Parcel | CM进程                       | 组件        |
| ------ | ----- | ------ | ---------------------------- | ----------- |
| node01 | MySQL | Parcel | Activity Monitor             | NN NM RM DN |
| node02 |       |        | Alert Publisher Event Server | DN NM       |
| node03 |       |        | Host Monitor Service Monitor | DN NM       |

#### 3. 环境搭建

- 网络配置

  ```shell
  vi /etc/sysconfig/network
  vi /etc/hosts
  echo "172.19.169.145 node01" >> /etc/hosts
  echo "172.19.169.147 node02" >> /etc/hosts
  echo "172.19.169.146 node03" >> /etc/hosts
  ```

- ssh免密登录

  ```shell
  ssh-keygen -t rsa  #生成秘钥
  scp -p ~/.ssh/id_rsa.pub root@<remote_ip>:/root/.ssh/authorized_keys
  ```

- 关闭防火墙

  ```shell
  service iptables stop
  chkconfig iptables off
  
  systemctl stop firewalld
  systemctl disable firewalld
  iptables -F
  ```

- 关闭selinux

  ```shell
  setenforce 0
  vi /etc/selinux/config  (SELINUX=disabled)
  ```

- 配置jdk环境变量

  ```shell
  chown -R root:root /usr/java/jdk1.8.0_231
  vi /etc/profile
  export JAVA_HOME=/usr/java/jdk1.8.0_231
  export PATH=:$JAVA_HOME/bin:$PATH
  source /etc/profile
  ```

- 时间同步（安装NTP）

  ```shell
  chkconfig ntpd on
  service ntpd start
  ntpdate -u s2m.time.edu.cn
  # 时区同步
  timedatectl set-timezone Asia/Shanghai
  
  ```

- 安装配置mysql   https://github.com/Hackeruncle/MySQL

  ```shell
  # 卸载mysql
  yum remove mysql mysql-devel mysql-server mysql-libs compat-mysql51
  # 解压
  tar -zxvf mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz -C /usr/local/
  # 移动并创建子目录
  mv mysql-5.7.11-linux-glibc2.5-x86_64 mysql
  mkdir mysql/arch mysql/data mysql/tmp
  # 编辑my.cnf
  vi /etc/my.cnf
  # 创建用户mysqladmin和组dba
  groupadd -g 101 dba
  useradd -u 514 -g dba -G root -d /usr/local/mysql mysqladmin
  cp /etc/skel/.*  /usr/local/mysql
  # 配置环境变量
  vi mysql/.bashrc
  export MYSQL_BASE=/usr/local/mysql
  export PATH=${MYSQL_BASE}/bin:$PATH
  # 修改权限
  chown  mysqladmin:dba /etc/my.cnf 
  chmod  640 /etc/my.cnf  
  chown -R mysqladmin:dba /usr/local/mysql
  chmod -R 755 /usr/local/mysql 
  # 配置服务及开机自启动
  cp support-files/mysql.server /etc/rc.d/init.d/mysql
  chmod +x /etc/rc.d/init.d/mysql
  chkconfig --add mysql
  # 查看密码
  cat mysql/data/hostname.err | grep password
  # 登录
  su mysqladmin
  service mysql start
  mysql -uroot -p
  # 重置密码
  mysql> alter user root@localhost identified by '123456';
  mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' ;
  mysql> flush privileges;
  # 重启
  service mysql restart
  ```

- 在mysql中，创建CDH元数据库 用户 和 amon的服务的库 用户

  ```shell
  create database cmf default character set utf8;
  create database amon default character set utf8;
  create database reportsmanager default character set utf8;
  grant all privileges on cmf.* to 'cmf'@'%' identified by '123456';
grant all privileges on amon.* to 'amon'@'%' identified by '123456';
  grant all privileges on reportsmanager.* to 'reportsmanager'@'%' identified by '123456';
  flush privileges;
  ```
  
- 下载第三方依赖包

  ```reStructuredText
  chkconfig  python  bind-utils  psmisc libxslt  zlib
  sqlite  cyrus-sasl-plain  cyrus-sasl-gssapi  fuse
  fuse-libs  redhat-lsb
  ```

- 选择第一台部署 amon进程，需要mysql-jdbc.jar

  ```shell
  cp mysql-connector-java-5.1.47.jar /usr/share/java/mysql-connector-java.jar
  
  ```

#### 4. rpm部署CM

```shell
ls cm6.3.1/RPMS/x86_64
```

- cloudera-manager-daemons-6.3.1-1466458.el6.x86_64.rpm

  这个是公共包，server和agent都需要用到

- cloudera-manager-server-6.3.1-1466458.el6.x86_64.rpm

- cloudera-manager-agent-6.3.1-1466458.el6.x86_64.rpm

- cloudera-manager-server-db-2-6.3.1-1466458.el6.x86_64.rpm

  这个是官方指定数据库postgresql，我们用mysql代替

**配置server：**

```shell
# 有网的情况下
yum install cloudera-manager-daemons-6.3.1-1466458.el6.x86_64.rpm
# 没有网的情况下
rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el6.x86_64.rpm --nodeps --force
rpm -ivh cloudera-manager-server-6.3.1-1466458.el6.x86_64.rpm --nodeps --force
# 编辑CM server的数据库配置文件
vi /etc/cloudera-scm-server/db.properties
# 启动server服务
service cloudera-scm-server start
# 查看server状态
service cloudera-scm-server status
```

```properties
# Copyright (c) 2012 Cloudera, Inc. All rights reserved.
#
# This file describes the database connection.
#

# The database type
# Currently 'mysql', 'p：ostgresql' and 'oracle' are valid databases.
com.cloudera.cmf.db.type=mysql

# The database host
# If a non standard port is needed, use 'hostname:port'
com.cloudera.cmf.db.host=node01:3306

# The database name
com.cloudera.cmf.db.name=cmf

# The database user
com.cloudera.cmf.db.user=cmf

# The database user's password
com.cloudera.cmf.db.password=123456

# The db setup type
# After fresh install it is set to INIT
# and will be changed post config.
# If scm-server uses Embedded DB then it is set to EMBEDDED
# If scm-server uses External DB then it is set to EXTERNAL
com.cloudera.cmf.db.setupType=EXTERNAL
```

**配置agent：**

```shell
cd /home/liurunze/lrz_software/tools/cm6.3.1/RPMS/x86_64/
rpm -ivh cloudera-manager-daemons-6.3.1-1466458.el6.x86_64.rpm --nodeps --force
rpm -ivh cloudera-manager-agent-6.3.1-1466458.el6.x86_64.rpm --nodeps --force
# 修改agent配置文件
vi /etc/cloudera-scm-agent/config.ini
# 启动agent服务
service cloudera-scm-agent start
# 查看agent状态
service cloudera-scm-agent status
```

**登录CM页面：**

http://192.168.200.100:7180/cmf/login

**配置parcel文件：**

```shell
# 下载httpd服务
yum install -y httpd
mkdir /var/www/html/cdh6_parcel
# 将三个文件移动到/var/www/html文件夹下
mv manifest.json /var/www/html/cdh6_parcel/
mv CDH-6.3.1-1.cdh6.3.1.p0.1470567-el6.parcel.sha1 /var/www/html/cdh6_parcel/CDH-6.3.1-1.cdh6.3.1.p0.1470567-el6.parcel.sha
mv CDH-6.3.1-1.cdh6.3.1.p0.1470567-el6.parcel /var/www/html/cdh6_parcel
# 启动httpd服务
service httpd start
# 访问这三个文件
http://192.168.200.100/cdh6_parcel/
# 将这个地址配置到浏览器中，然后下一步自动安装parcel
```

**检查问题并解决隐患：**

```shell
# 禁止使用透明大页面压缩
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# 主机由于交换而运行状况不良
echo 10 > /proc/sys/vm/swappiness
```



#### 部署过程中遇到的问题

- 部署mysql
  
  1. /etc/my.cfg 文件没有配置  user=mysqladmin
  
- 部署cm-server
  1. 包下载错误：应该下载centos6对应的cm
  2. 如果访问不了http://192.168.200.100:7180/cmf/login，去查看/var/log/cloudera-scm-server/下面的日志，有报错信息
  
- 清空mysql的cmf

  ```shell
  drop database cmf;
  show databases;
  ```

- 删除server和agent

  ```shell
  # 1.删除server和agent的rpm包
  rpm -qa | grep cloudera
  rpm -e cloudera-manager-server-6.3.1-1466458.el6.x86_64
  
  rpm -e --nodeps cloudera-manager-daemons-6.3.1-1466458.el6.x86_64 cloudera-manager-agent-6.3.1-1466458.el6.x86_64
  # 2.删除opt下文件
  rm -rf /opt/cloudera
  # 3.删除/var/lib下文件
  rm -rf /var/lib/cloudera-scm-*
  ```

- 服务启动命令

  ```shell
  service cloudera-scm-server start
  service cloudera-scm-server stop
  service cloudera-scm-server restart
  service cloudera-scm-agent start
  service cloudera-scm-agent stop
  service cloudera-scm-agent restart
  ```

- centos6.X 默认python版本是2.6，hue需要2.7的

  centos7.X 默认python版本是2.7

- java和jdbc都有默认路径

- 



