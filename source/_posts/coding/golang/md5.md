---
title: Golang计算MD5的写法和性能
date: 2020-03-05 21:10:00
tags: 
- Golang
- MD5
category: Golang
---
用过PHP的童鞋知道在PHP里面md5很简单，是一个内置函数，可以直接调用：
```bash
jwang@jun:~$ php -a
Interactive mode enabled

php > echo md5("12345");
827ccb0eea8a706c4c34a16891f84e7b
```

纠正一个错误的说法，很多人一直把md5叫作加密算法，实际上md5并不是加密，它既不是对称加密，也不是非对称加密，它只是一个摘要函数，一般被用于签名或者校验数据完整性。

虽然现在有文章说不推荐使用md5了，因为碰撞几率比较大，实际上，这个几率非常非常非常低，大只是相对于其它摘要函数来说，纯自然的情况下基本不可能碰撞，虽然可以用工具构造出来，但非常复杂。如果实在不放心，可以用sha1或者sha256，或者两者集合起来用，速度会慢一点，但安全性高一点，总之，md5由于速度快，简单易用，现在用的还是蛮多的。

<!--more-->

言归正传，在Go的标准库里面并没有md5这个函数，但是在**crypto**包里面确实有相关实现，需要自己动手组装拼凑一下，网上流传的写法有很多种：

## 1.第一种

```go
func MD5(s string) string {
    hash := md5.New()
    _, err := hash.Write([]byte(s))
    if err != nil {
        panic(err)
    }
    sum := hash.Sum(nil)
    return fmt.Sprintf("%x\n", sum)
}
```
平时用到可能只是copy过来，没仔细看，今天来仔细看一下，首先，这个 md5.New() 返回的是一个结构体 digest：

<img src = "/images/2020/2020-03-06_21-34.png" />

这个结构体成员啥意思呢？其实细说起来，这和md5的算法有关了，咱也不知道，咱也不敢问！

但是仔细看一下这个结构体的方法，你会发现有一个叫Write，还有一个叫Sum，如果你英语不错，你可以看懂，Write就是把数据写到刚才这个结构体里面，先甭管它咋写，肯定是有算法规则，感兴趣可以研究研究。Sum稍微有点不一样，但是有一个参数，和一个返回值，这个方法的意思是把参数追加到进去并且返回摘要，由于我们之前已经写进去了，所以参数为nil即可。

<img src = "/images/2020/2020-03-06_21-39.png" /> 

可见，一个md5方法Go就整了6行代码，老板看你代码写这么多，又可以加薪了，Go确实是好语言。

## 2.第二种

如果你仔细看了这个包里面的 md5.go 文件，你会发现最下面有一个公开的方法Sum，仔细一看，这就是刚才我写的那个简化版：
```go
// Sum returns the MD5 checksum of the data.
func Sum(data []byte) [Size]byte {
    var d digest
    d.Reset()
    d.Write(data)
    return d.checkSum()
}
```

所以我们的方法可以简化为：
```go
func MD5(s string) string {
    sum := md5.Sum([]byte(s))
    return fmt.Sprintf("%x\n", sum)
}
```
当然sum变量在这２个方法里面都是多余的，可以简化为一行代码即可。

## 3.第三种
还有一种方式是使用io库的方法往里面写，主要是因为degest实现了**io.Writer**接口
```go
func MD5(s string) string {
    hash := md5.New()
    _, _ = io.WriteString(hash, s)
    return hex.EncodeToString(hash.Sum(nil))
}
```

其中最后Sprintf方法是为了把结果转化成小写十六进制，也可以用**hex.EncodeToString**方法替代。

## 4.性能对比
这几种方式大同小异，理论上讲应该没有什么性能差距，不过既然Go自带Benchmark，我们就测一下吧：
```go
func BenchmarkMD5(b *testing.B) {
    for i := 0; i < b.N; i++ {
        MD5("12345678901234567890")
    }
}
```
结果如下：
```bash
jwang@jun:~/Documents/Work/learnGo/Std/md5$ go test -bench=.
goos: linux
goarch: amd64
BenchmarkMD5_1-12       10000000               359 ns/op
BenchmarkMD5_2-12       10000000               356 ns/op
BenchmarkMD5_3-12       10000000               163 ns/op
BenchmarkMD5_4-12       10000000               296 ns/op
PASS
ok      _/home/jwang/Documents/Work/learnGo/Std/md5     11.757s
```
不测不知道，一测吓一跳，其实前2个方法差不多很正常，但是第三个方法性能很好，其主要原因是因为Sprintf的性能比较差导致，不过**md5.New()**这种写法也比较慢。

## 5.最佳写法
最终得出结论，性能最高的md5写法是这种，推荐大家使用：
```go
func MD5(s string) string  {
    sum := md5.Sum([]byte(s))
    return hex.EncodeToString(sum[:])
}

```

## 6.Sha1
最后说个题外话，Go里面Sha1的写法和Md5几乎一致，只需要要把md5改成sha1即可：
```go
func Sha1(s string) string  {
    sum := sha1.Sum([]byte(s))
    return hex.EncodeToString(sum[:])
}
```
我也测了一下性能，它们之间的差距很小，md5是163ns/op，sha1是206ns/op，毕竟sha1比md5长一点。。。
