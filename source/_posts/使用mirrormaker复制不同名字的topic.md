---
title: 使用mirrormaker复制不同名字的topic
date: 2020-12-17 20:19:07
tags: [kafka,mirrormaker]
---
mirrormaker是kafka自带的topic复制工具，可以让两个集群之间的数据互相导入同步。 很多时候，我们需要把其中一个集群的某个topic数据，复制到另一个集群，但使用不同名字的topic。
官方好像并没有提供这样功能，但留了一个message.handler的入口，我们可以通过额外的插件来实现这样的功能。

首先放上这个插件地址： https://github.com/opencore/mirrormaker_topic_rename.git

使用方法：

1. 直接下载编译好的jar文件:
```
wget https://github.com/opencore/mirrormaker_topic_rename/files/2024649/mirrormaker_topic_rename.zip
unzip mirrormaker_topic_rename.zip
```
2. 添加环境变量：
```    
# 由于执行mirrormaker可能需要导出比较大的数据量，所以建议适当加些内存。这个根据需要来调试。
export KAFKA_HEAP_OPTS="-Xmx3G"
# 2.这个必写，很关键
export CLASSPATH=/mirrormaker_topic_rename/target/mmchangetopic-1.0-SNAPSHOT.jar
# 3. 源kafka集群 后面均用其来代替
export SOURCE_KAFKAS=xxx:9092
# 4. 目标kafka集群 后面均用其来代替
export SOURCE_KAFKAS=xxx:9092
```
<!-- more -->

3. 在目标kafka集群 提前创建好topic
```
./kafka-topics.sh  --create --topic test_sink_topic  --bootstrap-server $SINK_KAFKAS --partitions 3 --replication-factor 3 --config retention.ms=86400000

```

4. 执行同步命令
```
kafka-mirror-maker.sh --consumer.config /consumer.properties --producer.config producer.properties --whitelist test_source_topic  --message.handler com.opencore.RenameTopicHandler --message.handler.args 'test_source_topic,test_sink_topic'

```

5. consumer.properties 和 producer.properties

这两个文件需要提前自己创建出来，我发现其实官方好像文档写的也不全。 我这里先贴出来我自己用的。

consumer.properties
```
bootstrap.servers=$SOURCE_KAFKAS(这个地方需要看着改改)
group.id=mirror_maker-group
enable.auto.commit=true
auto.offset.reset=earliest
auto.commit.interval.ms=1000
partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor
```

producer.properties
```
bootstrap.servers=$SOURCE_KAFKAS(这个地方需要看着改改)
acks=1
linger.ms=100
batch.size=16384
retries=3
compression.type=none
```
