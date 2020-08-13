---
title: kafka添加broker重新分配partition
date: 2020-08-13 17:25:07
tags: ["kafka"]
---
当kafka添加broker时，我们需要重新分配partition，使其流量平衡，这里整理出相关的命令以供日常使用。
说明： 这里以topic test和test1为例。

0. 前提变量
   ```
   export ZOOKEEPERS=zookeeper:2181 #自己根据需要改,新版本基本上用不到这个
   export KAFKAS=kafka:9092 #自己根据需要改
   ```

1. 修改topic的partitions
   ```
   ./kafka-topics.sh --zookeeper $ZOOKEEPERS --alter --topic test --partitions 6
   ./kafka-topics.sh --zookeeper $ZOOKEEPERS --alter --topic test1 --partitions 6
   ```

2. 数据迁移--生成迁移计划json文件

   手动生成一个json文件topic.json
   ```
   {
     "topics": [
        {"topic": "test"},
        {"topic": "test1"}
    ],
    "version": 1
  }
  ```

  使用 --generate生成迁移计划，将test和test1的partition均匀扩充到所有机器上

  ```
  ./kafka-reassign-partitions.sh  --zookeeper $ZOOKEEPERS --topics-to-move-json-file topic.json  --broker-list  "1,2,3,4,5,6" --generate
  ```

  生成的结果类似于这样:
  ```
  Current partition replica assignment

  {
    "version":1,
    "partitions":[.....]
  }
  Proposed partition reassignment configuration

  {
    "version":1,
    "partitions":[.....]
  }

  ```

  我们将 "Proposed partition reassignment configuration" 后面的json复制下来生成reassignment.json 这个文件
  ```
  #reassignment.json

  {
    "version":1,
    "partitions":[.....]
  }

  ```

3. 数据迁移--执行扩容

   利用上一步生成的reassignment.json执行扩容命令：
   ```
   ./kafka-reassign-partitions.sh --zookeeper $ZOOKEEPERS --reassignment-json-file reassignment.json --execute

   ```

4. 验证结果

   ```
   ./kafka-reassign-partitions.sh --zookeeper $ZOOKEEPERS --reassignment-json-file reassignment.json --verify

   ```

   大概长这个样子:

   ```
   ...

   Reassignment of partition test-x completed successfully
   Reassignment of partition test1-x completed successfully
   ...

   ```
