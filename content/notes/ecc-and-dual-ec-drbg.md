title: 椭圆曲线和Dual_EC_DRBG后门
slug: ecc-and-dual-ec-drbg
date: 2013-12-20
tags: 密码学, 椭圆曲线, 比特币, ECC, Dual_EC_DRBG, NSA

这两天爆出NSA收买RSA将NSA推荐的算法，Dual_EC_DRBG，设置为BSafe中默认的随机数生成算法。为了推广这种算法，其实早在2007年NSA就执意将其写入了NIST（美国国家标准技术管理委员会）的确定性随机数生成器推荐标准（Special Publication 800-90，SP800-90），即使它速度极慢。同年Dual_EC_DRBG就在Crypto 2007会议上被指出存在被植入后门的可能，但不能认定算法设计者是否知道后门的参数，也不能认定NIST有意在算法中植入后门。2013年9月Edward Snowden泄漏的内部备忘录显示NSA曾向某加密算法植入后门，再到12月路透社爆出RSA被收买，现在看起来是NSA精心选取参数值下后门。有意思的是，比特币也使用了ECC来生成随机数，但是比特币神秘创始人中本聪在2009年发明比特币的时采用了小众的secp256k1的曲线参数而不是有问题的secp256r1（NIST P-256），神奇地躲过了密码学子弹。

那么Dual_EC_DRBG漏洞在哪，为什么随机数生成器的漏洞会让加密算法被攻破？先要从椭圆曲线和随机数生成器说起。

椭圆曲线
=====

椭圆曲线密码学（Elliptic curve cryptography，ECC）由Koblitz和Miller分别独立提出，是迄今为止安全性最高的一种公钥密码算法。椭圆曲线E的形式是：

$$y^2=x^3+ax+b, a,b \in F_p$$

有限域Fq，q一般是2的m次方或大素数p，也可以是某素数p的m次方。参数a,b确定椭圆曲线E。

它的安全性基于有限域椭圆曲线离散对数问题(Elliptic Curve Discrete Logarithm Problem,ECDLP）的难解性：

>给定素数p和椭圆曲线E，对Q=dG，在已知G，Q的情况下求出小于p的正整数d。可以证明由d和G计算Q容易，而由Q和G计算d则是指数级的难度。

G对应为椭圆曲线E的基点，d为私钥由自己选择，然后生成对应公钥Q=dG，得到密钥对(d, Q)，私钥自己保存，(p, a, b, G, Q)是已知的，但是由p和G计算私钥d很难，保证了加密算法的强度。

还有2种公钥密码体制分别基于大整数分解问题(Integer Factorization Problem,IFP)的难解性，如RSA，和有限域离散对数问题(Discrete Logarithm Problem,DLP)的难解性，如Diffie-Hellman、ElGamal。但是ECC问题要比DLP问题更难解，并且ECC用更少的密钥长度提供相同的加密强度。

这次出问题的Dual_EC_DRBG，全称Dual Elliptic Curve Deterministic Random Bit Generation，是基于椭圆曲线的确定性随机数生成器。

随机数生成器
=====

随机数在密码学中具有十分重要的地位，被广泛用于密钥产生、初始化向量、时间戳、认证挑战码、密钥协商、大素数产生等等方面。随机数产生器用于产生随机数，分为非确定性随机数生成器和确定性随机数生成器两类：

非确定性随机数生成器(NRBG)基于完全不可预测的物理过程。确定性随机数生成器（伪随机数生成器，DRBG）在指定初始种子之后，利用某些算法产生一个确定的比特序列，“确定性”因此得名。如果种子是保密的而且算法设计得很完善，那么可以认为输出的比特序列是不可预测的。

加密算法需要高度随机性的“种子”来生成密钥，因此随机数生成器会直接影响加密强度和安全性，脆弱的随机数生成器被攻破会击溃整个加密算法，比如Android随机生成数字串安全密钥中的漏洞就导致了部分比特币的失窃。

Dual_EC_DRBG后门
=====

这个后门的发现可以追溯到六年前，NIST在2007年发布了确定性随机数产生器推荐标准SP800-90修订版。这是一个推荐性的标准，描述了一些产生随机比特流的方法，这些随机比特流可能被用于直接或者间接的产生随机数，并用于相关使用密码学的应用中。NIST SP800-90提供了4种标准算法：Hash_DRBG（基于SHA系列算法）、HMAC_DRBG（基于HMAC函数）、CTR_DRBG(基于分组密码函数AES)、Dual_EC_DRBG（基于椭圆曲线）。

在4个月后的Crypto 2007会议上，Dan Shumow和Niels Fergusonsuo作了一个报告，宣称标准中的Dual_EC_DRBG设计存在严重漏洞，具体可以参看报告[On the Possibility of a Back Door in the NIST SP800-90 Dual Ec Prng](http://rump2007.cr.yp.to/15-shumow.pdf)...智商不够用了，我的理解是：

P, Q是ECC上两个基点，且满足eQ=P，点Q常量可以确定万能钥匙。如果攻击者知道私钥d满足Q=dP，计算eQ=P是容易的，存在a*e=1 mod p。反过来如果攻击者知道e,以及一些已知的随机值，攻击者可以计算出点G，然后可能能确定DRBG的内部状态，并预测DRBG后来的输出。一个“局外人”可能并不知道这种特殊关系的存在，但是如果知道了P和Q之间关系的“密钥”，攻击者可以利用Dual_EC_DRBG存在的后门得到该算法后续所有的随机数值，从而破坏不可预测的性质，预测其使用的密钥，进而攻破整个加密算法。

DES S-BOX的传说
===
DES中S盒来路不明是密码学届的一个著名的谜团，也和NSA有关。1975，IBM刚搞出DES（Data Encryption Standard）时候，NSA就对S-Box做了修改，并将密钥长度由128bit缩减为56bit，但未说明原因，直到80年代差分分析被公开学术界发现后，人们发现针对DES16轮的差分攻击效率甚至不如穷举，于是怀疑在设计DES的S盒和加密轮数时，差分攻击的方法就已经被发现了，而NSA那个修改证明是为了对抗可能的差分分析。


