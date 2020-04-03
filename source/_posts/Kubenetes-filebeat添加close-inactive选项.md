---
title: Kubenetes filebeat添加close_inactive选项
date: 2020-04-03 15:10:04
tags: [k8s,filebeat]
---
最近要修改k8s集群上filebeat的配置文件，其中很重要的一项就是配置 close-inactive 的时间。 默认时间是5min，我们觉得时间有点长了，决定改为1min钟。我们部署filebeat是已经用helm封装好了，所以修改配置文件很简单，只需要改 values 即可。
原始的 values 文件内容如下:


```
...
filebeatConfig:
  filebeat.yml: |-      
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
            max_bytes: 102400                                   

    output.logstash:
      hosts: ['logstash-logstash.elk-system.svc.cluster.local:5044']
...
```

<!-- more -->

添加完:
```
...
filebeatConfig:
  filebeat.yml: |-      
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
            max_bytes: 102400
            close_inactive: 1m                                   

    output.logstash:
      hosts: ['logstash.elk-system.svc.cluster.local:5044']
...
```

之后helm 重新部署 filebeat 即可。

坑点需要注意的其实就是  close_inactive: 1m  这个字段的位置，别写错就行。官方文档没有特别针对这块有demo。特此记录一下。
