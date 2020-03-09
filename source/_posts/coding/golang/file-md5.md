---
title: Golang计算文件MD5
date: 2020-03-07 10:42:22
tags: 
- Golang
- MD5
category: 编程开发
---
前面这篇文章[<Golang里面MD5的写法和性能>](https://wangbjun.site/2020/coding/golang/md5.html)介绍了如何计算字符串的md5，下面我们来说说如何计算文件的md5。

<!--more-->

## 1.错误的方式
有人说，文件可以读取成字符串，然后再计算就可以了，如下：
```go
func FileMD5(filePath string) (string, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return "", err
    }
    all, err := ioutil.ReadAll(file)
    if err != nil {
        return "", err
    }
    return MD5(string(all)), nil
}
```
此方法确实没问题，但是需要考虑一个问题，假如文件比较大呢？比如有好几个GB，如果按这个做法也得占用好几个GB内存，肯定存在问题。

经过我测试，在实际运行中，这种方式占用的内存是文件大小的好几倍，1个GB的文件需要大概4个GB的内存，太恐怖了。

## 2.正确的方式
```go
func FileMD5(filePath string) (string, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return "", err
    }
    hash := md5.New()
    _, _ = io.Copy(hash, file)
    return hex.EncodeToString(hash.Sum(nil)), nil
}
```
经过实际测试发现占用内存几乎非常非常少，这里大家就会发现md5.New()的用途所在了，简单分析一下为什么这种方式占用内存少。

首先要了解**io.Copy**方法的含义，可以先看看注释：
```go
// Copy copies from src to dst until either EOF is reached
// on src or an error occurs. It returns the number of bytes
// copied and the first error encountered while copying, if any.
//
// A successful Copy returns err == nil, not err == EOF.
// Because Copy is defined to read from src until EOF, it does
// not treat an EOF from Read as an error to be reported.
//
// If src implements the WriterTo interface,
// the copy is implemented by calling src.WriteTo(dst).
// Otherwise, if dst implements the ReaderFrom interface,
// the copy is implemented by calling dst.ReadFrom(src).
func Copy(dst Writer, src Reader) (written int64, err error) {
    return copyBuffer(dst, src, nil)
}
```
可以看出来，它底层调用了一个copyBuffer，这个方法底层在copy的时候会临时分配一个buffer缓存区，默认大小32k，每次只会占用32k大小内存，如果想自定义缓存区大小可以使用**CopyBuffer**:
```go
// CopyBuffer is identical to Copy except that it stages through the
// provided buffer (if one is required) rather than allocating a
// temporary one. If buf is nil, one is allocated; otherwise if it has
// zero length, CopyBuffer panics.
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
    if buf != nil && len(buf) == 0 {
        panic("empty buffer in io.CopyBuffer")
    }
    return copyBuffer(dst, src, buf)
}
```
最后配合Sum方法，每次计算32k，不断循环计算，直到算完，所以几乎不占用内存。


## 3.总结
如果计算的文件都是小文件，内存比较大的话，追求速度的话可以使用第一种方法，如果你计算的文件非常大，务必使用第二种方法，不然内存会爆掉。

