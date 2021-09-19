# [RocketMQ 使用](https://rocketmq.apache.org/)

> 当前时间官方最新版本 4.8.0 但是始终启动不成功，所以选择了 4.7.1 版本。
> 使用的 jdk 版本是 jdk8，因为官方默认是不支持 jdk11 的，如果要使用的话现需要修改一些 jvm 启动参数。

## linux

### 安装

官网下载的二进制版本（zip），解压即可。

### 修改 jvm 内存分配（如果需要的话）

1. vim bin/runbroker.sh

2. vim bin/runserver.sh

> 尤其是 runserver.sh，默认分配的内存非常大，很可能导致启动失败。

### 启动 mqnamesrv 和 mqbroker（默认 localhost）

> nohup sh bin/mqnamesrv & tail -f ~/logs/rocketmqlogs/namesrv.log
> nohup sh bin/mqbroker -n localhost:9876 & tail -f ~/logs/rocketmqlogs/broker.log

### 启动 mqnamesrv 和 mqbroker（指定 ip）

> nohup sh bin/mqnamesrv -n 192.168.220.130:9876 & tail -f ~/logs/rocketmqlogs/namesrv.log
>
> nohup sh bin/mqbroker -n 192.168.220.130:9876 & tail -f ~/logs/rocketmqlogs/broker.log

### 如果需要在一台电脑上连接另外一台服务器上部署的 RocketMQ，则需要做以下调整

1. 修改 conf 文件夹下 broker.conf 的配置，至少添加以下两行配置：

   > namesrvAddr = 192.168.220.130:9876
   >
   > brokerIP1 = 192.168.220.130

2. 启动 broker 时指定 IP 以及 配置文件：

   > nohup sh bin/mqnamesrv -n 192.168.220.130:9876 & tail -f ~/logs/rocketmqlogs/namesrv.log
   >
   > nohup sh bin/mqbroker -n 192.168.220.130:9876 autoCreateTopicEnable=true -c conf/broker.conf & tail -f ~/logs/rocketmqlogs/broker.log

### 停止 mqnamesrv 和 mqbroker

> sh bin/mqshutdown broker
>
> sh bin/mqshutdown namesrv

### 测试发送消息

> export NAMESRV_ADDR=localhost:9876
>
> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer

### 测试接收消息（打开另一个窗口）

> export NAMESRV_ADDR=localhost:9876
>
> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer

---

## windows

### 安装

官网下载的二进制版本（zip），解压即可（与 linux 相同）。

### 修改 jvm 内存分配（如果需要的话）

1. runserver.cmd

2. runbroker.cmd

> 尤其是 runbroker.cmd，默认分配的内存非常大，很可能导致启动失败。

### 设置环境变量

1. 设置系统环境变量：

   > ROCKETMQ_HOME="D:\programs\java\RocketMQ\downloads\rocketmq"
   >
   > NAMESRV_ADDR="localhost:9876"

2. 或者在 PowerShell 中设置临时环境变量：

   > $Env:ROCKETMQ_HOME="D:\programs\java\RocketMQ\downloads\rocketmq"
   >
   > $Env:NAMESRV_ADDR="localhost:9876"

3. 在 PowerShell 中查看环境变量：

   - 查看所有环境变量：Get-ChildItem env:
   - 查看某一环境变量具体值（例如 JAVA_HOME）：$Env:JAVA_HOME

### 启动 mqnamesrv 和 mqbroker（默认 localhost，在 powershell 中）

> .\bin\runserver.cmd
>
> .\bin\mqbroker.cmd -n localhost:9876 autoCreateTopicEnable=true

### 停止 mqnamesrv 和 mqbroker（在 powershell 中）

> 直接关闭窗口或者 Ctrl+C

### 测试发送消息（在 powershell 中）

> $Env:NAMESRV_ADDR="localhost:9876"
>
> .\bin\tools.cmd org.apache.rocketmq.example.quickstart.Producer

### 测试接收消息（在 powershell 中）

> $Env:NAMESRV_ADDR="localhost:9876"
>
> .\bin\tools.cmd org.apache.rocketmq.example.quickstart.Consumer

## 查看 java 进程

> 可以使用 jps 命令来查看正在运行的 java 进程
