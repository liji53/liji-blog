---
title: 定制docker镜像
date: 2021-06-10 02:38:06
tags:
---

# 定制docker镜像

如果你写出来的应用，领导要求要考虑部署简单、可维护，这时候你应该想到docker，以我工作中的环境为例，部署需要用到httpd、mongodb、svn、python3等。但去hub docker上找相应的镜像，往往只能找到基础镜像，因此需要我们定制自己的镜像 。

### 准备工作

##### 1. 下载docker

首先需要下载docker，我的环境是centos 7.6

由于是公司服务器环境，使用curl会连接失败，但好在yum能用

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# 更新源地址
sudo yum-config-manager --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 安装Docker Engine-Community
sudo yum install docker-ce docker-ce-cli containerd.io
```

安装完成之后，启动docker

```shell
sudo systemctl start docker
```

##### 2. 准备基础镜像

我选的基础镜像是centos/httpd

```shell
docker pull centos/httpd
```

但由于环境问题，会出现连接失败，因此需要手动打包镜像

先从能上网的地方下载镜像，比方先在windows上下载镜像，并打包

```shell
C:\Users\liji>docker pull centos/httpd
C:\Users\liji>docker images
REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
centos          latest    300e315adb2f   6 months ago    209MB
liji1211/hexo   V1        f03e20ea2889   11 months ago   289MB
spurin/hexo     latest    f03e20ea2889   11 months ago   289MB
centos/httpd    latest    2cc07fbb5000   2 years ago     258MB
```

```shell
# 打包镜像
C:\Users\liji>docker save -o centos_httpd.tar 2cc07fbb5000
```

把本地打包的centos_httpd.tar 上传到服务器，并加载镜像

```shell
# 加载镜像
docker load -i centos_httpd.tar
```

### 定制镜像

定制镜像，有2种方法: 第一种是基于基础镜像，在容器中直接修改下载各种软件，然后再把容器保存成镜像即可。另一种是通过Dockerfile来定制镜像，有点像写makefile，最后通过docker build来创建镜像。下面分别介绍：

##### 使用Dockerfile定制

1. 从网上下载mongodb的安装包，放在与Dockerfile同一级目录下

   下载地址：https://www.mongodb.com/try/download/community?jmp=nav

2. 新建Dockerfile，文件内容如下：

```shell
FROM centos/httpd
ENV LANG en_US.UTF-8
COPY mongodb-org-* /home/
RUN mkdir /root/.pip/
COPY pip.conf /root/.pip/
RUN cd /etc/yum.repos.d/ \
    && curl -O http://mirrors.aliyun.com/repo/Centos-7.repo  \
    && rm CentOS-Base.repo; mv Centos-7.repo CentOS-Base.repo \
    && yum makecache; yum clean all; yum makecache; yum -y update \
    && yum install -y openssl \
    && yum install -y openssl-devel \
    && yum install -y python36 \
    && pip3 install lxml \
    && yum install -y subversion \
    && cd /home/ \
    && rpm -ivh mongodb-org-server-4.0.24-1.el6.x86_64.rpm \
    && rpm -ivh mongodb-org-shell-4.0.24-1.el6.x86_64.rpm \
    && mkdir /home/data; mkdir /home/log; touch /home/log/mongod.log 
```

3. Dockerfile命令解释：

   FROM：指定基础镜像，会先从本地镜像仓库搜索，如果本地没有，则会网上搜索

   ENV：设置系统环境变量

   RUN：执行在系统里的命令，注意：有多条命令时，尽量写成一条命令

   COPY：把本地文件拷贝到镜像里，这里pip.config内容如下

   ```shell
   [global]
   trusted-host=mirrors.aliyun.com
   index-url=http://mirrors.aliyun.com/pypi/simple/
   ```

4. 构建镜像

   ```shell
   docker build -t test:v1 .
   ```

   至此，一个安装了httpd、python3.6、mongodb、Svn的镜像就完成

##### 手动修改容器镜像

1. 基于前面的准备工作，我们先启动、进入容器

   ```shell
   docker run -it centos/httpd /bin/bash
   ```

2. 进入容器之后，首先更新yum源

   ```shell
   cd /etc/yum.repos.d/ 
   curl -O http://mirrors.aliyun.com/repo/Centos-7.repo  
   rm CentOS-Base.repo -f; mv Centos-7.repo CentOS-Base.repo
   yum makecache;yum clean all;yum makecache;yum -y update
   ```

3. 下载openssl、python3.6、svn

   ```shell
   yum install -y openssl
   yum install -y openssl-devel 
   yum install -y python36
   yum install -y subversion
   ```

4. 更新pip源

   ```shell
   mkdir /root/.pip/
   cat > /root/.pip/pip.conf << EOF
   [global]
   trusted-host=mirrors.aliyun.com
   index-url=http://mirrors.aliyun.com/pypi/simple/
   EOF
   ```

5. 退出容器，按Ctrl+P，再按Ctrl+Q；然后把mongodb的安装包拷贝到容器中

   ```shell
   # 356e587e2f04是容器id
   docker cp mongodb-org-server-4.0.24-1.el6.x86_64.rpm 356e587e2f04:/home
   docker cp mongodb-org-shell-4.0.24-1.el6.x86_64.rpm 356e587e2f04:/home
   ```

6. 再次进入容器，安装mongodb，并启动mongodb

   ```shell
   # 进入容器
   docker exec -it 356e587e2f04 /bin/bash
   # 安装mongodb
   cd /home/ 
   rpm -ivh mongodb-org-server-4.0.24-1.el6.x86_64.rpm 
   rpm -ivh mongodb-org-shell-4.0.24-1.el6.x86_64.rpm 
   mkdir /home/data; mkdir /home/log; touch /home/log/mongod.log
   # 启动mongodb，但一旦重启容器、新建容器，mongodb需要重新启动
   mongod --dbpath /home/data/ --logpath /home/log/mongod.log --fork
   ```

7. 退出容器，并制作镜像

   ```shell
   docker commit 356e587e2f04 test:v2
   ```

   至此，镜像制作完成

##### 比较2种方式

```
[root@null mongo]# docker images
REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
test           v2        3cd3fd149fd0   45 seconds ago   1.17GB
test           v1        150221daf971   4 hours ago      934MB
```

dockerfile的优势：

1. 镜像的体积会更小
2. 可复用