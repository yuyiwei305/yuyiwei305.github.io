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
CONFIG_STORAGE_TOPIC:	trade_source_configs
CONNECT_CONNECTOR_CLIENT_CONFIG_OVERRIDE_POLICY:		All	 	
CONNECT_MAX_REQUEST_SIZE:		10485760	 	
CONNECT_PRODUCER_BUFFER_MEMORY:		45554432	 	
CONNECT_TOPIC_CREATION_ENABLE:		true	 	
GROUP_ID:		1	 	
HEAP_OPTS:		-Xmx4G -Xms4G	 	
OFFSET_FLUSH_INTERVAL_MS:		10000	 	
OFFSET_FLUSH_TIMEOUT_MS:		60000	 	
OFFSET_STORAGE_TOPIC:		trade_source_offsets	 	
STATUS_STORAGE_TOPIC:		trade_source_statuses

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
  "name": "source-connector-datetest",  
  "config": {  
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",  
    "database.hostname": "data-sync-db.cluster-xxxxxxxx.ap-northeast-1.rds.amazonaws.com",  
    "database.port": "3306",
    "database.user": "root",
    "database.password": "xxxxxx",
    "database.server.name": "source_datetest",  
    "table.include.list": "data_source6.GridShareBonus_activity2inviteinfo",
    "time.precision.mode": "connect",
    "database.history.kafka.bootstrap.servers": "b-1.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092,b-2.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092,b-3.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092",  
    "database.history.kafka.topic": "schema-changes.datetest",
    "transforms":"unwrap",
    "transforms.unwrap.type":"io.debezium.transforms.UnwrapFromEnvelope",
    "transforms.unwrap.drop.tombstones":"false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "include.schema.changes": "false",
    "max.queue.size": "81290",
    "max.batch.size": "20480",
    "topic.creation.default.cleanup.policy": "delete",
    "topic.creation.default.replication.factor": 3,
    "topic.creation.default.partitions": 1,
    "topic.creation.default.retention.ms": 7776000000,
    "topic.creation.default.compression.type": "zstd",
    "producer.override.compression.type": "zstd",
    "producer.override.buffer.memory": 167772160,
    "producer.override.max.request.size": 314572800,
    "producer.override.acks":"1"
  }
}

创建源同步配置:
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://kafka-connect-base.kafka-connect/connectors/ -d '{"name":"source-connector-datetest","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","tasks.max":"1","database.hostname":"data-sync-db.cluster-xxxxxxxx.ap-northeast-1.rds.amazonaws.com","database.port":"3306","database.user":"root","database.password":"xxxxxx","database.server.name":"source_datetest","table.include.list":"data_source6.GridShareBonus_activity2inviteinfo","time.precision.mode":"connect","database.history.kafka.bootstrap.servers":"b-1.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092,b-2.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092,b-3.data-sync-msk-clu.xxxxxx.c2.kafka.ap-northeast-1.amazonaws.com:9092","database.history.kafka.topic":"schema-changes.datetest","transforms":"unwrap","transforms.unwrap.type":"io.debezium.transforms.UnwrapFromEnvelope","transforms.unwrap.drop.tombstones":"false","key.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter.schemas.enable":"false","value.converter":"org.apache.kafka.connect.json.JsonConverter","value.converter.schemas.enable":"false","include.schema.changes":"false","max.queue.size":"81290","max.batch.size":"20480","topic.creation.default.cleanup.policy":"delete","topic.creation.default.replication.factor":3,"topic.creation.default.partitions":1,"topic.creation.default.retention.ms":7776000000,"topic.creation.default.compression.type":"zstd","producer.override.compression.type":"zstd","producer.override.buffer.memory":167772160,"producer.override.max.request.size":314572800,"producer.override.acks":"1"}}'



查看结果


curl -H "Accept:application/json" http://kafka-connect-base.kafka-connect/connectors

./kafka-topics.sh  --list   --bootstrap-server $KAFKAS

kafka-console-consumer.sh --bootstrap-server $KAFKAS --topic source_connector.data_source.customers --from-beginning


```

### 输出器配置
```
{
    "name": "trade-service-sink",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "tasks.max": "1",
        "topics": "trade_service_source.trade_service.GridShareBonus_binancereward",
        "connection.url": "jdbc:mysql://xxx:3306/trade_service_sink",
        "connection.user": "root",
        "connection.password": "xxx",
        "transforms": "unwrap,dropTopicPrefix,TimestampConverter",
        "transforms.dropTopicPrefix.type": "org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.dropTopicPrefix.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
        "transforms.dropTopicPrefix.replacement": "$3",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
        "transforms.TimestampConverter.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
        "transforms.TimestampConverter.field": "create_time",
        "transforms.TimestampConverter.format": "yyyy-MM-dd HH:mm:ss.SSS",
        "transforms.TimestampConverter.target.type": "string",
        "auto.create": "false",
        "auto.evolve": "false",
        "insert.mode": "upsert",
        "pk.fields": "id",
        "pk.mode": "record_value",
        "table.name.format": "${topic}"
    }
}


curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://kafka-connect-main.kafka-connect/connectors/ -d '{"name":"trade-service-sink","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector","tasks.max":"1","topics":"trade_service_source.trade_service.GridShareBonus_binancereward","connection.url":"jdbc:mysql://xxx:3306/trade_service_sink","connection.user":"root","connection.password":"xxx","transforms":"unwrap,dropTopicPrefix,TimestampConverter","transforms.dropTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter","transforms.dropTopicPrefix.regex":"([^.]+)\\.([^.]+)\\.([^.]+)","transforms.dropTopicPrefix.replacement":"$3","transforms.unwrap.type":"io.debezium.transforms.ExtractNewRecordState","transforms.unwrap.drop.tombstones":"false","transforms.TimestampConverter.type":"org.apache.kafka.connect.transforms.TimestampConverter$Value","transforms.TimestampConverter.field":"create_time","transforms.TimestampConverter.format":"yyyy-MM-dd HH:mm:ss.SSS","transforms.TimestampConverter.target.type":"string","auto.create":"false","auto.evolve":"false","insert.mode":"upsert","pk.fields":"id","pk.mode":"record_value","table.name.format":"${topic}"}}'

curl -H "Accept:application/json" http://kafka-connect-main.kafka-connect/connectors

```
