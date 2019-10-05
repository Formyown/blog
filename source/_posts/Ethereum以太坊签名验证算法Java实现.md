---
title: Ethereum以太坊签名验证算法Java实现
tags: [Java,以太坊,Ethereum,加密]
categories: [以太坊]
author: Formyown
date: 2019-02-16 15:41:00
mathjax: true
---
## 开始之前
在验证之前，我们需要这些基本的信息: 签名，原文，地址（或者公钥）

![](lizi.jpg)

Sig: `0x5f3c4ab309427d25cdc34a41cb432610c6c817e76ce9aec0c9336365063687ff29c92963edd6935f365d5f28c146f227881b626c7db18e87b9baf2089729741b1c`

Message: `签名这段文字以验证你是这个账户的持有者。`

Address: `0x1c52C6F743351fC9b97AE4Fe32194A588398FE69`



## 处理原文
首先我们需要将原为处理为Hash形式
```
MessageHash = keccak256(prefix + message)
```
这个`prefix`是个什么东西？

我们来看看签名在生成的时候是如何计算的。在Ethereum官方文档中，关于`eth_sig`的描述是这样的：

![原文地址: https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_sign](eth_sig.png)

可疑的地方是`sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message)))`，我们可以看到在message之前添加了一段话：`"\x19Ethereum Signed Message:\n" + len(message)`。真正被签名的数据不是单纯的message，而是在message之前添加了这句话（prefix）后整体做了hash运算。所以，我们在计算hash的时候，也需要照做。

```Java
String message = "签名这段文字以验证你是这个账户的持有者。";
byte[] msgBuffer = message.getBytes("UTF-8");
byte[] msgPrefix =  ("\u0019Ethereum Signed Message:\n" + msgBuffer.length).getBytes("UTF-8"); 
```
现在已经将原始消息和prefix都转换成了字节数组的形式，接下来需要把这两部分拼接起来
```Java
byte[] msg = new byte[msgPrefix.length + msgBuffer.length];
System.arraycopy(msgPrefix, 0, msg, 0, msgPrefix.length);
System.arraycopy(msgBuffer, 0, msg, msgPrefix.length, msgBuffer.length);
```
毫无难度，利用`System.arraycopy`方法可以轻松得拷贝数组，得到`msg`后，就可以计算`keccak256(msg)`了
```Java
import org.ethereum.crypto.cryptohash.Keccak256;

Keccak256 keccak256 = new Keccak256();
byte[] msgHash = keccak256.copy().digest(msg);
```
这里使用了`ethereumj`的加密包，当然你也可以使用其他的包或者自己实现keccak256。

> 结果：msgHash = 767cab717f7e9a3f56765c89f39887b9934a8e78ad4d0a2fb1fd0d009bb7012b (32字节，HEX)

好了，到目前为止`msgHash`已经拿到了，接下来就要处理签名了

## 处理签名
再来看一眼签名: `0x5f3c4ab309427d25cdc34a41cb432610c6c817e76ce9aec0c9336365063687ff29c92963edd6935f365d5f28c146f227881b626c7db18e87b9baf2089729741b1c`
长度是65个字节。

然而这段签名是由三部分组成的。它们分别是 `r`, `s`, `v`。`r`和`s`长度都是32字节，`v`是一个字节。下面的代码将会吧签名拆分成上述部分。
```Java
import org.bouncycastle.util.encoders.Hex;

byte[] r = Hex.decode(sig.substring(0, 64));
byte[] s = Hex.decode(sig.substring(64, 128));
byte v = Hex.decode(sig.substring(128, 130))[0];
```
> 结果：
r = 5f3c4ab309427d25cdc34a41cb432610c6c817e76ce9aec0c9336365063687ff (32字节，HEX)
s = 29c92963edd6935f365d5f28c146f227881b626c7db18e87b9baf2089729741b (32字节，HEX)
v = 28 (10进制)

得到这三个数据以后，就可以进入下一个环节辣


## 什么是椭圆曲线数字签名算法

> 若果你不想了解或者已经了解椭圆曲线的细节，可以跳过此小节

### ECDSA，EC，和Secp256k1

以太坊使用的是`Elliptic Curve Digital Signature Algorithm (ECDSA，椭圆曲线数字签名算法)`进行签名。可是`Elliptic Curve(EC，椭圆曲线)`又是个什么东西？
为了有一个直观感受，这里有一个简单但是通用的方程

$y^2 = x^3 + ax + b$

其中`a`都是常数`b`，不同的取值会形成不同的曲线方程

![各种EC的形状](ecs.png)

> 在这里https://www.desmos.com/calculator/9vv5aklebf，你可以调整`a`和`b`的值来观察不同的取值

这些方程和加密有什么联系么？当然！不过首先要了解什么是`群`以及`加法`

#### 群(Groups) 和阿贝尔群

群：若 __非空__ 集合 𝔾, 定义在 𝔾 上的一个二元运算：加法，符号`+`表示，满足：

1. 封闭性：存在$a,b\in𝔾$, $(a+b)\in𝔾$

2. 结合律：存在$a,b,c\in𝔾$, $(a+b)+c = a+(b+c)$

3. 交换律：存在$a,b\in𝔾$, $a+b = b+a$

4. 单位元：集合 𝔾 中有一个确切的值即单位元$0$，使得 $a+0=0+a=a$

5. 逆元：集合 𝔾 中每个成员都有其相反数，即$a+b=0$

则$(𝔾,+)$称为一个群，或者加法群也叫阿贝尔群。

#### 椭圆曲线上的加法

假设曲线$y^2 = x^3 + 7$上有两个点，$P$和$Q$

$P = (x_1, y_1)$

$Q = (x_2, y_2)$

![P,Q](ec_pq.png)

过 $P$ 和 $Q$ 做一条直线，相交于曲线 $R'$

![P,Q,R](ec_pqr.png)

过 $R'$ 做 $X$ 轴垂线并相交于曲线 $R$

![P,Q,R,R'](ec_pqrr'.png)


那么 __$P + Q = R$__

##### 如何计算 $P + Q$

> 访问 https://www.desmos.com/calculator/fttnxuzryp 查看更多关于计算的细节

我们已经知道了 $P$ 和 $Q$,又
$P = (x_1, y_1)$
$Q = (x_2, y_2)$
那么过这两点直线的斜率
$k = (y_2-y_1)/(x_2-x_1)$

设 $R=(x_3,y_3)$
$x_3 = k^2 - x_1 - x_2$
$y_3 = k(x_1 - x_3) - y_1$

#### 椭圆曲线和阿贝尔群
上述对于椭圆曲线上加法的描述和解析使我们得知$P$$Q$$R$都是曲线上的点，且满足阿贝尔群的定义。不难证明椭圆曲线上的加法是一个阿贝尔群。

#### 椭圆曲线上的乘法
现在我们已经知道了如何计算加法，那么如何计算$2P$的值呢？
其实$2P$就是$P + P$。还记得过两点做直线么？这里只有一个点P, 相当于P和Q重合，那么这段直线的斜率就是过P点的切线的斜率，直接求导。

$k = 3x_1^2/2y_1$

![](ec_pq_same.png)

这样带入上述公式
$R=(x_3,y_3)$
$x_3 = k^2 - x_1 - x_2$
$y_3 = k(x_1 - x_3) - y_1$
就能得出$R=2P$的值了

如何计算$3P$的值呢？
现在我们已经知道了$2P$的值，那么只需要计算$2P + P = 3P就可以啦
设$Q = 2P$
过$P$ 和 $Q$做直线，相交于$R'$, 过$R'$做$X$轴垂线相交于曲线另外一点$R$
那么$R$就是这里的$3P$的值啦

依此类推，我们可以计算$nP(n>0)$的值

#### 离散的特性

> 有空的同学务必要试一试

掏出纸和笔，设定随机点$P$画出$7P$的位置。

你会发现$1P,2P,3P....7P$的位置相差很大，没有规律可循。如果给定两点$P$以及$7P$的值，你能求出乘数$7$吗？
即便是只有7，这里的计算量也是很大的，如果给定$P$和$nP$($n$是一个$2^256$的数字),那么逆推出$n$的难度就相当大了，以人类文明现有水平，很难在短时间（几百年）计算出结果。

这就是为什么椭圆曲线数字签名算法安全性这么高了。

#### 签名过程

用户持有公钥$K$和私钥$k$, 以及原文$M$

#### 验证过程


## 签名验证过程

，更具体的,是`secp256k1`椭圆曲线。

https://en.bitcoin.it/wiki/Secp256k1

此椭圆曲线的方程是


$y^2 = x^3 + 7$


这个方程的图像看起来是这个样子：

![secp256k1图像](secp256k1.png)
https://www.desmos.com/calculator/ialhd71we3




![茫然](mangran.jpg)


## 从r,s,v中反推出公钥



