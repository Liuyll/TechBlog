###SSL校验过程

我们先概述一下SSL的校验过程(这只是一个非常简略的图，我们旨在讨论实现细节)

![image](https://img-blog.csdn.net/20160310160503593)
#### 注意：

+ https并不是完全的非对称加密,它是由非对称加密和对称加密两部分组成,这种方式在密码学上被称作`数字信封`
+ https的对称加密密钥交换过程中,是采用非对称加密(公钥利用证书的公钥)
+ https的数据传输加密过程中,是采用对称加密
+ 传输之前,server和client会沟通对称加密算法
+ 很多同学会疑惑秘钥交换过程不会出现窃听吗，有一种基于`离散对数`的秘钥交换算法`diffie-hellman秘钥交换`，该算法实现了<b>计算上不可行</b>的秘钥交换

###SSL双向校验过程

如果服务端需要验证客户端，那么客户端也要签发证书。

![image](https://img-blog.csdn.net/20160310160519781)
+ 跟单向认证的区别是,服务器必须要验证客户端身份后,才会与客户端商量加密方法



### 技术细节

#### 对称加密秘钥是如何协商来的

准确的说，不同版本的TLS，对称加密秘钥生成的方法不同。

##### RSA

最简单的方法是通过RSA，利用RSA的公钥来加密。步骤如下：

1. client hello 此时客户端发送一个random(random1)
2. server hello 此时服务端发送一个random(random2)
3. client 生成一个premaster key(random3)
4. client server公钥加密premaster key
5. server 利用私钥解密得到premaster key，并通过random{1,2,3} 生成master screat，此作为对称秘钥进行传输



##### DH+RSA

这是一种复杂的秘钥协商方式，通过DH来进行秘钥交换，RSA来签名秘钥，确保不被中间人攻击(中间人分别对两方进行秘钥交换)

步骤如下：

1. client发送大素数P和本原根
2. client和server分别选择一个数，记为x,y(此数整个通信过程不发送)
3. client计算并发送g^x mod P
4. server计算并发送g^y mod P
5. client和server分别以g^y(x) mod P作为底数 y,x
6. 最后的秘钥为g^xy mod P



每次session，g和P都是不同的。所以每次session都不依赖于其他秘钥，是完美的前向安全。



#### 第三步是怎么验证证书合法性的？

我们回顾一下握手步骤:

1. client 请求
2. 服务端发起挑战随机数，公钥和证书
3. 客户端验证证书

那么第三步究竟是怎么验证证书的呢？

要从下面讲起：

##### X509证书在https应用上怎么才能验证可信？

三个步骤：

1. CA私钥签名证书，返回公钥给客户
2. 客户挂上证书，留着CA公钥给自己的客户检查
3. 客户以公钥解密证书，若能解密，则继续验证信息

有心人可以发现，第二步的CA公钥是服务端发送给浏览器做验证的，那么这一步是否可以出现攻击行为？比如替换该公钥为自签名证书的公钥？然后伪造是CA签发的证书。

也有同学会细心的发现，抓包并没有发现Https会通过CA验证证书公钥合法性。

引入两个概念:



##### OSCP  or CRL

###### CRL(Certificate Revocation List)

CRL 维护了一个黑名单，上面是撤销的证书号，浏览器可以定期从上面下载(update etc..)。

同时浏览器也会维护一个trustStore来弥补CRL可能出现的问题

###### OCSP(Online Certificate Status Protocal 在线证书状态协议)

OCSP 意味着浏览器任何一次https握手都会访问CA验证公钥。

OCSP stamping 这是服务器维护的验证方案，它帮你远程



#### 浏览器维护了什么？

实际上，浏览器只维护了根证书

> 这是知乎的证书签发链。
>
> ![image-20200218164808634](/Users/liuyl/Library/Application Support/typora-user-images/image-20200218164808634.png)

浏览器的工作简而言之就是:

1. 通过根证书验证CA是否属于合法下属CA

2. 通过下属CA公钥验证证书

   

##### 技术细节

事实上，浏览器只会最先获得`*.zhihu.com`这个证书，它是怎么验证上层证书的呢？

1. 浏览器通过服务器发送的公钥验证`*.zhihu.com`
2. 继续验证签发`*.zhihu.com`的CA证书`GeoTrust RSA CA 2018`
3. 用根证书`DigiCert Global Root CA`公钥验证`GeoTrust RSA CA 2018`
4. 若`*.zhihu.com`不是伪造证书，则根证书必然验证成功，vice versa

> 根证书由三次元力量强行介入，用户不捣乱，则绝对安全
>
> 这个证书链就是PKI(public key infrastructure) PGP(PKI X.509)

以这样的方式，浏览器实现了不用OCSP就能验证证书，这就是三次元的力量