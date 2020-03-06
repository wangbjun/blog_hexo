---
title: Protobuf入门和实战
date: 2019-10-22 17:15:43
tags: Golang
category: 编程开发
---

## 1.简介

Protobuf（Google Protocol Buffer）是 Google公司内部的混合语言数据标准，目前已经开源，支持多种语言（C++、C#、Go、JS、Java、Python、PHP），它是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

说简单点，Protobuf就是类似JSON、XML这样的数据交换格式，当今互联网JSON是最流行的格式了，XML用的也挺多，最初接触到Protobuf是因为gRPC默认使用它作为数据编码，相比于JSON和XML，它更小，更快！

<!--more-->

举个例子：如果我们想表达一个人名字叫John，年龄是28岁，邮箱是jdoe@gmail.com这样的结构化数据，并且需要在互联网上传输

- 使用XML表示如下：
```
 <person>
    <name>John</name>
    <age>28</age>
    <email>jdoe@example.com</email>
  </person>
```

- 使用JSON表示如下：
```
{
    name: John,
    age: 28,
    email: jdoe@example.com
}
```

- 使用Protobuf表示如下：
```
message Person {
    string name = 1;
    int32 age = 2;
    string email = 3;
}
```

从可读性和表达能力上看，XML最好，JSON其次，而Protobuf这个其实只是一个DSL，用来定义数据结构和类型，实际生成的数据是二进制的，不可读，但Protobuf追求的是性能和速度，关于它们之间的对比，后面再说，咱们先说用法。

## 2.安装环境
Protobuf的使用比较麻烦，首先需要安装Protobuf的编译工具(Protocol Buffers compiler)，Ubuntu环境下自带编译环境，其它平台可自行安装
```
jwang@jwang:~$ protoc --version
libprotoc 3.8.0
```
然后还需要安装不同语言的运行环境，具体可以参考[github.com/protocolbuffers/Protobuf](https://github.com/protocolbuffers/Protobuf)

## 3.编写proto文件
proto其实是一种DSL语法，这个proto文件最终会使用protoc编译成不同语言的文件，然后在程序里面调用，这也是Protobuf跨平台的关键。关于proto文件的语法这里不详细介绍，建议大家参考[官方文档](https://developers.google.com/protocol-buffers/docs/proto3)，东西很多，也很详细。

我这里拿一个简单实际的例子（person.proto）来说明一下，建议大家使用Goland安装一个插件，这样有颜色还可以检查语法：

- 第一行syntax是声明proto语法版本，如果不声明默认是2，建议使用3版本
- 然后是package也就包，这个影响到最后生成的go文件的包
- 后面message是用来声明一个数据对象，我觉得可以理解为结构体struct，这个数据对象有自己的数据成员，每个字段有类型和默认值。
- proto的数据类型有标量类型和枚举类型，由于不同语言的数据类型不太一样，所以这里的类型和实际语言的类型有一个对应转换关系，具体可以参考官方文档
- repeated 相当于声明一个数组，比如在上面的例子，意思就是car是一个string类型的数组
- message可以嵌套声明，也可以引用一个类型
- 最迷惑的东西估计就是后面那个1,2,3,4...了，据官方文档的说法是为了在二进制格式里面标记数据，在每一个message里面必须是唯一的，从最小的1开始，一直可以到2的29次方-1，也就是536870911，但是19000到19999是保留的数字。

基本语法还是挺简单的，不过有些深入的用法这里没有介绍到，想要了解的话务必查看官方文档，不过定义数据结构和类型只是第一步，接下来我们还要使用protoc把这个文件编译成对应语言的文件。

## 4.编译proto文件

以Go语言为例，建议切换到proto文件目录执行命令：
```
protoc --go_out=. person.proto
```
其中--go_out表示输出go版本的，其它语言把go替换就行了，比如--php_out、--java_out,=后面是需要输出的目录，我选择.表示当前目录，当然你也可以指定输入和输出目录，最后面则是需要编译的文件，可以指定单个文件，也可以使用通配符同时编译多个文件。

执行完命令之后，你会发现当前目录多了一个person.pb.go文件，这是一个标准的go语法文件，里面主要是一个结构体和一些getter函数，其它的我也不太懂是什么意思就不说了，但是并不影响我们使用。

## 5.使用

以Go为例，我们需要安装一个[运行库](github.com/golang/Protobuf/proto)，其它语言也差不多，官方针对每一个语言都有一个单独的介绍文档，务必查阅一下。

下面是一个完整的案例：
```go
package main

import (
    "fmt"
    "github.com/golang/Protobuf/proto"
    "io/ioutil"
    "os"
)

func main() {
        //实例化模型对象，填充数据
    p := &Person{
        Id:    1,
        Name:  "jun",
        Age:   25,
        Money: 24.5,
        Car:   []string{"car1", "car2"},
        Phone: &Person_Phone{Number: "0551-12323232", Type: "1"},
        Sex:   Person_female,
    }

    //Marshal序列化
    out, err := proto.Marshal(p)
    if err != nil {
        panic(err)
    }
        //序列化得到结果是二进制的，是不可读的，所以这里保存到文件
    file, _ := os.OpenFile("out", os.O_CREATE|os.O_WRONLY, 0666)
    _, _ = file.Write(out)
    _ = file.Close()

    //unMarshal还原数据，从文件里面读取
    in, _ := os.Open("out")
    bytes, err := ioutil.ReadAll(in)
    if err != nil {
        panic(err)
    }
    p1 := &Person{}
    err = proto.Unmarshal(bytes, p1)
    if err != nil {
        panic(err)
    }
        //调用string()方法打印，也可以使用其生成的getter函数
    fmt.Printf("%s\n", p1.String())
        fmt.Printf("%d\n", p1.GetId)
}
```

## 6.与JSON对比

由于XML目前很少使用在Web API接口上，所以这里就不对比了，主要看一下和JSON的对比，包含2个方面：速度和大小。

为了测试，我在proto文件里面又加了一个数据对象，表示一个组里面有多个person对象
```
message Group {
    repeated Person person = 1;
}
```
分别测试有1,10,100个对象的时候对比情况，测试代码如下：
```go
func BenchmarkProto(b *testing.B) {
    g := &Group{}
    for i := 0; i < 100; i++ {
        p := &Person{
            Id:    int32(i),
            Name:  "测试名称",
            Age:   int32(25 * i),
            Money: 240000.5,
            Car:   []string{"car1", "car2", "car3", "car4", "car5", "car7", "car6", "car21", "car22",},
            Phone: &Person_Phone{Number: "0551-12323232", Type: "1"},
            Sex:   Person_female,
        }
        g.Person = append(g.Person, p)
    }
    b.ResetTimer()
        b.N = 1000
    for i := 0; i < b.N; i++ {
        out, err := proto.Marshal(g)
        if err != nil {
            panic(err)
        }

        g1 := &Group{}
        err = proto.Unmarshal(out, g1)
        if err != nil {
            panic(err)
        }
    }
}

func BenchmarkJson(b *testing.B) {
    g := &Group{}
    for i := 0; i < 100; i++ {
        p := &Person{
            Id:    int32(i),
            Name:  "测试名称",
            Age:   int32(25 * i),
            Money: 240000.5,
            Car:   []string{"car1", "car2", "car3", "car4", "car5", "car7", "car6", "car21", "car22",},
            Phone: &Person_Phone{Number: "0551-12323232", Type: "1"},
            Sex:   Person_female,
        }
        g.Person = append(g.Person, p)
    }
    b.ResetTimer()
        b.N = 1000
    for i := 0; i < b.N; i++ {
        out, err := json.Marshal(g)
        if err != nil {
            panic(err)
        }

        g1 := &Group{}
        err = json.Unmarshal(out, g1)
        if err != nil {
            panic(err)
        }
    }
}
```
为了方便对比，指定了测试次数为1000次，测试结果如下：

在1个person的级别：

可以看出，理论上proto明显比json要快不少，每次操作大概是4-5倍差距。后面在10，100个person的级别的测试中，基本上都是保持在4-5倍性能的差距，这个结果也和网上大部分测试结果一致。

关于生成的数据大小，这里也简单测试了一遍，还是上面的例子，我使用了10个person，Protobuf生成的文件大小是1030个byte,json生成的文件大小是1842个byte。

需要注意一点，虽然在大小上Protobuf也领先很多，但是据网上文章介绍，在经过nginx的gzip压缩之后，这2者大小基本上差不多。

## 7.总结
Protobuf作为一种新的数据交换编码方式，虽然使用起来麻烦点，但是在性能和大小上面领先很多，可以用来替换json，使用在一些对性能要求高的场景，比如移动端设备通信。除此之外，目前Protobuf主要用在gRPC用作默认数据编码格式。
