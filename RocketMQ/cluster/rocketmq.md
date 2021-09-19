# [RocketMQ 使用](https://rocketmq.apache.org/)

## 集群搭建

### 两台虚拟机

> 192.168.220.138（master1 和 slave2）
>
> 192.168.220.139（master2 和 slave1）

1. 修改 hosts 文件（在两台虚拟机的 hosts 文件中都要添加以下配置的全部）

   ```shell
      sudo vim /etc/hosts
   ```

   ```hosts
      # nameServer:
      192.168.220.138    rocketmq-nameserver1
      192.168.220.139    rocketmq-nameserver2

      # broker
      192.168.220.138    rocketmq-master1
      192.168.220.138    rocketmq-slave2
      192.168.220.139    rocketmq-master2
      192.168.220.139    rocketmq-slave1
   ```

   - 重启网卡

     ```shell
         # sudo apt install network-manager
         sudo service network-manager restart
         # 无效的话重启虚拟机
     ```

2. 配置环境变量，方便操作 qocketMq（两台虚拟机都要操作）

   ```shell
      sudo vim /etc/profile
      source /etc/profile
      echo $PATH
   ```

   ```profile
      export ROCKETMQ_HOME=/home/jxz/Documents/rocketmq/rocketmq-all-4.8.0-bin-release
      export PATH=$ROCKETMQ_HOME/bin:$PATH
   ```

3. 创建消息存储路径（两台虚拟机都要操作）

   ```shell
      rm -rf mkdir /home/jxz/Documents/rocketmq/store*
      rm $ROCKETMQ_HOME/nohup.out
      rm /home/jxz/nohup.out
      rm ~/logs/rocketmqlogs/namesrv.log
      rm ~/logs/rocketmqlogs/broker.log

      mkdir /home/jxz/Documents/rocketmq/store-a
      mkdir /home/jxz/Documents/rocketmq/store-a/commitlog
      mkdir /home/jxz/Documents/rocketmq/store-a/consumequeue
      mkdir /home/jxz/Documents/rocketmq/store-a/index
      mkdir /home/jxz/Documents/rocketmq/store-b-s
      mkdir /home/jxz/Documents/rocketmq/store-b-s/commitlog
      mkdir /home/jxz/Documents/rocketmq/store-b-s/consumequeue
      mkdir /home/jxz/Documents/rocketmq/store-b-s/index
      ls /home/jxz/Documents/rocketmq/
      ls /home/jxz/Documents/rocketmq/store-a
      ls /home/jxz/Documents/rocketmq/store-b-s


      mkdir /home/jxz/Documents/rocketmq/store-b
      mkdir /home/jxz/Documents/rocketmq/store-b/commitlog
      mkdir /home/jxz/Documents/rocketmq/store-b/consumequeue
      mkdir /home/jxz/Documents/rocketmq/store-b/index
      mkdir /home/jxz/Documents/rocketmq/store-a-s
      mkdir /home/jxz/Documents/rocketmq/store-a-s/commitlog
      mkdir /home/jxz/Documents/rocketmq/store-a-s/consumequeue
      mkdir /home/jxz/Documents/rocketmq/store-a-s/index
      ls /home/jxz/Documents/rocketmq/
      ls /home/jxz/Documents/rocketmq/store-b
      ls /home/jxz/Documents/rocketmq/store-a-s
   ```

4. 修改 broker 配置文件（两台虚拟机都要操作）

   - 位置（双主双从，同步）：`/home/jxz/Documents/rocketmq/rocketmq-all-4.8.0-bin-release/conf/2m-2s-sync`

     1. broker-a.properties ==>(master1)
     2. broker-a-s.properties ==>(slave1)
     3. broker-b.properties ==>(master2)
     4. broker-b-s.properties ==>(slave2)

   - 138 虚拟机上配置 master1 和 slave2 对应的文件（broker-a.properties、broker-b-s.properties）
   - 139 虚拟机上配置 master2 和 slave1 对应的文件（broker-b.properties、broker-a-s.properties）

5. 修改启动脚本中 JVM 内存分配大小（如果机器需要的话）

   ```shell
      sudo vim bin/runserver.sh
      # JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      sudo vim bin/runbroker.sh
      # JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
   ```

6. 启动集群

   - 分别启动 138 和 139 上的 mqnamesrv

     ```shell
        cd /home/jxz/Documents/rocketmq/rocketmq-all-4.8.0-bin-release
        nohup sh bin/mqnamesrv &
        jps
     ```

   - 在 138 上启动 master1 和 slave2

     ```shell
        cd /home/jxz/Documents/rocketmq/rocketmq-all-4.8.0-bin-release
        # master1
        nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-sync/broker-a.properties &
        # nohup sh bin/mqbroker -n 192.168.220.138:9876;192.168.220.139:9876 -c $ROCKETMQ_HOME/conf/2m-2s-sync/broker-a.properties &
        # slave2
        nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-b-s.properties &
        # nohup sh bin/mqbroker -n 192.168.220.138:9876;192.168.220.139:9876 -c conf/2m-2s-sync/broker-b-s.properties &
     ```

   - 在 139 上启动 master2 和 slave1
     ```shell
        cd /home/jxz/Documents/rocketmq/rocketmq-all-4.8.0-bin-release
        # master2
        nohup sh bin/mqbroker -c $ROCKETMQ_HOME/conf/2m-2s-sync/broker-b.properties &
        # nohup sh bin/mqbroker -n 192.168.220.138:9876;192.168.220.139:9876 -c conf/2m-2s-sync/broker-b.properties &
        # slave1
        nohup sh bin/mqbroker -c conf/2m-2s-sync/broker-a-s.properties &
        # nohup sh bin/mqbroker -n 192.168.220.138:9876;192.168.220.139:9876 -c conf/2m-2s-sync/broker-a-s.properties &
     ```

7. 查查看日志

   ```shell
      jps
      cat $ROCKETMQ_HOME/nohup.out
      tail -f $ROCKETMQ_HOME/nohup.out
      tail -f /home/jxz/nohup.out
      tail -f ~/logs/rocketmqlogs/namesrv.log
      tail -f ~/logs/rocketmqlogs/broker.log
   ```

8. 停止

   ```shell
      sh bin/mqshutdown broker
      sh bin/mqshutdown namesrv
   ```

9. 在不改变任何默认配置的情况下启动集群的主从结点，通过 `tail -f ~/logs/rocketmqlogs/broker.log ` 就能得到所有的默认的配置了。

10. 千万注意每次启动集群时
    > 先启动两个 Master Broker 结点，再启动两个 Slave Broker 结点。
    >
    > 有时候如果 Broker 结点总是启动该不起来则可以：
    >
    > 1. 先删除 `/home/jxz/Documents/rocketmq/store*` 文件夹，
    > 2. 创建好 `commitlog、consumequeue、index` 三个文件夹，
    > 3. 不要创建 `checkpoint、abort` 者两个文件夹。
    >
    > 如果不做这些操作，总是会有各种莫名其妙的错误，有时连日志都不报错，但就是启动不了 Broker 结点。

## 工具

### mqadmin 工具

```shell
   $ROCKETMQ_HOME/mqadmin {command} {args}
```

### [rocketmq-cosole](https://github.com/apache/rocketmq-externals)

### [RocketMQ Dashboard](https://github.com/apache/rocketmq-dashboard)

1. 修改 `application.properties` 配置文件中的 `rocketmq.config.namesrvAddr=192.168.220.138:9876;192.168.220.138:9876`
2. 利用 Maven 打包，或者直接用 Idea 打开项目来打包
3. 将 jar 包放到本地或者上传服务器上然后利用 `java -jar xxx.jar` 命令启动
4. 访问 `启动主机:8080` 便可查看 RocketMQ 情况。
