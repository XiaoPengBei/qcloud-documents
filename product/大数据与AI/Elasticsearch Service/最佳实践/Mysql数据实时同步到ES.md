业务需要把mysql的数据实时同步到ES，实现低延迟的检索到ES中的数据或者进行其它数据分析处理。本文给出以同步mysql binlog的方式实时同步数据到ES的思路, 实践并验证该方式的可行性，以供参考。

## mysql binlog日志

mysql的binlog日志主要用于数据库的主从复制与数据恢复。binlog中记录了数据的增删改查操作，主从复制过程中，主库向从库同步binlog日志，从库对binlog日志中的事件进行重放，从而实现主从同步。
mysql binlog日志有三种模式，分别为：

```
ROW: 记录每一行数据被修改的情况，但是日志量太大
STATEMENT: 记录每一条修改数据的SQL语句，减少了日志量，但是SQL语句使用函数或触发器时容易出现主从不一致
MIXED: 结合了ROW和STATEMENT的优点，根据具体执行数据操作的SQL语句选择使用ROW或者STATEMENT记录日志
```
	
要通过mysql binlog将数据同步到ES集群，只能使用ROW模式，因为只有ROW模式才能知道mysql中的数据的修改内容。

以UPDATE操作为例，ROW模式的binlog日志内容示例如下：

```
SET TIMESTAMP=1527917394/*!*/;
BEGIN
/*!*/;
# at 3751
#180602 13:29:54 server id 1  end_log_pos 3819 CRC32 0x8dabdf01 	Table_map: `webservice`.`building` mapped to number 74
# at 3819
#180602 13:29:54 server id 1  end_log_pos 3949 CRC32 0x59a8ed85 	Update_rows: table id 74 flags: STMT_END_F
	
BINLOG '
UisSWxMBAAAARAAAAOsOAAAAAEoAAAAAAAEACndlYnNlcnZpY2UACGJ1aWxkaW5nAAYIDwEPEREG
wACAAQAAAAHfq40=
UisSWx8BAAAAggAAAG0PAAAAAEoAAAAAAAEAAgAG///A1gcAAAAAAAALYnVpbGRpbmctMTAADwB3
UkRNbjNLYlV5d1k3ajVbD64WWw+uFsDWBwAAAAAAAAtidWlsZGluZy0xMAEPAHdSRE1uM0tiVXl3
WTdqNVsPrhZbD64Whe2oWQ==
'/*!*/;
### UPDATE `webservice`.`building`
### WHERE
###   @1=2006 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='building-10' /* VARSTRING(192) meta=192 nullable=0 is_null=0 */
###   @3=0 /* TINYINT meta=0 nullable=0 is_null=0 */
###   @4='wRDMn3KbUywY7j5' /* VARSTRING(384) meta=384 nullable=0 is_null=0 */
###   @5=1527754262 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
###   @6=1527754262 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
### SET
###   @1=2006 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='building-10' /* VARSTRING(192) meta=192 nullable=0 is_null=0 */
###   @3=1 /* TINYINT meta=0 nullable=0 is_null=0 */
###   @4='wRDMn3KbUywY7j5' /* VARSTRING(384) meta=384 nullable=0 is_null=0 */
###   @5=1527754262 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
###   @6=1527754262 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
# at 3949
#180602 13:29:54 server id 1  end_log_pos 3980 CRC32 0x58226b8f 	Xid = 182
COMMIT/*!*/;
```
STATEMENT模式下binlog日志内容示例为：

```
SET TIMESTAMP=1527919329/*!*/;
update building set Status=1 where Id=2000
/*!*/;
# at 688
#180602 14:02:09 server id 1  end_log_pos 719 CRC32 0x4c550a7d 	Xid = 200
COMMIT/*!*/;
```

从ROW模式和STATEMENT模式下UPDATE操作的日志内容可以看出，ROW模式完整地记录了要修改的某行数据更新前的所有字段的值以及更改后所有字段的值，而STATEMENT模式只单单记录了UPDATE操作的SQL语句。我们要将mysql的数据实时同步到ES， 只能选择ROW模式的binlog, 获取并解析binlog日志的数据内容，执行ES document api，将数据同步到ES集群中。

## mysqldump工具
mysqldump是一个对mysql数据库中的数据进行全量导出的一个工具.
mysqldump的使用方式如下：

```
mysqldump -uelastic -p'Elastic_123' --host=172.16.32.5 -F webservice > dump.sql
```
上述命令表示从远程数据库172.16.32.5:3306中导出database:webservice的所有数据，写入到dump.sql文件中，指定-F参数表示在导出数据后重新生成一个新的binlog日志文件以记录后续的所有数据操作。
dump.sql中的文件内容如下：

```
-- MySQL dump 10.13  Distrib 5.6.40, for Linux (x86_64)
--
-- Host: 172.16.32.5    Database: webservice
-- ------------------------------------------------------
-- Server version	5.5.5-10.1.9-MariaDBV1.0R012D002-20171127-1822

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `building`
--

DROP TABLE IF EXISTS `building`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `building` (
  `Id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `BuildingId` varchar(64) NOT NULL COMMENT '虚拟建筑Id',
  `Status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '虚拟建筑状态：0、处理中；1、正常；-1，停止；-2，销毁中；-3，已销毁',
  `BuildingName` varchar(128) NOT NULL DEFAULT '' COMMENT '虚拟建筑名称',
  `CreateTime` timestamp NOT NULL DEFAULT '2017-12-03 16:00:00' COMMENT '创建时间',
  `UpdateTime` timestamp NOT NULL DEFAULT '2017-12-03 16:00:00' COMMENT '更新时间',
  PRIMARY KEY (`Id`),
  UNIQUE KEY `BuildingId` (`BuildingId`)
) ENGINE=InnoDB AUTO_INCREMENT=2010 DEFAULT CHARSET=utf8 COMMENT='虚拟建筑表';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `building`
--

LOCK TABLES `building` WRITE;
/*!40000 ALTER TABLE `building` DISABLE KEYS */;
INSERT INTO `building` VALUES (2000,'building-2',0,'6YFcmntKrNBIeTA','2018-05-30 13:28:31','2018-05-30 13:28:31'),(2001,'building-4',0,'4rY8PcVUZB1vtrL','2018-05-30 13:28:34','2018-05-30 13:28:34'),(2002,'building-5',0,'uyjHVUYrg9KeGqi','2018-05-30 13:28:37','2018-05-30 13:28:37'),(2003,'building-7',0,'DNhyEBO4XEkXpgW','2018-05-30 13:28:40','2018-05-30 13:28:40'),(2004,'building-1',0,'TmtYX6ZC0RNB4Re','2018-05-30 13:28:43','2018-05-30 13:28:43'),(2005,'building-6',0,'t8YQcjeXefWpcyU','2018-05-30 13:28:49','2018-05-30 13:28:49'),(2006,'building-10',0,'WozgBc2IchNyKyE','2018-05-30 13:28:55','2018-05-30 13:28:55'),(2007,'building-3',0,'yJk27cmLOVQLHf1','2018-05-30 13:28:58','2018-05-30 13:28:58'),(2008,'building-9',0,'RSbjotAh8tymfxs','2018-05-30 13:29:04','2018-05-30 13:29:04'),(2009,'building-8',0,'IBOMlhaXV6k226m','2018-05-30 13:29:31','2018-05-30 13:29:31');
/*!40000 ALTER TABLE `building` ENABLE KEYS */;
UNLOCK TABLES;

/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2018-06-02 14:23:51
```
从以上内容可以看出，mysqldump导出的sql文件包含create table, drop table以及插入数据的sql语句，但是不包含create database建库语句。

## 使用go-mysql-elasticsearch开源工具同步数据到ES

go-mysql-elasticsearch是用于同步mysql数据到ES集群的一个开源工具，项目github地址：
[https://github.com/siddontang/go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch)

go-mysql-elasticsearch的基本原理是：如果是第一次启动该程序，首先使用mysqldump工具对源mysql数据库进行一次全量同步，通过elasticsearch client执行操作写入数据到ES；然后实现了一个mysql client,作为slave连接到源mysql,源mysql作为master会将所有数据的更新操作通过binlog event同步给slave， 通过解析binlog event就可以获取到数据的更新内容，之后写入到ES.

另外，该工具还提供了操作统计的功能，每当有数据增删改操作时，会将对应操作的计数加1，程序启动时会开启一个http服务，通过调用http接口可以查看增删改操作的次数。

### 使用限制：

1. mysql binlog必须是ROW模式
2. 要同步的mysql数据表必须包含主键，否则直接忽略，这是因为如果数据表没有主键，UPDATE和DELETE操作就会因为在ES中找不到对应的document而无法进行同步
3. 不支持程序运行过程中修改表结构
4. 要赋予用于连接mysql的账户RELOAD权限以及REPLICATION权限, SUPER权限：
   GRANT REPLICATION SLAVE ON *.* TO 'elastic'@'172.16.32.44';
   GRANT RELOAD ON *.* TO 'elastic'@'172.16.32.44';
   UPDATE mysql.user SET Super_Priv='Y' WHERE user='elastic' AND host='172.16.32.44';
		   
### 使用方式

1. git clone https://github.com/siddontang/go-mysql-elasticsearch
2. cd go-mysql-elasticsearch/src/github.com/siddontang/go-mysql-elasticsearch
3. vi etc/river.toml, 修改配置文件，同步172.16.0.101:3306数据库中的webservice.building表到ES集群172.16.32.64:9200的building index(更详细的配置文件说明可以参考项目文档)
		
	```
	# MySQL address, user and password
	# user must have replication privilege in MySQL.
	my_addr = "172.16.0.101:3306"
	my_user = "bellen"
	my_pass = "Elastic_123"
	my_charset = "utf8"
	
	# Set true when elasticsearch use https
	#es_https = false
	# Elasticsearch address
	es_addr = "172.16.32.64:9200"
	# Elasticsearch user and password, maybe set by shield, nginx, or x-pack
	es_user = ""
	es_pass = ""
	
	# Path to store data, like master.info, if not set or empty,
	# we must use this to support breakpoint resume syncing.
	# TODO: support other storage, like etcd.
	data_dir = "./var"
	
	# Inner Http status address
	stat_addr = "127.0.0.1:12800"
	
	# pseudo server id like a slave
	server_id = 1001
	
	# mysql or mariadb
	flavor = "mariadb"
	
	# mysqldump execution path
	# if not set or empty, ignore mysqldump.
	mysqldump = "mysqldump"
	
	# if we have no privilege to use mysqldump with --master-data,
	# we must skip it.
	#skip_master_data = false
	
	# minimal items to be inserted in one bulk
	bulk_size = 128
	
	# force flush the pending requests if we don't have enough items >= bulk_size
	flush_bulk_time = "200ms"
	
	# Ignore table without primary key
	skip_no_pk_table = false
	
	# MySQL data source
	[[source]]
	schema = "webservice"
	tables = ["building"]
	[[rule]]
	schema = "webservice"
	table = "building"
	index = "building"
	type = "buildingtype"
	```

4. 在ES集群中创建building index, 因为该工具并没有使用ES的auto create index功能，如果index不存在会报错 
5. 执行命令：./bin/go-mysql-elasticsearch -config=./etc/river.toml
6. 控制台输出结果：
	
	```
	2018/06/02 16:13:21 INFO  create BinlogSyncer with config {1001 mariadb 172.16.0.101 3306 bellen   utf8 false false <nil> false false 0 0s 0s 0}
	2018/06/02 16:13:21 INFO  run status http server 127.0.0.1:12800
	2018/06/02 16:13:21 INFO  skip dump, use last binlog replication pos (mysql-bin.000001, 120) or GTID %!s(<nil>)
	2018/06/02 16:13:21 INFO  begin to sync binlog from position (mysql-bin.000001, 120)
	2018/06/02 16:13:21 INFO  register slave for master server 172.16.0.101:3306
	2018/06/02 16:13:21 INFO  start sync binlog at binlog file (mysql-bin.000001, 120)
	2018/06/02 16:13:21 INFO  rotate to (mysql-bin.000001, 120)
	2018/06/02 16:13:21 INFO  rotate binlog to (mysql-bin.000001, 120)
	2018/06/02 16:13:21 INFO  save position (mysql-bin.000001, 120)
	```
7. 测试：向mysql中插入、修改、删除数据，都可以反映到ES中

### 使用体验
* go-mysql-elasticsearch完成了最基本的mysql实时同步数据到ES的功能，业务如果需要更深层次的功能如允许运行中修改mysql表结构，可以进行自行定制化开发。
* 异常处理不足，解析binlog event失败直接抛出异常
* 据作者描述，该项目并没有被其应用于生产环境中，所以使用过程中建议通读源码，知其利弊。

## 使用mypipe同步数据到ES集群
mypipe是一个mysql binlog同步工具，在设计之初是为了能够将binlog event发送到kafka, 当前版本可根据业务的需要也可以自定以将数据同步到任意的存储介质，项目github地址 [https://github.com/mardambey/mypipe](https://github.com/mardambey/mypipe).

### 使用限制

1. mysql binlog必须是ROW模式
2. 要赋予用于连接mysql的账户REPLICATION权限
   GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'elastic'@'%' IDENTIFIED BY 'Elastic_123'
3. mypipe只是将binlog日志内容解析后编码成Avro格式推送到kafka broker, 并不是将数据推送到kafka，如果需要同步到ES集群，可以从kafka消费数据后，再写入ES
4. 消费kafka中的消息(mysql insert, update, delete操作及具体的数据)，需要对消息内容进行Avro解析，获取到对应的数据操作内容，进行下一步处理；mypipe封装了一个KafkaGenericMutationAvroConsumer类，可以直接继承该类使用，或者自行解析
5. mypipe只支持binlog同步，不支持存量数据同步，也即mypipe程序启动后无法对mysql中已经存在的数据进行同步
		
### 使用方式

1. git clone https://github.com/mardambey/mypipe.git
2. ./sbt package
3. 配置mypipe-runner/src/main/resources/application.conf

	```
	mypipe {
	
	  # Avro schema repository client class name
	  schema-repo-client = "mypipe.avro.schema.SchemaRepo"
	
	  # consumers represent sources for mysql binary logs
	  consumers {
	
	    localhost {
	      # database "host:port:user:pass" array
	      source = "172.16.0.101:3306:elastic:Elastic_123"
	    }
	  }
	
	  # data producers export data out (stdout, other stores, external services, etc.)
	  producers {
	
	    kafka-generic {
	      class = "mypipe.kafka.producer.KafkaMutationGenericAvroProducer"
	    }
	  }
	
	  # pipes join consumers and producers
	  pipes {
	
	    kafka-generic {
	      enabled = true
	      consumers = ["localhost"]
	      producer {
	        kafka-generic {
	          metadata-brokers = "172.16.16.22:9092"
	        }
	      }
	      binlog-position-repo {
	        # saved to a file, this is the default if unspecified
	        class = "mypipe.api.repo.ConfigurableFileBasedBinaryLogPositionRepository"
	        config {
	          file-prefix = "stdout-00"     # required if binlog-position-repo is specifiec
	          data-dir = "/tmp/mypipe/data" # defaults to mypipe.data-dir if not present
	        }
	      }
	    }
	  }
	}
	``` 

4. 配置mypipe-api/src/main/resources/reference.conf，修改include-event-condition选项，指定需要同步的database及table
	
	```
	include-event-condition = """ db == "webservice" && table =="building" """
	```
5. 在kafka broker端创建topic: webservice_building_generic, 默认情况下mypipe以"${db}_${table}_generic"为topic名，向该topic发送数据
	
6. 执行：./sbt "project runner" "runMain mypipe.runner.PipeRunner"
7. 测试：向mysql building表中插入数据，写一个简单的consumer消费mypipe推送到kafka中的消息
8. 消费到没有经过解析的数据如下：

	```
	ConsumerRecord(topic=u'webservice_building_generic', partition=0, offset=2, timestamp=None, timestamp_type=None, key=None, value='\x00\x01\x00\x00\x14webservice\x10building\xcc\x01\x02\x91,\xae\xa3fc\x11\xe8\xa1\xaaRT\x00Z\xf9\xab\x00\x00\x04\x18BuildingName\x06xxx\x14BuildingId\nId-10\x00\x02\x04Id\xd4%\x00', checksum=128384379, serialized_key_size=-1, serialized_value_size=88)
	```

### 使用体验

* mypipe相比go-mysql-elasticsearch更成熟，支持运行时ALTER TABLE，同时解析binlog异常发生时，可通过配置不同的策略处理异常
* mypipe不能同步存量数据，如果需要同步存量数据可通过其它方式先全量同步后，再使用mypipe进行增量同步
* mypipe只同步binlog， 需要同步数据到ES需要另行开发