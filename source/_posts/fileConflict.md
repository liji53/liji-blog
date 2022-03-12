---
title: 没有登录认证下解决文件冲突方案
date: 2021-10-8 21:37:29
tags:
categories: python杂项
---

# web、后台解决文件冲突

### 需求背景

背景是之前给团队做了几个提升软件质量、提升工作效率的工具，这些工具需要通过web来修改后台程序的配置，但随着使用的人越来越多，并发的问题也越来越突出，今天我们主要解决在没有登录认证的情况下，多个用户同时操作文件冲突的问题。

### 寻找解决方法

多人同时操作文件，如何保证文件的并发控制，拿到这个问题，我们首先想到了以下几种解决方向：

1. 通知的方式：简单来说就是让其他人知道现在有人正在修改文件，请不要修改文件，并及时刷新页面
2. 冲突的方式：修改文件并提交时，判断文件的修改时间，如果读配置的时间在修改时间之后，则不允许修改
3. 文件锁的方式：类似svn提交代码的操作，在修改文件之前先对文件加锁，修改完成之后再解锁，保证原子性

经过技术评估，方案2最简单，方案1和方案3都需要花点精力。但从用户角度来说，方案3最好，因此最终选择3。再回到技术上，要实现方案3，需要考虑以下几个技术要点：

1. 服务端需要知道是哪个client在修改配置文件；

2. 什么情况下加锁，释放锁(切换文件的读写模式、切换到其他文件、关闭刷新网页)；
3. 异常情况下如何保证锁释放（断网、浏览器奔溃、加了文件锁但不在电脑前了）

### 方案设计

上面的几个技术要点，解决如下：

1. 虽然没有用户体系，但要识别客户端，可以通过cookie的方案来解决，client随机生成id，server根据client id来记录文件的锁定情况
2. 正常情况下，切换文件模式、关闭刷新网页 这些自然靠js自己判断
3. 异常情况下，我们可以参考保活机制来实现自动解锁；页面长时间不操作，则可以通过js判断

时序图如下：

![](Images\filelock_design.png)

### 代码实现

##### 1.识别客户端

由于我们不需要鉴权认证，仅仅只要能区别客户端用户就行，因此cookie可以客户端自己生成，只需要确保唯一性。关键代码如下(作为C++开发，写前端的代码，大家将就下):

```javascript
// 查询浏览器本地cookie
function get_local_cookie(){
    cook_list = document.cookie.split(';')
    for (var i = 0; i < cook_list.length; i++){
        var arr = cook_list[i].split('=')
        if (arr[0] == 'name' && arr[1].substring(0, 9) == 'autotest_'){
            return arr[1]
        }
    }
    return ''
}
// 获取cookie，没有则生成cookie，有则获取当前cookie
if (get_cookie() == ''){
    url = window.location.href + 'cgi-bin/login.cgi'
    url += "?name=autotest_" + Math.round(Math.random()*10000) + Date.parse(new Date())
    var request = new XMLHttpRequest();
    request.open("GET", url, true);
    request.send(null);
}
```

其实没必要通过服务器返回set-cookie，js可以直接生成存储cookie。后端代码login.cgi：

```python
'''
http协议交互格式：
get请求：
http://192.168.0.1:8088/cgi-bin/login.cgi?name=(autotest_random+timestamp)
response header:
'Set-Cookie': name=autotest_random+timestamp
'''
def create_cookie():
    # 获取数据
    form = cgi.FieldStorage()
    site_name = form.getvalue('name')
    # Path 需要设置为/ 否则js无法获取
    return 'Set-Cookie: name=%s; Path=/' % site_name

print ('Content-Type: text/html')
print (create_cookie())
print ('HTTPOnly: false')
print ()
```

##### 2.保活机制+页面活动检测

保活是为了在断网、网页异常的异常情况下，后台能够检测到client异常，并自动进行解锁。

这里js直接用setInterval，如果页面非活动状态会有问题

```javascript
// 后台启动定时器，发送保活包，如果长时间未保活，后台自动解锁
myWorker = new Worker("js/keeplive.js");
myWorker.postMessage(_get_url('keeplive'));
// 检查页面是否有人操作，长时间无人操作，则进行文件解锁
function eventFunc(){
    myWorker.postMessage("init");
}
var body = document.querySelector("html");
body.addEventListener("click", eventFunc);
body.addEventListener("keydown", eventFunc);
body.addEventListener("mousemove", eventFunc);
body.addEventListener("mousewheel", eventFunc);
myWorker.onmessage = function(e) {
    is_timeout = e.data
    if (is_timeout){
        cgiHttp('releaseLock')
        alert("长时间未操作，页面强制刷新！")
        location.reload()
    }
}
```

keeplive.js 主要用来发保活包，以及判断页面长时间未操作，代码如下：

```javascript
var url = ''
var init = 0
var interval = 30 * 1000         // 保活发送间隔30s
var pageTimeout = 20 * 60 * 1000 // 页面超时20分钟
var is_timeout = false
onmessage = function(e) {
    if (e.data == 'init'){
        init = 0
    }
    else{
        url = e.data
    }
}
setInterval(function(){
    if (url == '' || is_timeout == true){
        return
    }
    var request = new XMLHttpRequest();
    request.open("GET", url, true);
    request.send(null);
    init += 1
    if (init >= pageTimeout/interval){
        postMessage(true); /// 通知主线程刷新页面 
        is_timeout = true
    }
}, interval);
```

keeplive.cgi 是后台用来接收保活包的，并把包转发给管道，由喂狗程序去判断是否异常并解锁

```python
'''
http协议交互格式：
get请求：
http://10.20.147.33:8088/cgi-bin/keeplive.cgi?cook=autotest_***
response:
'''
# todo 管道没有加锁保护，并发存在问题
def _feed_dog(cookie_name):
    pipe_name = commVariable.pipe_name + 'feedDog'
    if not os.path.exists(pipe_name):
        os.mkfifo(pipe_name)
    pipe_fd = os.open(pipe_name, os.O_WRONLY)
    os.write(pipe_fd, cookie_name.encode())

form = cgi.FieldStorage()
cookie = form.getvalue('cook')
_feed_dog(cookie)

print ('Content-Type: text/html')
print ()
```

喂狗程序代码

```python
g_lock_times = 3*60 # 3分钟
lock_manager = '/tmp/filelock_manager'
pipe_name = "/tmp/pipefeedDog"
g_mutex = threading.Lock()
g_file_lock_variable = {}
# 更新内存中文件锁的时间戳
def update_lock_timestamp(cookie):
    g_mutex.acquire()
    for system in g_file_lock_variable:
        for business in g_file_lock_variable[system]:
            if cookie == g_file_lock_variable[system][business][0]:
                g_file_lock_variable[system][business][1] = int(time.time())
    g_mutex.release()
# 从文件中更新到内存中
def _update_lock_from_file(data):
    # 删除文件锁已经不存在的
    for system in list(g_file_lock_variable.keys()):
        if not data.__contains__(system):
            g_file_lock_variable.pop(system)
            continue
        for business in list(g_file_lock_variable[system].keys()):
            if not data[system].__contains__(business):
                g_file_lock_variable[system].pop(business)
    # 更新新增的文件锁
    for system in data:
        for business in data[system]:
            if not g_file_lock_variable.__contains__(system):
                g_file_lock_variable[system] = {business: [data[system][business], int(time.time())]}
                continue
            if not g_file_lock_variable[system].__contains__(business):
                g_file_lock_variable[system][business] = [data[system][business], int(time.time())]

# 删除超时的文件锁
def _delete_timeout_lock():
    isChange = False
    for system in g_file_lock_variable:
        for business in list(g_file_lock_variable[system].keys()):
            # 超时，删除该文件锁
            if (int(time.time()) - g_file_lock_variable[system][business][1]) >= g_lock_times:
                g_file_lock_variable[system].pop(business)
                isChange = True
    return isChange
# 把内存中的文件锁状态跟新到文件中
def update_file_lock():
    if not os.path.exists(lock_manager):
        return
    with open(lock_manager, 'r+', encoding="utf-8") as fd:
        try:
            data = json.load(fd)
        except:
            data = {}
        g_mutex.acquire()
        _update_lock_from_file(data)
        isChange = _delete_timeout_lock()
        if isChange:
            fd.seek(0)
            fd.truncate()
            content = {s: {b: g_file_lock_variable[s][0] for b in g_file_lock_variable[s]} for s in g_file_lock_variable}
            fd.write(json.dumps(content))
        g_mutex.release()

def check_timeout():
    update_file_lock()
    threading.Timer(int(g_lock_times/3), check_timeout).start()

# 启动检查是否文件锁是否过期的定时器
check_timeout()

# 从管道中读cookie,更新文件锁的最新情况
if not os.path.exists(pipe_name):
    os.mkfifo(pipe_name)
while True:
    pipe_fd = os.open(pipe_name, os.O_RDONLY) # 阻塞
    cook = os.read(pipe_fd, 100)
    update_lock_timestamp(cook)
    os.close(pipe_fd)
```

##### 3.关闭刷新页面，解锁

靠js判断页面关闭、刷新时发送请求

```javascript
// 关闭页面时，进行文件解锁
window.onbeforeunload = function(){
    url = window.location.href + 'cgi-bin/releaseLock.cgi'
    const formData = new FormData();
    formData.append("system", g_system)
    formData.append("subSystem", g_sub_system)
    formData.append("business", g_business)
    formData.append("fileName", g_fileName)
    formData.append("cook", g_cookie)
    window.navigator.sendBeacon(url, formData)
}
// 刷新页面时，则强制解锁该client的所有文件锁
if (performance.navigation.type == 1){
    cgiHttp('releaseLock')
    console.log(g_business)
}
```

##### 4.切换模式

配置的编辑使用了jsonEditor，只需要监听mode的切换即可

```javascript
var jsonOptions = {
    mode: 'view',
    modes: ['view', 'code', 'tree'],
    onError: function(err) {
        alert(err.toString());
    },
    onModeChange: function(newMode,oldMode){
        if ((newMode == 'code' || newMode == 'tree') && oldMode == 'view'){
            cgiHttp('getLock')
            if (g_response.hasOwnProperty('success') && g_response['success'] != true){
                alert('本文件已经被锁定，请稍后再试')
                g_jsonEditor.setMode('view')
            }
        }
        else if (newMode == 'view' && ((oldMode == 'code' || oldMode == 'tree'))){
            cgiHttp('releaseLock')
            console.log(g_response)
        }
    }
};
```

### 最终效果

![](Images\tool_demo.png)

