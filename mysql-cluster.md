# 集群
MySQL Cluster是一个基于NDB Cluster存储引擎的完整的分布式数据库系统,在无共享存储设备的情况下实现的一种完全分布式数据库系统。
通过合理的配置，可以将服务请求在多台物理机上分发实现负载均衡 ；同时内部实现了冗余机制，在部分服务器宕机的情况下，整个集群对外提供的服务不受影响，从而能达到99.999%以上的高可用性。
官方文档(https://dev.mysql.com/doc/refman/8.0/en/mysql-cluster-overview.html)

## SQL API 节点
也就是我们常说的MySQL Server。主要负责实现一个数据库在存储层之上的所有事情，比如连接管理，Query 优化和响应 ，Cache 管理等等，只有存储层的工作交给了NDB 数据节点去处理了。也就是说，在纯粹的MySQL Cluster 环境中的SQL 节点，可以被认为是一个不需要提供任何存储引擎的MySQL服务器，因为他的存储引擎有Cluster 环境中的NDB 节点来担任。所以，SQL 层各MySQL服务器的启动与普通的MySQL Server 启动也有一定的区别，必须要添加ndbcluster参数选项才行。我们可以添加在my.cnf配置文件中，也可以通过启动命令行来指定。
## NDB 数据节点
也就是上面说的NDB Cluster。最初的NDB是一个内存式存储引擎，当然也会将数据持久化到存储设备上。但是最新的NDB Cluster存储引擎已经改进了这一点，可以选择数据是全部加载到内存中还是仅仅加载索引数据。NDB 节点主要是实现底层数据存储功能，来保存Cluster 的数据。每一个Cluster节点保存完整数据的一个fragment，也就是一个数据分片（或者一份完整的数据，视节点数目和配置而定），所以只要配置得当，MySQL Cluster在存储层不会出现单点的问题。一般来说，NDB 节点被组织成一个一个的NDB Group，一个 NDB Group实际上就是一组存有完全相同的物理数据的NDB节点群。
上面提到了NDB 各个节点对数据的组织，可能每个节点都存有全部的数据也可能只保存一部分数据，主要是受节点数目和参数来控制的。首先在 MySQL Cluster主配置文件（在管理节点上面，一般为 config.ini）中，有一个非常重要的参数叫NoOfReplicas，这个参数指定了每一份数据被冗余存储在不同节点上面的份数，该参数一般至少应该被设置成2，也只需要设置成2就可以了。因为正常来说，两个互为冗余的节点同时出现故障的概率还是非常小的，当然如果机器和内存足够多的话，也可以继续增大来更进一步减小出现故障的概率。此外，一个节点上面是保存所有的数据还是一部分数据还受到存储节点数目的限制。NDB 存储引擎首先保证NoOfReplicas参数配置的要求来使用存储节点，对数据进行冗余，然后再根据节点数目将数据分段来继续使用多余的NDB节点。分段的数目为节点总数除以NoOfReplicas 所得。
## Manage 管理节点
管理节点负责整个Cluster集群中各个节点的管理工作,包括集群的配置,启动关闭各节点,对各个节点进行常规维护，以及实施数据的备份恢复等。管理节点会获取整个Cluster环境中各节点的状态和错误信息，并且将各 Cluster 集群中各个节点的信息反馈给整个集群中其他的所有节点。由于管理节点上保存了整个Cluster 环境的配置，同时担任了集群中各节点的基本沟通工作，所以他必须是最先被启动的节点。

# 部署
MySQL Cluster 至少需要一个管理节点，一个SQL节点，两个NDB节点。
本例为一个管理节点，两个SQL节点，两个NDB节点。

```json
[ndbd(NDB)]     2 node(s)
id=2    @192.168.231.26  (mysql-5.5.62 ndb-7.2.35, Nodegroup: 0, *)
id=3    @192.168.231.19  (mysql-5.5.62 ndb-7.2.35, Nodegroup: 0)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @192.168.231.27  (mysql-5.5.62 ndb-7.2.35)

[mysqld(API)]   2 node(s)
id=4    @192.168.231.50  (mysql-5.5.62 ndb-7.2.35)
id=5    @192.168.231.30  (mysql-5.5.62 ndb-7.2.35)

```
## 软件安装
每个节点执行
```shell
tar zxvf mysql-cluster-gpl-7.2.35-linux-glibc2.12-x86_64.tar.gz
mv mysql-cluster-gpl-7.2.35-linux-glibc2.12-x86_64 /usr/local/mysql
yum install libaio* -y
groupadd mysql
useradd -g mysql mysql
chown -R mysql:mysql /usr/local/mysql
```

## 文件配置

### 管理节点
```shell
vim /etc/ndb-config.ini

[ndbd default]
NoOfReplicas= 2
DataMemory= 1000M
IndexMemory= 1000M
TimeBetweenWatchDogCheck= 30000
DataDir= /var/lib/mysql-cluster
MaxNoOfOrderedIndexes= 512
LockPagesInMainMemory=1
MaxNoOfConcurrentOperations=500000
TimeBetweenLocalCheckpoints=20
TimeBetweenGlobalCheckpoints=1000
TimeBetweenEpochs=100
TimeBetweenWatchdogCheckInitial=60000
TransactionInactiveTimeout =50000
MaxNoOfExecutionThreads=8
BatchSizePerLocalScan=512
[ndb_mgmd default]
DataDir= /var/lib/mysql-cluster

[ndb_mgmd]
Id=1
HostName= 192.168.231.27
DataDir=/var/lib/mysql-cluster
[ndbd]
Id= 2
HostName= 192.168.231.26
datadir=/usr/local/mysql/data

[ndbd]
Id= 3
HostName= 192.168.231.19
datadir=/usr/local/mysql/data

[mysqld]
Id= 4
HostName=192.168.231.50
[mysqld]
Id=5
HostName=192.168.231.30

[tcp default]
PortNumber= 63132


/usr/local/mysql/bin/ndb_mgmd --initial -f /etc/ndb-config.ini
netstat -langput | grep mgm
tcp        0      0 0.0.0.0:1186            0.0.0.0:*               LISTEN      16830/ndb_mgmd
tcp        0      0 127.0.0.1:56944         127.0.0.1:1186          ESTABLISHED 16830/ndb_mgmd
tcp        0      0 127.0.0.1:1186          127.0.0.1:56944         ESTABLISHED 16830/ndb_mgmd

```

### NDB数据节点

```shell
vim /etc/my.cnf


[MYSQLD]
ndbcluster                      # run NDB engine
character_set_server=utf8
ndb-connectstring=192.168.231.27  # location of MGM node

# Options for ndbd process:
[MYSQL_CLUSTER]
ndb-connectstring=192.168.231.27  # location of MGM node


/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --user=mysql

/usr/local/mysql/bin/ndbd --initial
```

### SQL API节点
```shell
vim /etc/my.cnf

[MYSQLD]
ndbcluster                      # run NDB engine
character_set_server=utf8
ndb-connectstring=192.168.231.27  # location of MGM node

# Options for ndbd process:
[MYSQL_CLUSTER]
ndb-connectstring=192.168.231.27  # location of MGM node


/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --user=mysql

/usr/local/mysql/bin/mysqld_safe --user=mysql
```

## 检验
### 状态检验
管理节点执行```/usr/local/mysql/bin/ndb_mgm -e show```
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @192.168.231.26  (mysql-5.5.62 ndb-7.2.35, Nodegroup: 0, *)
id=3    @192.168.231.19  (mysql-5.5.62 ndb-7.2.35, Nodegroup: 0)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @192.168.231.27  (mysql-5.5.62 ndb-7.2.35)

[mysqld(API)]   2 node(s)
id=4    @192.168.231.50  (mysql-5.5.62 ndb-7.2.35)
id=5    @192.168.231.30  (mysql-5.5.62 ndb-7.2.35)
表明所有节点正常工作

或 ```/usr/local/mysql/bin/ndb_mgm```进入控制台

查看内存使用
ndb_mgm> all report memoryusage
-- NDB Cluster -- Management Client --
ndb_mgm> all report memoryusage;
Connected to Management Server at: localhost:1186
Node 2: Data usage is 0%(25 32K pages of total 32000)
Node 2: Index usage is 0%(21 8K pages of total 128032)
Node 3: Data usage is 0%(25 32K pages of total 32000)
Node 3: Index usage is 0%(21 8K pages of total 128032)

### 数据检验
在ID4节点：
执行```/usr/local/mysql/bin/mysql```进入mysql
执行```show databases;```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| ndbinfo            |
| performance_schema |
| test               |
+--------------------+
执行```create database testdb;```
执行```use testdb;```
执行```create table t(i int,name varchar(10)) engine=ndbcluster;```
执行```insert into t values(1,'this_sql1');```


在ID5节点：
执行```show databases;```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| ndbinfo            |
| performance_schema |
| test               |
| testdb             |
+--------------------+
执行```use testdb;```
执行``` select * from t;```
+------+-----------+
| i    | name      |
+------+-----------+
|    1 | this_sql1 |
+------+-----------+
1 row in set (0.01 sec)

表明数据在集群正常插入读取

## 管理
关闭集群顺序：SQL节点->数据节点->管理节点 或者使用# ./ndb_mgm -e shutdown命令关闭集群

启动集群顺序: 管理节点->数据节点->SQL节点