# kubernetes 无认证二进制部署

> Github地址：https://github.com/1849805767/kubernetes/
>
> QQ：1849805767

----

## 1	操作系统初始化

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
```

---

## 2	Etcd

```sh
# 1.下载二进制包
cd ~/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz

# 2.创建工作目录，解压二进制包
mkdir /opt/etcd/{bin,cfg} -p
tar zxvf etcd-v3.5.1-linux-amd64.tar.gz
mv etcd-v3.5.1-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

# 3.创建配置文件
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.189.132:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.189.132:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.189.132:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.189.132:2379"
ETCD_INITIAL_CLUSTER="etcd=http://192.168.189.132:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

# 4.创建etcd系统服务
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
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 5.启动并设置开机启动  --  [后续步骤不再重复]
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

## 3	下载kubernetes

```sh
# 1.下载二进制包。网址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.13.md
# 只需要下载server，里面包含了Master和worker node二进制
wget https://dl.k8s.io/v1.13.0/kubernetes-server-linux-amd64.tar.gz

# 2.将所需的组件放到/opt/下
mkdir -p /opt/kubernetes/{bin,cfg,logs}
tar zxvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes/server/bin/
cp kube-apiserver kube-scheduler kube-controller-manager kubelet kube-proxy /opt/kubernetes/bin/
cp kubectl /usr/bin
```

## 4	kube-apiserver

```sh
# 1.kube-apiserver 配置文件
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=http://192.168.189.132:2379 \
--insecure-bind-address=0.0.0.0 \
--insecure-port=8080 \
--advertise-address=192.168.189.132 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--service-node-port-range=30000-32767 \
--enable-swagger-ui=true"
EOF

# 2.systemd管理apiserve
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
```

## 5	kube-controller-manager

```sh
# 1.创建部署文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--kubeconfig=/opt/kubernetes/cfg/kubeconfig.yaml"
EOF

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
```

```sh
# kubeconfig.yaml
apiVersion: v1
kind: Config
users:
- name: client
  user:
clusters:
- name: default
  cluster:
    server: http://192.168.189.132:8080
contexts:
- context:
    cluster: default
    user: client
  name: default
current-context: default
```

## 6	kube-scheduler

```sh
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--kubeconfig=/opt/kubernetes/cfg/kubeconfig.yaml"
EOF

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
```

## 7	Docker

```sh
# 1.下载地址
wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.9.tgz 

# 2.解压二进制docker
tar zxvf docker-20.10.9.tgz 
mv docker/* /usr/bin/

# 3.systemd管理docker
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

# 4.修改镜像源
mkdir /etc/docker 
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
EOF
```

## 8	kubelet

```sh
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-node-q \
--kubeconfig=/opt/kubernetes/cfg/kubeconfig.yaml"
EOF

# 2.systemd管理kubelet
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
```

## 9	kube-proxy

```sh
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \
 --v=2 \
 --log-dir=/opt/kubernetes/logs \
 --kubeconfig=/opt/kubernetes/cfg/kubeconfig.yaml"
EOF

# 2.systemd管理kube-proxy
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
```

---

## 10	tools

```
while((1))
do
  clear && kubectl get node,deployment,svc,pod,endpoints --all-namespaces -o wide
  sleep 3s
done
```

---

![image-20220303234107624](C:\Users\Q-DELL_G7\AppData\Roaming\Typora\typora-user-images\image-20220303234107624.png)