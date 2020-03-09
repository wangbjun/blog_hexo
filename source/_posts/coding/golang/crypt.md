---
title: Golang常见加密算法实现
date: 2020-03-08 17:02:00
tags: 
    - Golang
    - RSA
    - AES
category: Golang
---
说完Go里面的md5的用法，这篇文章咱说说用的比较多的加密方式在Go里面如何实现。首先，科普一下，一般待加密的内容被叫作明文，加密使用的关键元素被称为秘钥，加密的结果被称为密文，当然其中还有一个非常关键的加密算法。

一般加密算法可分为对称加密和非对称加密这两个分类，这两者区别很明显，对称加密是指我们拿到秘钥和密文可以解密出明文，在加密和解密时使用的是同一个秘钥；而非对称加密算法需要两个密钥来进行加密和解密，这两个密钥是公开密钥（public key，简称公钥）和私有密钥（private key，简称私钥）。

<!--more-->

本篇文章不是讲如何在Go里面实现这些加密算法，仔细看一下Go标准库里面**crypto**库，你会发现其实Go已经实现很多了加密算法，但是很多人不知道咋用，正如MD5算法一样，它没有提供一个非常简单易用的对外接口，需要你自己再封装一遍，这一点非常不好。。。

<img src="/images/2020-03-08_17-00.png" />

今天我们主要说两种最常用的加密算法：AES对称加密、RSA非对称加密。

## AES对称加密
AES（Advanced Encryption Standard）是最常见的对称加密算法，但是这个算法分很多模式，不同模式的实现方式又有很大差异，比如ECB、CBC、OFB、CFB，详细技术细节这里就不多说了。

有几点需要注意，AES对加密key的长度要求必须固定为16、24、32位，也就是128、192、256比特，所以又有一个AES-128、AES-192、AES-256这种叫法，位数越大安全性越高但加密速度越慢。最关键是对明文长度也有要求，必须是分组长度长度的倍数，AES加密数据块分组长度必须为128bit也就是16位，所以这块又涉及到一个填充问题，而这个填充方式可以分为PKCS7和PKCS5等方式，不得不说是真麻烦。

本文以CBC模式为例来介绍，CBC又有点特殊，它需要一个iv偏移量，iv不一样，结果也不一样所以更安全,这个偏移量必须和分组大小长度一样，也是16位，其实如何去生成iv和填充明文才是最麻烦的地方，但标准库里面并没有给出示例，我网上找了下，先看一个简单的实现：
```go
func AesCBCEncrypt(plainText []byte, key []byte) []byte {
    // 生成加密用的block
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(err)
    }
    // 对IV有随机性要求，但没有保密性要求，所以常见的做法是将IV包含在加密文本当中
    cipherText := make([]byte, aes.BlockSize+len(plainText))
    iv := cipherText[:aes.BlockSize]
    if _, err := io.ReadFull(rand.Reader, iv); err != nil {
        panic(err)
    }
    mode := cipher.NewCBCEncrypter(block, iv)
    mode.CryptBlocks(cipherText[aes.BlockSize:], plainText)

    return cipherText
}
```
实际应用中，秘钥我们可以指定为固定位数，但是需要加密的内容往往是不固定长度的，所以需要做填充，同时在解密的时候就需要去除填充，这里总结了2种填充方法，一个是PKCS7，网上也有些文章称之为PKCS5，另一个是0填充。
```go
func PKCS7Padding(src []byte, blockSize int) []byte {
    padding := blockSize - len(src)%blockSize
    padText := bytes.Repeat([]byte{byte(padding)}, padding)
    return append(src, padText...)
}

func PKCS7UnPadding(origData []byte) []byte {
    length := len(origData)
    unpadding := int(origData[length-1])
    return origData[:(length - unpadding)]
}

func ZeroPadding(src []byte, blockSize int) []byte {
    padding := blockSize - len(src)%blockSize
    padText := bytes.Repeat([]byte{0}, padding)
    return append(src, padText...)
}

func ZeroUnPadding(origData []byte) []byte {
    return bytes.TrimFunc(origData,
        func(r rune) bool {
            return r == rune(0)
        })
}
```
解密的过程首先是要提取出iv，然后解密，最后去除填充得到明文，代码如下：
```go
func AesCBCDecrypt(cipherText []byte, key []byte) []byte {
    block, err := aes.NewCipher(key)
    if err != nil {
        panic(err)
    }
    if len(cipherText) < aes.BlockSize {
        panic("cipher text too short")
    }
    iv := cipherText[:aes.BlockSize]
    cipherText = cipherText[aes.BlockSize:]

    if len(cipherText)%aes.BlockSize != 0 {
        panic("cipher text is not a multiple of the block size")
    }

    mode := cipher.NewCBCDecrypter(block, iv)
    mode.CryptBlocks(cipherText, cipherText)

    return cipherText
}
```
最后，咱们来看一个简单的使用示例：
```go
func main() {
    // 需要被加密的内容，需要填充
    var src = "Hello，我是一个测试加密内容你知道吗？？？"
    // key必须是16\24\32位
    var key = "1234567890123456"

    // 使用了PKCS7填充法
    cipherText := AesCBCEncrypt(PKCS7Padding([]byte(src), aes.BlockSize), []byte(key))
    // 为方便展示，用base64编码
    fmt.Printf("cipherText text is %s\n", base64.StdEncoding.EncodeToString(cipherText))

    // 解密
    plainText := AesCBCDecrypt(cipherText, []byte(key))
    fmt.Printf("plain text is %s\n", PKCS7UnPadding(plainText))
}
// 由于每次iv是随机的，所以结果都不一样，但是解密之后的明文都正确
// cipherText text is gFGf2lw9EQzQGxUtJGFQWDOaP3uU9CVWvLWCpSbeb9zrJqLUbSjS6d6GljtleGCFPFLWZZZ4a1RvKxR8wVT0d/U0cn8F4nwhEnun4Ba3t0M=
// plain text is Hello，我是一个测试加密内容你知道吗？？？
```
## RSA非对称加密
RSA非对称加密需要一对秘钥，一个公钥，一个私钥，公钥加密之后私钥才能解密，私钥加密之后公钥才能解密，其最广泛的应用莫过于https、ssh，安全性高，但是速度相对较慢。

首先，我们得生成一对秘钥，方法有很多种，我们可以用工具生成，比如在Linux下面可以使用openssl命令生成：
```bash
# 生成私钥
openssl genrsa -out private.pem 2048
# 生成公钥
openssl rsa -in private.pem -outform PEM -pubout -out public.pem
```
也可以使用一些工具生成，保存起来，值得一提的是这个秘钥的格式还有很多说法，这里暂不细说，如果遇到问题了，不妨留意一下。

Go的crypto库提供了一些方法来进行rsa的加密和解密操作，不过同样我们还得自己组装起来，先看一下加密：
```go
func RsaEncrypt(plainText []byte, keyPath string) []byte {
    // 读取公钥
    file, _ := os.Open(keyPath)
    fileInfo, _ := file.Stat()
    data := make([]byte, fileInfo.Size())
    _, _ = file.Read(data)
    // pem解码
    block, _ := pem.Decode(data)
    publicKey, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        panic(err)
    }
    cipherText, err := rsa.EncryptPKCS1v15(rand.Reader, publicKey.(*rsa.PublicKey), plainText)
    if err != nil {
        panic(err)
    }
    return cipherText
}
```
然后是解密：
```go
func RsaDecrypt(cipherText []byte, keyPath string) []byte {
    // 读取私钥
    file, _ := os.Open(keyPath)
    fileInfo, _ := file.Stat()
    data := make([]byte, fileInfo.Size())
    _, _ = file.Read(data)

    // pem解码
    block, _ := pem.Decode(data)
    privateKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
    if err != nil {
        panic(err)
    }
    plainText, err := rsa.DecryptPKCS1v15(rand.Reader, privateKey, cipherText)
    if err != nil {
        panic(err)
    }
    return plainText
}
```

综合示例：
```go
func main() {
    var src = "1234567890"
    cipherText := RsaEncrypt([]byte(src), "/path/to/public.pem")

    // base64编码输出
    fmt.Printf("%s\n", base64.StdEncoding.EncodeToString(cipherText))

    // 解密
    plainText := RsaDecrypt(cipherText, "/path/to/private.pem")
    fmt.Printf("%s\n", plainText)
}
```
RSA加密每次的结果都不一样，安全性虽高，但是也有缺点，速度慢，而且加密的内容不能太大，最大不能超过秘钥的长度，比如说这个例子里面秘钥是2048位的，也就是256字节，如果超过了可能就需要你特殊处理了，比如分割成多段依次加密。

总之，在Go里面使用加密的话需要根据实际情况调整，不同加密方式的实现细节有很多不一样的地方，好在标准库大部分都实现了，只需要我们花点功夫研究一下咋去使用。

