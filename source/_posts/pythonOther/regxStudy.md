---
title: 正则表达式入门
date: 2021-12-03 02:22:13
tags:
---
# 正则表达式从入门到工作
最近工作中用到了很多正则表达式，多少需要记录下使用心得。但一直犹豫要不要写这篇文章，因为网上的正则表达式太多，但想到，写笔记是为了以后的方便，还是决定做吧，而且除了要写基础的，更要写一些特色出来。

## 是什么
简单来说，正则表达式就是用来对文件(字符串)进行匹配的语言(规则)，匹配到之后呢，就可以对文本进行增删改查，增删改查跟语言相关，因此不涉及这方面知识，当然示例代码用的是python

## 正则表达式的广泛应用
最开始接触正则是在shell命令下的grep、sed、awk，后来用的python、js等语言也都支持(包括C++在C11标准中也支持了正则).擅长用工具的人，还能发现像notepad、Everything等各种工具都支持正则,正则表达式就像基础功能一样，不管在编程语言中，还是工具中都有广泛的应用

## 学习资料，网站
正是如此的广泛的应用，因此网上学习资料是不缺的。
  
在线测试用具：https://tool.oschina.net/regex
  
常用正则表达式(转)：https://blog.csdn.net/sirobot/article/details/89478951
  
正则表达式原理(转)：https://zhuanlan.zhihu.com/p/107836267

## 正则表达式入门
### 基础语法
##### 1. 匹配普通文本
```python
re.search(r'hello world','example hello world!')
# <re.Match object; span=(8, 19), match='hello world'>
```
匹配任意一个字符.(除了换行符\n)
```python
re.search(r'..llo','example hello world!')
#<re.Match object; span=(8, 13), match='hello'>
```
##### 2. 匹配字符组[]
虽然叫字符组，但其实匹配的还是一个字符，只不过这个字符是一个范围
```python
# 匹配axample，bxample，cxample,[abc]表示a或b或c
re.search(r'[abc]xample','cxample hello world!')
# <re.Match object; span=(0, 7), match='cxample'>
```
如果字符很多，可以使用-来表示一个范围的字符
```python
# 匹配axample，bxample，cxample,[a-c]等价[abc]
re.search(r'[a-c]xample','cxample hello world!')
# <re.Match object; span=(0, 7), match='cxample'>
```
还可以用^来表示非的意思
```python
# 匹配axample，bxample，cxample,[^d-z]等价[abc]
re.search(r'[^d-z]xample','cxample hello world!')
# <re.Match object; span=(0, 7), match='cxample'>
```
##### 3. 转义字符 \\
转义字符主要是为了匹配一些无法表达的特殊字符，如\t,\n等。转义字符跟字符集一样，仅匹配一个字符
```python
# 这里\t匹配制表符
re.search(r'\texample','    example hello world!')
#<re.Match object; span=(0, 8), match='\texample'>
```
转义字符也可以用来表示一类字符，如\w,\d,\s等  
```python
# 这里\w 等价[a-z0-9_]
re.search(r'\w','   example hello world!')
# <re.Match object; span=(1, 2), match='e'> 
```

### 数量词
前面的基础语法都只能匹配一个字符，如果需要匹配多次，下面的数量词就派上用场了,数量词需要在字符之后
##### 1. 基础数量词
匹配0次或者无限次*\
匹配1次或者无限次+\
匹配0次或者1次？
```python
# 匹配至少一个空白字符 后接 至少一个[a-z0-9_]字符 
re.search(r'\s+\w+','example hello_world!')
# <re.Match object; span=(7, 19), match=' hello_world'>
# 匹配0个或一个空白字符 后接 至少一个[a-z0-9_]字符
re.search(r'\s?\w+','example hello_world!')
# <re.Match object; span=(0, 7), match='example'>
```
##### 2. 匹配诺干次
{m} 匹配前一个字符m次\
{n,m} 匹配前一个字符n至m次
```python
# 匹配数字 4到5次
re.search(r'\d{4,5}','1 22 333 4444')
# <re.Match object; span=(9, 13), match='4444'>
```

### 字符边界(位置匹配)
字符边界简单来说就是匹配字符在哪个位置, 但除了^和$常用，平常用的比较少。对于边界字符的理解，可以把位置看成空字符，即""
##### 1. 简单常用位置符号
^表示开头\
$表示结尾
```python
# 不匹配'11 21'
re.search(r'^\d+ 22$','11 22')
# <re.Match object; span=(0, 5), match='11 22'>
```
##### 2. 其他位置符号
\b是单词边界，就是\w和\W之间的位置\
\B是\b的反面，即非单词边界
```python
# 从结果可以看到，还包括\w和^、$之间的位置
re.sub(r'\b',"@",'1.Hello world')
# '@1@.@Hello@ @world@'
```
##### 3. 先行断言
(?=exp) 其中exp是一个子表达式，即exp前面的位置。\
(?!exp) 与上面相反
```python
# 把位置理解成"",这里就是在ll字符前面的空字符替换成@
re.sub(r'(?=ll)',"@",'1.Hello worlld')
# '1.He@llo wor@lld'
re.sub(r'(?!ll)',"@",'1.Hello worlld')
# '@1@.@H@el@l@o@ @w@o@rl@l@d@'
```

### 分组和引用
前面讲了字符组[ab], 但这只能表示一个字符, 如果想要匹配ab或者cd就无能为力了；\
又或者ab*, *只作用于一个字符，你想要把ab作为一个整体匹配多次，同样无能为力。
分组用于解决这些问题，分组也可以认为是子表达式
##### 1. 分组
(exp) 其中exp是子表达\
|式      表示左右任意匹配一个
```python
# 匹配'.'前面是数字或者字母的
re.search(r'(\d+|[a-zA-Z]+)\.','1.hello')
# <re.Match object; span=(0, 2), match='1.'>
```
##### 2. 引用
引用是为了对重复出现的文本进行匹配，要引用需要先分组\
(exp)  会自动产生编号，从1开始;\<number> 引用编号为number的分组匹配到的字符串\
(?:exp) 不会捕获，即不会有编号\
(?P\<name\>exp) 定义一个命名分组;(?P=name)引用别名为name的分组匹配到的字符串
```python
# 常用于匹配html标签
re.search(r'<([a-z]+)>.*</\1>','<span>xxx</span>')
# <re.Match object; span=(0, 16), match='<span>xxx</span>'>
# (?:span) 不捕获，没有编号
re.search(r'<(?:span)><(div)>.*</\1>','<span><div>xxx</div>')
# <re.Match object; span=(0, 20), match='<span><div>xxx</div>'>
# 使用别名
re.search(r'<(?P<name1>\w+)><(?P<name2>h[1-5])>.*</(?P=name2)></(?P=name1)>','<html><h1>xxx</h1></html>')
# <re.Match object; span=(0, 25), match='<html><h1>xxx</h1></html>'>
```

## 正则表达式不能干的事
几乎所有讲正则表达式的博客，都是在说正则表达式无所不能，这里我结合实际，把实际中无法直接用正则匹配的情况说下，当然也可能是我水平不够，写不出来。
##### 1. 不能(^abc)
字符集可以[^abc]，表示非abc的字符；但没有表示非abc字符串的表达式，(^abc),^表示的是开头
##### 2. 不能就近匹配
比方“select a from select a into b”, 我希望匹配离into最近的select，但事与愿违

## 总结
我写的内容虽然没有覆盖全正则表达式的所有语法，但作为入门足够了。正如前面写的，正则表达式在工具中也有广泛应用，在平常工作中可以多用正则表达式来查找文件，查找奇奇怪怪的内容，对工作效率提升up