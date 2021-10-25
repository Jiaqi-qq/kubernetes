# Ubuntu Server 20.04.3 LTS 初始配置

```
sudo passwd root # 设置root密码
apt update && apt upgrade # 更新

# 下载缺少的模块
apt install -y openssh-server net-tools 

# 修改sshd_config，允许root登录
sudo vi /etc/ssh/sshd_config # 修改PermitRootLogin为yes
service ssh restart # 重启ssh服务

# 生成密钥
ssh-keygen
```

