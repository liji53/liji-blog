---
title: 探索difflib(初探匹配算法)
date: 2021-09-10 08:04:57
tags:
categories: python杂项
---

# 让difflib展示更智能

继上一篇文章[pytest测试报告自定义比较内容](https://liji53.github.io/2021/09/09/pytestHtml/)，我们修改了pytest-html的源码，用difflib的html比对方式生成了新的测试报告。但用了之后发现，difflib的行比对效果极差，如果存在几处(测了下3个字符以上)不一致的地方就整行变红色，而不是只显示差异字符。如下图：

![](Images\difflib_org.png)

我想要的效果是beyond compare这种(只把差异字符标红，而并不是整行变红)

![](Images\beyond_compare.png)

网上百度difflib的实现原理，居然完全空白，于是只能自己看源代码分析原因。

首先我们知道difflib有3种差异模式："Added " ; "Changed"; "Deleted"。

现在我们要找为什么我们期望的"Changed"会变成"Added"。

### difflib源码分析

##### 1. 生成html(table)的源码

```python
def make_table(self,fromlines,tolines,fromdesc='',todesc='',context=False,numlines=5):
    # 省略......
    return table.replace('\0+','<span class="diff_add">'). \
                replace('\0-','<span class="diff_sub">'). \
                replace('\0^','<span class="diff_chg">'). \
                replace('\1','</span>'). \
                replace('\t','&nbsp;')
```

##### 2. 匹配度导致的整行变红

下面就定位到_fancy_replace 这个函数，这个函数作用是把原字符串替换成标注差异的字符串

```python
"""	
Example:
>>> d = Differ()
>>> results = d._fancy_replace(['abcDefghiJkl\n'], 0, 1,
...                            ['abcdefGhijkl\n'], 0, 1)
>>> print(''.join(results), end="")
- abcDefghiJkl
?    ^  ^  ^
+ abcdefGhijkl
?    ^  ^  ^
"""
def _fancy_replace(self, a, alo, ahi, b, blo, bhi):
    best_ratio, cutoff = 0.74, 0.75
    cruncher = SequenceMatcher(self.charjunk)
    # 省略......
    if cruncher.real_quick_ratio() > best_ratio and \
            cruncher.quick_ratio() > best_ratio and \
            cruncher.ratio() > best_ratio:
        best_ratio, best_i, best_j = cruncher.ratio(), i, j
    # 省略......
```

这里的逻辑是匹配度要达到0.74，就能展示字符差异，而不是整行差异

##### 3. 匹配度与预期不符

```python
# Where T is the total number of elements in both sequences, and
# M is the number of matches, this is 2.0*M / T.
def ratio(self):
    matches = sum(triple[-1] for triple in self.get_matching_blocks())
    return _calculate_ratio(matches, len(self.a) + len(self.b))
```

按照源码的注释，如果只有个别字符不一样，匹配度应该是很高的。在我自己的例子中，只有5个字符不一样，2个比对的字符串分别有250个字符，按照公式，匹配度应该有2\*(250-5)/(250\*2)=0.98，但实际却只有0.6多

##### 4. 匹配的块少了

这个函数会返回所有匹配的内容。Match(a=3, b=2, size=2)，表示左边字符串第3位开始，右边字符串第2位开始相同，相同字符个数为2个

```python
"""
>>> s = SequenceMatcher(None, "abxcd", "abcd")
>>> list(s.get_matching_blocks())
[Match(a=0, b=0, size=2), Match(a=3, b=2, size=2), Match(a=5, b=4, size=0)]
"""
def get_matching_blocks(self):
    # 省略......
    while queue:
        alo, ahi, blo, bhi = queue.pop()
        i, j, k = x = self.find_longest_match(alo, ahi, blo, bhi)
        if k:   # if k is 0, there was no matching block
            matching_blocks.append(x)
            if alo < i and blo < j:
                queue.append((alo, i, blo, j))
            if i+k < ahi and j+k < bhi:
                queue.append((i+k, ahi, j+k, bhi))
    matching_blocks.sort()
```

期望是所有匹配的子串都能找到返回，但实际却缺少了几个Match

##### 5. 自动垃圾启发式计算惹的祸

这个函数看字面意思是找最长的字串，但实际是有条件的。

```python
def find_longest_match(self, alo=0, ahi=None, blo=0, bhi=None):
    # 省略......
    for i in range(alo, ahi):
        # look at all instances of a[i] in b; note that because
        # b2j has no junk keys, the loop is skipped if a[i] is junk
        j2lenget = j2len.get
        newj2len = {}
        for j in b2j.get(a[i], nothing):
            # a[i] matches b[j]
            if j < blo:
                continue
            if j >= bhi:
                break
            k = newj2len[j] = j2lenget(j-1, 0) + 1
            if k > bestsize:
                besti, bestj, bestsize = i-k+1, j-k+1, k
        j2len = newj2len
    # 省略......
```

这个函数里有一个关键变量b2j， 这个变量维护了字符串中频率很低的字符，以及坐标位置。这么做的目的是为了提高运行效率，毕竟如果要比较的字符串很大，每一个字符都比较很影响效率。因此通过比较个别“冷门”字符就能快速匹配。

自动垃圾启发式计算的定义， 引用官方原文：

**SequenceMatcher支持使用启发式计算来自动将特定序列项视为垃圾。 这种启发式计算会统计每个单独项在序列中出现的次数。 如果某一项（在第一项之后）的重复次数超过序列长度的 1% 并且序列长度至少有 200 项，该项会被标记为“热门”并被视为序列匹配中的垃圾。 这种启发式计算可以通过在创建SequenceMatcher时将 `autojunk` 参数设为 `False` 来关闭。**

### 结论

原因找到了，只要匹配的字符串长度**超过200字符**，就可能匹配度变低。比较遗憾的是HtmlDiff类并没有把autojunk参数暴露出来，因此还是要通过修改源码才行, 修改如下：

```python
def _fancy_replace(self, a, alo, ahi, b, blo, bhi):
    best_ratio, cutoff = 0.74, 0.75
    #cruncher = SequenceMatcher(self.charjunk)
    cruncher = SequenceMatcher(self.charjunk, autojunk=False)
```

注意：这么改，**效率会变低**

最后上个效果图：

![](Images\finial_show.png)



