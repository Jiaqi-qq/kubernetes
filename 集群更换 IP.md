# 集群更换 IP

---

| 角色 | 旧IP           | 新IP            |
| ---- | -------------- | --------------- |
| m1n1 | 192.168.30.137 | 192.168.189.128 |
| m2n2 | 192.168.30.138 | 192.168.189.129 |
| m3   | 192.168.30.139 | 192.168.189.130 |
| n3   | 192.168.30.140 | 192.168.189.131 |

---

## 1	更改ip信息

```sh
# 1.更改/etc/hosts文件
192.168.189.128 m1n1
192.168.189.129 m2n2
192.168.189.130 m3
192.168.189.131 n3

192.168.189.128 k8s-worker1
192.168.189.129 k8s-worker2
192.168.189.131 k8s-worker3

# 将hosts发送给到其他机器
scp /etc/hosts root@m2n2:/etc/hosts
```



## 2	Etcd（master）

```sh
cd ~/TLS/etcd

# 1.更换证书申请文件
vi server-csr.json

# 2.生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

# 3.拷贝etcd证书至/opt/etcd/ssl/目录中
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/

# 4.更改配置文件
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"    
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.189.128:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.89.128:2379"
 
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.189.128:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.189.128:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.189.128:2380,etcd-2=https://192.168.189.129:2380,etcd-3=https://192.168.189.130:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

# 5.将配置文件、证书拷贝到其余master中
scp -r m1n1:/opt/etcd/{cfg,ssl} /opt/
	
# 6.分别在master2和master3节点修改 ETCD_NAME、IP
vi /opt/etcd/cfg/etcd.conf

# 7.重启
systemctl restart etcd
	# 如果不能启动，可以删除/var/lib/etcd/default.etcd的信息（可能是因为etcd是之前已经存在的，但是配置文件里是new，可以改成existing试试）
	rm -r /var/lib/etcd/default.etcd/member/

# 8.查看集群状态
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.189.128:2379,https://192.168.189.129:2379,https://192.168.189.130:2379" endpoint health
```

## 3	kube-apiserver（master）

```sh
cd ~/TLS/k8s/

# 1.更换证书申请文件
vi server-csr.json

# 2.生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

# 3.拷贝到master1工作目录
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/

# 4.更改配置文件 --etcd-servers --bind-address --advertise-address=
vi /opt/kubernetes/cfg/kube-apiserver.conf

# 5.其他master拉取新配置文件并修改 --bind-address=本机IP
scp m1n1:/opt/kubernetes/cfg/kube-apiserver.conf /opt/kubernetes/cfg/

# 6.所有master重启服务
systemctl restart kube-apiserver.service
```

## 4	kubelet（worker node）

```sh
# 1.在master1更改kubeconfig.sh脚本文件
cd ~/TLS && vi kubeconfig.sh # KUBE_APISERVER
bash kubeconfig.sh # 运行脚本

# 2.拷贝bootstrap.kubeconfig、ca.pem到 所有worker节点
scp m1n1:~/TLS/bootstrap.kubeconfig /opt/kubernetes/cfg/
# 删除变更节点下ssl证书目录的kubelet-client-*文件
rm /opt/kubernetes/ssl/kubelet-client-*
systemctl restart kubelet.service

# master审批node加入集群
kubectl get csr
kubectl certificate approve ...
kubectl get node # 所有worker都应该存在了
```

## 5	kube-proxy

```sh
# 1.在master1更改kube-proxy.sh脚本文件
cd ~/TLS && vi ~/TLS/kube-proxy.sh # 修改KUBE_APISERVER
bash kube-proxy.sh # 执行脚本

# 2.kube-proxy.kubeconfig到所有worker节点
scp m1n1:~/TLS/kube-proxy.kubeconfig /opt/kubernetes/cfg/

systemctl restart kube-proxy.service
```

## 6	CNI网络

```sh
# 在 master1 节点执行
kubectl apply -f kube-flannel.yml
```

## 7	授权 apiserver 访问 kubelet

```sh
# 在 master1 节点执行
kubectl apply -f apiserver-to-kubelet-rbac.yaml

# 3.验证yaml
kubectl -n kube-system get clusterrole|grep system:kube-apiserver-to-kubelet
kubectl -n kube-system get clusterrolebinding|grep system:kube-apiserver
# 以上命令有返回结果代表apiserver授权访问成功。
```

## 8	Dashboard部署

```sh
# 1.部署
kubectl apply -f recommended.yaml

# 2.创建service account并绑定默认cluster-admin管理员集群角色：
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')

# 3.使用输出的token登录Dashboard
https://worker节点任意ip:30001
```

## 9	CoreDNS部署

```sh
# 1.部署
kubectl apply -f coredns.yaml

# 5.DNS解析测试
kubectl run -it --rm dns-test --image=busybox:1.28 sh 
/# nslookup kubernetes
```

