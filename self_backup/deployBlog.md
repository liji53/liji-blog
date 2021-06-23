---
title: deployBlog
date: 2021-04-27 11:00:44
tags:
---

# 用docker，hexo，github部署博客

hexo+github的部署教程，网上详细教程一堆。如果你觉得安装node.js、npm以及再用npm安装各种模块(安装后还会有大量的安装残余)不爽，那欢迎参考本次教程。我们将使用docker来封装运行hexo，docker的hexo镜像非自己制作，我们网上下载

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

   安装完成之后，cmd下执行docker version，如下图所示，说明安装成功

   ![](Images\docker_versionCommand.png)

3. 修改镜像仓库的配置，最好自己申请一个阿里云的镜像，速度最快

   ![](Images\docker_imageModif.png)

### docker启动hexo

hexo的镜像，从Docker官方的公共仓库找，网址：https://hub.docker.com/search?q=hexo

我选的是spurin/hexo

1. 下载spurin/hexo镜像

   ![](Images\hexo_pull.png)

2.  创建容器，命令如图所示

   ![](Images\hexo_create.png)

   详细参数说明：

   --name   给容器指定名称，方便多个博客的管理

   -e HEXO_SERVER_PORT   指定hexo server的端口，本地预览效果时有用

   -e GIT_USER    github账户的账户名称

   -e GIT_EMAIL    github的注册email

   **-v    路径映射，把hexo的app文件夹映射到本地**

   -p   将容器内部使用的网络端口随机映射到我们使用的主机上

3.  启动容器，可能需要一段时间。会自动创建SSH密钥，并根据你的用户名和邮箱自动执行gitconfig

   ![](Images\hexo_start.png)

4.  检查--本地预览效果，出现以下图片说明部署成功了 ，网址：http://localhost:4001/

   ![](Images\hexo_localhost.png)

### 替换hexo主题

hexo的主题，我们用的是hueman，地址：https://github.com/ppoffice/hexo-theme-hueman

1. 进入容器（也可以不进入容器，后续步骤直接在windows的映射路径上操作）

   ![](Images\theme_enterbash.png)

2. 下载主题

   ![](Images\theme_download.png)

   如果出现下面这个问题(fatal: unable to access 'https://github.com/ppoffice/hexo-theme-hueman.git/': gnutls_handshake() failed: The TLS connection was non-properly terminated.)

   则该改成 **git clone git://github.com/ppoffice/hexo-theme-hueman.git themes/hueman**

3. 修改hueman的默认配置名称

   ![](Images\theme_modif.png)

4. 修改hexo的全局配置，使用hueman的主题，在windows的映射路径中修改，本例的路径是E:\blog（在容器中修改，没有vim，需要下载，下载命令 apt update && apt -y install vim）

   ![](Images\theme_config.png)

5. 检查--本地预览效果，出现以下图片说明部署成功了，网址：http://localhost:4001/

   ![](Images\theme_example.png)

   如果效果不对，可以重启容器，命令：

   docker restart liji53.github.com && docker logs --follow liji53.github.com

### 搭建github博客

首先，要注册一个github的账号，注册的邮箱必须要验证，这步就不贴图了

1. 创建仓库

   ![](Images\github_create.png)

2. 仓库的名字必须是username.github.io，其中username是你的用户名

   ![](Images\github_name.png)

### 配置SSH Key

当你本地写完博客，提交代码时，必须有github权限才可以，直接使用用户名和密码每次都要输入，很不方便。因此我们使用ssh key来解决这个问题

1. 前面搭建docker+hexo的过程中，已经给我们生成了ssh密钥，不在需要调用ssh-keygen

   密钥的路径在本地映射路径下，参考：

   ![](Images\ssh_pub.png)

2. 记事本打开`.ssh\id_rsa.pub`文件，复制里面的内容，打开你的github主页，进入个人设置->SSH and GPG keys -> New SSH key, 将刚复制的内容粘贴到key哪里，title随便填

   ![](Images\ssh_setting.png)

   ![](Images\ssh_new.png)

### 用hexo写博客并上传

到这一步，环境已经搭建完成了

1. 在windows下进入source/_posts，用markdown随便写几句

   ![](Images\blog_src.png)

2. 进入docker容器，用hexo生成博客静态网页

   ![](Images\blog_generate.png)

3. 上传博客

   ![](Images\blog_upload.png)

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

   

   



