# HTTPS技术鉴赏

[https技术鉴赏](https://www.bilibili.com/video/BV1uY4y1D7Ng/?spm_id_from=333.1387.favlist.content.click&vd_source=aff299492aef997f739ddf5bb0ae722e)

**HTTPS**；常称为HTTP over TLS、HTTP over SSL或HTTP Secure）。

HTTPS经由[HTTP](https://zh.wikipedia.org/wiki/HTTP)进行通信，但利用[SSL/TLS](https://zh.wikipedia.org/wiki/传输层安全)来加密数据包。HTTPS开发的主要目的，是提供对网站服务器的身份认证，以保护交换资料的隐私与完整性。

## 对称加密

发送方和接收方必须事先共享同一个密钥，并且保证该密钥的安全性。

## 非对称加密

密钥成对出现（公钥和私钥）

![image-20250117222021685](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117222021685.png)

1. 服务器先将自己的公钥发送给浏览器
2. 浏览器生成一个随机的数据并用服务器的公钥进行加密。再发送给服务器
3. 服务器用自己的私钥解密，此时双方得到同一个随机数据。这个随机数据便可以作为对称加密的密钥。

在这个过程中即便攻击者拦截到公钥也无济于事，因为公钥无法解密它自身加密的数据

![image-20250117222552807](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117222552807.png)

SSL握手

![image-20250117223043927](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117223043927.png)

![image-20250117222627080](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117222627080.png)

1. 攻击者在服务器发送公钥给浏览器的过程中将公钥拦截，并替换成自己的公钥再发送给浏览器

2. 浏览器无法知道这个公钥是否被篡改过，仍然用它来加密作为后续对称加密公钥的随机数据
3. 攻击者收到加密的随机数据后，可以通过自己的私钥解密得到明文，然后再用服务器的公钥对其加密再发送给服务器，服务器用自己的私钥解密。攻击者就像一个黑中介，两头骗

![image-20250117230158245](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117230158245.png)

这就需要引入一个第三方角色，CA机构。CA(Certificate Authority)证书授权

![image-20250117230904218](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117230904218.png)

服务器将自己的公钥、域名等信息放在一起形成一个数据集合，然后拿着这份数据找到CA机构

CA机构有自己的公私钥对，用私钥对数据集合进行加密得到一个密文，该密文就是数字签名。

![image-20250117231042324](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117231042324.png)

然后把签名和原始明文放在一起发送给服务器的管理员，这就是所谓的TLS证书。

![image-20250117231314742](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117231314742.png)

现在服务器不再直接发送公钥，而是发送能够表明自己身份的证书，浏览器拿到证书后需要先进行验证而不是选择直接相信

浏览器拿CA机构的公钥对证书中的密文进行解密，如果解密的结果和证书中的明文一致则验证通过。

然后从证书中提取出公钥加密随机数据发送给服务器

浏览器内置了可信CA机构，只有被系统或浏览器内置的CA机构颁发的证书才能通过浏览器的验证

![image-20250117232007662](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20250117232007662.png)