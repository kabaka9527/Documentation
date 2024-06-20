# 通过 Docker-Compose 启动面板

## 需先安装 Docker + Docker-compose

```bash
sudo su
curl -sSL https://get.daocloud.io/docker | sh
apt update && apt install docker-compose
```

-   现已支持 docker 容器内调用宿主机 docker 来启动 `应用实例`

    -   注意：如果要 `修改挂载目录` 只需要修改 `.env` 文件中的 `INSTALL_PATH`, 目录结尾不要有斜线!!！

-   若不修改任何配置 则您的所有数据将会保存在宿主机的 `/opt/docker-mcsm` 下

-   若您使用 unraid 搭建 docker-mcsm, 那么根据 unraid 的机制, 您的数据必须保存到 /mnt/user/appdata 下才能重启服务器不丢失数据。所以请修改 `.env` 文件中 INSTALL_PATH 为 `INSTALL_PATH=/mnt/user/appdata`。

    -   此时 docker-mcsm 的所有数据会保存到 `/mnt/user/appdata/docker-mcsm` 目录下
    
-   修改 `挂载目录` 后，需要将修改前构建的容器和镜像全部删除，否则修改不生效。

<br />

## 编写两个 Dockerfile

面板分为网页前端（Web）和守护进程后端（Daemon），所以需要先备好两个 Dockerfile 文件。

### Web

```dockerfile
FROM registry.cn-guangzhou.aliyuncs.com/kabaka/docker_build:node20
ARG INSTALL_PATH=/opt/docker-mcsm
ARG TZ=Asia/Shanghai
ENV TZ=${TZ}
ARG INSTALL_ADDRESS=https://github.com/MCSManager/MCSManager/releases/latest/download/mcsmanager_linux_release.tar.gz
ARG Package=mcsmanager_linux_release.tar.gz
RUN sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list.d/debian.sources
RUN apt update -y && apt install unzip tar wget curl -y
RUN mkdir $INSTALL_PATH
WORKDIR $INSTALL_PATH
RUN wget -O $Package $INSTALL_ADDRESS && tar vxf $Package && rm -r $Package && rm -r daemon
WORKDIR $INSTALL_PATH/web
CMD ["node", "app.js"]
```

复制并保存文件名为 `dockerfile-web` 的文件

### Daemon

```dockerfile
FROM registry.cn-guangzhou.aliyuncs.com/kabaka/docker_build:node20
ARG INSTALL_PATH=/opt/docker-mcsm
ARG TZ=Asia/Shanghai
ENV TZ=${TZ}
ARG INSTALL_ADDRESS=https://github.com/MCSManager/MCSManager/releases/latest/download/mcsmanager_linux_release.tar.gz
RUN sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list.d/debian.sources
RUN apt update -y && apt install unzip tar wget curl -y
RUN mkdir $INSTALL_PATH
WORKDIR $INSTALL_PATH
RUN wget -O $Package $INSTALL_ADDRESS && tar vxf $Package && rm -r $Package && rm -r web
WORKDIR $INSTALL_PATH/daemon
CMD ["node", "app.js"]
```

复制并保存文件名为 `dockerfile-daemon` 的文件

<br />

## Docker-compose.yml

```yml
version: "3"
services:
    mcsm-web:
        container_name: mcsm-web
        build:
            context: .
            dockerfile: dockerfile-web
            args:
                INSTALL_PATH: ${INSTALL_PATH-/opt/docker-mcsm}
                TZ: ${TZ-Asia/Shanghai}
        network_mode: "host"
        restart: always
        environment:
            - PUID=0
            - PGID=0
            - UMASK=022
        volumes:
            - ${INSTALL_PATH-/opt/docker-mcsm}/web/data:${INSTALL_PATH-/opt/docker-mcsm}/web/data
            - ${INSTALL_PATH-/opt/docker-mcsm}/web/logs:${INSTALL_PATH-/opt/docker-mcsm}/web/logs
            - ${INSTALL_PATH-/opt/docker-mcsm}/daemon/data/Config:${INSTALL_PATH-/opt/docker-mcsm}/daemon/data/Config:ro
    mcsm-daemon:
        container_name: mcsm-daemon
        build:
            context: .
            dockerfile: dockerfile-daemon
            args:
                INSTALL_PATH: ${INSTALL_PATH-/opt/docker-mcsm}
                TZ: ${TZ-Asia/Shanghai}
        network_mode: "host"
        restart: always
        environment:
            - PUID=0
            - PGID=0
            - UMASK=022
        volumes:
            - ${INSTALL_PATH-/opt/docker-mcsm}/daemon/data:${INSTALL_PATH-/opt/docker-mcsm}/daemon/data
            - ${INSTALL_PATH-/opt/docker-mcsm}/daemon/logs:${INSTALL_PATH-/opt/docker-mcsm}/daemon/logs
            - /var/run/docker.sock:/var/run/docker.sock:ro
```
如果你不想锁死mcsm的路径，可以考虑以下配置：
```yml
services:
    mcsm-web:
        #... 重复内容，省略
        volumes:
            - mcsm-web-data:${INSTALL_PATH-/opt/docker-mcsm}/web/data
            - mcsm-web-logs:${INSTALL_PATH-/opt/docker-mcsm}/web/logs
            - mcsm-web-config:${INSTALL_PATH-/opt/docker-mcsm}/daemon/data/Config:ro
    mcsm-daemon:
        #... 重复内容，省略
        volumes:
            - mcsm-daemon-data:${INSTALL_PATH-/opt/docker-mcsm}/daemon/data
            - mcsm-daemon-logs:${INSTALL_PATH-/opt/docker-mcsm}/daemon/logs
            - /var/run/docker.sock:/var/run/docker.sock:ro
Volumes:
    mcsm-web-data:
        extrnal: false
    mcsm-web-logs:
        extrnal: false
    mcsm-web-config:
        extrnal: false
    mcsm-daemon-data:
        extrnal: false
    mcsm-daemon-logs:
        extrnal: false
    mcsm-daemon-config:
        extrnal: false
```
复制并保存文件名为 `docker-compose.yml` 的文件

<br />

## .env

```.env
INSTALL_PATH=/opt/docker-mcsm
TZ=Asia/Shanghai
```

复制并保存文件名为 `.env` 的文件

<br />

### 最后

把四个文件放到一个文件夹内，您可以通过进入到这个目录

```shell

docker-compose up -d # 运行 web 和 daemon

docker-compose up -d mcsm-web # 仅运行 web

docker-compose up -d mcsm-daemon # 仅运行 daemon

```

-   发布版中不携带 java,如需运行 java 程序请在 `面板->环境镜像->环境镜像管理->新建镜像` 中自行构建

    -   实例设置中的 `进程启动方式` 选择 `虚拟化容器`

-   关闭服务器请进入到 docker-compose.yml 文件目录运行 `docker-compose stop`

    -   运行 `docker-compose down` 来移除容器

<br />

### 更新 docker-mcsm

```

docker-compose exec mcsm-web bash -c "cd ../ && wget https://github.com/MCSManager/MCSManager/releases/latest/download/mcsmanager_linux_release.tar.gz && tar vxf mcsmanager_linux_release.tar.gz && rm -r mcsmanager_linux_release.tar.gz" # 更新 web

docker-compose exec mcsm-daemon bash -c "cd ../ && wget https://github.com/MCSManager/MCSManager/releases/latest/download/mcsmanager_linux_release.tar.gz && tar vxf mcsmanager_linux_release.tar.gz && rm -r mcsmanager_linux_release.tar.gz" # 更新 daemon

docker-compose restart

```

安装包名建议一致。

基于[zijiren233](https://github.com/zijiren233/docker-mcsm)大佬原文档修改
