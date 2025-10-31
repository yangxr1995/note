
# image
## 拉取image
### 设置加速
#### registry-mirrors
```bash
sudo vim /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://docker.xuanyuan.me"
    ]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

#### DockerHub
```bash
# 无需修改配置，拉取时通过命令行指定 DockerHub
docker pull docker.io/library/mysql:5.7
```

#### registry-mirrors 和 DockerHub 的差别
- DockerHub：
  - Docker 官方的公共镜像仓库（https://hub.docker.com），存储了大量官方和社区开发的镜像（如 nginx、mysql 等），是 Docker 镜像的主要源头。
  - 镜像的「存储源」，所有官方和社区镜像的原始版本都托管在这里。
- registry-mirrors：
  - Docker 客户端的一项配置，用于指定一个或多个镜像加速器（代理服务器），这些加速器会缓存 DockerHub 上的镜像，用户通过它们拉取镜像时可以提高速度。
  - 镜像的「加速通道」，本身不生产镜像，而是缓存 DockerHub 的内容，用户通过它拉取镜像时：
    - 若加速器已缓存该镜像，直接从加速器下载（速度更快）；
    - 若未缓存，加速器会先从 DockerHub 拉取并缓存，再提供给用户。


```bash
# 直接使用 DockerHub：拉取镜像时无需额外配置，命令默认指向 DockerHub，例如：
docker pull mysql:5.7  # 等价于 docker pull docker.io/library/mysql:5.7
```

使用 registry-mirrors：需要在 Docker 配置文件（如 daemon.json）中指定加速器地址，之后所有 docker pull 命令会自动通过加速器拉取，例如：
```json
{
  "registry-mirrors": ["https://docker.xuanyuan.me"]
}
```
配置后，执行 `docker pull mysql:5.7` 会自动通过 `https://docker.xuanyuan.me` 代理拉取，无需手动修改镜像地址。


## 更新image
使用 docker commit 命令。
这相当于将容器当前的状态（包括所有修改）保存为一个新的镜像。
步骤如下：
查看当前运行的容器先获取要基于的容器 ID 或名称：
```bash
docker ps
输出示例：
plaintext
CONTAINER ID   IMAGE          COMMAND       CREATED       STATUS       PORTS     NAMES
a46312d5d066   original-image "/bin/bash"   2 hours ago   Up 2 hours             my-container
```
基于容器创建新镜像使用 docker commit 命令，格式：
```bash
docker commit [容器ID或名称] [新镜像名称]:[标签]
```
示例：
```bash
docker commit a46312d5d066 my-new-image:v1.0
```
验证新镜像查看本地镜像列表，确认新镜像已创建：
```bash
docker images | grep my-new-image
```
注意事项：
docker commit 会将容器的所有文件系统变更打包为新镜像，包括临时文件等，可能导致镜像体积过大。
推荐的最佳实践是使用 Dockerfile 构建镜像（可重复性更强），docker commit 仅适合临时测试场景。
如果需要添加元数据（如作者、说明），可以使用 -m 和 -a 参数：
```bash
docker commit -m "添加了自定义配置" -a "作者名称" a46312d5d066 my-new-image:v1.0
```

# container
## 管理容器
### 列出正在运行的容器
```bash
# 所有当前运行中的容器，包括容器 ID、名称、使用的镜像、启动时间等信息。
docker ps
# 查看所有容器（包括已停止的），可以使用 
docker ps -a
```

### 停止容器
```bash
docker stop <容器ID或容器名称>
```

### 删除容器
删除容器前需要确保容器处于停止状态，如果容器正在运行，需要先停止它。
```bash
# 步骤 1：停止容器
docker stop my-container
# 步骤 2：删除容器
docker rm my-container
```

### 强制删除正在运行的容器
如果需要直接删除正在运行的容器（不先停止），可以使用 -f 参数强制删除：
```bash
docker rm -f <容器ID或容器名称>
```

## 常用组合操作
```bash
# 停止所有运行中的容器：
docker stop $(docker ps -q)

# 删除所有已停止的容器：
docker rm $(docker ps -aq)

# 强制删除所有容器（包括运行中的）：
docker rm -f $(docker ps -aq)
```

## 运行 container
### 前后台运行
#### 前台运行
```bash
docker run  -it [--name <容器名>] <镜像名>
```
- 参数:
  - -i（interactive）：保持标准输入（STDIN）打开，即使没有附加到容器
  - -t（tty）：分配一个伪终端（pseudo-TTY）
- 特点：
  - 启动后会直接进入容器的交互式终端，允许你与容器内的进程进行交互（如输入命令）
- 适用场景：
  - 需要手动操作容器内部的情况，例如调试、运行需要交互的程序

示例：
```bash
# 会启动 Ubuntu 容器并直接进入 bash 终端
docker run -it ubuntu bash 
```
#### 后台运行
```bash
docker run -d：
```
- 参数:
  - -d（detached）：以分离模式运行容器，即后台运行
- 特点：
  - 容器启动后会在后台运行，不会进入容器终端，而是返回一个容器 ID
- 适用场景：
  - 运行长期服务（如 Web 服务器、数据库等），不需要实时交互

示例：
```bash
# 会在后台启动 Nginx 容器
docker run -d nginx 
```

如果使用 -d 启动后又想进入容器，可以使用 `docker exec -it <容器ID或名称> <命令>` 命令，
例如 
```bash
docker exec -it mycontainer bash。
```

### 文件挂载
要在 Docker 中运行镜像并挂载文件（或目录），可以使用 docker run 命令的 -v（或 --volume）参数实现。挂载可以让容器内访问主机的文件，或在容器停止后保留数据。
基本语法
```bash
docker run -v 主机路径:容器内路径 [其他参数] 镜像名称 [命令]
```
常用场景示例
1. 挂载主机目录到容器
将主机的 /host/path 目录挂载到容器的 /container/path 目录：
```bash
docker run -it -v /host/path:/container/path --name mycontainer 镜像名
```
容器内对 /container/path 的修改会实时同步到主机的 /host/path
如果主机路径不存在，Docker 会自动创建该目录

2. 挂载单个文件到容器
将主机的 /host/file.txt 文件挂载到容器的 /container/file.txt：
```bash
docker run -it -v /host/file.txt:/container/file.txt 镜像名 /bin/bash
```

注意：挂载单个文件时，容器内对应的路径必须是文件（不能是目录）
修改容器内的 /container/file.txt 会同步到主机的源文件

3. 只读挂载（防止容器修改主机文件）
添加 :ro 标记设置为只读：
```bash
docker run -v /host/path:/container/path:ro 镜像名
```

容器内只能读取 /container/path 的内容，无法修改

4. 使用匿名卷（自动创建数据卷）
如果只指定容器内路径，Docker 会创建匿名卷并挂载：
```bash
docker run -v /container/path 镜像名
```
匿名卷会在容器删除后保留，可通过 docker volume ls 查看

5. 使用命名卷（更易管理的持久化存储）
```bash
# 创建命名卷
docker volume create myvolume

# 挂载命名卷到容器
docker run -v myvolume:/container/path 镜像名
```

命名卷比匿名卷更容易识别和管理，适合长期保存数据

注意事项
路径格式：
Linux/macOS 使用 / 开头的绝对路径（如 /home/user/data）；
Windows 使用 /c/Users/user/data 或 C:\Users\user\data（需根据 Docker 版本调整）

权限问题：
容器内进程的用户 ID 可能与主机不同，可能需要通过 -u 参数指定用户，
或调整主机文件权限挂载目录时，容器内原目录的内容会被主机目录覆盖（但不会删除容器镜像内的文件）

通过挂载功能，可以实现容器与主机的数据共享，
或为容器提供持久化存储，是 Docker 常用的重要功能。

