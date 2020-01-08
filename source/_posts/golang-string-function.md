---
title: Golang字符串处理函数浅析
date: 2019-02-12 20:15:43
tags: Golang
category: 编程开发
---

很多从PHP转Go的小伙伴经常会怀恋PHP丰富的字符串函数，Go的标准库针对字符串的操作函数虽然不少但是还是不够丰富，很多时候还得自己造，下面我就结合PHP里面字符串的操作函数来说说Go里面怎么实现。

<!--more-->

## String
Go是强类型语言，有一个单独的字符串类型 string，如果熟悉Go语言的人应该了解string底层是切片，切片底层是数组，所以字符串也叫字符数组。

举个最简单的例子，有一个字符串 12ab34cd56, 我们要获取其第3到第5个字符之间的元素怎么做呢？

熟悉PHP的童鞋可以会想到PHP里面有一个 substr的函数可以做到，但是Go里面呢？

我们打开IDE看一下，其实标准库里面的 strings 包已经有非常多的函数了，大约有20多个，包含常见的trim、index、replace、contain等功能，但是没有找到我们想要的？

![](https://show-file.che001.com/fileserver/img?id=3031e623-10bd-11ea-85f9-0242ac144108)

其实很简单，因为string本质上是切片，所以我们可以直接使用切片来分割字符串：
```go
package main

import (
	"fmt"
)

func main() {
	str := "12ab34cd56"

	fmt.Printf("%s\n", str[2:4]) //ab

	fmt.Printf("%s\n", str[3:]) //b34cd56

	fmt.Printf("%s\n", str[:3]) //12a
}
```
切片的切割用法就不多说了，从0角标开始，包含开始，不包含结束。

不过这种写法是有bug的，它只可以针对单字节字符，针对中文这种多字节字符串就不可以了，PHP里面也一样，PHP里面针对多字节字符有一个 mbstring 扩展，也有 mb_substr 这样的函数专门处理多字节字符。

## Rune
在国内编程，大部分时候不可避免的要处理中文字符串，所以像计算长度、切割一定要处理好多字节的问题，Go里面针对多字节的字符有一个rune类型，针对上面的这个问题，我们这样做：
```go
package main

import (
	"fmt"
)

func main() {
	str := "我爱学习Go语言"

	rStr := []rune(str)

	fmt.Printf("%s\n", string(rStr[2:4])) //学习

	fmt.Printf("%s\n", string(rStr[3:])) //习Go语言

	fmt.Printf("%s\n", string(rStr[:3])) //我爱学
}
```

这种方式完全没问题，如果说问题可能就在于多了一次内存分配，那rune到底是什么呢？

rune类型在Go里面实际上是int32的别名：
```go
// byte is an alias for uint8 and is equivalent to uint8 in all ways. It is
// used, by convention, to distinguish byte values from 8-bit unsigned
// integer values.
type byte = uint8

// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
type rune = int32
```
byte是8位，可以表示-128-127之间的数，用来存储单字节字符刚好，但是中文一般使用2-3个字节表示，byte就无能为力了，但是int32用来表示世界上所有字符也绰绰有余。

```go
r := "我爱学习Go语言"

fmt.Printf("%s\n", r)
fmt.Printf("%v\n", []byte(r))
fmt.Printf("%v\n", []rune(r))

//运行结果如下：
我爱学习Go语言
[230 136 145 231 136 177 229 173 166 228 185 160 71 111 232 175 173 232 168 128]
[25105 29233 23398 20064 71 111 35821 35328]
```

从上面的例子也可以说明，中文“我”实际上是以230 136 145 3个字节表示的，但是在rune类型里面是以25105表示的，这个数是Unicode编码的10进制表现形式。

所以，我们可以把一个字符串先转成rune数组，然后再使用切片切割。

## for...range
字符串本质上是字符数组，所以有也可以使用range遍历，而且range在迭代字符串的时候也是按字符遍历的，我们也可以利用这点分割字符串：
```go
func SubString(str string, start, end int) string {
	var n, i, k int

	for k = range str {
		if n == start {
			i = k
		}
		if n == end {
			break
		}
		n++
	}

	return str[i:k]
}
```

## reverse
再看一个比较常见的PHP函数，反转字符串，在Go标准库里面也没有相应的实现

如果只考虑单字节我们可以很容易写出下面的代码：
```go
func ReverseString(str string) string {
	b := []byte(str)
	for i, j := 0, len(b)-1; i < len(b)/2; i, j = i+1, j-1 {
		b[i], b[j] = b[j], b[i]
	}
	return string(b)
}
```

如果考虑到中文等多字节字符可以参考下面这种方式：
```go
func ReverseRuneString(s string) string {
	var start, size, end int
	buf := make([]byte, len(s))
	for end < len(s) {
		_, size = utf8.DecodeRuneInString(s[start:])
		end = start + size
		copy(buf[len(buf)-end:], s[start:end])
		start = end
	}
	return string(buf)
}
```

## 推荐
除了字符串之外，PHP的数组功能也很强大，如果你不想自己造轮子，可以使用现成的第三方库，下面简单介绍一下几个项目：

### 1.https://github.com/syyongx/php2go
这个项目是使用Go实现PHP内置的函数库，东西比较多，不过这个库里面并没有特殊处理多字节字符串，需要注意一下。

### 2.https://github.com/thinkeridea/go-extend
这个项目收集了一些常用的操作函数，辅助更快的完成开发工作，并减少重复代码，都是一些比较实用的函数，虽然没有第一个那么全。

### 3.https://github.com/jianfengye/collection
Collection包目标是用于替换golang原生的Slice，使用场景是在大量不追求极致性能，追求业务开发效能的场景。

