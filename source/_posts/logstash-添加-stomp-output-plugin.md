---
title: logstash 添加 stomp output plugin
date: 2020-04-16 14:59:41
tags: [logstash,plugin]
---
最近由于工作需要，要将logstash采集到的数据，经过过滤发送到mq中，直接用默认的logstash镜像并不能支持。查了下资料，发现有相关的插件，可以支持。在此记录一下。

### 准备阶段
logstash 部署在k8s集群中，之前我们已经通过helm完整的封装，在此我们只需要对于原始镜像去做修改，让它支持stomp output即可。


名称|地址|说明
-|-|-|
logstash基础镜像 | docker.elastic.co/logstash/logstash-oss:7.5.1 |  所用到的基础image |
logstash-output-stomp | github.com/kfogle/logstash-output-stomp.git  |  stomp 插件，这里用了个网友修改过的，原因是官方插件对https支持不够好 |



<!-- more -->

现在我们开始下载对应的资源：

```
#1.下载基础镜像
docker pull docker.elastic.co/logstash/logstash-oss:7.5.1
#2.创建空的工作目录
mkdir workdir
#3.下载插件
cd workdir
git clone https://github.com/kfogle/logstash-output-stomp.git
```
### 制作dockerfile

做好以上准备之后可以动手制作镜像了，现在开始编写dockerfile

```
#在工作目录下创建dockerfile
touch Dockerfile
vi Dockerfile
```
Dockerfile 文件如下：

```
FROM docker.elastic.co/logstash/logstash-oss:7.5.1
COPY ./logstash-output-stomp /usr/share/logstash/logstash-output-stomp
RUN echo "gem \"logstash-output-stomp\", :path => \"/usr/share/logstash/logstash-output-stomp\"" >> /usr/share/logstash/Gemfile
RUN bin/logstash-plugin install --no-verify
```

编写好之后直接docker build 镜像即可：

```
# -t 后面的镜像名字基于项目自己定这里给出个例子
docker build -t logstash/logstash-oss:7.5.1-plugin .
```
到此，新制作的logstash镜像已经支持stomp output 了。

### logstash.conf 示例

这里给出一个 logstash output 例子，看下如何使用:

```
...
output {
        if [kubernetes][namespace] == "logtest" {
          stomp {
              host => "xxxxxxxxxx-1.mq.us-west-2.amazonaws.com"
              port => "61614"
              destination => "/queue/collect_event"
              user => "xxx"
              password => "xxxxxx"
          }
        }
      }
...
```

大致使用方法就是这样。
