# kubernetes集群部署 —— kubeadm

## 安装前准备

> 1. 准备1台master，2核、4G内存、20G硬盘，2台node，4核，8G内存、40G硬盘。
> 2. 系统 Ubuntu Server 20.04.3 LTS

## Ubuntu配置

```shell
# 下载缺少的模块
apt install net-tools

# 配置ssh可以root登录
vi /etc/ssh/sshd_config # 将PermitRootLogin 改为 yes
service ssh restart # 重启ssh服务
```

## 环境准备

```shell
# 根据规划设置主机名（3台机器分别运行）
hostnamectl set-hostname master1
hostnamectl set-hostname node1
hostnamectl set-hostname node2

#在master添加hosts(在msster上运行)
cat >> /etc/hosts << EOF
192.168.0.200 master01
192.168.0.201 node01
192.168.0.202 node02
EOF

#设置免登录(在msster上运行)
ssh-keygen
ssh-copy-id root@node1
ssh-copy-id root@node2

#把hosts文件复制到node01\02(在msster上运行)
scp /etc/hosts root@node01:/etc/hosts
scp /etc/hosts root@node02:/etc/hosts

#关闭防火墙(在3台机运行) 【*】
systemctl stop firewalld && systemctl disable firewalld

#关闭selinux(在3台机运行) 【*】
sed -i 's/enforcing/disabled/' /etc/selinux/config && setenforce 0

#关闭swap(在3台机运行)
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab


#时间同步(在3台机运行)
apt install ntpdate -y && timedatectl set-timezone Asia/Shanghai && ntpdate time.windows.com
```

## 安装Docker

```shell
# Update the apt package index and install packages to allow apt to use a repository over HTTPS:
apt update
apt install ca-certificates curl gnupg lsb-release

# Add Docker’s official GPG key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Use the following command to set up the stable repository.
 echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
# 下载 Docker Engine
apt update
apt-get install docker-ce docker-ce-cli containerd.io

# 更换镜像
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors" : [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://cr.console.aliyun.com/"
  ]
}
EOF

# 重启docker
systemctl restart docker

# 将用户加入到docker组
groupadd docker

```

## 安装kubelet、kubeadm、kubectl

- kubeadm：用来初始化集群的指令。
- kubelet：在集群中的每个节点上用来启动 pod 和容器等。
- kubectl：用来与集群通信的命令行工具。

> kubeadm 不能 帮您安装或者管理 kubelet 或 kubectl，所以您需要确保它们与通过 kubeadm 安装的控制平面的版本相匹配。 如果不这样做，则存在发生版本偏差的风险，可能会导致一些预料之外的错误和问题。 然而，控制平面与 kubelet 间的相差一个次要版本不一致是支持的，但 kubelet 的版本不可以超过 API 服务器的版本。 例如，1.7.0 版本的 kubelet 可以完全兼容 1.8.0 版本的 API 服务器，反之则不可以。

```shell
# 添加阿里云密钥
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

# 添加kubernetes阿里云源
apt-get update && apt-get install -y apt-transport-https
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt update

# 安装
apt install kubelet=1.18.20-00 kubeadm=1.18.20-00 kubectl=1.18.20-00
```

## 准备集群镜像

```
# kubeadm init前，先准备k8s运行所需的容器
# 查询所需镜像
kubeadm config images list

# 写sh脚本，拉取镜像
cat >> alik8simages.sh << EOF
#!/bin/bash
list='kube-apiserver:v1.18.20
kube-controller-manager:v1.18.20
kube-scheduler:v1.18.20
kube-proxy:v1.18.20
pause:3.2
etcd:3.4.3-0
coredns:1.6.7'
for item in \$list
  do
    docker pull registry.aliyuncs.com/google_containers/\$item
    docker tag registry.aliyuncs.com/google_containers/\$item k8s.gcr.io/\$item
    docker rmi registry.aliyuncs.com/google_containers/\$item
  done
EOF

# 运行脚本
bash alik8simages.sh
```

## 集群初始化

```
# 创建集群（只在master上运行） apiserver-addvertise-address指定master节点ip，剩下两个固定写法。
kubeadm init \
--kubernetes-version=v1.18.20 \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12 \
--apiserver-advertise-address=192.168.30.128
```

​	成功则会看到如下信息

![image-20211025110018052](D:\k8s_vm\kubernetes文档\image-20211025110018052.png)

![image-20211025110018052](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20211025110018052.png)

```
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
# 查看nodes
kubectl get nodes

# 将node节点加入（在node上运行）
kubeadm join 192.168.30.128:6443 --token 3im9ok.4lqwo0a2rnfy8s3y \
    --discovery-token-ca-cert-hash sha256:7d21087042950694fa65414ead4342b6ea0eee5fa2a7171f4c2b848037fbb6d1
```

## 部署CNI网络插件

​	只在master上运行，插件使用的是DaemonSet的控制器，它会在每个节点上都运行

```
# 使用配置文件启动fannel
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

# 查看运行情况
kubectl get pods -n kube-system
kubectl get nodes
```

## 测试kubernetes集群

```
# 在kubernetes集群中创建一个pod，验证是否正常运行
kubectl create deployment nginx --image=nginx # kubectl get pod查看状态，running之后再执行后续操作
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
```

![image-20211025115724590](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20211025115724590.png)

通过三台机器任意 ip:30701即可访问

