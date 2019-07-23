# 前言
* 初入docker门的新人，容器化概念越来越流行，希望这篇小白文能帮助到想使用docker做部署的大家。其中有什么遗漏，错误欢迎大家指出!
* 这篇文章适用于简单的部署单机应用，想部署集群化的进阶操作还需要加入k8s，对这部实践完成后再发一份集群化的部署文章。

## 服务器基本信息

* 操作系统:Centos 7.4 

## 部署概要图

![部署概要图](https://github.com/FYKANG/docker_note/raw/master/img/devops.png)


## 环境搭建

### docker

 * 安装docker

    * 安装所需依赖

      ```shell
      $ sudo yum install -y yum-utils \
        device-mapper-persistent-data \
        lvm2
      ```

   * 更新稳定仓库
   
     ```shell
     $ sudo yum-config-manager \
         --add-repo \
         https://download.docker.com/linux/centos/docker-ce.repo
     ```
   
   * 安装最新版本的docker
   
     ```shell
     $ sudo yum install docker-ce docker-ce-cli containerd.io
     ```
   
   * 启动docker
   
     ```shell
     $ sudo systemctl start docker
     ```
   
   * 设置开机启动
   
     ```shell
     $ sudo systemctl enable docker
     ```

### docker-compose

* 安装docker-compse

  * 下载安装包

    ```shell
  	sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    ```

  * 赋予可执行文件权限

    ```shell
    $ sudo chmod +x /usr/local/bin/docker-compose
    ```


## docker镜像准备

* 所需镜像
  
  * gogs 
  * jenkins
  * letsencrypt-nginx-proxy-companion
  * nginx-proxy
  * open-jdk1.8
  * mysql:5.7 
  * redis

* 镜像拉取
  ```shell
  $ docker pull gogs/gogs
  ```

  ```shell
  $ docker pull jenkins
  ```

  ```shell
  $ docker pull jrcs/letsencrypt-nginx-proxy-companion
  ```

  ```shell
  $ docker pull jwilder/nginx-proxy
  ```

  ```shell
  $ docker pull openjdk:8-jdk-alpine
  ```

  ```shell
  $ docker pull mysql:5.7
  ```
  
  ```shell
  $ docker pull redis
  ```
  
* 通过docker-compse启动镜像

  * 分别创建3个文件夹comm,project,nginx-proxy里面都放docker-compose.yml文件

    ```shell
    mkdir /web/docker/comm && mkdir /web/docker/project && mkdir /web/docker/nginx-proxy
    ```

    * 配置comm下的docker-compose.yml文件

      ```yaml
      version: "3"
      services:
       redis: #service应用名称
        image: docker.io/redis #镜像地址
        container_name: redis #容器名称
        restart: always #自动重启
        command: redis-server --requirepass redis_pwd #设置redis连接密码
        ports: #映射6379端口到宿主机的6379端口
         - 6379:6379
        volumes: #挂载数据到宿主机中做数据持久化
         - /web/data/redis:/data
         - /etc/localtime:/etc/localtime
       db:
        image: docker.io/mysql:5.7
        container_name: mysql
        restart: always
        ports: #映射3306端口到宿主机的3306端口
         - 3306:3306
        environment: #设置mysql连接密码
         MYSQL_ROOT_PASSWORD: mysql_pwd 
        volumes: #挂载数据到宿主机中做数据持久化
         - /etc/localtime:/etc/localtime
         - /web/data/mysql:/var/lib/mysql
       git-gogs:
        image: gogs/gogs
        container_name: git-gogs
        restart: always
        environment: #(VIRTUAL_HOST与LETSENCRYPT_HOST填写的内容一致)
         - VIRTUAL_PORT=3000 #转发端口
         - VIRTUAL_HOST=git.xx.xx #域名转发(填上自己的域名)
         - LETSENCRYPT_HOST=git.xx.xx #ssl证书申请域名(填上自己的域名)
         - LETSENCRYPT_EMAIL=xx@xx.xx #ssl证书到期通知邮箱(填上自己的邮箱)
        volumes: #挂载数据到宿主机中做数据持久化
         - /etc/localtime:/etc/localtime
         - /web/data/gogs:/data/git/gogs-repositories
        expose: #对宿内部docker网络开放3000端口
         - 3000
      # ports: #映射3000端口到宿主机的3000端口
      #  - 3000:3000
        links: #进行容器连接,连接上面配置好的db服务别名为mysql 
         - db:mysql
       jenkins:
        image: jenkins
        container_name: jenkins
        restart: always
      # ports: #映射8080端口到宿主机的8080端口
      #  - 8080:8080
        environment:
         - VIRTUAL_PORT=8080
         - VIRTUAL_HOST=jenkins.xx.xx
         - LETSENCRYPT_HOST=jenkins.xx.xx
         - LETSENCRYPT_EMAIL=jenkins.xx.xx
        volumes: #挂载数据到宿主机中做数据持久化
         - /etc/localtime:/etc/localtime
         - /web/data/jenkins:/var/jenkins_home
      networks: # 设置内部网络nginx-proxy
        default:
          external:
            name: nginx-proxy
      
      ```
    
    * 配置nginx-proxy下的docker-compose.yml文件
    
      ```yaml
      version: "3"
      services:
       nginx-proxy:
        image: jwilder/nginx-proxy
        container_name: nginx-proxy
        ports:
         - 80:80
         - 443:443
        restart: always
        volumes:
         - html:/usr/share/nginx/html #docker内部数据卷挂载html
         - dhparam:/etc/nginx/dhparam #docker内部数据卷挂载dhparam
         - vhost:/etc/nginx/vhost.d #docker内部数据卷挂载vhost
         - certs:/etc/nginx/certs:ro #docker内部数据卷挂载certs
         - /var/run/docker.sock:/tmp/docker.sock:ro #挂载宿主机docker.sock用于监听容器启动情况
        labels: # 设置一个标签
         - "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy"
       letsencrypt-nginx-proxy-companion:
        restart: always
        image: jrcs/letsencrypt-nginx-proxy-companion
        container_name: letsencrypt-nginx-proxy-companion
        volumes:
         - /web/config/nginx:/etc/nginx/nginx.conf 
         - certs:/etc/nginx/certs:rw
         - vhost:/etc/nginx/vhost.d
         - html:/usr/share/nginx/html
         - /var/run/docker.sock:/var/run/docker.sock:ro
      volumes: #设置数据卷
        certs:
        html:
        vhost:
        dhparam:
      networks: #设置内部网络nginx-proxy
        default:
          external:
            name: nginx-proxy
      ```
    
    * project中的docker-compse.yml文件稍后结合项目再配置
    
  * 启动redis,mysql,jenkins,gogs,letsencrypt-nginx-proxy-companion,nginx-proxy容器
    
    * 挂载目录授权
    
      ```shell
      $ sudo chown -R 1000 /home/yk/data
      ```
    
    * 启动容器
    
      * 启动comm下配置的容器
    
        ```shell
        $ cd /web/docker/comm && docker-compose up -d
        ```

      * 启动nginx-proxy下的容器

        ```shell
        $ cd /web/docker/nginx-proxy && docker-compose up -d
        ```
    
## 项目Dockerfile配置

* 后端项目配置

  * 在项目根目录新建一个文件docker/dev/Dockerfile内容如下

    ```dockerfile
    FROM openjdk:8-jdk-alpine
    COPY app.jar /app/app.jar
    ENV JAVA_OPTS=""
    EXPOSE 8080
    ENV TIME_ZONE=Asia/Shanghai
    ENTRYPOINT exec java $JAVA_OPTS -jar /app/app.jar --spring.profiles.active=dev #springboot的环境配置
    ```

* 前端项目配置

  * 在项目根目录新建一个文件docker/dev/Dockerfile内容如下

    ```dockerfile
    FROM nginx
    COPY ./dist /usr/share/nginx/html
    ```

* 配置服务器中的/web/docker/proj中的docker-compose.yml文件

  ```yml
  version: "3"
  services:
   web-dev:
    image: registry.cn-shenzhen.aliyuncs.com/xx/xx #这里填写你的docker云仓库项目地址
    container_name: web-dev
    environment:
     - VIRTUAL_PORT=8080 
     - VIRTUAL_HOST=web-dev.xx.xx 
     - LETSENCRYPT_HOST=web-dev.xx.xx 
     - LETSENCRYPT_EMAIL=xx@xx.xx
    volumes:
     - /etc/localtime:/etc/localtime
    expose:
     - 8080
   web-cms-dev:
    restart: always
    image: registry.cn-shenzhen.aliyuncs.com/xx/xx #这里填写你的docker云仓库项目地址
    container_name: web-cms-dev
    environment:
     - VIRTUAL_PORT=8080 
     - VIRTUAL_HOST=web-cms-dev.xx.xx 
     - LETSENCRYPT_HOST=web-cms-dev.xx.xx 
     - LETSENCRYPT_EMAIL=xx@xx.xx
    volumes:
     - /etc/localtime:/etc/localtime
   javaee-api-dev:
    image: registry.cn-shenzhen.aliyuncs.com/xx/xx #这里填写你的docker云仓库项目地址
    container_name: javaee-api-dev
    environment:
     - MYSQL_USER=mysql_user
     - MYSQL_PWD=mysql_pwd
     - REDIS_PWD=redis_pwd
     - VIRTUAL_PORT=8080 
     - VIRTUAL_HOST=javaee-api-dev.xx.xx 
     - LETSENCRYPT_HOST=javaee-api-dev.xx.xx 
     - LETSENCRYPT_EMAIL=xx@xx.xx
    volumes:
     - /etc/localtime:/etc/localtime
    external_links:
     - redis:redis
     - db:mysql
  networks:
    default:
      external:
        name: nginx-proxy
  
  ```

  

## gogs配置

* gogs简介

  * Gogs 的目标是打造一个最简单、最快速和最轻松的方式搭建自助 Git 服务。使用 Go 语言开发使得 Gogs 能够通过独立的二进制分发，并且支持 Go 语言支持的 **所有平台**，包括 Linux、Mac OS X、Windows 以及 ARM 平台。

* 访问gogs:

  * url地址为docker-compose.yml中配置好的LETSENCRYPT_HOST域名地址git.xx.xx

  * 如果想要通过ip地址访问则需要映射出3000端口到宿主机中,

    ```yml
    ports: #映射3000端口到宿主机的3000端口
      - 3000:3000
    ```

* 首次启动gogs

  * 数据库主机:通过link进行容器连接可以填mysql:3306，如数据库不通过link方式启动填对应ip地址即可
  * 应用url和域名：填写docker-compose 中配置的ssl证书申请地址

![gogs初始化](https://github.com/FYKANG/docker_note/raw/master/img/gogs_init.png)


* 配置webhook

  * 推送地址
    * 填上jenkin访问地址
    * 注意project_token需要配合jenkin地址填写
  * 密钥文本
    * 配合jenkin填写

  ![jenkins_hook](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_hook.png)


## docker云仓库配置

* 阿里云控制台:<https://homenew.console.aliyun.com/>

* 申请docker云仓库

![ali_image](https://github.com/FYKANG/docker_note/raw/master/img/ali_image.png)


* 创建docker命名空间

![ali_namespace](https://github.com/FYKANG/docker_note/raw/master/img/ali_name_space.png)


## jenkins配置

* jenkins介绍

  * jenkins是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。

    jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序

* 访问jenkins:

  * url地址为docker-compose.yml中配置好的LETSENCRYPT_HOST域名地址jenkins.xx.xx

  * 如果想要通过ip地址访问则需要映射出8080端口到宿主机中,

    ```yml
    ports: #映射8080端口到宿主机的8080端口
      - 8080:8080
    ```

* 首次启动jenkins
  
  * 先进入容器
  
    ```shell
    $ docker exec -it jenkins /bin/bash
    ```
  
  * 查看文本内容
  
    ```shell
    $ cat /var/jenkins_home/secrets/initialAdminPassword
    ```
  
  * 将文本内容张贴到Administrator password 中点击Continue
  
  * 退出容器
  
    ```shell
    $ exit
    ```
  

 ![jenkins_key](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_key.jpg)


* 安装插件

  * 在 **Customize Jenkins** 页面内， 您可以安装任何数量的有用插件作为您初始步骤的一部分。

    两个选项可以设置:

    - **安装建议的插件** - 安装推荐的一组插件，这些插件基于最常见的用例.
    - **选择要安装的插件** - 选择安装的插件集。当你第一次访问插件选择页面时，默认选择建议的插件。

  * 进入插件安装页面，搜索点击安装即可

    * Manage Jenkins->pluginManager

  * 需要用到的jenkins插件有

    * `Docker plugin `- 用于进行docker构建
    * `Git` -git管理
    * `Gogs plugin` -配合gogs使用
    * `Maven Integration` -maven构建管理工具,构建后端项目
    * `NodeJS` -node环境工具,构建前端项目
    * `Publish Over SSH`-进行ssh连接
    * `Generic Webhook Trigger Plugin`-设置webhook触发条件

* 配置ssh

  * 进入配置页面

    Manage Jenkins->configure system

  * 配置ssh
    
    ![jenkins_key](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_ssh.png)


* 配置云docker
  
  * 使用安全的TLS方式部署
  
    ```shell
    $ openssl genrsa -aes256 -out ca-key.pem 4096       # 生成CA私钥
    #生成CA公钥，也就是证书(注意：Common Name (e.g. server FQDN or YOUR name) []:$HOST)
    $ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
    $ openssl genrsa -out server-key.pem 4096       # 生成服务器私钥
    $ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr  # 用私钥生成证书请求文件
    $ echo subjectAltName = IP:$HOST,IP:127.0.0.1 > extfile.cnf
    # 将Docker守护程序密钥的扩展使用属性设置为仅用于服务器身份验证：
    $ echo extendedKeyUsage = serverAuth >> extfile.cnf
    $ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
    ```
  
    
  
  * 配置客户端证书
  
    ```shell
    $ openssl genrsa -out key.pem 4096      # 客户端私钥
    $ openssl req -subj '/CN=client' -new -key key.pem -out client.csr      # 客户端证书请求文件
    # 要使密钥适配客户端身份验证，请创建扩展配置文件：
    $ echo extendedKeyUsage = clientAuth >> extfile.cnf
    $ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf
    # 删除证书请求文件：
    $ rm -v client.csr server.csr
    # 默认的私钥权限太开放了，为了更加的安全，我们需要更改证书的权限，删除写入权限，限制阅读权限（只有你能查看）：
    $ chmod -v 0400 ca-key.pem key.pem server-key.pem
    # 证书文件删除其写入权限：
    $ chmod -v 0444 ca.pem server-cert.pem cert.pem
    ```
  
    
  
  * 证书部署
  
    ```shell
    # centos
    $ sudo vi /etc/sysconfig/docker
        如：OPTIONS='--selinux-enabled --log-driver=journald --tlsverify=true'
        DOCKER_CERT_PATH=/etc/docker
    $ sudo service docker restart
    
    
    # daemon.json
    $ sudo vi /etc/docker/daemon.json
    {
      "tlsverify": true,
      "tlscert": "/var/docker/server-cert.pem",
      "tlskey": "/var/docker/server-key.pem",
      "tlscacert": "/var/docker/ca.pem",
      "hosts": [
        "tcp://0.0.0.0:2376",
        "unix:///var/run/docker.sock"
      ]
    }
    # 启动守护进程
    $ dockerd
    ```
  
  * 在系统管理->系统设置->cloud->	Docker Host UR填写 `tcp://主机ip:2376`
  
  * 配置 Server credentials
  
  * 选择类型Docker Host Certificate Authentication,将生成的证书如下图填写 
  
  ![jenkins_key](https://github.com/FYKANG/docker_note/raw/master/img/server_credentials.png)

* 配置node.js环境
  
  * 进入配置页面
    
    * Manage Jenkins->Global Tool Configuration
    
  * 选择需要的node.js版本
  
   ![node_config](https://github.com/FYKANG/docker_note/raw/master/img/node_config.png)

* 配置git凭证
  
  ![jenkins_docker_config](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_docker_config.png)

* 配置docker仓库登录凭证
  
  * 访问凭证可以从这里查看
  
  ![docker_login](https://github.com/FYKANG/docker_note/raw/master/img/docker_login.png)

  
  * 凭证配置
  

![docker_login](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_docker_config.png)


* 配置项目构建过程
  
  * 后端java项目配置
  
    * New Item->构建一个maven项目
  
  ![docker_login](https://github.com/FYKANG/docker_note/raw/master/img/jenkins_project_config.png)

  
  * 前端vue项目配置
  
    * 前端vue项目和后端大同小异，有2点需要改动
  
    * New Item->Freestyle project
  
    * 选择配置好的node.js环境
 ![docker_vue_config](https://github.com/FYKANG/docker_note/raw/master/img/docker_vue_config.png)

  * 执行的构建命令和后端的maven不一样使用npm进行构建
  

![docker_build_config](https://github.com/FYKANG/docker_note/raw/master/img/docker_build_config.png)


 * 其他配置按需修改即可

## 访问项目

web-dev.xx.xx 

web-cms-dev.xx.xx 

javaee-api-dev.xx.xx 