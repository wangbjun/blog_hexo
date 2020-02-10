---
title: Golang测试用例编写
date: 2020-02-10 18:15:00
tags: Golang
category: 编程开发
keywords: golang,单元测试，功能测试
---

如果你看过很多开源库的源码，你会发现大部分项目都有很多详细的测试代码，一般来说测试覆盖率越高说明这个项目的质量越高，所以好的项目测试是少不了的。很多公司对代码测试覆盖率也有要求，不为别的，只为更好的代码质量。

## 1.分类

虽然业界有一直开发模式叫做测试驱动开发（TDD），但是了解的人都知道```TDD```对开发要求太高了，它要求你先写测试用例然后再写代码，需要你写代码之前思考很多，需要大量时间，我实际开发中并没有采用过这种模式，估计国内都应该很少。

我们可以粗略的把测试用例简单划分为2种类型，一种是```单元测试```，它是针对某个模块、函数、方法的测试，另一种是```功能测试（集成测试）```，它是针对整个项目功能是否可用的测试。举个例子，你写个了Web服务接口，单元测试可能是针对这个接口里面调用的一个函数测试，而功能测试就是测试这个接口是否可用，因为一个接口可能调用了多个函数。

## 2.单元测试
Golang里面的测试和其它大部分语言的测试不多，只不过表示形式略有不同，比如Go的单元测试通常情况下是和被测试的代码放在一起的，以```xxx_test.go```命名并且测试的函数名必须以```Test```开头。

<!--more-->

例如：
math.go 有以下2个函数
```go
package main

import "errors"

func Div(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}
```
如果想测试这个文件，那么测试文件名字就应该叫 math_test.go
```go
package main
import (
	"testing"
)
func TestDiv(t *testing.T) {
	i, err := Div(4, 2)
	if err != nil {
		t.Fail()
	}
	if i != 2 {
		t.Fail()
	}
}
```
切换到工作目录下执行 ```go test```即可，这个命令有很多附加参数，比如说```-v```可以查看详细情况，```-coverprofile```可以看测试覆盖率。
```
jwang@jun:~/Documents/Work/test$ go test -v -coverprofile=c.out
=== RUN   TestDiv
--- PASS: TestDiv (0.00s)
PASS
coverage: 66.7% of statements
ok      _/home/jwang/Documents/Work/test       0.001s
```

根据测试函数参数类型的不同，Go里面把测试又细分为```*testing.T和*testing.B```，其实B是性能基准测试，通常用来测试算法性能，这里就不多说了。

单元测试的目的就是尽可能的覆盖到所有情况，说白了，就是枚举各种情况，根据输入的参数人工推导正确的结果，然后和实际得出的结果做比对，如果失败则说明程序有bug，比如上面的例子明显没有覆盖到所有情况，只达到了66.7%。

上面这段测试代码主要是没有覆盖到被除数为0的情况，下面完善一下：
```go
package main
import (
	"testing"
)
func TestDiv(t *testing.T) {
	i, err := Div(4, 2)
	if err != nil {
		t.Fail()
	}
	if i != 2 {
		t.Fail()
	}

	i, err = Div(4, 0)
	if err == nil {
		t.Fail()
	}
}
```
重新执行```go test```会发现覆盖率达到了100%，也就是所有语句都覆盖到。

>请注意，覆盖率达到100%并不意味着代码没有问题。

## 3.表格测试
表格测试严格来说并不是一种测试类型，只是一种测试方式，就是一种套路，上面的例子里面，我们需要手动构造每一个测试的入参和出参后执行、断言结果，有很多重复代码，我们可以使用表格测试优化一下：
```go
package main

import (
	"testing"
)
func TestDiv(t *testing.T) {
	var tests = []struct {
		a        int
		b        int
		expected int
		err      error
	}{
		{4, 2, 2, nil},
		{4, 1, 4, nil},
		{5, 2, 2, nil},
		{4, 0, 0, DivisionByZeroError},
	}

	for _, v := range tests {
		i, err := Div(v.a, v.b)
		if i != v.expected || err != v.err {
			t.Errorf("input %d, %d, expected %d, got %d", v.a, v.b, v.expected, i)
		}
	}
}
```
这种方式比较简洁，参数一目了然，而且方便扩展添加新的用例，这里需要注意一下那个error，可以先定义一个自定义的error方便判断，同时使用了```t.Errorf```格式化入参和出参方便排查错误。

>为了更方便的断言结果，我们可以使用第三方的assert库，Github上面也有很多开源的测试库，可以简化你的操作，更快速的编写测试用例。

## 4.功能测试
功能测试就和你用Postman去测试一样，我们需要把这个服务启动起来，然后模拟用户的操作，去测试结果是否符合预期。测试本身是个非常广泛的话题，有很多种方式，这里我只说说平时用的比较多的Http服务的接口测试。

首先，我们需要了解一下Go里面Http服务的创建方式，最简单的方式莫过于下面这种：
```go
package main

import (
	"net/http"
	"strconv"
)

func main() {
	http.HandleFunc("/div", DivHandler)
	_ = http.ListenAndServe(":8888", nil)
}

func DivHandler(writer http.ResponseWriter, request *http.Request)  {
	a := request.PostFormValue("a")
	b := request.PostFormValue("b")

	paramA, _ := strconv.Atoi(a)
	paramB, _ := strconv.Atoi(b)

	i, err := Div(paramA, paramB)
	if err != nil {
		_, _ = writer.Write([]byte("error"))
		return
	}
	_, _ = writer.Write([]byte(strconv.Itoa(i)))
}
```

上面这段代码就是使用了Go自带的http库创建了一个Web服务，它提供了一个接口，功能和之前的函数一样，如果出错的话就返回error。

我们可以使用Go的一个http recorder对这个http服务进行测试，方法如下：
```go
package main

import (
	"io/ioutil"
	"net/http"
	"net/http/httptest"
	"net/url"
	"strings"
	"testing"
)

func TestDivHandler(t *testing.T)  {
	recorder := httptest.NewRecorder()

	params := url.Values{}
	params.Add("a", "42")
	params.Add("b", "2")

	request, _ := http.NewRequest("POST", "/div", strings.NewReader(params.Encode()))
	request.Header.Add("Content-Type", "application/x-www-form-urlencoded")

	DivHandler(recorder, request)

	if recorder.Result().StatusCode != 200 {
		t.Error("Test failed")
	}

	body, _ := ioutil.ReadAll(recorder.Result().Body)
	if string(body) != "21" {
		t.Error("Test failed")
	}
}
```
在这个测试用例里面，我主要测试了2点，一个是返回码是不是200，另外测试了一下正常的返回结果。不过很明显，我这里并没有覆盖到异常情况。

很多Go的Web框架，比如Beego和Gin，框架本身会多一层路由，但是测试方式大同小异，主要还是使用http recorder来实现，这里就不多说了。

## 5.总结
这里介绍的只是最简单测试方式，实际开发中想要完全做好测试还有很多问题，比如有些系统有很多外部依赖，在测试的时候可能还要借助于mock。再比如有很多Web服务还涉及到数据库层，想要完整测试还要做好数据回滚。

国内很多公司对测试要求并不严格，很多公司都不要求写测试，有些虽然有测试覆盖率要求，但是也是为了应付（代码都写完了，后面再加测试也就图个心理安慰），测试用例并无法保证代码质量，我觉得真正想提高代码质量还是得靠```code review```。








