---
title: oracle数据迁移
date: 2021-09-02 06:31:01
tags:
---

# 数据迁移总结(oracle)

此类博客太多的坑，都不完整，要不执行着就报错执行不下去了。总的来说，能不用oracle就不要用，学习成本太高。

### 方法一:客户端exp/imp(不推荐)

##### 1. 环境准备，下载工具

下载地址：

https://www.oracle.com/cn/database/technologies/instant-client/winx64-64-downloads.html

下载以下2个文件：

  instantclient-basic-windows.x64-12.2.0.1.0.zip

  instantclient-tools-windows.x64-12.2.0.1.0.zip

注意：exp和imp工具在tools里，**版本12.2.0.1.0** 上才有，

##### 2. 环境安装，配置TNS文件

解压之后，在exp、imp工具的同级目录下新增Network/Admin/tnsnames.ora文件（环境变量看自己的需求）

tnsnames.ora文件内容参考：

```shell
ora11g_test = 
    (DESCRIPTION =
        (ADDRESS_LIST =
            (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.0.1)(PORT = 1521))
        )
        (CONNECT_DATA =
            (SERVICE_NAME = helowinXDB)
        )
    )
```

##### 3.使用方法

使用之前，需要**建立好表空间、用户**，这里不详细讲

```shell
exp.exe usr/password@ora11g_test file=usr_export.dmp owner=usr
imp.exe usr/password@target_oracle file=usr_export.dmp full=y
```

##### 4. 存在问题

说实在的，报的问题实在太多，很多解决不了，下面是随便罗列的几个，因此放弃这个方法.

```
EXP-00113: Feature New Composite Partitioning Method is unsupported. 
EXP-00107: Feature (BINARY XML) of column XML_CONTENT in table
ORA-00904: "DUMMYFLAG": invalid identifier
```

### 方法二：expdp和impdp（不推荐）

经历方法一的失败，网上看到有数据泵的方式，因此再次尝试

##### 1. 环境&资料

使用数据泵需要10g以上的oracle server版本，同时需要在服务端运行，相关的教程建议看

https://hevodata.com/learn/export-data-from-oracle-using-expdp/#i1

https://docs.oracle.com/cd/E11882_01/server.112/e22490/dp_export.htm#SUTIL200

https://oracle-base.com/articles/10g/oracle-data-pump-10g#TableExpImp

##### 2. 使用方法

连接本地数据库

```
sqlplus / as sysdba
```

创建连接远程数据库(本地的话不需用)(网上很多教程没有提到这一步，如果不创建，生成的dmp文件在远程的oracle服务器上)

```sql
SQL>create public database link orcl11g connect to system identified by oracle using '(DESCRIPTION =(ADDRESS_LIST =(ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.0.2)(PORT = 1521)))(CONNECT_DATA =(SERVICE_NAME = test)))';
```

创建dmp文件本地存储路径，并赋予权限

```sql
SQL>create directory data_dir as '/home/oracle/back/data';
SQL>Grant read,write on directory data_dir to test;
```

shell下，导出数据(有多种导出模式：导整个数据库、按表空间导、按用户导、按表名导、按查询条件导)

```shell
mkdir -p /home/oracle/back/data
expdp system/system dumpfile=export.dmp directory=data_dir network_link=orcl11g schemas=test
```

导入数据（注意：**需要先建表空间、用户**，即使按整个数据库模式）

```shell
impdp test/*** dumpfile=export.dmp directory=data_dir schemas=test
```

##### 3. 问题

用了方法二，问题同样很多，有些报错解决不了。因此这个方法也放弃

```shell
ORA-14460
ORA-26059
.......
```

### 方法三: 写脚本

也试过用plsql的导出用户对象、导出表的方式，但也是动不动就出问题，而且无法实现自动化，想要灵活的定制只能自己写脚本同步数据。下面是相关代码，

工具：python

##### 1. 整体代码结构

```python
import cx_Oracle
import os
g_table_space_name_list = [['TEST1_DATA', 'test1dat.dbf', 1024],
                          ['TEST2_DATA', 'test2dat.dbf', 1024]]
                         
grant_privilege = ["CONNECT","RESOURCE","DBA","UNLIMITED TABLESPACE",
                   "select any table","create any table", "drop any table"]

g_user_name_list = [["LJ_TEST", "TEST1_DATA", grant_privilege],
                  ["LJ_TEST2", "LJ_TEST2", grant_privilege]]

exist_table_space_str = "select count(*) from dual where exists(" \
                    "select * from v$tablespace a where a.name = upper('%s'))"
create_table_space_str = "CREATE TABLESPACE %s DATAFILE " \
                         "'/home/oracle/app/oracle/oradata/helowin/%s' " \
                "SIZE %dM EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO"
exist_user_name_str = "select count(*) from dual where exists(" \
                    "select * from all_users a where a.username = upper('%s'))"
create_user_name_str = "CREATE USER %s IDENTIFIED BY test " \
                       "DEFAULT TABLESPACE %s TEMPORARY TABLESPACE TEMP"
query_table_data_str = "select * from %s.%s"
insert_table_data_str = "insert into %s.%s (%s) values (%s)"
delete_table_data_str = "truncate table %s.%s"

if __name__ == '__main__':
    client = OracleClient()
    client.create_table_space()
    client.create_user_name()
    client.create_table()
    client.sync_data()
```

##### 2.连接数据库

```python
class OracleClient:
    def __init__(self):
        os.environ['NLS_LANG'] = 'SIMPLIFIED CHINESE_CHINA.utf8'
        # 待同步的数据库
        dsn = cx_Oracle.makedsn("192.168.0.1", '1521', service_name='test')
        self.m_client = cx_Oracle.connect('system', 'oracle', dsn)
        self.m_cursor = self.m_client.cursor()
        # 数据源
        src_dsn = cx_Oracle.makedsn("192.168.0.2", '1521', service_name='test')
        self.m_src_client = cx_Oracle.connect('system', 'oracle', src_dsn)
        self.m_src_cursor = self.m_src_client.cursor()
    def __del__(self):
        self.m_cursor.close()
        self.m_client.close()
        self.m_src_cursor.close()
        self.m_src_client.close()
```

##### 3. 建立表空间

```python
    def create_table_space(self):
        for table_space_name in g_table_space_name_list:
            self.m_cursor.execute(exist_table_space_str % table_space_name[0])
            data = self.m_cursor.fetchall()
            # todo: if table space created, should drop and recreate.
            if data[0][0] != 0:
                continue
            self.m_cursor.execute(create_table_space_str %
                    (table_space_name[0], table_space_name[1], table_space_name[2]))
            self.m_client.commit()
```

##### 4. 建用户

```python
    def create_user_name(self):
        for user_name in g_user_name_list:
            self.m_cursor.execute(exist_user_name_str % user_name[0])
            data = self.m_cursor.fetchall()
            # todo: if user created, should drop and recreate.
            if data[0][0] != 0:
                continue
            self.m_cursor.execute(create_user_name_str %
                                  (user_name[0], user_name[1]))
            for privilege in user_name[2]:
                self.m_cursor.execute("GRANT %s TO %s" % (privilege, user_name[0]))
            self.m_client.commit()
```

##### 5. 建表

```python
    def _get_table_name_list(self):
        src_table_name_list = {}
        dst_table_name_list = {}
        for user_name in g_user_name_list:
            self.m_src_cursor.execute(query_table_name_str % user_name[0])
            src_data = self.m_src_cursor.fetchall()
            src_table_name_list[user_name[0]] = [x[0] for x in src_data]
            self.m_cursor.execute(query_table_name_str % user_name[0])
            data = self.m_cursor.fetchall()
            dst_table_name_list[user_name[0]] = [x[0] for x in data]
        return src_table_name_list, dst_table_name_list

    def _create_table(self, user_name, table_name):
        query_table_name_str = "select TABLE_NAME from all_tables where OWNER = '%s'"
		query_create_table_str="select dbms_metadata.get_ddl('TABLE','%s','%s') from dual"
        self.m_src_cursor.execute(query_create_table_str % (table_name, user_name))
        create_table_obj = self.m_src_cursor.fetchall()
        create_table_str = create_table_obj[0][0].read()
        self.m_cursor.execute(create_table_str)
        self.m_client.commit()

    def create_table(self):
        src_table_name_list, dst_table_name_list = self._get_table_name_list()
        for user_name in src_table_name_list:
            for table_name in src_table_name_list[user_name]:
                # todo, if table created, should recreated
                if table_name in dst_table_name_list[user_name]:
                    continue
                self._create_table(user_name, table_name)
```

##### 6. 同步数据

```python
    def _sync_data(self, user_name, table_name):
        self.m_src_cursor.execute(query_table_data_str % (user_name, table_name))
        titles = ','.join([i[0] for i in self.m_src_cursor.description])
        data_placeholder = ','.join([':'+str(i+1) for i in range(len(self.m_src_cursor.description))])
        try:
            all_table_data = self.m_src_cursor.fetchall()
        except cx_Oracle.DatabaseError as msg:
            print("\033[1;31;40mError[%s.%s]:%s \033[0m" % (user_name, table_name, msg))
            return
        # if contains old data,should drop
        self.m_cursor.execute(delete_table_data_str % (user_name, table_name))
        self.m_client.commit()
        try:
            self.m_cursor.executemany(insert_table_data_str % (
                user_name, table_name, titles, data_placeholder), all_table_data)
        except cx_Oracle.DatabaseError as msg:
            print("\033[1;31;40mError[%s.%s]:%s \033[0m" % (user_name, table_name, msg))
            return
        self.m_client.commit()

    def sync_data(self):
        src_table_name_list, dst_table_name_list = self._get_table_name_list()
        for user_name in src_table_name_list:
            for table_name in src_table_name_list[user_name]:
                if table_name not in dst_table_name_list[user_name]:
                    self._create_table(user_name, table_name)
                self._sync_data(user_name, table_name)
```

