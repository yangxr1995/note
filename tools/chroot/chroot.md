
以ubuntu18环境为例

```bash
# 搜索所有ubuntu镜像
docker search ubuntu
# 建议拉取带应用的
docker pull ubuntu/nginx
# 创建chroot的根目录
mkdir rootfs
# 将docker镜像的文件导出到rootfs
docker export $(docker create ubuntu/nginx) | tar -C rootfs -xvf -
# 修改DNS
cp /etc/resolv.conf /root/rootfs/etc/resolv.conf
# ubuntu镜像是24版本，更换国内源需要下载24的证书
wget http://archive.ubuntu.com/ubuntu/pool/main/c/ca-certificates/ca-certificates_20241223_all.deb
# 切换当前进程的根目录
chroot rootfs bash
apt install openssl
# 安装证书
dpkg -i ca-certificates_20241223_all.deb
```

修改`/etc/apt/sources.list.d/ubuntu.sources`
```bash
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
# Types: deb-src
# URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
# Suites: noble noble-updates noble-backports
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 以下安全更新软件源包含了官方源与镜像站配置，如有需要可自行修改注释切换
Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# Types: deb-src
# URIs: http://security.ubuntu.com/ubuntu/
# Suites: noble-security
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# 预发布软件源，不建议启用

# Types: deb
# URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
# Suites: noble-proposed
# Components: main restricted universe multiverse
# Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

# # Types: deb-src
# # URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
# # Suites: noble-proposed
# # Components: main restricted universe multiverse
# # Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

```bash
apt update
```
编辑`/etc/fstab`
```bash
proc    /proc   proc    defaults    0   0
sysfs   /sys    sysfs   defaults    0   0
devpts  /dev/pts devpts  gid=5,mode=620 0 0
debugfs /sys/kernel/debug debugfs defaults 0 0
```

```bash
mount -a
```

