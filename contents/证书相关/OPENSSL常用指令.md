# 常用指令收录

## 生成RSA公私密钥

1. 生成指定长度私钥(此处为2048位)

  ```shell
  $ openssl genrsa -out rsa_private_key.pem 2048
  ```

1. 生成私钥对应的公钥

  ```shell
  $ openssl rsa -in rsa_private_key.pem -out rsa_public_key.pem -pubout
  ```

1. 对私钥进行PKCS#8编码

  ```shell
  openssl pkcs8 -topk8 -in rsa_private_key.pem -out pkcs8_rsa_private_key.pem -nocrypt
  ```

> Note: 步骤1.中的私钥文件编码格式为PKCS#1，步骤3.中生成的私钥文件编码格式为PKCS#8，据说Java不支持PKCS#1格式。

## PKCS

The Public-Key Cryptography Standards (PKCS)是由美国RSA数据安全公司及其合作伙伴制定的一组公钥密码学标准，其中包括证书申请、证书更新、证书作废表发布、扩展证书内容以及数字签名、数字信封的格式等方面的一系列相关协议

- PKCS#1：定义RSA公开密钥算法加密和签名机制，主要用于组织PKCS#7中所描述的数字签名和数字信封
- PKCS#3：定义Diffie-Hellman密钥交换协议
- PKCS#5：描述一种利用从口令派生出来的安全密钥加密字符串的方法。使用MD2或MD5 从口令中派生密钥，并采用DES-CBC模式加密。主要用于加密从一个计算机传送到另一个计算机的私人密钥，不能用于加密消息
- PKCS#6：描述了公钥证书的标准语法，主要描述X.509证书的扩展格式
- PKCS#7：定义一种通用的消息语法，包括数字签名和加密等用于增强的加密机制，PKCS#7与PEM兼容，所以不需其他密码操作，就可以将加密的消息转换成PEM消息
- PKCS#8：描述私有密钥信息格式，该信息包括公开密钥算法的私有密钥以及可选的属性集等
- PKCS#9：定义一些用于PKCS#6证书扩展、PKCS#7数字签名和PKCS#8私钥加密信息的属性类型
- PKCS#10：描述证书请求语法
- PKCS#11：称为Cyptoki，定义了一套独立于技术的程序设计接口，用于智能卡和PCMCIA卡之类的加密设备
- PKCS#12：描述个人信息交换语法标准。描述了将用户公钥、私钥、证书和其他相关信息打包的语法
- PKCS#13：椭圆曲线密码体制标准
- PKCS#14：伪随机数生成标准
- PKCS#15：密码令牌信息格式标准

## 加密：算法/模式/填充

| Algorithm | Modes | Paddings |
|:----------|:------|:---------|
| AES | CBC<br> CFB<br> CTR<br> CTS<br> ECB<br> OFB | NoPadding<br> PKCS5Padding<br> ISO10126Padding |
| AES | GCM | NOPADDING |
| AES_128 | CBC<br> ECB | NoPadding<br> PKCS5Padding |
| AES_128 | GCM | NoPadding |
| AES_256 | CBC<br> ECB | NoPadding<br> PKCS5Padding|
| AES_256 | GCM | NoPadding |
| ARC4 | ECB | NoPadding |
| BLOWFISH | CBC<br> CFB<br> CTR<br> CTS<br> ECB<br> OFB | NoPadding<br> PKCS5Padding<br> ISO10126Padding |
| DES | CBC<br> CFB<br> CTR<br> CTS<br> ECB<br> OFB | NoPadding<br> PKCS5Padding<br> ISO10126Padding |
| DESede | CBC<br> CFB<br> CTR<br> CTS<br> ECB<br> OFB<br> | NoPadding<br> PKCS5Padding<br> ISO10126Padding |
| RSA | ECB<br> NONE | NoPadding<br> OAEPPadding<br> PKCS1Padding |
| RSA |              | OAEPwithSHA-1andMGF1Padding<br> OAEPwithSHA-256andMGF1Padding |
| RSA |              | OAEPwithSHA-224andMGF1Padding OAEPwithSHA-384andMGF1Padding OAEPwithSHA-512andMGF1Padding |

## RSA加密中的Padding

1. RSA_PKCS1_PADDING 填充模式，最常用的模式
  要求: 输入：必须 比 RSA 钥模长(modulus) 短至少11个字节, 也就是　RSA_size(rsa) – 11 如果输入的明文过长，必须切割，然后填充。
  输出：和modulus一样长
  根据这个要求，对于1024bit的密钥，block length = 1024/8 – 11 = 117 字节
1. RSA_PKCS1_OAEP_PADDING
  输入：RSA_size(rsa) – 41
  输出：和modulus一样长
1. RSA_NO_PADDING　　不填充
  输入：可以和RSA钥模长一样长，如果输入的明文过长，必须切割，　然后填充
  输出：和modulus一样长

## AES加密模式和填充

| 算法/模式/填充 | 16字节加密后数据长度 | 不满16字节加密后长度 |
|:------------:|:-----------------:|:-----------------:|
| AES/CBC/NoPadding | 16 | 不支持 |
| AES/CBC/PKCS5padding | 32 | 16 |
| AES/CBC/ISO10126Padding | 32 | 16 |
| AES/CFB/NoPadding | 16 | 原始数据长度 |
| AES/CFB/PKCS5Padding | 32 | 16 |
| AES/CFB/ISO10126Padding | 32 | 16 |
| AES/ECB/NoPadding | 16 | 不支持 |
| AES/ECB/PKCS5Padding | 32 | 16 |
| AES/ECB/ISO10126Padding | 32 | 16 |
| AES/OFB/NoPadding | 16 | 原始数据长度 |
| AES/OFB/PKCS5Padding | 32 | 16 |
| AES/OFB/ISO10126Padding | 32 | 16 |
| AES/PCBC/NoPadding | 16 | 不支持 |
| AES/PCBC/PKCS5Padding | 32 | 16 |
| AES/PCBC/ISO10126Padding | 32 | 16 |

> PKCS5Padding填充的原则是: 如果长度少于16个字节，需要补满16个字节，补(16-len)个(16-len)。例如： 123这个节符串是3个字节，16-3= 13,补满后如：123+(13个十进制的13)， 如果字符串长度正好是16字节，则需要再补16个字节的十进制的16。
> 采用NoPadding填充的数据解密前，可用数据长度对16进行取余看是否为0，不为零就是错的。其他填充方式解密数据前，可以检测最后一个字节的数据值是否在（0,16】之间的一个整数值，不是的话为错。

## Message Authentication Code（Mac: 消息认证码）

密码学中，通信实体双方使用的一种验证机制，保证消息数据完整性的一种工具。构造方法由M.Bellare提出，安全性依赖于Hash函数，故也称带密钥的Hash函数。消息认证码是基于密钥和消息摘要所获得的一个值，可用于数据源发认证和完整性校验。
在发送数据之前，发送方首先使用通信双方协商好的散列函数计算其摘要值。在双方共享的会话密钥作用下，由摘要值获得消息验证码。之后，它和数据一起被发送。接收方收到报文后，首先利用会话密钥还原摘要值，同时利用散列函数在本地计算所收到数据的摘要值，并将这两个数据进行比对。若两者相等，则报文通过认证。

| 算法 |
|:----:|
| DESMAC |
| DESMAC/CFB8 |
| DESedeMAC |
| DESedeMAC/CFB8 |
| DESedeMAC64 |
| DESwithISO9797 |
| HmacMD5 |
| HmacSHA1 |
| HmacSHA224 |
| HmacSHA256 |
| HmacSHA384 |
| HmacSHA512 |
| ISO9797ALG3MAC |
| PBEwithHmacSHA |
| PBEwithHmacSHA1 |
| PBEwithHmacSHA224 |
| PBEwithHmacSHA256 |
| PBEwithHmacSHA384 |
| PBEwithHmacSHA512 |

## 参考

<http://blog.csdn.net/chaijunkun/article/details/7275632/>
<https://baike.baidu.com/item/PKCS/1042350?fr=aladdin>
<http://blog.csdn.net/aaaaatiger/article/details/2525561>
<http://blog.csdn.net/guyongqiangx/article/details/74930951>
<http://blog.csdn.net/Levilly/article/details/51786595>
<http://blog.csdn.net/u013427969/article/details/52878676>
<https://developer.android.com/reference/javax/crypto/Cipher.html>
