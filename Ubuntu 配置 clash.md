# Ubuntu 配置 clash

```
# 下载clash for linux
# 下载最新版本 clash：https://github.com/Dreamacro/clash/releases
wget -O clash.gz https://github.com/Dreamacro/clash/releases/download/v1.7.1/clash-linux-amd64-v1.7.1.gz
 
# 解压到当前文件夹
gzip -f clash.gz -d

# 授予可执行权限
chmod +x clash

# 移动
mkdir /opt/clash
mv clash /opt/clash/clash

# 配置clash启动服务
sudo vi /etc/systemd/system/clash.service

# 初始化执行 clash 会默认在 ~/.config/clash/ 目录下生成配置文件和全球IP地址库：config.yaml 和 Country.mmdb 
# 我们指定 -d /opt/clash/
# 把下面的信息复制进去
[Unit]
Description=clash daemon

[Service]
Type=simple
User=root
ExecStart=/opt/clash/clash -d /opt/clash/
Restart=on-failure

[Install]
WantedBy=multi-user.target

# 将你的clash订阅链接yaml文件传到服务器上覆盖 /opt/clash/config.yaml

# 设置Clash开机自启
systemctl daemon-reload
systemctl enable clash
systemctl start clash

# 设置代理
cat >> /etc/profile << EOF

export http_proxy=http://localhost:7890
export https_proxy=http://localhost:7890
EOF

source /etc/profile # 生效

# 设置节点：用能访问到Ubuntu电脑的浏览器打开 http://clash.razord.top/#/proxies，即可使用
```

----

## sudo命令无法走代理

- sudo在切换成root用户时，evn不会保留这些环境变量，需要特别指定【不建议用此方法，可以先su成root用户再使用】

```
# 在 /etc/sudoers中，env_reset下添加
Defaults env_keep="http_proxy https_proxy ftp_proxy no_proxy
```

如果修改/etc/sudoers文件出现语法错误并保存后，再次执行sudo会出现错误。 这会陷入一种窘境：一方面需要root权限来再次修改，另一方面sudo却无法使用。

![image-20211026095823812](Ubuntu%20%E9%85%8D%E7%BD%AE%20clash.assets/image-20211026095823812.png)

解决方案有四种：

- 通过su，登录root。但是一些系统配置下，root是无法登录的，或者强密码设完即丢。
- 使用其它sudo类软件。但是，一般是没有安装的，而安装又需要sudo apt install ...。
- 重装系统，或者在恢复模式、U盘系统等情况下做文件操作……
- 通过pkexec visudo修改。还好一般发行版都留下了pkexec这道门。

还是用最后一种吧，其它三种都更麻烦。

---

## http://clash.razord.top/ 无响应

- [解决谷歌浏览器最新chrome94版本CORS跨域问题](https://zhuanlan.zhihu.com/p/414533145)



