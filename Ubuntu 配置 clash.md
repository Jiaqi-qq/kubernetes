# Ubuntu 配置 clash

```
# 下载clash for linux
# 下载最新版本 clash：https://github.com/Dreamacro/clash/releases
wget -O clash.gz https://github.com/Dreamacro/clash/releases/download/v1.7.1/clash-linux-amd64-v1.7.1.gz
 
# 解压到当前文件夹
gzip -f clash.gz -d

# 授予可执行权限
chmod +x clash

# 初始化执行clash
./clash

# 初始化执行 clash 会默认在 ~/.config/clash/ 目录下生成配置文件和全球IP地址库：config.yaml 和 Country.mmdb


```



