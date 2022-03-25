---
title: 数据监控(influxDb+grafana)
date: 2021-07-30 03:02:02
tags:
---

# 数据监控方案

### 背景&预备知识

​		经过一个月多月的程序运行，数据沉淀在mongodb中，接下来就要考虑统计报表了，数据统计能最直接的体现你的工作价值！

​		从网上找了一些数据监控的解决方案，得益于docker的简单部署，让我们在验证方案可行性上省去了大量时间。整体方案：1.使用docker部署InfluxDb和Grafana；2.用influxDb远程连接Mongodb，并用脚本生成influxdb数据；3.最后通过Grafana的web展示出来。

​		预备知识：

1. 了解InfluxDb，会使用python或其他语言读写influxdb数据库
2. 了解Grafana，会简单使用对应的web即可
3. 了解mongodb，由于数据源在mongodb，因此需要会从mongodb中读数据

### 环境部署

##### 1. 下载安装包

下载地址：

https://dl.grafana.com/oss/release/grafana-8.0.6-1.x86_64.rpm

https://dl.influxdata.com/influxdb/releases/influxdb-1.7.10.x86_64.rpm

##### 2. 生成docker镜像

这里我把2个软件合成了一个镜像。

启动脚本文件run-tool.sh:

```shell
service grafana-server start
influxd -config /etc/influxdb/influxdb.conf
```

dockerfile文件：

```dockerfile
FROM centos:7
COPY influxdb-1.7.10.x86_64.rpm /home/
COPY grafana-8.0.6-1.x86_64.rpm /home/
COPY run-tool.sh /
RUN cd /etc/yum.repos.d/ \
    && curl -O http://mirrors.aliyun.com/repo/Centos-7.repo  \
    && rm CentOS-Base.repo; mv Centos-7.repo CentOS-Base.repo \
    && yum clean all; yum makecache; yum -y update \
    && yum install -y /sbin/service; yum install -y fontconfig \
    && yum install -y urw-fonts \
    && cd /home/ \
    && rpm -ivh influxdb-1.7.10.x86_64.rpm \
    && rpm -ivh grafana-8.0.6-1.x86_64.rpm \
    && chmod +x /run-tool.sh
CMD /run-tool.sh
```

生成镜像命令：

```shell
docker build -t tool:monitor .
```

##### 3. 启动容器

```shell
docker run --name monitor --privileged -it -p 192.168.0.1:8086:8086 -p 192.168.0.1:3000:3000 -v /home/liji/docker/tmp:/mnt tool:monitor
```

验证是否部署成功，登录grafana的web界面查看，存在以下登录界面说明部署成功：

![](Images\deploy_grafana.png)

### 数据生成

数据生成这个环节是粘合剂，把业务生成的数据通过某种规则转成直观的统计数据，是最核心的步骤。本环节我是通过python对mongodb里的数据进行统计，仅贴部分代码：

##### 1. 连接mongodb、连接influxdb

```python
import datetime
import pymongo
from influxdb import InfluxDBClient
# 连接mongodb
url = "mongodb://" + g_mongo_ip + ':' + g_mongo_port
mongo_client = pymongo.MongoClient(url)
# 连接influxdb
influx_client = InfluxDBClient(g_influx_ip, g_influx_port, database='test')
influx_client.create_database(g_influx_database)  # 没有则创建
```

##### 2. 统计mongodb的数据(DIY)

```python
mongo_db = mongo_client[mongo_db_name]
mongo_db[mongo_col_name].find({"xxx": {"$exists": True}}).count()
```

##### 3. 写入influxdb(DIY)

```python
points = [{
        "measurement": mongo_db_name,
        "time": datetime.datetime.utcnow().isoformat("T"),
        "fields": {
            "current_problem_count":100,
            "resolve_problem_count":20      
        }
    }]
influx_client.write_points(points)
```

##### 4. 验证数据是否写入

在安装了influxdb的环境中运行influx，具体命令可以百度

```shell
[root@123106ce0db7 /]# influx
Connected to http://localhost:8086 version 1.7.10
InfluxDB shell version: 1.7.10
> show databases 
name: databases
name
----
_internal
test
```

### 界面展示

这个环节是对grafana的界面操作，我也是刚入门，基本操作如下：

##### 1. 配置数据源

![](Images\grafana_datasource.png)

选择influxdb以及选择连接地址端口

![](Images\grafana_datasetting.png)

选择要连接的数据库，不用怕错误，点击“save & test”的时候，会自动帮你测试连接情况

![](Images\grafana_datasetting2.png)

##### 2. 配置展示面板

![](Images\grafana_panel.png)

选择要展示的数据

![](Images\grafana_addData.png)

##### 3. 结果呈现

这个效果是我的效果，暂时已经符合我的预期了哈

![](Images\grafana_statics.png)

网上盗的图，参考

![](Images\grafana_other.png)

### 总结

整个数据监控部署实现差不多花了2天半时间，在这个过程中让我第一次接触influxdb时间序列数据库，也第一次体验了数据监控的方案。作为开发，使用了你以前没用过的技术，确实很爽，但也要及时总结、温故而知新。



