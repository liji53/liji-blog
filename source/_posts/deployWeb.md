---
title: 花最小的学习成本部署web服务
date: 2021-07-05 11:51:02
tags:
---

# 用httpd给程序搭个web界面

当你写完程序，需要做推广时，必不可少的需要有个界面。基于这么一个简单背景，花了5天的时间，给自己的程序搞了个web界面，由于之前没有web的实战经验，又不能花大量精力在学习web上，因此有了这篇学习记录

### 预备知识

主要储备知识还是在前端这块：

1. 了解html、css、js，html和js可以在实际过程中边学边用
2. 了解http协议，主要是出问题的时候，可以抓包快速确定是前端还是后端的问题
3. 了解httpd、cgi，主要是要部署web服务器，以及写后端脚本

### 总体思路

##### 1.明确web做什么

既然是花最小的学习成本部署web服务，那就需要清楚页面要做什么事，然后针对性的去学习。例如我这次web要做的是读写后台的配置文件。理想的服务端目录结构是这样的：

```json
{
    "系统1":{
        "子系统1":{
            "业务1":"配置文件1",
            "业务2":"配置文件2"
        }
    }
}
```

web要修改的就是“配置文件1，配置文件2”。基于这个结构，web页面需要3层导航，分别表示 “系统”、‘’子系统“、“业务”，还需要一个能显示配置，同时能修改配置的地方。

##### 2.找一个静态web页面模板

找静态的web页面是为了让我们后续动态生成html有参考模板，同时网上的web页面会比自己从零开始写的要好看，不需要自己搞CSS。

##### 3.部署服务器

这里采用httpd作为服务器，使用cgi进行交互。httpd部署过程忽略(建议直接用docker)

##### 4.确定交互协议

找到合适的模板，部署好服务之后，就要考虑如何与服务器交互了，一般采用http协议，交互的数据格式为json。具体协议设计忽略

##### 5.动态生成html页面

这里主要是通过ajax来动态更新页面。

### 找静态web页面模板

我的情况比较简单，只要能体现3层导航的web页面即可，可以从官网的demo中找

地址：https://getbootstrap.com/docs/5.0/getting-started/download/

其他专业模板(太高端，没玩过)：https://www.w3cschool.cn/msv2es/qmaj1pyd.html

由于我没有经验，找了1个小时才找到合适的模板，模板长这样：

![](Images\static_web.png)

接着去掉内容，只留下html骨架代码。可以看到三个地方留了id属性，用于后续动态生成html

```html
<!-- 最上面的导航 -->
<div class="navbar navbar-fixed-top">
    <div class="navbar-inner">
        <div class="container-fluid">
            <!-- id后面动态生成的时候用到 -->
            <div class="nav-collapse" id='header_system'></div>
        </div>
    </div>
</div>
<div class="container-fluid">
    <div class="row-fluid">
        <!-- 左边的导航栏 -->
        <div class="span3">
            <div class="well sidebar-nav" id='left_config'></div>
        </div>
        <!-- 右边配置展示 -->
        <div class="span7">
            <div style="padding: 10px 10px 10px;" id='right_config'></div>
        </div>
    </div>
</div>
```

### 部署服务

下载httpd的镜像，启动容器，命令如下：

```shell
docker pull centos/httpd
docker run --name=myHttpd -it -p 192.168.0.1:8089:80 \
  -v /home/liji/apache/html:/var/www/html/ \
  -v /home/liji/apache/cgi-bin:/var/www/cgi-bin/ \
  -v /home/liji/apache/conf:/etc/httpd/conf  \
  centos/httpd
```

修改httpd.conf，前面已经做好路径映射，直接修改/home/liji/apache/conf/httpd.conf

```xml
<Directory "/var/www/cgi-bin">
    AllowOverride None
    SetHandler  cgi-script
    Options  ExecCGI
    Require all granted
</Directory>
```

重启httpd容器

```
docker restart myHttpd
```

cgi脚本可以用shell，也可以用python， 如果用python则还需要在容器内安装python，可以参考上一篇博客

### 测试CGI

在html中随便加个能发请求的button，用于测试

```html
<button type="button" onclick="cgiHttp('test')">测试CGI</button>
```

js发送请求代码如下：

```javascript
function cgiHttp(func){
    url = window.location.href + 'cgi-bin/' + func + '.cgi&arg=test' 
    var request = new XMLHttpRequest()
    request.onreadystatechange = function() {
        if (request.readyState==4 && request.status==200){
             alert(request.responseText)
        }
    }
    request.open("GET", url, true)
    request.send(null)
}
```

web服务端，在/home/liji/apache/cgi-bin目录下新建test.cgi，内容如下：

```python
import cgi, cgitb
import json
form = cgi.FieldStorage()
# 获取数据
site_arg = form.getvalue('arg')
print("Content-type:text/html")
print()
print("cgi test success! arg = %s" % site_arg)
```

遇到问题的时候首先抓包，如果返回的是500内部服务错误，直接到/etc/httpd/logs/目录下，查看error_log文件

典型的3种错误有：

**1.权限问题**

```shell
(13)Permission denied: exec of '/var/www/cgi-bin/test.cgi' failed
```

这种可能是test.cgi 没有执行权限，chmod +x  test.cgi 解决

如果你写的cgi脚本需要修改其他目录下的文件，则还需要修改相应目录的权限，参考命令 chown -R apache:apache  /home/test

**2.执行失败**

```shell
(2)No such file or directory: exec of '/var/www/cgi-bin/previousPick.cgi' failed
```

检查test.cgi 是否存在**windows字符**，这个问题我当时花了半小时才找到。vi -b test.cgi，进入文件看有没有 ^M 字符 

**3.字符编码**

```shell
UnicodeEncodeError: 'ascii' codec can't encode characters in position 4-9: ordinal not in range(128)
```

这个问题比较恶心，在实际的环境中，脚本需要修改其他目录下的配置文件，而这个目录存在中文，就会报错。我用的是python3.6版本，直接运行脚本不会出错，但用apache用户运行就有问题，即使我在httpd.conf中加入环境变量**SetEnv PYTHONIOENCODING utf-8**依然没有解决，最后只能不用中文的目录来规避

### 协议交互

假设现在设计了4个协议，对应服务端有4个脚本，分别是getSystem.cgi、getBusiness.cgi、getConfig.cgi、setConfig.cgi，js代码如下：

```javascript
/// 根据协议的设计，拼装http请求，为了展示效果不冗余，只拼装1个协议
function _get_url(func){
    url = window.location.href + 'cgi-bin/' + func + '.cgi'
	if(func == 'getSystem'){
        url += ("?system=" + g_system)
    }else{
        alert("功能尚未实现，敬请期待！")
        return null
    }
    return url
}
function cgiHttp(func){
    url = _get_url(func) 
    if (url == null){
        return
    }
    var request = new XMLHttpRequest()
    request.onreadystatechange = function() {
        if (request.readyState==4 && request.status==200){
            response_config = JSON.parse(request.responseText)
            /// 动态加载html页面
            load_func = '_load' + func + 'Response(response_config)'
            eval(load_func)
        }
    }
    request.open("GET", url, true)
    request.send(null)
}
```

### 动态更新页面

根据前面的协议交互，我们已经能收到服务器的响应了，下面就是拼装html。先确定我们的目标长什么样：

```html
<div class="nav-collapse" id="header_system">
    <ul class="nav">
        <li class="active" id="system_system1">
            <a onclick="systemClick('system1')">system1</a></li>
        <li class="disabled" id="system_system2">
        	<a onclick="systemClick('system2')">system2</a></li>
        <li class="disabled" id="system_system3">
            <a onclick="systemClick('system3')">system3</a></li>
    </ul>
    <p class="navbar-text pull-right">contract  liji37951</p>
</div>
```

重点是class="active" 、 id="system_system1"、systemClick('system1') 这几个字段是动态的。

```javascript
function _loadgetSystemResponse(systemConfig){
    headerInnerHtml = '<ul class="nav">'
    for (var i = 0; i < systemConfig.length; i++){
        if (i == 0){ 
            headerInnerHtml += '<li class="active"'
            g_system = systemConfig[i]
        }else{
            headerInnerHtml += '<li class="disabled"'
        }
        headerInnerHtml += ' id="system_' + systemConfig[i]
                        + '"><a onclick="systemClick(\'' 
                        + systemConfig[i] + '\')">' 
                        + _toChinese(systemConfig[i])+'</a></li>'
    }
    headerInnerHtml += '</ul><p class="navbar-text pull-right">contract liji</p>'
    document.getElementById("header_system").innerHTML=headerInnerHtml
}
```

下面就是实现systemClick这个函数。

```javascript
function systemClick(system){
    if (document.getElementById("system_"+system).className == 'disabled'){
        document.getElementById("system_"+g_system).className='disabled'
        g_system = system
        document.getElementById("system_"+system).className='active'
        cgiHttp('getBusiness', g_system)
        cgiHttp('getConfig', g_business)
    }
}
```

其他左边的导航栏，右边的配置内容， 方法是一致的。

### 总结

这篇文章，无法直接拷贝代码，帮你完成web的搭建。但我写这篇文章，是为了下次遇到类似问题时提供一个思路，这次花5天时间搞定，下次能2天时间搞定，这就是这篇文章的价值。









