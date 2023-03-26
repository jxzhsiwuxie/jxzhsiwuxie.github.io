# 部署

[资料](https://pan.baidu.com/disk/main?from=homeSave#/index?category=all&path=%2F%E7%BC%96%E7%A8%8B%2F%E5%B0%9A%E7%A1%85%E8%B0%B7Kubernetes%E6%95%99%E7%A8%8B)

## 网络选择

使用一主两从的集群部署
>使用网段 `192.168.120.0/24`，虚拟机网卡使用 `NAT` 模式。

1. `k8s-master01` ip 地址 `192.168.120.110`
2. `k8s-node01` ip 地址 `192.168.120.120`
3. `k8s-node02` ip 地址 `192.168.120.121`

>VMWare 设置网络为 `NAT` 模式，同时关闭 `DHCP` 获取 `ip` 的方式
> 将网关设置为 192.168.120.2

```sh
vi /etc/sysconfig/network-scripts/ifcfg-ens33

# k8s-master01
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=3468e44b-32d5-4eb4-b6c9-61c417adb084
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.120.110
NETMASK=255.255.255.0
GATEWAY=192.168.120.2
DNS1=192.168.120.2
DNS2=114.114.114.114

# k8s-node01
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=3468e44c-32d5-4eb4-b6c9-61c417adb084
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.120.120
NETMASK=255.255.255.0
GATEWAY=192.168.120.2
DNS1=192.168.120.2
DNS2=114.114.114.114

# k8s-node02
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=3468e44d-32d5-4eb4-b6c9-61c417adb084
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.120.121
NETMASK=255.255.255.0
GATEWAY=192.168.120.2
DNS1=192.168.120.2
DNS2=114.114.114.114
```

### 固定主机名

```sh
# 查看当前主机名
uname -n
hostname
# localhost.localdomain

# 设置当前主机名同时修改 hosts 文件
# master01 节点执行
hostnamectl set-hostname k8s-master01
# ndoe01 节点执行
hostnamectl set-hostname k8s-node01
# ndoe02 节点执行
hostnamectl set-hostname k8s-node02

# 三个节点都执行（或者在 master01 上执行之后将 hosts 文件拷贝到另外两台机器上）
cat /etc/hosts

# 向 hosts 追加 ip 主机名 映射
cat >> /etc/hosts <<EOF

192.168.120.110 k8s-master01
192.168.120.120 k8s-node01
192.168.120.121 k8s-node02
EOF

# 再次查看 hosts 内容
cat /etc/hosts

# 将 hosts 文件拷贝到 node01 和 node02 机器对应的目录中
scp /etc/hosts root@k8s-node01:/etc/hosts
scp /etc/hosts root@k8s-node02:/etc/hosts

# 查看是否设置成功，在 master 和 node 之间通过主机名 ping
ping k8s-master01
ping k8s-node01
ping k8s-node02
```

### 安装一些过程中用到的依赖（三台机器都安装）

```sh
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git
```

### 设置防火墙为 Iptables 并设置空规则（三台机器都要执行）

```sh
# 临时关闭防火墙 && 永久关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
# 安装 Iptables、设置开机自启、设置规则为空并保存
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save
```

### 关闭 Swap 交换分区以及关闭 selinux（三台机器都要执行）

```sh
# 关闭swap 且 写入到 /etc/fstab 文件中，保证重启机器也生效
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭selinux
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 调整内核参数（三台机器都要执行）

```sh
cat > kubernetes.conf <<EOF
# 必须
net.bridge.bridge-nf-call-iptables=1
# 必须
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
# 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.swappiness=0
# 不检查物理内存是否够用
vm.overcommit_memory=1
# 开启 OOM
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
# 必须
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
# 立即生效
sysctl -p /etc/sysctl.d/kubernetes.conf

# 出现如下报错，暂时不用管，升级内核版本之后就会存在这些文件
# sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: 没有那个文件或目录
# sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: 没有那个文件或目录
# net.ipv4.ip_forward = 1
# net.ipv4.tcp_tw_recycle = 0
# vm.swappiness = 0
# vm.overcommit_memory = 1
# vm.panic_on_oom = 0
# fs.inotify.max_user_instances = 8192
# fs.inotify.max_user_watches = 1048576
# fs.file-max = 52706963
# fs.nr_open = 52706963
# net.ipv6.conf.all.disable_ipv6 = 1
# sysctl: cannot stat /proc/sys/net/netfilter/nf_conntrack_max: 没有那个文件或目录

```

### 调整系统时区（三台机器都执行）

```sh
# 查看当前系统时区（不是 Asia/Shanghai 则设置一下）
timedatectl status
# 设置系统时区为 中国/上海
timedatectl set-timezone Asia/Shanghai
# 将当前的 UTC 时间写入硬件时钟
timedatectl set-local-rtc 0
# 重启依赖于系统时间的服务
systemctl restart rsyslog
systemctl restart crond
```

### 关闭系统不需要服务（三台机器都执行）

```sh
# 邮件服务
systemctl stop postfix && systemctl disable postfix
```

### 设置日志模式从 rsyslogd 改为 systemd-journald（三台机器都执行）

```sh
# 持久化保存日志的目录
mkdir /var/log/journal
mkdir /etc/systemd/journald.conf.d

# 创建 systemd-journald 日志配置文件
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 syslog
ForwardToSyslog=no
EOF

# 重启 systemd-journald 日志
systemctl restart systemd-journald
```

### 升级系统内核（三台机器都要执行）

>CentOS 7.x 系统自带的 3.10.x 内核存在一些 Bugs，导致运行的 Docker、Kubernetes 不稳定

```sh
# 查看内核linux版本
cat /proc/version
# Linux version 3.10.0-1160.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) ) #1 SMP Mon Oct 19 16:18:59 UTC 2020
# 安装
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
yum --enablerepo=elrepo-kernel install -y kernel-lt
# 设置开机从新内核启动
grub2-set-default 'CentOS Linux (5.4.238-1.el7.elrepo.x86_64) 7 (Core)'
# 重启
reboot
# 查看是否使用新版本内核
uname -sr
# 5.4.238-1.el7.elrepo.x86_64
```

### kube-proxy开启ipvs的前置条件（三台机器都要执行）

```sh
modprobe br_netfilter

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

> 最后一步报错 `modprobe: FATAL: Module nf_conntrack_ipv4 not found.`
> 因为高版本的 linux 内核中 `nf_conntrack_ipv4` 和 `nf_conntrack_ipv6` 两者都被弃用
> 合并为 `nf_conntrack`

[参考1](https://github.com/kubernetes-sigs/kubespray/issues/7176)
[参考2](https://www.cnblogs.com/Lusai/p/16701240.html)

```sh
modprobe br_netfilter

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
```

### 安装 Docker 软件（三台机器都要）

```sh
yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加阿里云docker镜像仓库
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装最新版 docker
yum update -y && yum install -y docker-ce
# 已安装:
#   docker-ce.x86_64 3:23.0.1-1.el7    

# 更新完之后，重新设置一下启动时默认使用新版本内核，否则重启后内核版本又会回到之前的老版本
grub2-set-default 'CentOS Linux (5.4.238-1.el7.elrepo.x86_64) 7 (Core)'
# 重启
reboot
# 查看内核版本
uname -sr
# 5.4.238-1.el7.elrepo.x86_64

# 启动 docker 并设置为开机自启
systemctl start docker && systemctl enable docker

## 创建 /etc/docker 目录
mkdir /etc/docker

# 配置 daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://gqs7xcfd.mirror.aliyuncs.com","https://hub-mirror.c.163.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  }
}
EOF

# 创建 docker 配置文件目录
mkdir -p /etc/systemd/system/docker.service.d
# 重启docker服务
systemctl daemon-reload && systemctl restart docker && systemctl enable docker

# 查看docker版本
docker version
```

### 安装 kubeadm、kubelet 和 kubectl （三台机器都需要）

```sh
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum -y install kubeadm-1.25.8 kubectl-1.25.8 kubelet-1.25.8

# 设置kubelet开机自启
# 安装好kubelet后先不用启动，当集群初始化的时候会自动启动kubelet，选择启动kubelet会报错
systemctl enable kubelet.service
```

### 初始化主节点

```sh
# 指定k8s的版本、pod网络地址段、service资源网络地址段、apiserver地址
# apiserver-advertise-address 指定为执行的 master ip
kubeadm init \
--kubernetes-version=1.19.6 \
--apiserver-advertise-address=0.0.0.0 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.245.0.0/16 \
--image-repository registry.aliyuncs.com/google_containers \
--apiserver-advertise-address=192.168.100.10


```

```sh
# 先将 kubeadm 默认的初始化文件打印到当前目录下的 kubeadm-config.yaml 文件中，方便更改
kubeadm config print init-defaults > kubeadm-config.yaml
cat kubeadm-config.yaml

#1）修改 advertiseAddress 值为当前 master 的 ip：192.168.120.110
#2）修改 kubernetesVersion 的版本为之前安装的版本 1.25.8
#3）networking 下面添加 podSubnet: "10.244.0.0/16" 为了与 flannel 网络插件对的上
#4）在最后面添加上将调度方式改为 ipvs 调度的配置：
# ---
# apiVersion: kubeproxy.config.k8s.io/v1alpha1
# kind: KubeProxyConfiguration
# featureGates:
#   SupportIPVSProxyMode: true
# mode: ipvs
#5）修改 imageRepository 为阿里镜像源仓库
```

[参考](https://huangzhongde.cn/istio/Chapter2/Chapter2-4.html)
[参考2](https://blog.51cto.com/u_10112066/6145616)

```sh
rm -f kubeadm-config.yaml

cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  # 当前 master01 的 ip
  advertiseAddress: 192.168.120.110
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
# 修改镜像拉去的仓库地址
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
# 刚才安装的版本
kubernetesVersion: 1.25.8
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  # 添加 flannel 网络插件用的网段
  podSubnet: 10.244.0.0/16
scheduler: {}
---
# 添加网络调度模式为 ipvs
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
EOF

# 查看镜像
kubeadm config images list --config kubeadm-config.yaml
# 预先拉去镜像
kubeadm config images pull --config kubeadm-config.yaml
# 语法验证
kubeadm init --config kubeadm-config.yaml --dry-run
# 出现如下错误：
# [ERROR CRI]: container runtime is not running: output: time="2023-03-26T02:20:17+08:00" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
# 参考 https://www.cnblogs.com/immaxfang/p/16721407.html
mv /etc/containerd/config.toml /etc/containerd/config.toml.20230326.bk
systemctl restart containerd
# 再次验证成功

```

```sh
# 验证配置文件是否正确
kubeadm init --config kubeadm-config.yaml --dry-run
# 利用初始化文件初始化 kubeadm
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.log
```
