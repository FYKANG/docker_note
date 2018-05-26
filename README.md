# DOCKER 笔记

## 目录

1. [参考资料](#参考资料)

2. [docker简介](#docker简介)

3. [docker安装](#docker安装)

4. [使用镜像](#使用镜像)

5. [镜像定制](#镜像定制)


### 学习资料

* [Docker — 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)

### docker简介

* **什么是Docker**
  * Docker 使用 Google 公司推出的 Go 语言 进行开发实现，基于 Linux 内核的 cgroup， namespace，以及 AUFS 类的 Union FS 等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术。
* **Docker的优势**
  * 更高效的利用系统资源
  * 更快速的启动时间
  * 一致的运行环境
  * 持续交付和部署
  * 更轻松的迁移
  * 更轻松的维护和扩展

### docker安装

* **安装依赖包**

  ```linux
    $sudo yum install -y yum-utils\device-mapper-persistent-data\lvm2
  ```

* **更新 yum 软件源缓存，并安装 docker-ce**

  ```linux
  $sudo yum makecache fast
  $sudo yum install docker-ce
  ```

* **启动 Docker CE**

  ```linux
  $sudo systemctl enable docker
  $sudo systemctl start docker
  ```

* **测试 Docker 是否安装正确(输出以下信息这表示启动成功)**

  ```linux
  $ docker run hello-world
  Unable to find image 'hello-world:latest' locally
  latest: Pulling from library/hello-world
  9bb5a5d4561a: Pull complete
  Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
  Status: Downloaded newer image for hello-world:latest

  Hello from Docker!
  This message shows that your installation appears to be working correctly.

  To generate this message, Docker took the following steps:
  1. The Docker client contacted the Docker daemon.
  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
      (amd64)
  3. The Docker daemon created a new container from that image which runs the
      executable that produces the output you are currently reading.
  4. The Docker daemon streamed that output to the Docker client, which sent it
      to your terminal.

  To try something more ambitious, you can run an Ubuntu container with:
  $ docker run -it ubuntu bash

  Share images, automate workflows, and more with a free Docker ID:
  https://hub.docker.com/

  For more examples and ideas, visit:
  https://docs.docker.com/engine/userguide/
  ```

* **国内镜像加速设置**
  * Ubuntu 16.04+、Debian 8+、CentOS 7
    * 对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）

    ```json
    {
      "registry-mirrors": [
        "https://registry.docker-cn.com"
      ]
    }
    ```
    * 重新启动服务

    ```linux
    $sudo systemctl daemon-reload
    $sudo systemctl restart docker
    ```
    * 生效测试

    ```linux
    $docker info

    ```
* **注意事项**
  * 默认情况下，docker 命令会使用 Unix socket 与 Docker 引擎通讯。而只有 root 用户和 docker 组的用户才可以访问 Docker 引擎的 Unix socket。出于安全考虑，一般 Linux 系统上不会直接使用 root 用户。因此，更好地做法是将需要使用 docker 的用户加入 docker 用户组。

    * 建立 docker 组：

    ```linux
    $sudo groupadd docker

    Registry Mirrors:
    https://registry.docker-cn.com/

    ```

    * 将当前用户加入 docker 组：

    ```linux
    $sudo usermod -aG docker $USER
    ```

### 使用镜像

* **镜像获取**

```linux
$docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```

* **查询镜像**

```linux
docker image ls
```

![查询镜像](https://github.com/FYKANG/docker_note/raw/master/img/docker_image_ls.png)

* **使用Dockerfile定制镜像**
  * 在一个空白目录中，建立一个文本文件，并命名为 Dockerfile：

  ```linux
  $mkdir mynginx
  $cd mynginx
  $vi Dockerfile
  ```

  * 内容

  ```docker
  FROM nginx
  RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
  ```

  * 构建镜像
    * 在 Dockerfile 文件所在目录执行：

    ```linux
    $docker build -t nginx:v1 .
    ```

  * 注意事项
    1. 每一个run都会构建一层镜像，只构建一层镜像的多语句执行语法

    ```docker
    FROM debian:jessie

    RUN yum vim
        && yum  install git
    ```
* **Dockerfile命令**
  * COPY 复制文件 
    * COPY 指令将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。

    ```docker
    COPY <源路径>... <目标路径>
    COPY ["<源路径1>",... "<目标路径>"]
    COPY package.json /usr/src/app/
    ```

    * <目标路径> 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径

  * ADD 更高级的复制文件
    * 语法与COPY基本保持一致
    * 在下载url资源中如有tar，gzip, bzip2 以及 xz格式压缩包将会自动解压
    * 除需自动解压的场景外，并不推荐使用
    * **注意**：ADD 指令会令镜像构建缓存失效，从而可能会令镜像构建变得比较缓慢。
  * CMD 容器启动命令
    * shell 格式：CMD <命令>
    * exec 格式：CMD ["可执行文件", "参数1", "参数2"...]
    * 参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
