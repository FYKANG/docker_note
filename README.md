# DOCKER 笔记

## 目录

1. [参考资料](#参考资料)

2. [docker简介](#docker简介)

3. [docker安装](#docker安装)

4. [使用镜像](#使用镜像)

5. [镜像定制](#镜像定制)

6. [容器的使用](#容器的使用)

7. [镜像的推送](#镜像的推送)


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
    * **注意**：ADD 指令会令镜像构建缓存失效，从而可能会令镜像构`建变得比较缓慢。
  * CMD 容器启动命令
    * shell 格式：`CMD <命令>`
    * exec 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
    * 参数列表格式：`CMD ["参数1", "参数2"...]`。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
    * **注意**：区分主进程，当主进程进程结束后容器退出，使用shell格式时主进程为sh
  * ENTRYPOINT 入口点
    * 用于带参启动
    * `ENTRYPOINT [ "curl", "-s", "http://ip.cn" ]`
  * ENV 设置环境变量
    * `ENV <key> <value>`
    * `ENV <key1>=<value1> <key2>=<value2>...`
  * ARG 构建参数
    * 与ENV相同用于设置环境变量
    * Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg <参数名>=<值> 来覆盖。
  * VOLUME 定义匿名卷
    * `VOLUME ["<路径1>", "<路径2>"...]`
    * `VOLUME <路径>`
    * 示例
      ```linux
      --volume /opt/gitlab/config:/etc/gitlab #将容器中的/etc/gitlab挂载到宿主机中的/opt/gitlab/config目录
      ```
    * 对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中
  * EXPOSE 声明端口
    * 格式为 `EXPOSE <端口1> [<端口2>...]`
    * **注意** 要将 EXPOSE 和在运行时使用 `-p <宿主端口>:<容器端口>` 区分开来。-p，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。
  * WORKDIR 指定工作目录
    * `WORKDIR <工作目录路径>`
    * 适用场景：进入下层容器时改变工作目录（cd命名在当层容器目录改变后进入下层容器时不会有效）
  * USER 指定当前用户
    * `USER <用户名>`
    * **注意** 这个用户必须是事先建立好的，否则无法切换。
  * HEALTHCHECK 健康检查
    * `HEALTHCHECK [选项] CMD <命令>`：设置检查容器健康状况的命令
    * `HEALTHCHECK NONE`：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
    * HEALTHCHECK 支持下列选项：
      * `--interval=<间隔>`：两次健康检查的间隔，默认为 30 秒；
      * `--timeout=<时长>`：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
      * `--retries=<次数>`：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
  * ONBUILD
    * ONBUILD 是一个特殊的指令，它后面跟的是其它指令，比如 RUN, COPY 等，而这些指令，在当前镜像构建时并不会被执行。只有当以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。

### 容器的使用

* 启动容器
  * 启动一个 bash 终端，允许用户进行交互。

  ```linux
  $docker run -t -i ubuntu:14.04
  ```

  * 其中，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， -i 则让容器的标准输入保持打开。在交互模式下，用户可以通过所创建的终端来输入命令，例如
  * 当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：
    * 检查本地是否存在指定的镜像，不存在就从公有仓库下载
    * 利用镜像创建并启动一个容器
    * 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
    * 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
    * 从地址池配置一个 ip 地址给容器
    * 执行用户指定的应用程序
    * 执行完毕后容器被终止
  * 启动已终止容器
    `docker container start`
* 后台运行
  * 通过添加 -d 参数来实现
  `docker run -d ubuntu:17.10`
  * 使用 -d 参数启动后会返回一个唯一的 id，也可以通过 `docker container ls` 命令来查看容器信息。
  * 要获取容器的输出信息，可以通过 `docker container logs` 命令。
* 终止容器
  * `docker container stop`
  * 进行交互中的容器退出使用`exit`或`Ctrl+d`
  * 终止状态的容器可以用 `docker container ls -a` 命令查看
  * 重启容器`docker container restart`
* 进入容器
  * `docker attach 243c`其中的`243c`可以通过`docker container ls`查询获的`CONTAINER ID`
  * `docker exec` 后边可以跟多个参数。
    * 只使用`-i` 参数，没有分配伪终端，无Linux 命令提示符，但命令执行结果仍然可以返回。
    * `-i` `-t`参数一起使用时，有 Linux 命令提示符
    * 更多参数可查看`docker exec --help`
    * **注意** 如果从这个 stdin 中 `exit`，不会导致容器的停止。
* 导出和导入容器
  * 导出容器 `docker export`
    ```liunx
    $ docker container ls -a
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS               NAMES
    7691a814370e        ubuntu:14.04        "/bin/bash"         36 hours ago        Exited (0) 21 hours ago                       test
    $ docker export 7691a814370e > ubuntu.tar
    ```
  * 导入容器快照 `docker import`
    ```linux
    $ cat ubuntu.tar | docker import - test/ubuntu:v1.0
    $ docker image ls
    REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
    test/ubuntu         v1.0                9d37a6082e97        About a minute ago   171.3 MB
    ```
* 删除容器
  * `docker container rm 容器名称`
  * 清理所有处于终止状态的容器 `$ docker container prune`
  
### 镜像的推送
  * 登录私有仓库(以阿里云仓库为例)
  ```
  docker login --username=***** registry.cn-shengzhen.aliyuncs.com
  ```
  * 标记 TAG
  ```
  sudo docker tag [ImageId] registry.cn-shenzhen.aliyuncs.com/ykthink/fire:[镜像版本号]
  ```
  * 推送镜像
  ```
  sudo docker push registry.cn-shenzhen.aliyuncs.com/ykthink/fire:[镜像版本号]
  ```
  
### 常用命令

* `sudo docker inspect 容器 #查看容器配置信息`
* `sudo docker logs 容器 #查看容器日志`

### 关于CentOS7配置阿里云加速镜像（用官方文档方法失败的话可以用这个方法试试）
 编辑
 ```
 vim /etc/sysconfig/docker
 ```
 修改--registry-mirror=`你的镜像地址`
 ```
 OPTIONS='--selinux-enabled --log-driver=journald --registry-mirror=http://xxx.xxx.io'
 ```
