---
title: Windows 使用 hexo 蹚的一些坑
date: 2020-04-04 10:15:07
tags: [blog,hexo]
---

一般情况下本人写blog会用公司的mac来搞,适逢清明放假所以把环境整理了下,用家里的windows来写。整个过程中还是有一些坑的,特此记录下。

# 1. nodejs 版本问题
由于很少在家coding,家中的nodejs版本过低。导致在安装hexo出现各种问题,网上搜了下资料发现现在的nodejs版本管理工具已经很成熟了，随便找了一个叫做gnvm 还挺好使。
安装很简单不需要复杂的操作,直接下载就能用。
GNVM 是一个简单的 Node.js 多版本管理器，类似 nvm nvmw nodist。很简单直接用了。

### 安装 & 验证
直接下载到nodejs所在目录即可，如果没安装nodejs，自己找一个目录放进去就好。由于我是之前就有所以就直接放进去了

```
curl -L https://github.com/Kenshin/gnvm-bin/blob/master/64-bit/gnvm.exe?raw=true -o gnvm.exe
```

下载其实也没有按照这个方式下载，原因是 ssl 证书问题，我是直接把链接拖到chrome里下载就完事了。在 cmd 下，输入 gnvm version，如有 版本说明 则配置成功。
接下来安装/升级 nodejs

<!-- more -->

查看目前最新的 nodejs版本
```
gnvm node-version
```

安装最新的nodejs版本
```
gnvm install latest
```
查看当前本地的版本
```
gnvm ls
```

此时不出意外就已经装好了最新版本的nodejs，npm也同步更新到了最新版本。
```
node -v
npm -v
```
# 2. windows PowerShell: 因为在此系统中禁止执行脚本 解决办法

安装好hexo 之后发现 hexo命令无法执行 报错提示 ："因为在此系统中禁止执行脚本

查了下资料发现：
首次在计算机上启动 Windows PowerShell 时，现用执行策略很可能是 Restricted（默认设置）。
Restricted 策略不允许任何脚本运行。
若要在本地计算机上运行您编写的未签名脚本和来自其他用户的签名脚本，请使用以下命令将计算机上的，执行策略更改为 RemoteSigned：

```
set-executionpolicy remotesigned
```

选择 Y 然后就可以执行 hexo 命令了。


此时所有的坑蹚完，万事大吉。
