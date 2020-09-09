---
title: 用swoole TCP Server 和mongodb做数据备份和恢复
date: 2016-05-18 18:52:40
tags: [mongo,php-mongo,swoole,tcp]
---

应用场景是，单个用户写的数据是8-10条/S JSON数据包

采用架构如下图所示：
![架构图](Swoole-Mongo.png)

采用的是

- php7.0.6
- swoole-1.8.5版本
- php扩展mongodb1.1.6  而目前mongo-1.6.14不支持php7 这样导致连接Mongo函数不一样，后面会讲到。
- 服务器 Ubuntu 14.04 64位 2核 4G
- 阿里云的Mongodb服务器

到目前阿里云Mongodb还不支持sharding,后续会进行支持。目前是使用双节点。摘录[MongoDB杭州用户交流会](https://yq.aliyun.com/articles/7557)的一段话:

>阿里云目前已提供对MongoDB复制集（Replica Set）的支持，默认会为用户创建包含3个数据节点的复制集，其中一个Primary、一个Secondary，以及一个Hidden节点。Primary、Secondary对用户可见，用户可以自定义ReadPreference，Hidden节点对用户不可见，目前主要用于实例数据备份以及自动的failover，当有Primary或Secondary节点挂掉时，Hidden会被自动切换为Secondary，保证用户的服务不受影响。

### mongodb服务器
阿里云服务器目前只支持ECS访问不支持外网方面

- [公网连接mongodb windows篇](https://help.aliyun.com/knowledge_detail/13052608.html#通过公网连接云数据库MongoDB--ECS Windows篇)

- [公网连接mongodb linux篇](https://help.aliyun.com/knowledge_detail/13052572.html#通过公网连接云数据库MongoDB--ECS Linux篇)

用如下命令检测是否连上mongodb

- ping  dds-xxxxxxxx.mongodb.rds.aliyuncs.com
- telnet  dds-xxxxxxxxxxxx.mongodb.rds.aliyuncs.com 3717
- mongo --host dds-xxxxxxxxxxxxx1.mongodb.rds.aliyuncs.com:3717 --authenticationDatabase admin -u root -p
- mongo --host dds-xxxxxxxxxxxxxx2.mongodb.rds.aliyuncs.com:3717 --authenticationDatabase admin -u root -p

如果本身server服务器用的ECS，还是要设置一下，

### 搭建TCP-SEVER服务器

1. 买好服务器之后习惯性的把服务器服务升级到最新。（这个是强迫症）

```
$ apt-get update 

$ apt-get upgrade
```

2. 安装最新的php7.0.6环境

```powershell
apt-get install python-software-properties

apt-get install software-properties-common

LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php

apt-get update

apt-cache show php7.0-dev  



Package: php7.0-dev
Source: php7.0
Priority: optional
Section: php
Installed-Size: 4658
Maintainer: Debian PHP Maintainers <pkg-php-maint@lists.alioth.debian.org>
Architecture: amd64
Version: 7.0.6-12+donate.sury.org~trusty+1
Recommends: dh-php, pkg-php-tools
Depends: autoconf (>= 2.63), automake (>= 1.11), libpcre3-dev, libssl-dev, php7.0-cli (>= 7.0.6-12+donate.sury.org~trusty+1), php7.0-common (= 7.0.6-12+donate.sury.org~trusty+1), shtool, libtool
Filename: pool/main/p/php7.0/php7.0-dev_7.0.6-12+donate.sury.org~trusty+1_amd64.deb
Size: 505784
MD5sum: bb75fadf2fc0bae22d7a26999b9ac0e2
SHA1: a85e13a259faae1377813d7b1e5c344c8ede8d09
SHA256: 79617d3b79e054129a2e980534885ab794ec18c5052894e71d1da37ec07b2aff
Description-en: Files for PHP7.0 module development
 This package provides the files from the PHP7.0 source needed for compiling
 additional modules.

PHP (recursive acronym for PHP: Hypertext Preprocessor) is a widely-used
 open source general-purpose scripting language that is especially suited
 for web development and can be embedded into HTML.
Description-md5: cab4eaaf141b1f52bc2814eea2492ab2

apt-get install php7.0-dev
```

3. 安装mongo扩展 和swoole扩展

```
使用pecl进行安装mongo和swoole扩展。
这里记住是mongodb扩展不是mongo扩展。这两个扩展在使用上面不一样，后面会讲到。这里我们安装
mongodb

搜索mongodb
$ pecl search mongo


pecl库更新到最新
$ pecl channel-update pecl.php.net

安装mongodb
$ pecl install mongodb

问题：
1. 这里如果安装mongo会提示不支持Php7.0以上版本：
	pecl/mongo requires PHP (version >= 5.3.0, version <= 5.99.99),
	installed version is 7.0.6-12+donate.sury.org~trusty+1
2. configure: error: Cannot find OpenSSL's libraries
	 解决方法：  ln -s /usr/lib/x86_64-linux-gnu/libssl.so  /usr/lib


安装swoole
$ pecl install swoole

查找php.ini 把swoole.so和mongodb加入进去
$ php --ini  

$ php -ri|grep swoole

```

4. PHP swoole代码

```php
$serv = new swoole_server("0.0.0.0", 9501);

$serv->set(array(
	'worker_num' => 8, //工作进程数量
	'daemonize' => false, //是否作为守护进程
));

$serv->on('connect', function ($serv, $fd) {
	echo "Client:Connect.\n";
});

$serv->on('receive', function ($serv,$fd, $from_id, $data) {
	
	$arr = json_decode($data);
	
	//$array=array('column_name'=>'col'.rand(100,999),'column_exp'=>'xiaocai');
	$connection = new MongoClient("mongodb://root:密码@主机ID1.mongodb.rds.aliyuncs.com:3717,主机2.mongodb.rds.aliyuncs.com:3717/admin?replicaSet=副本集名称");
	//	$connection = new MongoClient("mongodb://localhost:27017");
	var_dump($connection);
	$roomid = "roomid".'_123';
	$collection = $connection->test->$roomid;
	$collection->insert($arr);
	
	$document = $collection->findOne();
	var_dump($document);
	
	$serv->send($fd, 'Swoole: ' . $data);
	$serv->close($fd);
});

$serv->on('close', function ($serv, $fd) {
	echo "Client: Close.\n";
});

$serv->start();

```
如果执行上面的代码的话，会报`PHP Fatal error:  Uncaught Error: Class 'MongoClient' not found`
这里要说明一下MongoClient是扩展Mongo的内置函数。
可以通过

```php
print_r(get_declared_classes());
```
查看函数列表
Mongo的函数列表

```
    [156] => MongoClient
    [157] => Mongo
    [158] => MongoDB
    [159] => MongoCollection
    [160] => MongoCursor
    [161] => MongoCommandCursor
    [162] => MongoGridFS
    [163] => MongoGridFSFile
    [164] => MongoGridFSCursor
    [165] => MongoWriteBatch
    [166] => MongoInsertBatch
    [167] => MongoUpdateBatch
    [168] => MongoDeleteBatch
    [169] => MongoId
    [170] => MongoCode
    [171] => MongoRegex
    [172] => MongoDate
    [173] => MongoBinData
    [174] => MongoDBRef
    [175] => MongoException
    [176] => MongoConnectionException
    [177] => MongoCursorException
    [178] => MongoCursorTimeoutException
    [179] => MongoGridFSException
    [180] => MongoResultException
    [181] => MongoWriteConcernException
    [182] => MongoDuplicateKeyException
    [183] => MongoExecutionTimeoutException
    [184] => MongoProtocolException
    [185] => MongoTimestamp
    [186] => MongoInt32
    [187] => MongoInt64
    [188] => MongoLog
    [189] => MongoPool
    [190] => MongoMaxKey
    [191] => MongoMinKey
```

Mongodb的函数列表

```
	[100] => MongoDB\Driver\Command
    [101] => MongoDB\Driver\Cursor
    [102] => MongoDB\Driver\CursorId
    [103] => MongoDB\Driver\Manager
    [104] => MongoDB\Driver\Query
    [105] => MongoDB\Driver\ReadConcern
    [106] => MongoDB\Driver\ReadPreference
    [107] => MongoDB\Driver\Server
    [108] => MongoDB\Driver\BulkWrite
    [109] => MongoDB\Driver\WriteConcern
    [110] => MongoDB\Driver\WriteConcernError
    [111] => MongoDB\Driver\WriteError
    [112] => MongoDB\Driver\WriteResult
    [113] => MongoDB\Driver\Exception\LogicException
    [114] => MongoDB\Driver\Exception\RuntimeException
    [115] => MongoDB\Driver\Exception\UnexpectedValueException
    [116] => MongoDB\Driver\Exception\InvalidArgumentException
    [117] => MongoDB\Driver\Exception\ConnectionException
    [118] => MongoDB\Driver\Exception\AuthenticationException
    [119] => MongoDB\Driver\Exception\SSLConnectionException
    [120] => MongoDB\Driver\Exception\WriteException
    [121] => MongoDB\Driver\Exception\BulkWriteException
    [122] => MongoDB\Driver\Exception\ExecutionTimeoutException
    [123] => MongoDB\Driver\Exception\ConnectionTimeoutException
    [124] => MongoDB\BSON\Binary
    [125] => MongoDB\BSON\Javascript
    [126] => MongoDB\BSON\MaxKey
    [127] => MongoDB\BSON\MinKey
    [128] => MongoDB\BSON\ObjectID
    [129] => MongoDB\BSON\Regex
    [130] => MongoDB\BSON\Timestamp
    [131] => MongoDB\BSON\UTCDateTime
```
所以要使用Php7.0以上版本需要重新写连接mongdo的数据库
查找php文档[Mongodb driver](http://php.net/manual/zh/set.mongodb.php)

修改成为mongodb的一个demo	连接修改代码如下：

```php
$serv = new swoole_server("0.0.0.0", 9501);

$serv->set(array(
	'worker_num' => 8, //工作进程数量
	'daemonize' => false, //是否作为守护进程
));

$serv->on('connect', function ($serv, $fd) {
	echo "Client:Connect.\n";
});

$serv->on('receive', function ($serv, $fd, $from_id, $data) {
	
	$arr = json_decode($data);
	
	$bulk = new MongoDB\Driver\BulkWrite(['ordered' => true]);
	
	$bulk->insert($arr);
	
	$connection = new MongoDB\Driver\Manager("mongodb://localhost:27017");
	
	try {
	
		$result = $connection->executeBulkWrite('test.roomid', $bulk);
	
	} catch (MongoDB\Driver\Exception\BulkWriteException $e) {
		
			$result = $e->getWriteResult();
	
			if ($writeConcernError = $result->getWriteConcernError()) {
				printf("%s (%d): %s\n",	$writeConcernError->getMessage(),
				$writeConcernError->getCode(),
				var_export($writeConcernError->getInfo(), true));
			}
			// Check if any write operations did not complete at all
			foreach ($result->getWriteErrors() as $writeError) {
				printf("Operation#%d: %s (%d)\n",
				$writeError->getIndex(),
				$writeError->getMessage(),
				$writeError->getCode()
			);
		}
	} catch (MongoDB\Driver\Exception\Exception $e) {
	printf("Other error: %s\n", $e->getMessage());
	exit;
}
	
	printf("Inserted %d document(s)\n",$result->getInsertedCount());
	printf("Updated  %d document(s)\n",$result->getModifiedCount());
	
	$serv->send($fd, 'Swoole: ' . $data);
	
	$serv->close($fd);
});

$serv->on('close', function ($serv, $fd) {

	echo "Client: Close.\n";
	
});

$serv->start(); 
```
模拟请求数据

```
netcat 127.0.0.1 9501
```

END

