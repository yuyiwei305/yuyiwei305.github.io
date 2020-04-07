---
title: Mac OS 解决时间不同步问题
date: 2020-04-07 21:34:12
tags: [mac]
---

清明假期回来发现工作的mac电脑时间差了个几十秒，稍微影响了工作，在此记录下如何解决该问题：

确认mac当前版本为: 10.15.2

简单看了下mac当前管理时间的命令是sntp,所以打开终端直接输入命令:

```
sudo sntp -sS pool.ntp.org
```
<!-- more -->

发现提示报错:

```
kod_init_kod_db(): Cannot open KoD db file /var/db/ntp-kod: No such file or directory
```

那么需要执行：
```
sudo touch /var/db/ntp-kod
sudo chmod 666 /var/db/ntp-kod
#再执行这句
sudo sntp -sS pool.ntp.org
```
至此 问题解决完毕。

参考链接: https://apple.stackexchange.com/questions/117864/how-can-i-tell-if-my-mac-is-keeping-the-clock-updated-properly
