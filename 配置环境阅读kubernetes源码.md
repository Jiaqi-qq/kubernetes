# VSCode阅读kubernetes源码（WSL）

---

> 最好在Linux平台下阅读，在Windows环境中，不支持软连接，有问题。

---

## 1	配置go环境

```sh
# 1.官网：https://golang.org/dl/
wget wget https://golang.google.cn/dl/go1.17.2.linux-amd64.tar.gz
tar -C /usr/local -zxvf  go1.17.2.linux-amd64.tar.gz

# 2.设置GOPATH目录
mkdir -p /home/q/go/{src,pkg,bin}

sudo vim /etc/profile 
# 加入环境变量
export GOROOT=/usr/local/go
export GOPATH=/home/q/go
export PATH=$PATH:$GOROOT/bin

source /etc/profile

# 查看版本
go version
```

---

## 2	下载 k8s 源码

```sh
# 下载k8s源码 gitee地址：git@gitee.com:mirrors/Kubernetes.git
mkdir kubernetes && cd kubernetes
wget git@github.com:kubernetes/kubernetes.git -b release-1.18
chmod 0777 kubernetes-release-1.18.zip
unzip kubernetes-release-1.18.zip

# 软链接源码到 $GOPATH/src下
ln -sf /home/q/kubernetes/kubernetes-release-1.18/staging/src/k8s.io/ $GOPATH/src
```

