# kubernetes 无认证二进制部署

> Github地址：https://github.com/1849805767/kubernetes/
>
> QQ：1849805767

----

## 1	操作系统初始化

```shell
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config # 永久
setenforce 0 # 临时

# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
swapoff -a # 临时

# ----------------------------
# 本机 ip : 192.168.189.132
# 本机 名称 q-virtual-machine
# ----------------------------
```

---

## 2	Etcd

```shell
# 1.下载二进制包
cd ~/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz

# 2.创建工作目录，解压二进制包
mkdir /opt/etcd/{bin,cfg,data} -p
tar zxvf etcd-v3.5.1-linux-amd64.tar.gz
mv etcd-v3.5.1-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

# 3.创建配置文件
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd"
ETCD_DATA_DIR="/opt/etcd/data/default.etcd"
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

```shell
# 1.下载二进制包。网址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md
# 只需要下载server，里面包含了Master和worker node二进制
wget https://dl.k8s.io/v1.18.20/kubernetes-server-linux-amd64.tar.gz

# 2.将所需的组件放到/opt/下
mkdir -p /opt/kubernetes/{bin,cfg,logs}
tar zxvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes/server/bin/
cp kube-apiserver kube-scheduler kube-controller-manager kubelet kube-proxy /opt/kubernetes/bin/
cp kubectl /usr/bin
```

## 4	kube-apiserver

```shell
# 1.kube-apiserver 配置文件
# 重点: 准入控制器删除ServiceAccount
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=http://192.168.189.132:2379 \
--insecure-bind-address=0.0.0.0 \
--port=8080 \
--advertise-address=192.168.189.132 \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-swagger-ui=true \
--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
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

```shell
# 1.创建部署文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--master=http://192.168.189.132:8080"
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

## 6	kube-scheduler

```shell
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--master=http://192.168.189.132:8080"
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

```shell
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

```shell
# 1.创建配置文件
# pod-infra-container-image: 指定pod名称空间容器将使用的镜像, 默认k8s.gcr.io/pause:3.2无法使用
# kubeconfig: 指定如何连接到API服务器, 省略则启用独立模式
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=q-virtual-machine \
--kubeconfig=/opt/kubernetes/cfg/kubeconfig.yaml \
--pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0"
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

```shell
cat > /opt/kubernetes/cfg/kubeconfig.yaml << EOF
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
EOF
```

## 9	kube-proxy

```shell
# 1.创建配置文件
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \
 --v=2 \
 --log-dir=/opt/kubernetes/logs \
 --master=http://192.168.189.132:8080"
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

## 10	其他

```shell
# shell脚本
cat > kubectl.sh << EOF
while((1))
do
  clear && kubectl get node,deployment,svc,pod,endpoints --all-namespaces -o wide
  sleep 3s
done
EOF

# 添加可执行权限
chmod +x kubectl.sh

# nginx.yaml
cat > nginx.yaml << EOF
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
      nodePort: 30001 # 暴露出来可访问的port
  selector: # 后端pod标签
    app: nginx
EOF

# 应用nginx
kubectl apply -f nginx.yaml
```

---

![image-20220304215248008](D:\Program Files (x86)\VMware\VMware Virtual Machine\k8s_vm\笔记——kubernetes\kubernetes 无认证二进制部署.assets\image-20220304215248008-16464019715271.png)