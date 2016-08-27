#RSA加密和解密
最近在项目中使用到了RSA加密算法，查询很多资料，在这里做个笔记
##基础知识

1.什么是RSA？
RSA是一种非对称加密算法，常用来对传输数据进行加密，配合上数字摘要算法，也可以进行文字签名。

2.RSA加密中padding？
padding即填充方式，由于RSA加密算法中要加密的明文是要比模数小的，padding就是通过一些填充方式来限制明文的长度。

3.加密和加签有什么区别？
**加密**：公钥放在客户端，并使用公钥对数据进行加密，服务端拿到数据后用私钥进行解密;
**加签**：私钥放在客户端，并使用私钥对数据进行加签，服务端拿到数据后用公钥进行验签。
前者完全为了加密；后者主要是为了防恶意攻击，防止别人模拟我们的客户端对我们的服务器进行攻击，导致服务器瘫痪。

##基本原理
RSA使用“密钥对”对数据进行加密解密，在加密解密前需要先生存公钥（Public Key）和私钥（Private Key）。
**公钥(Public key)**: 用于加密数据. 用于公开, 一般存放在数据提供方, 例如iOS客户端。
**私钥(Private key)**: 用于解密数据. 必须保密, 私钥泄露会造成安全问题。

iOS中的Security.framework提供了对RSA算法的支持，这种方式需要对密匙对进行处理, 根据public key生成证书, 通过private key生成p12格式的密匙。想想jave直接用字符串进行加密解密简单多了。

##生成证书
RSA加密这块公钥、私钥必不可少的。**Apple是不支持直接使用字符串进行加密解密的，推荐使用p12文件。**这边教大家去生成在加密中使用到的所有文件，并提供给Java使用。

**生成模长为1024bit的私钥**
```
openssl genrsa -out private_key.pem 1024
```
**生成certification require file**
```
openssl req -new -key private_key.pem -out rsaCertReq.csr
```
**生成certification 并指定过期时间**
```
openssl x509 -req -days 3650 -in rsaCertReq.csr -signkey private_key.pem -out rsaCert.crt
```
**生成公钥供iOS使用**
```
openssl x509 -outform der -in rsaCert.crt -out public_key.der
```
**生成私钥供iOS使用 这边会让你输入密码，后期用到在生成secKeyRef的时候会用到这个密码**
```
openssl pkcs12 -export -out private_key.p12 -inkey private_key.pem -in rsaCert.crt
```
**生成pem结尾的公钥供Java使用**
```
openssl rsa -in private_key.pem -out rsa_public_key.pem -pubout
```
**生成pem结尾的私钥供Java使用openssl**
```
pkcs8 -topk8 -in private_key.pem -out pkcs8_private_key.pem -nocrypt
```
**以上的步骤都是在终端上完成。**

##RSA参考文章

####使用X509生成P12文件，通过苹果原生的加密库来加密解密
http://www.jianshu.com/p/a1bad1e2be55

http://witcheryne.iteye.com/blog/2171850

https://github.com/PanXianyue/XYCryption


####使用openssl库来加密解密，需要到第三方库openssl，好处是安卓，iOS通用，使用openssl有可能内存泄漏
http://blog.csdn.net/xyxjn/article/details/17225809

http://www.kuqin.com/shuoit/20150312/345182.html

https://github.com/x2on/OpenSSL-for-iPhone




