---
title: pytest测试报告自定义比较内容
date: 2021-09-09 08:48:22
tags:
categories: python杂项
---

# pytest-html自定义比较内容

pytest通过assert进行断言，而断言产生的错误内容是python自带的错误解释。但这个错误内容长这样，是不是一脸懵。

![](Images\pytest_assert.png)

我想要的效果是像beyond compare那样：

![](Images\beyond_compare.png)

### 预备知识&资料

遇到问题，先从网上找资料，其中有几篇文章对了解pytest还是很有帮助的

资料：

https://www.cnblogs.com/yoyoketang/p/9748718.html

https://www.cnblogs.com/linuxchao/p/linuxchao-pytest-html.html

https://www.cnblogs.com/yoyoketang/p/14108144.html

但这些文章，并不是我想要的，于是只能自己从pytest-html的源码入手，定制测试报告

### 定位pytest-html源码

pytest-html的源码不多，还是很容易理解的，实现都在pytest_html/plugins.py文件中

##### 1. 分析测试报告的html

通过下图，可以看到我们要修改的测试报告的内容位于

![](Images\html_extra.png)

##### 2.定位源码

通过找生成div(class='log')的代码，可以定位到以下代码，这部分的代码就是用来生成div标签的html

```python
def _populate_html_log_div(self, log, report):
    if report.longrepr:
        # longreprtext is only filled out on failure by pytest
        #    otherwise will be None.
        #  Use full_text if longreprtext is None-ish
        #   we added full_text elsewhere in this file.
        text = report.longreprtext or report.full_text
        if html_log_content is None:
            for line in text.splitlines():
                separator = line.startswith("_ " * 10)
                if separator:
                    log.append(line[:80])
                else:
                    exception = line.startswith("E   ")
                    if exception:
                        log.append(html.span(raw(escape(line)), class_="error"))
                    else:
                        log.append(raw(escape(line)))
            log.append(html.br())
```

### 生成目标html

直接上代码， 这里用到了difflib，**不要忘记import difflib**

由于代码是通过Full diff来判断的，因此调用pytest的时候**不要忘加-vv参数选项**

```python
@staticmethod
def _my_diy_html_log_div(text):
	if text.find('Full diff') != -1:
        idx1 = text.find('AssertionError: assert') + len('AssertionError: assert')
        idx2 = text.find('E             At index')
        compare_str = text[idx1:idx2]
        left = compare_str[:compare_str.find('==')].replace('\\n','\n')
        right = compare_str[compare_str.find('==')+len('=='):].replace('\\n','\n')
        diff = difflib.HtmlDiff()
        diff_content = diff.make_file(left.splitlines(), right.splitlines())
        return diff_content
    return None
```

调用者代码如下：

```python
def _populate_html_log_div(self, log, report):
    if report.longrepr:
        text = report.longreprtext or report.full_text
        # add by liji
        html_log_content = self._my_diy_html_log_div(text)
        if html_log_content is None:
            for line in text.splitlines():
                separator = line.startswith("_ " * 10)
                if separator:
                    log.append(line[:80])
                else:
                    exception = line.startswith("E   ")
                    if exception:
                        log.append(html.span(raw(escape(line)), class_="error"))
                    else:
                        log.append(raw(escape(line)))
                log.append(html.br())
        else:
            log.append(raw(html_log_content))
```

代码最后一句：log.append(raw(html_log_content))，必须要加raw，否则报告长这样

![](Images\notraw_html.png)

这是因为log.append(html)，字符串html会被转义，'<'、'>'、'&'等会被转义

通过看py/_xmlgen.py 的源码，可以看到raw不会escape

```python
    def __object(self, obj):
        #self.write(obj)
        self.write(escape(unicode(obj)))

    def raw(self, obj):
        self.write(obj.uniobj)
```

### 最后效果展示

![](Images\finial_show.png)

格式是difflib的html格式，大致满足要求了！