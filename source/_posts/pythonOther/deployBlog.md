---
title: 史上最详细-用docker部署博客
date: 2021-04-27 11:00:44
tags:
---

# 用docker，hexo，github部署博客

hexo+github的部署教程，网上详细教程一堆。如果你觉得安装node.js、npm以及再用npm安装各种模块(安装后还会有大量的安装残余)不爽，那欢迎参考本次教程。我们将使用docker来封装运行hexo，hexo镜像非自己制作，我们网上下载

### 预备知识

通过本次教程，虽然可以完成搭建，但还是建议多储备点知识，这样在遇到环境差异时，能快速排查问题。

1. 了解docker，会用常用命令；参考：https://www.runoob.com/docker/docker-tutorial.html
2. 了解hexo，会用常用命令；参考：https://hexo.io/zh-cn/docs/
3. 了解git、github、markdown

### docker安装

这里只讲win10安装docker，其他环境，请参考其他博客，网上资料很多，不怕搞不定

1. 启用Hyper，Hyper是win10自带的虚拟化技术，docker会在Hyper虚拟出来的环境上运行

   ![](Images\docker_hyper.png)

2. 安装docker，下载地址：https://docs.docker.com/docker-for-windows/install/

   一路next即可

   安装完成之后，cmd下执行docker version，如下所示，说明安装成功

   ```shell
   C:\Users\hspcadmin>docker version
   Client: Docker Engine - Community
    Cloud integration: 1.0.12
    Version:           20.10.5
    API version:       1.41
    Go version:        go1.13.15
    Git commit:        55c4c88
    Built:             Tue Mar  2 20:14:53 2021
    OS/Arch:           windows/amd64
    Context:           default
    Experimental:      true
   ```

3. 修改镜像仓库的配置，最好自己申请一个阿里云的镜像，速度最快

   ![](Images\docker_imageModif.png)

### docker启动hexo

hexo的镜像，从Docker官方的公共仓库找，网址：https://hub.docker.com/search?q=hexo

我选的是spurin/hexo (100K+下载量，说明用的人挺多的)

1. 下载spurin/hexo镜像(我已经提前下载，所以打印提示已经是最新版本)

   ```shell
   C:\Users\hspcadmin>docker pull spurin/hexo
   Using default tag: latest
   latest: Pulling from spurin/hexo
   Digest: sha256:4f8a4d8133d5b29d60b86782237673511b3c3b2081b767064d2b922b53bc2ef5
   Status: Image is up to date for spurin/hexo:latest
   docker.io/spurin/hexo:latest
   ```

2. 创建容器，命令如图所示

   ```shell
   C:\Users\hspcadmin>docker create --name=liji53.github.com -e HEXO_SERVER_PORT=4001 -e GIT_USER="liji53" -e GIT_EMAIL="liji_1211@163.com" -v E:/blog:/app -p 4001:4001 spurin/hexo
   666891af8d09cdc0443cfe157c3fcf10ab3f9739e72673cdf201a030c66d4c75
   ```

   详细参数说明：

   --name   给容器指定名称，方便多个博客的管理(非必须)

   -e HEXO_SERVER_PORT   指定hexo server的端口(注意这个是容器内部的hexo server端口)

   **-e GIT_USER    github账户的账户名称(必须与github上的账号保持一致，还没有的赶紧注册)**

   **-e GIT_EMAIL    github的注册email**

   **-v  路径映射，把hexo的app文件夹映射到本地**

   -p   将容器内部使用的网络端口随机映射到我们使用的主机上

3. 启动容器，需要一段时间。会自动初始化hexo、下载依赖的插件、创建SSH密钥等。(启动的打印很多，我这里仅展示部分)

   ```shell
   C:\Users\hspcadmin>docker start liji53.github.com && docker logs --follow liji53.github.com
   liji53.github.com
   ***** App directory exists and has content, continuing *****
   ***** App directory contains a requirements.txt file, installing npm requirements *****
   ***** App .ssh directory exists and has content, continuing *****
   ***** Running git config, user = liji53, email = liji_1211@163.com *****
   ***** Copying .ssh from App directory and setting permissions *****
   ***** Contents of public ssh key (for deploy) - *****
   ***** Starting server on port 4001 *****
   INFO  Validating config
   INFO  Start processing
   INFO  Hexo is running at http://localhost:4001 . Press Ctrl+C to stop.
   ```

4.  检查--本地预览效果，出现以下图片说明部署成功了 ，网址：http://localhost:4001/

   ![](Images\hexo_localhost.png)

### 替换hexo主题

hexo的主题，我们用的是hueman，地址：https://github.com/ppoffice/hexo-theme-hueman

1. 进入容器（也可以不进入容器，后续步骤可直接在windows的映射路径上操作）

   ```shell
   C:\Users\hspcadmin>docker exec -it  liji53.github.com bash
   root@666891af8d09:/app#
   ```

2. 下载主题

   ```shell
   root@666891af8d09:/app/themes# git clone https://github.com/ppoffice/hexo-theme-hueman.git themes/hueman
   Cloning into 'themes/hueman'...
   remote: Enumerating objects: 2272, done.
   remote: Counting objects: 100% (11/11), done.
   remote: Compressing objects: 100% (11/11), done.
   remote: Total 2272 (delta 1), reused 2 (delta 0), pack-reused 2261
   Receiving objects: 100% (2272/2272), 5.77 MiB | 37.00 KiB/s, done.
   Resolving deltas: 100% (1216/1216), done.
   Checking out files: 100% (241/241), done.
   ```

   如果出现下面这个问题(fatal: unable to access 'https://github.com/ppoffice/hexo-theme-hueman.git/': gnutls_handshake() failed: The TLS connection was non-properly terminated.)

   则该改成

    ```shell
    git clone git://github.com/ppoffice/hexo-theme-hueman.git themes/hueman
    ```

3. 修改hueman的默认配置名称

   ```shell
   root@666891af8d09:/app# mv themes/hueman/_config.yml.example  themes/hueman/_config.yml  
   ```

4. 修改hexo的全局配置，使用hueman的主题，在windows上修改，本例的文件路径是E:\blog\\_config.yml（在容器中修改，没有vim，需要下载，下载命令 apt update && apt -y install vim）

   打开_config.yml，找到theme这一行，改成hueman，其他\_config.yml的用法参考官网

   https://hexo.io/zh-cn/docs/configuration

   ```shell
   # Extensions
   ## Plugins: https://hexo.io/plugins/
   ## Themes: https://hexo.io/themes/
   theme: hueman
   ```

5. 检查--本地预览效果，出现以下图片说明部署成功了，网址：http://localhost:4001/

   ![](Images\theme_example.png)

   如果效果不对，试试重启容器，命令：

   ```shell
   docker restart liji53.github.com && docker logs --follow liji53.github.com
   ```

### 搭建github博客

首先，要注册一个github的账号，注册的邮箱必须要验证，这步就不贴图了

1. 创建仓库

   ![](Images\github_create.png)

2. 仓库的名字必须是username.github.io，其中username是你的用户名

   ![](Images\github_name.png)

### 配置SSH Key

当你本地写完博客，提交代码时，必须有github权限才可以，直接使用用户名和密码每次都要输入，很不方便。因此我们使用ssh key来解决这个问题

1. 前面搭建docker+hexo的过程中，已经提到hexo容器帮我们生成了ssh密钥，不需要调用ssh-keygen

   密钥的路径在本地映射路径下，参考：

   ![](Images\ssh_pub.png)

2. 记事本打开`.ssh\id_rsa.pub`文件，复制里面的内容，打开你的github主页，进入个人设置->SSH and GPG keys -> New SSH key, 将刚复制的内容粘贴到key哪里，title随便填

   ![](Images\ssh_setting.png)

   ![](Images\ssh_new.png)

### 用hexo写博客并上传

到这一步，环境已经搭建完成了，只剩考虑该写啥了。。。接下来我们简单验证下

1. 在容器环境下，用命令生成文件；在到windows环境下，打开test.md随便写几句

   ```shell
   root@666891af8d09:/app# hexo new test
   INFO  Validating config
   INFO  Created: /app/source/_posts/test.md
   ```

2. 进入容器，用hexo生成博客静态网页(会生成public目录，里面就是生成的网页)

   ```shell
   root@666891af8d09:/app# hexo g
   INFO  Validating config
   INFO  Start processing
   Deprecated as of 10.7.0. highlight(lang, code, ...args) has been deprecated.
   Deprecated as of 10.7.0. Please use highlight(code, options) instead.
   https://github.com/highlightjs/highlight.js/issues/2277
   INFO  Files loaded in 2.6 s
   INFO  Generated: content.json
   INFO  Generated: archives/index.html
   INFO  Generated: archives/2021/index.html
   INFO  Generated: index.html
   ```

3. 上传博客(用hexo d命令只会上传public目录，其他不会上传)

   ```shell
   root@666891af8d09:/app# hexo d
   INFO  Validating config
   INFO  Start processing
   Deprecated as of 10.7.0. highlight(lang, code, ...args) has been deprecated.
   Deprecated as of 10.7.0. Please use highlight(code, options) instead.
   https://github.com/highlightjs/highlight.js/issues/2277
   INFO  Files loaded in 2.48 s
   INFO  Generated: content.json
   INFO  Generated: archives/index.html
   ```

4. 检查--网站预览效果

   ![](Images\blog_web.png)

### 常见问题及参考

1. github访问慢、访问失败

   参考：https://zhuanlan.zhihu.com/p/15893854

2. 博客图片路径加载失败

   参考：https://blog.csdn.net/xjm850552586/article/details/84101345

3. 其他参考

   https://spurin.com/2020/01/04/Creating-a-Blog-Website-with-Docker-Hexo-Github-Free-Hosting-and-HTTPS/

   https://www.cnblogs.com/liuxianan/p/build-blog-website-by-hexo-github.html

   

   



