---
title: cryptography-对称密码
date: 2021-09-20 18:40:11
tags:
categories: 密码学
---
# 对称密码(共享密码)
对称加密算法简单来说，就是把数据尽可能的打乱，并能把打乱的数据还原。

### DES
DES是以每64位为单位对明文进行加密的对称加密算法。它的密钥长度为56位(不包括8位校验码)。
##### Feistel结构
Feistel网络是一种加密设计结构，而非加密算法。很多对称加密算法都采用这种设计结构。
如下图所示(三轮)：
![](Images/des_round.png)
每一轮解密的过程：
![](Images/des_decrypt.png)
这2张图中长得像网一样的结构就叫做Feistel网络。
对于Feistel网络，我们需要知道：
1. 无论轮函数怎样实现（即使论函数不可逆),只要子密钥正确，就可以解密，因此轮函数可以设计的无比复杂。
2. Feistel网络的轮数，可以任意增加，无论经过多少轮，都能解密

##### DES算法
DES是16轮的Feistel型加密算法。
在进行16轮加密之前，DES会先对明文做一个固定的初始置换IP（其实这个置换对安全没有任何用处），等到16轮加密完成之后还会做一次逆置换。所谓的置换其实就是按照固定的规则把第N位的数据放到其他位置，把数据打乱而已。
下面我们看下DES的轮函数流程：
![](Images/des_Func.png)
这里扩展置换就是把32位的数据按照固定的位置置换表扩展成48位。
S盒变换根据固定的算法，把48位的数据变成32位，S盒也是保证安全的关键。
P盒变换就是根据另一张固定的位置置换表(做法跟第一步的初始置换IP类似)把数据打乱。
可以看到每一个步骤都需要按照固定的格式进行转化，限于篇幅，如果对它感兴趣，可以看《密码学原理与实践》。
子密码的生成过程，可以参考博客：https://www.cnblogs.com/TheFutureIsNow/p/10794182.html

##### 3DES
DES已经可以被暴力破解了，而3DES就是用于增加DES的强度，其实就是对DES重复进行3次。如图所示：
![](Images/des_3des.png)
3DES的密钥长度为3*56=168位，其中中间的步骤不是加密而是解密这是为了兼容普通的DES。
3DES的主要问题就是运行速度并不快，安全性也不如AES。

### AES
AES是目前的主流对称加密算法，有128、192、256三种密钥长度。
##### AES算法
AES并没有使用Feistel结构，但也要经过多轮加密，加密的轮数与密钥的长度相关(128位密钥需要10轮)。
AES也是分组加密，每组128位，比起DES，加密、解密的过程中可以按字节、行、列为单位进行并行计算。
每一轮的加密步骤如图所示：
![](Images/aes_encrypt.png)
其中SubBytes、ShiftRows、MixColumns的算法实现，自行参考《密码学原理与实践》

##### AES性能
AES加解密的速度与数据长度线性相关。
从2008年开始，X86架构中开始支持AES-NI指令集，这种指令集能帮助AES加解密加速。
通过命令grep aes /proc/cpuinfo可以查看cpu是否支持。如果cpu支持，自然首选AES。
而在一些嵌入式领域中，如果不支持AES硬解，可以选择ChaCha20 算法。这个算法的性能比AES更加强大。

### 分组密码的模式
分组密码的模式指的是将明文分组之后，如果明文长度超过分组长度，需要对分组进行迭代加密，从而使整个明文都加密。
分组密码的模式并不是某一种加密算法特有的，是分组密文的迭代过程。
##### ECB模式
这种模式将明文分组之后各自加密成密文分组。如图所示：
![](Images/module_ECB.png)
如果存在多个相同的明文分组，会导致生成相同的密文分组，因此ECB存在风险，实际中不要使用

##### CBC模式
CBC(Cipher Block Chaining)模式中，明文分组需要与前一个密文分组进行异或运算，如图所示：
![](Images/module_CBC.png)
CBC模式的主要问题是加密的过程必须串行，无法并行，因此在效率上比较差。

##### CFB模式
CFB(Cipher FeedBack)模式中，会对前一个密文分组进行加密，如图所示：
![](Images/module_CFB.png)
CFB可以用重放攻击，所谓的重放攻击就是利用之前的数据再次发给接收人，从而欺骗接收者。
如图所示：
![](Images/module_CFB_attack.png)
这里成功的篡改了数据，因此CFB模式也不能使用。

##### CTR模式
CTR(CounTeR)模式通过对累加计数器加密来生成密钥流。如图所示：
![](Images/module_CFB_attack.png)
由于计数器的加密过程、明文的异或计算都可以并行，因此这种模式是实现中常用的模式(GCM模式中的C指的就是这个模式)
最常见的GCM模式，我们在最后进行补充。

### 消息认证码
消息认证码(message authentication code)用于确认数据的完整性,简称MAC。
##### 散列函数
介绍MAC之前，需要知道散列函数的概念。
散列函数能将任意长度单向计算出一个固定长度的值。如图所示：
![](Images/mac_hash.png)
所谓的单向指的是无法通过计算出的值反推出原文。
一般我们把计算出来的值称为摘要、指纹
而散列函数一般也叫哈希函数，常见的散列算法有MD5、SHA-1、SHA-256、SHA-384等。
其中MD5、SHA-1 已经被破解，这里的破解指的是已经找到2个不同的数据却具有相同的散列值。
散列函数是校验数据完整性（一致性）的核心，消息认证码、数字签名、认证等都必须依赖散列函数。
校验的过程如下图所示：
![](Images/mac_hash_finger.png)

##### 对散列函数的攻击
对散列函数的攻击，这里我们主要是了解下生日攻击。
这种攻击方式并不是通过寻找特定散列值的数据，而是找两个散列值相同的2条消息，这是对单向散列函数的“强抗碰撞性”的攻击。
生日攻击的破解难度比暴力破解的难度要小很多。
举个暴力破解的例子：
你去买房，通过房产局跟房东签订了电子合同，合同里写明了100w的房款，并且双方保留一份合同的散列值。
你是一名黑客，想要把合同的100w改成10w，于是你不停的修改电子合同，不停的计算散列值，期望找到一个相同的散列值，这样就能替换原合同了。
生日攻击的例子：
你同样去买房，但这次你不通过房产局签合同，你直接跟房东签合同，你拿出事先准备好的电子合同，并双方保留散列值。
晚上你偷偷摸摸入侵电脑把电子合同换成另一份事先准备好的具有相同散列值的合同。于是房东跟你毁约了，并赔偿了你一笔违约金。。。

##### 消息认证码的校验过程
仅靠数据摘要对数据进行验证在需要通信的场景中是不够的。攻击者完全可以同时篡改你的数据+指纹。
我们先看下消息认证码的过程：
![](Images/mac_procedure.png)
它的思想其实就是利用共享密钥是只有你我两人知道这个一个特征实现的。
注意这里共享密钥并不是用于加密数据的，只是跟消息进行组合。

##### HMAC的实现
消息认证码的实现可以有多种，我们大概看下HMAC的实现：
![](Images/mac_hmac.png)
如果仅通过HMAC机制仍然存在漏洞，攻击者可以重放攻击：
![](Images/mac_repeat.png)
因此在通信的过程中，消息里常常带一个随机数nonce字段，每次通信nonce都会发生变化，于是就无法重放攻击了。

### GCM模式
最后我们再来介绍下GCM模式，前面介绍的CBC、CFB、CTR等模式只能提供加密的功能，但不具备校验数据完整性的功能。

##### 工作原理
GCM模式同时具备了2种功能（AEAD），简单来说就是 CTR模式 + GMAC机制（另一种MAC）
它的实现过程（图片来自：https://zhuanlan.zhihu.com/p/376692295）：
![](Images/gcm_impl.jpg)

### 参考资料
书籍：
《图解密码技术》
《密码学原理与实践》（第二版）
网上资料：
https://www.cnblogs.com/TheFutureIsNow/p/10794182.html
https://zhuanlan.zhihu.com/p/376692295
