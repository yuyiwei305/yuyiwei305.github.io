---
title: 使用Debezium同步mysql数据到kafaka
date: 2020-10-20 20:13:57
tags: [Debezium,kafaka]
---

### 目标

目前已有mysql和kafka，需要利用Debezium将mysql的数据同步到kafka持久化保存，之后再通过jdbc sink导出kafka中的数据至其他mysql

Debezium是一个分布式平台，它可以将你现有的数据库变成事件流，因此应用程序可以看到并立即响应数据库中的每个行级变化。

Debezium建立在Apache Kafka之上，并提供Kafka Connect兼容的连接器，可以监控特定的数据库管理系统。Debezium在Kafka日志中记录数据变化的历史，从你的应用程序消费它们的地方。这使得您的应用程序可以轻松地正确和完整地消费所有的事件。即使你的应用程序意外停止，它也不会错过任何东西：当应用程序重新启动时，它将继续消耗它离开的事件。

### 准备工作

开始前需要准备好需要的资源

1. kafka  集群 （这里使用的是aws msk）
2. MYSQL 集群
3. kafka以及mysql的客户端命令行工具
4. kafka connect

<!-- more -->

### Debezium配置

Debezium 使用容器启动，官方会提供docker镜像，这里使用最新的 debezium/connect:1.3。启动时需要提供相关的环境变量

```
BOOTSTRAP_SERVERS: $KAFKAS
CONFIG_STORAGE_TOPIC:	source_configs
CONNECT_CONNECTOR_CLIENT_CONFIG_OVERRIDE_POLICY:		All	 	
CONNECT_MAX_REQUEST_SIZE:		10485760	 	
CONNECT_PRODUCER_BUFFER_MEMORY:		45554432	 	
CONNECT_TOPIC_CREATION_ENABLE:		true	 	
GROUP_ID:		1	 	
HEAP_OPTS:		-Xmx4G -Xms4G	 	
OFFSET_FLUSH_INTERVAL_MS:		10000	 	
OFFSET_FLUSH_TIMEOUT_MS:		60000	 	
OFFSET_STORAGE_TOPIC:		source_offsets	 	
STATUS_STORAGE_TOPIC:		source_statuses

```
启动完成之后 进入cmd 命令行确认 connect 运行情况，这里将启动之后的容器通k8s service 暴露出来，服务名为 kafka-connect-base.kafka-connect
```
查看版本
curl -H "Accept:application/json"  http://kafka-connect-base.kafka-connect/
{"version":"2.6.0","commit":"62abe01bee039651","kafka_cluster_id":"0lcUFkgrTQOtXcRJyEz8Fw"}

查看当前连接器
curl -H "Accept:application/json" http://kafka-connect-base.kafka-connect/connectors
[]

删除连接器
curl -X DELETE -H "Accept:application/json" http://kafka-connect-base.kafka-connect/connectors/xxx

```


### 源同步配置
```
{
  "name": "source-test",  
  "config": {  
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",  
    "database.hostname": "$MYSQL_HOST",  
    "database.port": "3306",
    "database.user": "xxx",
    "database.password": "xxx",
    "database.server.name": "source-test",  
    "database.include.list": "testdatabase",
    "table.exclude.list": "trade_service.django_admin_log,trade_service.django_content_type",
    "database.history.kafka.bootstrap.servers": "$KAFKAS",  
    "time.precision.mode": "connect",
    "database.history.kafka.topic": "schema-changes.source-test",
    "max.queue.size": "81290",
    "max.batch.size": "20480",
    "snapshot.locking.mode": "none",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://kafka-connect-schema-registry.kafka-connect",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://kafka-connect-schema-registry.kafka-connect",
    "topic.creation.default.cleanup.policy": "delete",
    "topic.creation.default.replication.factor": 3,
    "topic.creation.default.partitions": 1,
    "topic.creation.default.retention.ms": 7776000000,
    "producer.override.compression.type": "zstd",
    "producer.override.buffer.memory": 67108864,
    "producer.override.max.request.size": 62914560,
    "producer.override.acks":"1"
  }
}

创建源同步配置:
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://kafka-connect-base.kafka-connect/connectors/ -d '{"name":"source-test","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","tasks.max":"1","database.hostname":"$MYSQL_HOST","database.port":"3306","database.user":"xxx","database.password":"xxx","database.server.name":"source-test","database.include.list":"testdatabase","table.exclude.list":"trade_service.django_admin_log,trade_service.django_content_type","database.history.kafka.bootstrap.servers":"$KAFKAS","time.precision.mode":"connect","database.history.kafka.topic":"schema-changes.source-test","max.queue.size":"81290","max.batch.size":"20480","snapshot.locking.mode":"none","value.converter":"io.confluent.connect.avro.AvroConverter","value.converter.schema.registry.url":"http://kafka-connect-schema-registry.kafka-connect","key.converter":"io.confluent.connect.avro.AvroConverter","key.converter.schema.registry.url":"http://kafka-connect-schema-registry.kafka-connect","topic.creation.default.cleanup.policy":"delete","topic.creation.default.replication.factor":3,"topic.creation.default.partitions":1,"topic.creation.default.retention.ms":7776000000,"producer.override.compression.type":"zstd","producer.override.buffer.memory":67108864,"producer.override.max.request.size":62914560,"producer.override.acks":"1"}}'



查看结果


curl -H "Accept:application/json" http://kafka-connect-base.kafka-connect/connectors

./kafka-topics.sh  --list   --bootstrap-server $KAFKAS

kafka-console-consumer.sh --bootstrap-server $KAFKAS --topic source_connector.data_source.customers --from-beginning


```

### 输出器配置
```
{
    "name": "sink-test",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "tasks.max": "1",
        "topics.regex": "source-test.testdatabase.(.*)",
        "connection.url": "jdbc:mysql://xxx:3306/sink",
        "connection.user": "XXX",
        "connection.password": "XXXX",
        "transforms": "unwrap,dropTopicPrefix,IdMask",
        "transforms.dropTopicPrefix.type": "org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.dropTopicPrefix.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
        "transforms.dropTopicPrefix.replacement": "$3",
        "transforms.IdMask.type": "org.apache.kafka.connect.transforms.MaskField$Value",
        "transforms.IdMask.fields": "id",
        "transforms.IdMask.replacement": "0",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
        "auto.create": "false",
        "auto.evolve": "false",
        "insert.mode": "upsert",
        "pk.fields": "id",
        "batch.size": "500",
        "consumer.override.fetch.min.bytes": "1500000",
        "consumer.override.max.poll.records": "4000",
        "pk.mode": "record_value",
        "errors.tolerance": "all",
        "table.name.format": "${topic}"
    }
}


curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://kafka-connect-main.kafka-connect/connectors/ -d '{"name":"sink-test","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector","tasks.max":"1","topics.regex":"source-test.testdatabase.(.*)","connection.url":"jdbc:mysql://xxx:3306/sink","connection.user":"XXXX","connection.password":"XXXX","transforms":"unwrap,dropTopicPrefix,IdMask","transforms.dropTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter","transforms.dropTopicPrefix.regex":"([^.]+)\\.([^.]+)\\.([^.]+)","transforms.dropTopicPrefix.replacement":"$3","transforms.IdMask.type":"org.apache.kafka.connect.transforms.MaskField$Value","transforms.IdMask.fields":"id","transforms.IdMask.replacement":"0","transforms.unwrap.type":"io.debezium.transforms.ExtractNewRecordState","transforms.unwrap.drop.tombstones":"false","auto.create":"false","auto.evolve":"false","insert.mode":"upsert","pk.fields":"id","batch.size":"500","consumer.override.fetch.min.bytes":"1500000","consumer.override.max.poll.records":"4000","pk.mode":"record_value","errors.tolerance":"all","table.name.format":"${topic}"}}'

curl -H "Accept:application/json" http://kafka-connect-main.kafka-connect/connectors

```


# 日志记录

单独去每个节点上操作：
```
查看日志
curl http://localhost:8083/admin/loggers

修改日志等级
curl -X PUT -H "Content-Type:application/json" http://localhost:8083/admin/loggers/io.confluent.connect.jdbc -d '{"level": "DEBUG"}'
```

# 不同情况下的思路

a. 如果我们是两个db之间的数据同步，且数据量很大，那么可以考虑增量处理。

1. 源db创建clone， 找到 binlog
2. 从binlog 开始增量同步到kafka
3. sink，开始, 从kafak数据实时同步到 clone db。

b. db全量同步到kafka，且数据量很大，思路也一样
1. 创建clone
2. 从clone 同步数据（全量）
3. 切换到主，从主再同步数据（增量）
