---
title: kafka cmd 常用命令
date: 2020-05-27 20:41:10
tags:
---

整理了一些，以供自己日常使用：


0. 前提变量
```
export ZOOKEEPERS=zookeeper:2181 #自己根据需要改,新版本基本上用不到这个
export KAFKAS=kafka:9092 #自己根据需要改
```

1. 列出所有Topic:
```
kafka-topics.sh  --describe --bootstrap-server $KAFKAS
```
2. 创建一个topic
```
kafka-topics.sh  --create --topic test --bootstrap-server $KAFKAS --partitions 3 --replication-factor 2
```
3. 消费者订阅主题
```
kafka-console-consumer.sh --bootstrap-server $KAFKAS --topic test
```
4. 生产者向topic发送消息
```
kafka-console-producer.sh --broker-list $KAFKAS  --topic test
```


5. 压力测试
```
#生产者
kafka-run-class.sh org.apache.kafka.tools.ProducerPerformance --print-metrics --topic test --num-records 6000000 --throughput 100000 --record-size 100 --producer-props bootstrap.servers=$KAFKAS buffer.memory=67108864 batch.size=32768 linger.ms=20 acks=1

#消费者
kafka-consumer-perf-test.sh --broker-list $KAFKAS --messages 1000000 --threads 1 --topic test --print-metrics
```


6. 下载kafka安装包
```
#image: openjdk:8-slim-buster
apt-get update && apt-get install -y wget curl net-tools vim procps
wget https://archive.apache.org/dist/kafka/2.4.1/kafka_2.12-2.4.1.tgz  && tar -xzf kafka_2.12-2.4.1.tgz 
```
