---
title: Golang的常用数据结构
date: 2019-06-19 21:00:33
tags: Golang
category: 编程开发
---

闲着无事，随便写写，初学Go，望各位大神轻喷！Go自带的几个复合数据类型，基本数据类型咱就不说了，大部分语言常见的几种复合数据类型大概有数组、字典、对象等，不同语言叫法不一样，用法也有差异，比如说PHP里面数组其实严格来说不算数组。

<!--more-->

### 1.数组
Go里面的数组和C类似，是由**有序**的**固定长度**的**特定类型**元素组成。画重点，固定长度和特定类型。在很多弱类型的语言里面，数组非常随意，PHP的数组本质上是一个hash table，和C的数组差异太大，所以写惯了PHP再写Go的话这点需要注意。

#### 基础用法1:
```
package main

import "fmt"

func main() {
	var a [5]int

	a[1] = 1
	a[2] = 3

	var b [10]string

	b[0] = "a1"
	b[1] = "b2"
	b[2] = "c5"

	fmt.Printf("%v\n", a)
	fmt.Printf("%v\n", b)
}
---结果---
[0 1 3 0 0]
[a1 b2 c5       ]
```
从语法上看，Go定义数组的类型放在后面，这点写惯C系语言的估计蛋疼。数组也是通过索引下标访问，如果不初始化赋值的话，默认情况下，int类型的元素是0,string类型是空字符串。

#### 基础用法2
我们也可以不先定义，直接使用字面量初始化数组：
```
package main

import "fmt"

func main() {
	a := [...]int{1, 2, 3, 4, 5, 7}

	fmt.Printf("%v", a)
}
---结果---
[1 2 3 4 5 7]
```
在这种情况下，我们可以省略长度,使用3个点代替，编译器会自动判断。

#### 数组遍历
主要有两种方式：
```
package main

import "fmt"

func main() {
	a := [...]int{1, 2, 3, 4, 5, 7}

	for i := 0; i < len(a); i++ {
		fmt.Print(a[i])
	}
	
	for k, v := range a {
		fmt.Print(k, "->", v)
	}
}
```
如果知道长度的话可以使用for循环，否则可以使用for range 这种语法。

#### 数组函数
Go内置了一些函数可以操作数组，如果你使用了IDE的话，可以“点”出来：

![](http://ww1.sinaimg.cn/large/5f6e3e27ly1fzgltdja32j20g4054wen.jpg)

然而，append并不是用来操作数组的，其实它是用来操作变长数组的，即slice, 又称切片。

### 2.Slice（切片）
传统的数组长度固定，所以实际用途并不多，除非你明确知道自己想要多长的数组，很多时候我们需要的是一个可以改变长度大小的数组，在Go里面这类型被称为切片。

slice其实是从数组而来的，它和数组非常像，区别就在于slice没有固定长度，非常方便，所以平时一般都是用这个比较多。

#### 基础用法1:
```
package main

import "fmt"

func main() {
	var a []int

	a = append(a, 2)
	a = append(a, 1)
	a = append(a, 4)
	a = append(a, 5)

	fmt.Printf("%v", a)
}
```
区别就在于slice在定义的时候不需要指定长度，也不用3个点，但是这就意味着你不能使用索引下标的方法去赋值了，可以使用append函数去追加元素。

而且在使用slice的也需要注意下标，如果大于slice的长度也会出现 ```panic: runtime error: index out of range```。

#### 基础用法2
```
package main

import "fmt"

func main() {
	a := [...]int{1,2,3,4,5,6,7,8}

	s1 := a[0:]

	s2 := a[1:5]

	s3 := a[4:6]

	fmt.Printf("%v\n", a)
	fmt.Printf("%v\n", s1)
	fmt.Printf("%v\n", s2)
	fmt.Printf("%v\n", s3)
}
```
slice可以使用```[start:end]```这种语法从一个数组里面生成，比如```a[1:5]```意思是生成一个包含数组索引1到5的之间元素的slice。

>在Go里面不同长度但是同一类型的数组是不同类型的，比如你定义了2个int数组，一个长度为5，一个长度为10，他们其实并不是同一个类型，虽然都是int类型。cannot use a (type [10]int) as type [5]int in argument

所以在大部分时候我们需要的是一个slice，并不是一个数组。虽然这个2个用法基本上一毛一样。。。


### 3.Map
在很多语言里面，map被叫作字典，这个中文名称很亲切，字典就是一种key value结构，小时候大家都用过新华字典，字典的特征就是每一个字都对应一个解释。但是Go的map是无序的，这点大家需要注意。如果有童鞋写过PHP，会发现这个数据类型类似PHP里面的关联数组。

在Go里面，它和slice的区别就是slice的索引是数值，map的索引类型就丰富了，基本上常用数据类型都支持，甚至包括结构体。

#### 基础用法
和其它数组类型一样，map也支持先定义后赋值，或者直接使用字面量创建。但是如果使用先定义后赋值这种方式，map需要使用make初始化。
```
package main

import "fmt"

func main() {
	var m1 map[string]string

	m1 = make(map[string]string)

	m1["name"] = "Golang"
	m1["address"] = "BeiJin"

	m2 := map[string]string{
		"name": "GoLand",
		"addr": "ShangHai",
	}

	fmt.Printf("%v\n", m1)
	fmt.Printf("%v", m2)
}
---结果---
map[name:Golang address:BeiJin]
map[name:GoLand addr:ShangHai]
```

map可以使用for range 语法遍历，但是需要注意的是每次遍历的顺序是无序的。

如何判断一个key是否存在map里面？在PHP里面我们有一个array_key_exists函数，在Go里面写法略有不同：
```
age, ok := m1["age"]
if !ok {
    fmt.Println("age 不存在", age)
}
```
其实如果你不判断是否存在直接取也可以，并不会报错，只不过获取到的值是一个对应类型的零值。

### 4.结构体
Go的结构体也类似C，类似于现在很多面向对象的语言里面的类，往往用来存储一组相关联的数据，Go虽然不是一个完全面向对象的语言，但是使用结构体可以实现类似效果。

#### 基本用法
```
package main

import "fmt"

type Goods struct {
	name    string
	price   int
	pic     string
	address string
}

func main() {
	var goods Goods
	goods.name = "商品1"
	goods.price = 100
	goods.pic = "http://xxxx.jpg"
	goods.address = "中国"

	fmt.Printf("%v\n", goods)

	goods2 := Goods{
		name:    "商品2",
		price:   200,
		pic:     "http://xxxx.png",
		address: "日本",
	}

	fmt.Printf("%v", goods2)
}
---结果---
{商品1 100 http://xxxx.jpg 中国}
{商品2 200 http://xxxx.png 日本}
```
先定义后赋值或者字面量赋值都可以，值得一提的是在Go里面如果结构体或者其属性的首字母大写则表示该结构体或者属性可以被导出，也就是被其它包使用。结构体里面的属性成员的类型也可以是结构体，这就变相实现了类的继承。

既然结构体和类差不多，那类的方法在哪里定义呢？这点Go实现的就比较巧妙了！
```
func (g Goods) getName() string {
	return g.name
}
```
我们只需要在函数的前面放一个变量，就变成了方法。在很多语言里面，函数和方法区分不是很明显，大部分时候我们都是混着叫，但是在Go里面，方法指的是针对某一类型的函数。比如在上面的例子里面，这个**getName**函数就是针对**Goods**结构体的,用面向对象的说法就是一个类方法。所以我们可以使用 ```goods.getName()```的形式调用这个方法。
>上面的代码里那个附加的参数p，叫做方法的接收器（receiver），早期的面向对象语言留下的遗产将调用一个方法称为“向一个对象发送消息”。
 在Go语言中，我们并不会像其它语言那样用this或者self作为接收器；我们可以任意的选择接收器的名字。由于接收器的名字经常会被使用到，所以保持其在方法间传递时的一致性和简短性是不错的主意。这里的建议是可以使用其类型的第一个字母。
 
在Go里面我们可以为任何类型定义方法，无论是常见的int、string，还是map、struct都没问题，下面的例子里面就是为int类型扩展一个方法：
```
package main

import "fmt"

type MyInt int

func main() {
	myInt := MyInt(10)
	res := myInt.add(100)

	fmt.Printf("%d", res)
}

func (m MyInt) add(a int) int {
	return int(m) + a
}
---结果---
110
```
我们无法直接使用基本数据类型，但是我们可以起一个别名，纯属娱乐！


### 5.JSON
严格来说，JSON并不是一种数据类型，但是json是现在最流行的数据交换格式，Go对json的支持也很好，在Go里面主要通过结构体生成json，我们也可以把一个json转换成结构体。
```
package main

import (
	"encoding/json"
	"fmt"
)

type Goods struct {
	Name    string
	Price   int
	Address string `json:"address2"`
	Tag     string
}

func main() {
	goods := Goods{
		"商品1", 100, "中国", "特价",
	}

	bytes, err := json.Marshal(goods)

	if err != nil {
		panic(err)
	}

	fmt.Printf("%s", bytes)
}
---结果---
{"Name":"商品1","Price":100,"address2":"中国","Tag":"特价"}
```
把结构体转换成json可以使用Marshal方法，有一点需要注意: 结构体的属性成员首字母必须大写，但是可以使用注解的Tag标注转换成json之后的key名称。

json字符串转换成结构体步骤差不多：
```
package main

import (
	"encoding/json"
	"fmt"
)

type Goods struct {
	Name    string
	Price   int
	Address string `json:"address2"`
	Tag     string
}

func main() {
	jsonStr := `{"Name":"商品1","Price":100,"address2":"中国","Tag":"特价"}`

	goods := Goods{}

	err := json.Unmarshal([]byte(jsonStr), &goods)
	if err != nil {
		panic(err)
	}

	fmt.Printf("%v", goods)
}
---结果---
{商品1 100  特价}
```
这在我们平时写接口或者请求接口的时候非常好使，简单易用！

好了，今天就介绍这么多了，谢谢！
