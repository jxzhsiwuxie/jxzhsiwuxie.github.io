# VMware 安装 Ubuntu 虚拟机

## 软件与镜像准备

1. VMware：VMware-workstation-full-16
2. Ubuntu：[ubuntu-20.10-live-server-amd64](https://mirrors.tuna.tsinghua.edu.cn/)

## 安装

### 安装镜像（基本上选择默认的就可以了）

![img](./assets/2021-09-20_211722.png)
![img](./assets/2021-09-20_212101.png)
![img](./assets/2021-09-20_212459.png)
![img](./assets/2021-09-20_212718.png)
![img](./assets/2021-09-20_212901.png)
![img](./assets/2021-09-20_213351.png)
![img](./assets/2021-09-20_213509.png)
![img](./assets/2021-09-20_213625.png)
![img](./assets/2021-09-20_213659.png)
![img](./assets/2021-09-20_213728.png)
![img](./assets/2021-09-20_213803.png)
![img](./assets/2021-09-20_213837.png)
![img](./assets/2021-09-20_213941.png)
![img](./assets/2021-09-20_214017.png)
![img](./assets/2021-09-20_214050.png)
![img](./assets/2021-09-20_214213.png)

#### 注意这里选择安装 OpenSSH 以方便使用 XShell 等工具直接连接

![img](./assets/2021-09-20_214249.png)
![img](./assets/2021-09-20_214333.png)

#### 选择重启后，在虚拟机标题处（DockerHabor）右键->设置 移除之前添加的 ISO 镜像文件，然后按下 Enter 键

![img](./assets/2021-09-20_214502.png)
![img](./assets/2021-09-20_214604.png)
![img](./assets/2021-09-20_214604.png)

#### 虚拟机成功启动，利用之前设置的用户名和密码登录

![img](./assets/2021-09-20_214711.png)
![img](./assets/2021-09-20_214737.png)

### [开启 Ubuntu Server 的 root 用户的登录](https://ubuntu.com/server/docs/security-users)

在终端输入

```shell
sudo passwd
```

会有如下提示

> [sudo] password for username: (enter your own password) （当前用户 jxz 的密码，并且最近短时间内没有使用过 sudo 命令的话才会提示这个，否则直接进入下一个提示）  
> Enter new UNIX password: (enter a new password for root)  
> Retype new UNIX password: (repeat new password for root)  
> passwd: password updated successfully

![img](./assets/2021-09-20_222530.png)

然后重启

```shell
reboot
```

重启后就可以使用 root 用户登录了
![img](./assets/2021-09-20_223113.png)

### 设置静态 Ip，默认情况下还没有被配置网络

![img](./assets/2021-09-20_223419.png)

1. **VMware：编辑 -> 虚拟网络编辑器 -> 更改设置**
   ![img](./assets/2021-09-20_223754.png)

2. **取消选中：使用本地 DHCP 服务将 IP 地址分配给虚拟机，并记住子网 ip（192.168.220.0）**

![img](./assets/2021-09-20_224039.png)

3. **点击 NAT 设置，记住网关地址（192.168.220.2）**
   ![img](./assets/2021-09-20_224329.png)
   ![img](./assets/2021-09-20_224537.png)
   ![img](./assets/2021-09-20_224640.png)

4. 回到虚拟机中设置网卡 ens33 的 ip

   > 注意：Ubuntu18 固定 IP 的方式跟 Ubuntu18 之前版本的的配置方式不同，Ubuntu18 之前在/etc/network/interfaces 进行配置，Ubuntu18 及之后版本在/etc/netplan/\*.yaml 进行配置，如/etc/netplan/01-network-manager-all.yaml，如果/etc/netplan 目录下没有 yml 文件，则可以新建一个。

   ```shell
    # 这里已存在 00-installer-config.yml
    ls /etc/netplan/
   ```

   ![img](./assets/2021-09-20_225054.png)
   ![img](./assets/2021-09-20_230626.png)

   ```yaml
   network:
   version: 2
   ethernets:
     ens33: # 网卡名称
       dhcp4: no
       dhcp6: no
       addresses: [192.168.220.100/24] # 本机ip及掩码，最后一段不要取1或2
       gateway4: 192.168.220.2 # 前面记录的网关地址
       nameservers:
         addresses: [192.168.220.2] # DNS跟随网关地址一致，也可以改别的，如[114.114.114.114,8.8.8.8]
   ```

5. 使 ip 配置生效

   ```shell
    netplan apply
   ```

   ![img](./assets/2021-09-20_230709.png)

6. 设置远程以 ssh 登录 root 用户

   ```shell
    vi /etc/ssh/sshd_config
    # 将 PermitRootLogin prohibit_password 修改为
    # PermitRootLogin yes
    # 重启 SSH server
    systemctl restart sshd.service
   ```

   ![img](./assets/2021-09-20_232425.png)
   ![img](./assets/2021-09-20_232853.png)

7. 使用 XShell 或者其它工具登录虚拟机，以方便操作（server 版窗口字体太小了）
   ![img](./assets/2021-09-20_231025.png)
   ![img](./assets/2021-09-20_231104.png)
   ![img](./assets/2021-09-20_231133.png)
   ![img](./assets/2021-09-20_233018.png)

### [设置国内的镜像源（这里选择清华镜像源）](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)

1. 选择对应的版本
   ![img](./assets/2021-09-20_215725.png)
2. 备份原本默认配置然后修改为新的配置

   ```shell
    tee sources.list <<-'EOF'
    # 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-updates main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-updates main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-backports main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-backports main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-security main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-security main restricted universe multiverse

    # 预发布软件源，不建议启用
    # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-proposed main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-proposed main restricted universe multiverse
    EOF
   ```

   ![img](./assets/2021-09-20_234440.png)

3. 更新 apt 源
   ```shell
   # update 只检查是否有更新，实际不会更新；upgrade 实际更新
    apt update && apt upgrade
   ```

## 至此，VMware 中安装 Ubuntu 并设置静态 ip 完成。
