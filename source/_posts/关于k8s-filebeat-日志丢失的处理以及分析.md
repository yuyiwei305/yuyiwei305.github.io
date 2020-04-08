---
title: 关于k8s filebeat 日志丢失的处理以及分析
date: 2020-04-08 14:22:43
tags: [kubernetes,filebeat]
---

### 问题分析
最近几天一直在排查k8s集群业务日志丢失的问题。现在业务集群上大概有十几个node跑了几百个pod。平时业务经常会更新。偶尔排查日志的时候发现个别的业务日志会采集不到，
但是重启pod之后又可以采集到了。 这给问题排查造成了很大困扰。由于偶尔出现，所以不好去定位和排查相关的问题。
简单说下目前集群日志采用的架构：基础就是elk，每个node节点通过demeonset方式部署filebeat,filebeat的output设置为logstash，经过logstash简单的处理发送到electicserch，最后通过kibana展示 。
判断的思路： 结合出现问题的现象是个别的pod采集不到数据而不是整体另外重启相关pod后又可以重新采集日志。所以可以初步判断是filebeat的问题。
查询了下filebeat相关资料 了解到一条日志是如何被采集的：
<!-- more -->
---
从代码的实现角度来看，filebeat大概可以分以下几个模块：

input: 找到配置的日志文件，启动harvester
harvester: 读取文件，发送至spooler
spooler: 缓存日志数据，直到可以发送至publisher
publisher: 发送日志至后端，同时通知registrar
registrar: 记录日志文件被采集的状态
---
从上述几个模块也看出整个的采集流程。现在的问题就是定位，到底是这几个步骤的哪一步？



恰好我找到了一个服务日志采集不到的情况，不多说，重启该业务，同时查看filebeat报的什么日志：
```
#相关隐私信息已经改为xxxx
2020-04-07T06:46:51.741Z INFO add_kubernetes_metadata/kubernetes.go:68 add_kubernetes_metadata: kubernetes env detected, with version: v1.14.9-eks-502bfb
2020-04-07T06:46:51.741Z INFO kubernetes/util.go:94 kubernetes: Using pod name filebeat-filebeat-l7h55 and namespace filebeat to discover kubernetes node
2020-04-07T06:46:51.745Z INFO kubernetes/util.go:100 kubernetes: Using node xxx discovered by in cluster pod node query
2020-04-07T06:46:51.846Z INFO log/input.go:152 Configured paths: [/var/lib/docker/containers/xxxxxxxxxx/*.log]
2020-04-07T06:46:51.846Z INFO input/input.go:114 Starting input of type: container; ID: xxxxxxxxxx

```

### 问题处理
通过日志可以清楚的看到，filebeat走的是文件发现这个流程，也就是说是在第一步。接下来的解决思路就很直接,想办法让filebeat文件发现即可。已知重启可以进行发现，那么证明filebeat本身配置没啥问题，而出错的
pod也不是一开始就不能采集日志，那么大概率就是pod的日志文件被删除或者是重置或者是和已有的什么冲突了导致。接下来思路就是看看能否让filebeat自己去排查，比如某个日志不更新了，关闭相关句柄重新发现之类的配置。
果然有相关配置：close_inactive  clean_inactive clean_remove ignore_older 。
所以最终我的配置文件改为：
```
...
filebeat.autodiscover:
      providers:
        - type: kubernetes      
          in_cluster: true    
          hints.enabled: true
          hints.default_config:          
            type: container
            paths:
              - /var/lib/docker/containers/${data.kubernetes.container.id}/*.log
            processors:
              - add_kubernetes_metadata:        
              - add_fields:
                  target: kubernetes
                  fields:
                    name: "dev"
            max_bytes: 102400
            close_inactive: 1m
            ignore_older: 50m
            clean_inactive: 60m
...
```

大概希望的效果就是 一分钟如果没有收到日志就关闭句柄，60分钟之后清理相关文件。 但愿改完之后日志丢失的事儿可以解决吧。 后续具体的参数调优可以慢慢来。
ps: 无论解决还是没有解决后续的进展我再继续补充。

相关资料：  https://juejin.im/post/5d29376ae51d45507022702e
