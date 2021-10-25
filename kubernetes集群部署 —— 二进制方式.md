# kubernetes集群部署 —— 二进制方式

## 1. 创建多台虚拟机，安装Linux操作系统
> [Ubuntu Server 20.04.3 LTS 配置](https://github.com/1849805767/kubernetes/blob/master/Ubuntu%20Server%2020.04.3%20LTS%20%E5%88%9D%E5%A7%8B%E9%85%8D%E7%BD%AE.md)

## 2. 操作系统初始化

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久
setenforce 0 # 临时

# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
swapoff -a # 临时

# 根据规划设置主机名（3台机器分别运行）
hostnamectl set-hostname master2
hostnamectl set-hostname node3
hostnamectl set-hostname node4

#在master添加hosts(在master上运行)
cat >> /etc/hosts << EOF
192.168.30.131 master2
192.168.30.132 node3
192.168.30.133 node4
EOF

#设置免登录(在master上运行)
ssh-copy-id root@node3
ssh-copy-id root@node4

#把hosts文件复制到node3\4(在master上运行)
scp /etc/hosts root@node3:/etc/hosts
scp /etc/hosts root@node4:/etc/hosts

# 将桥接的IPv4流量传递到iptables的链（三台机器都执行）
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system # 生效

# 时间同步(在3台机运行)
apt install ntpdate -y && timedatectl set-timezone Asia/Shanghai && ntpdate time.windows.com
```

## 3. 为Etcd和apiserver自签证书

> cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用。
>
> 找任意一台服务器操作，这里使用master节点

```
# 在matster任意一台主机下载cfssl工具
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo

```



## 4. 部署Etcd集群

> Etcd是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储，所以先准备一个Etcd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群，可容忍1台机器故障，也可以使用5台组建集群，可容忍2台机器故障。

```

```

## 部署master组件

> kube-apiserver、kube-controller-manager、kube-scheduler、Etcd

## 部署node组件

> kubelet、kube-proxy、docker、Etcd

## 部署集群网络

