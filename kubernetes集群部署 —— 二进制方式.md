

# kubernetes集群部署 —— 二进制方式

---

> Github地址：https://github.com/1849805767/kubernetes/
>
> QQ：1849805767

---

## 服务器规划

> 3 master + 3 Node。
>
> 经过测试，两台etcd搭建集群时，是unhealthy的，换成三个就好了。

| 角色 |       IP       |                             组件                             |
| :--: | :------------: | :----------------------------------------------------------: |
| m1n1 | 192.168.30.137 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd<br />kubelet，kube-proxy，docker |
| m2n2 | 192.168.30.138 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd<br />kubelet，kube-proxy，docker |
|  m3  | 192.168.30.139 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
|  n3  | 192.168.30.140 |                 kubelet，kube-proxy，docker                  |

> 本文所有命令均在root权限下执行

---

## 1	创建多台虚拟机，安装Linux操作系统
> [Ubuntu Server 20.04.3 LTS 配置](https://github.com/1849805767/kubernetes/blob/master/Ubuntu%20Server%2020.04.3%20LTS%20%E5%88%9D%E5%A7%8B%E9%85%8D%E7%BD%AE.md)

---

## 2	操作系统初始化

```sh
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久
setenforce 0 # 临时

# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
swapoff -a # 临时

# 根据规划设置主机名（4台机器分别运行）
hostnamectl set-hostname m1n1
hostnamectl set-hostname m2n2
hostnamectl set-hostname m3
hostnamectl set-hostname n3

#在master添加hosts(在m1n1上运行)
cat >> /etc/hosts << EOF
192.168.30.137 m1n1
192.168.30.138 m2n2
192.168.30.139 m3
192.168.30.140 n3
EOF

#设置免登录(在m1n1上运行)
ssh-copy-id root@m2n2
ssh-copy-id root@m3
ssh-copy-id root@n3

#把hosts文件复制到其他机器(在m1n1上运行)
scp /etc/hosts root@m2n2:/etc/hosts
scp /etc/hosts root@m3:/etc/hosts
scp /etc/hosts root@n3:/etc/hosts

# 将桥接的IPv4流量传递到iptables的链（4台机器都执行）
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system # 生效

# 时间同步(在4台机运行)
apt install ntpdate -y && timedatectl set-timezone Asia/Shanghai && ntpdate time.windows.com
```

---

## 4	生成所需证书

### 	4.1	下载 cfssl  证书生成工具

> cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用。

```sh
# 在master任意一台主机下载cfssl工具
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

> 先在一台机器生成，其他机器需要使用时，拷贝过去即可。这里使用master2节点

### 	4.2	生成Etcd证书

```sh
# 创建证书目录
mkdir -p ~/TLS/{etcd,k8s}
cd ~/TLS/etcd

# 1.创建自签证书颁发机构CA
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "438000h"
    },
    "profiles": {
      "www": {
        "expiry": "438000h",
        "usages": [ "signing", "key encipherment", "server auth", "client auth" ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
  "CN": "etcd CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing"
    }
  ]
}
EOF

# 2.生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
> ca-key.pem  ca.pem

# 3.创建证书申请文件
#注：文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。
cat > server-csr.json << EOF
{
  "CN": "etcd",
  "hosts": [
  "192.168.30.137",
  "192.168.30.138",
  "192.168.30.139"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

# 4.etcd https证书生成
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

ls server*pem
> server-key.pem  server.pem

# 5.校验证书两种方式
cfssl-certinfo -cert server.pem
openssl x509  -noout -text -in server.pem
```

### 	4.2	生成 kube-apiserver 证书

```sh
# 切换工作目录
cd ~/TLS/k8s/

# 1.自签证书颁发机构（CA）
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "438000h"
    },
    "profiles": {
      "kubernetes": {
        "expiry": "438000h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
 
# 2.生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
> ca-key.pem  ca.pem

#3.创建证书申请文件
# hosts字段中IP为所有Master/LB等 IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。
# 10.0.0.1：这是后边dns要用的虚拟网络的网关,不用改	--service-cluster-ip-range=10.0.0.0/24 切忌
cat > server-csr.json << EOF
{
  "CN": "kubernetes",
  "hosts": [
    "10.0.0.1",
    "127.0.0.1",
    "192.168.30.137",
    "192.168.30.138",
    "192.168.30.139",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
 
# 4.生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

# 以上k8s证书暂时不用留到kube-apiserver使用。
```

### 	4.3	生成 kube-proxy 证书

```sh
# 切换工作目录
cd ~/TLS/k8s

# 1.自签证书颁发机构（CA）【已完成】
# 2.生成证书【已完成】

# 3.创建证书请求文件
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 4.生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

# 以上证书借给部署kube-proxy使用。
```

---

## 5	部署 Master 组件

> kube-apiserver、kube-controller-manager、kube-scheduler、Etcd

### 	5.1	 部署 Etcd 集群

> Etcd是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储。
>
> 为解决Etcd单点故障，应采用集群方式部署。使用3台组建集群，可容忍1台机器故障，也可以使用5台组建集群，可容忍2台机器故障。
>
> 以下在 master2 上操作,然后将所需内容拷贝到 master3

```sh
# 1.下载二进制包
cd ~/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz

# 2.创建工作目录，解压二进制包
mkdir /opt/etcd/{bin,cfg,ssl} -p # 其余master也要执行
tar zxvf etcd-v3.5.1-linux-amd64.tar.gz
mv etcd-v3.5.1-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

# 3.创建配置文件
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"    
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.30.137:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.30.137:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.30.137:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.30.137:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.30.137:2380,etcd-2=https://192.168.30.138:2380,etcd-3=https://192.168.30.139:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

	> ETCD_NAME：节点名称，集群中唯一
	  ETCD_DATA_DIR：数据目录
	  ETCD_LISTEN_PEER_URLS：集群通信监听地址（本机IP）
	  ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址（本机IP）
	  ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址（本机IP）
	  ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址（本机IP）
	  ETCD_INITIAL_CLUSTER：集群节点地址
	  ETCD_INITIAL_CLUSTER_TOKEN：集群Token
	  ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

# 4.拷贝etcd证书至/opt/etcd/ssl/目录中
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/

# 5.创建etcd系统服务
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 6.将所需文件拷贝到其余master中
	# etcd文件、配置文件、证书
	scp -r m1n1:/opt/etcd /opt/
	# 服务文件
	scp m1n1:/usr/lib/systemd/system/etcd.service /usr/lib/systemd/system/
	
# 7.其余master要修改配置文件中的 ETCD_NAME 和 ip
vi /opt/etcd/cfg/etcd.conf

# 8.启动并设置开机启动 所有master都要运行
systemctl daemon-reload
systemctl start etcd
# 注：在第一台节点上执行start后会一直卡着无法返回命令提示符，这是因为在等待其他节点准备就绪，继续启动其余节点即可
systemctl enable etcd

# 查看集群状态
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.30.137:2379,https://192.168.30.138:2379,https://192.168.30.139:2379" endpoint health
```

![image-20211026195346412](kubernetes%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2%20%E2%80%94%E2%80%94%20%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%B9%E5%BC%8F.assets/image-20211026195346412.png)

​	如果输出上面信息，就说明集群部署成功，如果有问题，使用 ```journalctl -u etcd``` 查看

### 	5.2	下载 kubernetes 

```sh
# 1.下载二进制包。网址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md
# 只需要下载server，里面包含了Master和worker node二进制
wget https://dl.k8s.io/v1.18.20/kubernetes-server-linux-amd64.tar.gz

# 2.将master所需的3个组件放到/opt/下
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} # master机器都执行
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin/
cp kubectl /usr/bin

# 3.其他master节点运行，拷贝master所需的 kube-apiserver kube-scheduler kube-controller-manager
scp m1n1:/opt/kubernetes/bin/* /opt/kubernetes/bin/
scp m1n1:/usr/bin/kubectl /usr/bin/
```

### 	5.3	部署 kube-apiserver

```sh
# 1.创建 kube-apiserver 配置文件
# 两个\第一个是转义符，第二个是换行符，使用转义符是为了使用EOF保留换行符。
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://192.168.30.137:2379,https://192.168.30.138:2379,https://192.168.30.139:2379 \
--bind-address=192.168.30.137 \
--secure-port=6443 \
--advertise-address=192.168.30.137 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-32767 \
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
 
# 2.拷贝之前生成的kube-apiserver证书到配置文件中的路径
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
 
# 3.配置token文件(启用TLS Bootstrapping机制)
# 3.1.生成token
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
f3b805164ab3482d6e690800f0bcd514

# 3.2.创建token文件	格式：token，用户名，UID，用户组
cat > /opt/kubernetes/cfg/token.csv << EOF
f3b805164ab3482d6e690800f0bcd514,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF

# 4.systemd管理apiserve
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
 
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
EOF

# 5.启动并设置开机启动
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver

# 6.授权kubelet-bootstrap用户允许请求证书
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
 
# 7.把master1节点上内容分别拷贝到在master2和master3节点上，注意token不变
# 配置文件分别修改--bind-address成本机ip，--advertise-address是SLB的ip地址不变化(已经在负载配置完成)
# 在其他master上执行
scp -r m1n1:/opt/kubernetes /opt/
vi /opt/kubernetes/cfg/kube-apiserver.conf # 修改--bind-address成本机ip地址
scp -r m1n1:/usr/lib/systemd/system/kube-apiserver.service /usr/lib/systemd/system/

# 8.所有master启动kube-apiserver服务
systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver
 
# 9.验证在浏览器上输入以下地址
https://192.168.30.137:6443/version # 这个地址是--advertise-address=192.168.30.137指定的
https://192.168.30.137:6443/version
https://192.168.30.138:6443/version
https://192.168.30.139:6443/version # 以上地址返回kubernetes版本信息说明正常
#或在主节点上执行
curl -k https://192.168.30.137:6443/version
#或者在各节点执行以下命令能看到etcd各节点为健康说明正常
kubectl get cs
```

```sh
启用 TLS Bootstrapping 机制：
TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。

kube-apiserver配置文件注解：
–logtostderr：启用日志
—v：日志等级
–log-dir：日志目录
–etcd-servers：etcd集群地址
–bind-address：监听地址
–secure-port：https安全端口
–advertise-address：集群通告地址
–allow-privileged：启用授权
–service-cluster-ip-range：Service虚拟IP地址段
–enable-admission-plugins：准入控制模块
–authorization-mode：认证授权，启用RBAC授权和节点自管理
–enable-bootstrap-token-auth：启用TLS bootstrap机制
–token-auth-file：bootstrap token文件
–service-node-port-range：Service nodeport类型默认分配端口范围
–kubelet-client-xxx：apiserver访问kubelet客户端证书
–tls-xxx-file：apiserver https证书
–etcd-xxxfile：连接Etcd集群证书
–audit-log-xxx：审计日志
```

### 	5.4	部署 kube-controller-manager

```sh
# 1.创建部署文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect=true \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1 \
--allocate-node-cidrs=true \
--cluster-cidr=10.244.0.0/16 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--experimental-cluster-signing-duration=438000h0m0s"
EOF
	> -–master：通过本地非安全本地端口8080连接apiserver。
	  -–leader-elect：当该组件启动多个时，自动选举（HA）
	  -–cluster-signing-cert-file/–cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致

# 2.systemd管理controller-manager
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

#3.启动并设置开机启动
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager

#4.其余master运行
scp m1n1:/opt/kubernetes/cfg/kube-controller-manager.conf  /opt/kubernetes/cfg/
scp m1n1:/usr/lib/systemd/system/kube-controller-manager.service /usr/lib/systemd/system/

systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager

#5.在所有节点执行命令以下能看到controller-manager状态为健康说明正常
kubectl get cs

```

###		5.4	部署 kube-scheduler

```sh
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF

	> --master:通过本地非安全本地端口8080连接apiserver
	  --leader-electl
	  
# 2.systemd管理scheduler
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 3.启动并设置开机启动
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler

# 4.其余master执行
scp m1n1:/opt/kubernetes/cfg/kube-scheduler.conf /opt/kubernetes/cfg/
scp m1n1:/usr/lib/systemd/system/kube-scheduler.service /usr/lib/systemd/system/
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler

# 5.在所有节点执行命令以下能看到kube-scheduler状态为健康说明正常
kubectl get cs
```

###		5.5	验证 master 集群

> 所有master节点上执行 ```kubectl get cs``` 能看到```controller-manager/scheduler/etcd-{0,1,2}```状态为```Healthy```
>
> ![image-20211026212951142](kubernetes%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2%20%E2%80%94%E2%80%94%20%E4%BA%8C%E8%BF%9B%E5%88%B6%E6%96%B9%E5%BC%8F.assets/image-20211026212951142.png)

---

## 6	部署 Worker Node

### 		6.1	安装Docker

```sh
# 下载地址
wget https://download.docker.com/linux/static/stable/x86_64/docker-18.06.3-ce.tgz 

# 解压二进制docker
tar zxvf docker-18.06.3-ce.tgz 
mv docker/* /usr/bin/

# 复制到其他机器
scp docker-18.06.3-ce.tgz node3:~/
scp docker-18.06.3-ce.tgz node4:~/

# 在其他机器上执行上述，解压二进制docker步骤
tar zxvf docker-18.06.3-ce.tgz 
mv docker/* /usr/bin/

# systemd管理docker【三台机器运行】
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF

# 修改镜像源
mkdir /etc/docker 
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
      "https://registry.docker-cn.com",
      "https://docker.mirrors.ustc.edu.cn"
    ]
}
EOF

# 启动并设置开启启动
systemctl daemon-reload
systemctl start docker
systemctl enable docker
```

### 		6.2	创建工作目录并拷贝二进制文件

```
# 1.创建工作目录
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} # 【所有node都执行】
 
# 2.所有node节点拷贝 kubelet、kube-proxy # 【所有node都执行】
scp m1n1:~/kubernetes/server/bin/kubelet /opt/kubernetes/bin/
scp m1n1:~/kubernetes/server/bin/kube-proxy /opt/kubernetes/bin/
```

### 		6.3	部署 kubelet

```sh
# 1.创建配置文件
	# --hostname-override=k8s-worker1 修改worker节点主机名
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-worker1 \
--network-plugin=cni \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yaml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0"
EOF
	> –-hostname-override：在集群中显示的主机名，唯一
	  –-network-plugin：启用CNI
	  –-kubeconfig：空路径，会自动生成，后面用于连接apiserver
	  –-bootstrap-kubeconfig：首次启动向apiserver申请证书
	  –-config：配置参数文件
	  –-cert-dir：kubelet证书生成目录
	  –-pod-infra-container-image：管理Pod网络容器的镜像 
	
# 2.配置参数文件
cat > /opt/kubernetes/cfg/kubelet-config.yaml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

# 3.生成bootstrap.kubeconfig文件 【切换到master1节点创建kubeconfig.sh脚本来生成】
cd ~/TLS
cat > ~/TLS/kubeconfig.sh << EOF
# !/bin/bash
	# 指定 apiserver 内网负载均衡地址
KUBE_APISERVER="https://192.168.30.137:6443" # apiserver IP:PORT 采用SLB ip
TOKEN="f3b805164ab3482d6e690800f0bcd514" # 与token.csv里保持一致
	# 生成 kubelet bootstrap kubeconfig 配置文件
	# 设置集群参数
kubectl config set-cluster kubernetes \
 --certificate-authority=/opt/kubernetes/ssl/ca.pem \
 --embed-certs=true \
 --server=${KUBE_APISERVER} \
 --kubeconfig=bootstrap.kubeconfig
	# 设置客户端认证参数
kubectl config set-credentials "kubelet-bootstrap" \
 --token=${TOKEN} \
 --kubeconfig=bootstrap.kubeconfig
	# 设置上下文参数
kubectl config set-context default \
 --cluster=kubernetes \
 --user="kubelet-bootstrap" \
 --kubeconfig=bootstrap.kubeconfig
	# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
EOF

bash kubeconfig.sh # 运行脚本

# 4.拷贝bootstrap.kubeconfig、ca.pem到 worker1
scp m1n1:~/TLS/bootstrap.kubeconfig /opt/kubernetes/cfg/
scp m1n1:~/TLS/k8s/ca*.pem /opt/kubernetes/ssl/

# 5.systemd管理kubelet
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 6.启动并设置开机启动
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet

# 7.【所有node执行】
	# 拷贝 kubelet.conf、bootstrap.kubeconfig、kubelet-config.yaml
	scp m1n1:/opt/kubernetes/cfg/kubelet.conf m1n1:/opt/kubernetes/cfg/bootstrap.kubeconfig  /opt/kubernetes/cfg/ /opt/kubernetes/cfg/kubelet-config.yaml 
	# 修改主机名
	sed -i '4,/hostname/s/k8s-worker1/k8s-worker2/g' /opt/kubernetes/cfg/kubelet.conf 
	sed -i '4,/hostname/s/k8s-worker1/k8s-worker3/g' /opt/kubernetes/cfg/kubelet.conf

	scp m1n1:/opt/kubernetes/ssl/ca*.pem /opt/kubernetes/ssl/ # 证书

	scp m1n1:/usr/lib/systemd/system/kubelet.service /usr/lib/systemd/system/ # 服务

	systemctl daemon-reload
	systemctl start kubelet
	systemctl enable kubelet

# 8.验证kublet证书请求
# 在master节点执行下列命令，结果出现Pending说明节点kubelet运行正常。
kubectl get csr
```

### 		6.4	master 批准 kubelet 证书申请并加入集群

```sh
# 1.在任何master节点查看kubelet证书请求
kubectl get csr

#2.批准申请三个worker节点
kubectl certificate approve node-csr-qurlzezsERVbbr5c5sFclK51E4lMXf0rYev8Hn2fwbA
kubectl certificate approve node-csr-qXZUagqydFSjOK_fWoiOnyoEdAcQTu6AV5BZaJytxlQ
kubectl certificate approve node-csr-EsCryiKDXz6p97u4THepKc5PYisu8krUTvCUIO-OWjg

#3.查看节点
kubectl get node
> NAME          STATUS     ROLES    AGE   VERSION
  k8s-worker1   NotReady   <none>   12m   v1.18.20
  k8s-worker2   NotReady   <none>   54s   v1.18.20
  k8s-worker3   NotReady   <none>   44s   v1.18.20
# 注：由于网络插件还没有部署，节点会没有准备就绪 NotReady
```

### 	6.5	部署 kube-proxy

```sh
# 1.在 work1 节点创建配置文件
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \
 --v=2 \
 --log-dir=/opt/kubernetes/logs \
 --config=/opt/kubernetes/cfg/kube-proxy-config.yaml"
EOF

# 2.配置参数文件
	# hostnameOverride: k8s-worker1 修改成 worker 节点的主机名
cat > /opt/kubernetes/cfg/kube-proxy-config.yaml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-worker1
clusterCIDR: 10.0.0.0/24
EOF
 
# 3.生成 kube-proxy.kubeconfig 配置文件 【切换到master1节点创建kube-proxy.sh脚本】
cd ~/TLS
cat > ~/TLS/kube-proxy.sh << EOF
#!/bin/bash
KUBE_APISERVER="https://192.168.30.137:6443" # apiserver IP:PORT 采用SLB ip
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=./k8s/kube-proxy.pem \
  --client-key=./k8s/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
EOF

bash kube-proxy.sh # 执行脚本

# kube-proxy.kubeconfig 放到worker1工作目录
scp m1n1:~/TLS/kube-proxy.kubeconfig /opt/kubernetes/cfg/

# 4.systemd管理kube-proxy
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 5.启动并设置开机启动
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy

# 6.验证kube-proxy是否有报错
journalctl -u kube-proxy

# 7.【所有node执行】
	scp m1n1:/opt/kubernetes/cfg/kube-proxy*    /opt/kubernetes/cfg/
	# hostnameOverride: k8s-worker1修改成worker节点的主机名
	sed -i '7,/hostnameOverride/s/k8s-worker1/k8s-worker2/g' /opt/kubernetes/cfg/kube-proxy-config.yaml
	sed -i '7,/hostnameOverride/s/k8s-worker1/k8s-worker3/g' /opt/kubernetes/cfg/kube-proxy-config.yaml

	scp m1n1:/usr/lib/systemd/system/kube-proxy.service /usr/lib/systemd/system/ # 服务

	systemctl daemon-reload
	systemctl start kube-proxy
	systemctl enable kube-proxy
```

### 	6.6	部署 CNI 网络

```sh
# 1.下载二进制文件，CNI地址：https://github.com/containernetworking/plugins/releases/
cd && wget https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz

# 2.解压并移动默认工作目录
mkdir /opt/cni && mkdir /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v1.0.1.tgz -C /opt/cni/bin/

# 【所有node执行】
	scp -r m1n1:/opt/cni /opt/

# 2.部署CNI网络
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yaml
 
# 下载完成后默认镜像地址无法访问，修改为docker hub镜像仓库。【这里我没改，因为能访问外网】
# sed -i -r "s#quay.io/coreos/flannel:.*-amd64#dockerhub能访问到的镜像地址#g" kube-flannel.yaml
 
# 3.在 master1 节点执行
kubectl apply -f kube-flannel.yaml
 
# 4.验证CNI网络是否部署成功
kubectl -n kube-system get pods
> NAME                    READY   STATUS    RESTARTS   AGE
  kube-flannel-ds-n9hr6   1/1     Running   0          17m
  kube-flannel-ds-wbrq6   1/1     Running   0          17m
  kube-flannel-ds-zmbkq   1/1     Running   0          17m
  
kubectl get node
> NAME          STATUS   ROLES    AGE   VERSION
  k8s-worker1   Ready    <none>   12h   v1.18.20
  k8s-worker2   Ready    <none>   12h   v1.18.20
  k8s-worker3   Ready    <none>   12h   v1.18.20
  
# 5.添加worker名ip到hosts【否则会显示找不到主机的错误】
cat >> /etc/hosts << EOF
192.168.30.137 k8s-worker1
192.168.30.138 k8s-worker2
192.168.30.140 k8s-worker3
EOF
```

---

## 8	授权 apiserver 访问 kubelet

```sh
# 1.在 master1 节点执行以下
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
 
# 2.运行该yaml文件
kubectl apply -f apiserver-to-kubelet-rbac.yaml
 
# 3.验证yaml
kubectl -n kube-system get clusterrole|grep system:kube-apiserver-to-kubelet
kubectl -n kube-system get clusterrolebinding|grep system:kube-apiserver
# 以上命令有返回结果代表apiserver授权访问成功。
```

---

## 9	新增 Worker Node

```sh
> 可以执行Worker Node部署中，【所有node执行】的操作

# 1.拷贝 worker node 涉及内容到新节点
scp -r m1n1:/opt/kubernetes/ /opt/
scp -r m1n1/usr/lib/systemd/system/{kubelet,kube-proxy}.service /usr/lib/systemd/system/
scp -r m1n1:/opt/cni /opt/
scp m1n1:/opt/kubernetes/ssl/ca.pem /opt/kubernetes/ssl/

# 2.删除kubelet证书和kubeconfig文件，这些文件是证书许可之后生成的
rm /opt/kubernetes/cfg/kubelet.kubeconfig
rm -f /opt/kubernetes/ssl/kubelet*

# 3.修改主机名
vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-worker4

vi /opt/kubernetes/cfg/kube-proxy-config.yaml
hostnameOverride: k8s-worker4

# 4.启动并设置开机启动
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
systemctl start kube-proxy
systemctl enable kube-proxy

# 5.在master上批准新node kubelet证书申请
kubectl get csr
kubectl certificate approve ....
```

---

## 10	kubernetes插件部署

### 	10.1	Dashboard部署

```sh
# 1.下载Dashboard的yaml文件。官方主页https://github.com/kubernetes/dashboard
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

# 2.默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部：
vim recommended.yaml
...
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort   # 在第40行左右添加service的类型为NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001   # 新增
  selector:
    k8s-app: kubernetes-dashboard

---
...

kubectl apply -f recommended.yaml

# 3.验证
kubectl -n kubernetes-dashboard get pod,svc
> pod状态处于Running说明部署成功

# 5.创建service account并绑定默认cluster-admin管理员集群角色：
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')

# 6.使用输出的token登录Dashboard
https://worker节点任意ip:30001
```

### 	10.2	CoreDNS部署

> 使用kubeadm方式，默认就使用了CoreDNS，而二进制安装需要手动操作

```sh
# 1.kubernetes与coredns版本对照表 1.18->1.6.7
https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md

# 2.编写coredns.yaml文件
```

<details>
    <summary>coredns.yaml</summary>
    <pre><code>
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop 
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.6.7
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.2    # dns ip
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
	</code></pre>


```sh
# 3.执行部署命令
kubectl apply -f coredns.yaml

# 4.查看验证
kubectl -n kube-system get pods,svc

# 查看configmap
kubectl get cm coredns -n kube-system -o yaml
# 编辑configmap
kubectl -n kube-system edit  cm coredns
# 重新调度pod使配置生效
kubectl get pods -n kube-system | grep coredns | awk '{print $1}' | xargs kubectl -n kube-system delete pod

# 5.DNS解析测试
kubectl run -it --rm dns-test --image=busybox:1.28 sh # 如果不指定版本号，其他版本的nslookup会出现问题

/# nslookup kubernetes
> Server:    10.0.0.2
  Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

  Name:      kubernetes
  Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local

/ # nslookup kubernetes-dashboard.kubernetes-dashboard.svc.cluster.local
> Server:    10.0.0.2
  Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local\

  Name:      kubernetes-dashboard.kubernetes-dashboard.svc.cluster.local
  Address 1: 10.0.0.29 kubernetes-dashboard.kubernetes-dashboard.svc.cluster.local

/ # nslookup nginx-svc # 后面有部署教程，可以先去部署nginx
> Server:    10.0.0.2
 Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

 Name:      nginx-svc
 Address 1: 10.0.0.178 nginx-svc.default.svc.cluster.local
 
# 问题1：coredns始终crashloopbackoff，无法启动
# CoreDNS启动后会通过宿主机的resolv.conf文件去获取上游DNS的信息，因为/etc/resolve.conf中存在 nameserver 127.0.0.53 回环地址造成循环引用。对于我的机器（Ubuntu Server 20.04.3 LTS）：
ll /etc/resolv.conf
> lrwxrwxrwx 1 root root 39 Aug 24 16:42 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
# 解决方案：
mv /etc/resolv.conf /etc/resolv.conf.back
ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf

# 问题2：无法ping通 nginx-svc
# 正常现象。[k8s iptables模式 svc 不支持ping协议 分析](https://www.yoyoask.com/?p=4742)
# 使用ubuntu20.04镜像，进入容器，下载curl，执行curl，就会出现nginx网页代码
curl nginx-svc
```

---

## 11	部署nginx测试

```sh
# 1.编写yaml文件
cat >> nginx.yaml << EOF
apiVersion: apps/v1 # API 版本号
kind: Deployment # 类型，如：Pod/ReplicationController/Deployment/Service/Ingress
metadata:
  name: nginx-app # Kind 的名称
spec:
  selector:
    matchLabels:
      app: nginx # 容器标签的名字，发布 Service 时，selector 需要和这里对应
  replicas: 2 # 部署的实例数量
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers: # 配置容器，数组类型，说明可以配置多个容器
      - name: nginx # 容器名称
        image: nginx:latest # 容器镜像
        imagePullPolicy: IfNotPresent # 只有镜像不存在时，才会进行镜像拉取
        ports:
        # Pod 端口
        - containerPort: 80
---
apiVersion: v1
kind: Service # 类型是service
metadata:
  name: nginx-svc
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      nodePort: 31000 # 暴露出来可访问的port
  selector: # 后端pod标签
    app: nginx
EOF

# 2.部署
kubectl apply -f nginx.yaml

# 3.查看
kubectl get pods,svc

# 4.通过nodeip:31000 即可访问
```

---



---

## 12	常用命令

```sh
# 查看所有service account
kubectl get sa --all-namespaces

# 删除pod
kubectl get deployment --all-namespaces # 查看deployment，如果只删除pod，那么还会重启
kubectl delete deployment nginx-app # 删除deployment

kubectl attach dns-test -c dns-test -i -t
kubectl exec dns-test -- cat /etc/resolv.conf
```

---

## 13	脚本

```
# 持续显示状态
cat >> kubectl-get.sh << EOF
while((1))
do
  clear && kubectl get deployment,svc,pod,endpoints --all-namespaces -o wide
  sleep 5s
done
EOF
```

