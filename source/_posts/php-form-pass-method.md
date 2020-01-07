---
title: 浅谈PHP前后端传参常见的几种方式
date: 2017-09-05 15:00:00
category: PHP
tags: 
    - PHP
---

在Web开发里面，有前后端之分，它们之间的交互主要通过传参的方式，但是这个传参也分几种形式，比如说Form表单提交、Ajax提交...今天我就在这里总结一下开发中常见的几种形式：

### 1. Form表单提交
这种方式是最原始最常见的方式，提交的时候也有可能是通过js触发，其请求头Content-Type为: application/x-www-form-urlencoded，示例如下：
##### 前端代码：
```
<body>
    <form action="backend.php" method="post">
        <label for="name">姓名:</label>
        <input type="text" id="name" name="name">
        <label for="name">年龄:</label>
        <input type="text" id="age" name="age">
        <input type="submit" value="提交">
    </form>
</body>
```

<!--more-->

##### 后端接收：
```
<?php
var_dump($_POST['name']);
var_dump($_GET['age']);
var_dump($_REQUEST['age]);
```
##### 请求头：
![](http://upload-images.jianshu.io/upload_images/3571187-3b1aa32292a0f9e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这种提交方式也是ajax默认的提交方式,请求参数是以key-value键值对的形式传递到后端,在PHP里面通$_POST等超全局变量就可以获取到,简单实用。其未经解析的原始的数据其实是：name=PHP&age=25
![](http://upload-images.jianshu.io/upload_images/3571187-c2d0236765996437.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2. JSON形式提交
这种形式，需要设置一下请求头Content-Type为application/json，实例如下：
##### 前端代码：
```
 $.ajax({
            type: 'POST',
            url: "backend.php",
            data: {
                'name': 'hello',
                'age': 15,
            },
            contentType: 'application/json',
            dataType: "json",
            success: function (data) {
                console.log(data);
            }
        });
```
##### 请求头：
![](http://upload-images.jianshu.io/upload_images/3571187-cbf271fe3dd4bad7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上面的截图可以看到，请求参数那里变成Request Payload，虽然格式上看上去和之前form提交差不多，但是这时候如果后台用$_POST这类方法是无法获取的，需要换一种方式：
```
$input = file_get_contents('php://input');
```
上面这种方式获取到的内容是字符串: name=Jun&age=15，在这个例子里面反而不容易处理了，实际上采用json这种方式提交的参数的话，一般都是把需要的数据封装成json格式提交，在js里面就是把数据放到对象里面，然后序列化：
```
        var data = {
            'name': 'Jun',
            'age': 15,
        };
        $.ajax({
            type: 'POST',
            url: "backend.php",
            data: JSON.stringify(data),
            contentType: 'application/json',
            dataType: "json",
            success: function (data) {
                console.log(data);
            }
        });
```
这是再查看请求头：
![](http://upload-images.jianshu.io/upload_images/3571187-7f20f6b265d2e8bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到参数变成json格式，这时候PHP后端就可以采用json_decode函数去获取参数：
```
$input = json_decode(file_get_contents('php://input'), true);
```

### 3.文件上传
> 一般上传图片等各种文件的时候用的到，Content-Type是 multipart/form-data

请求头类似如下：
```
------WebKitFormBoundary63FiWN3UoYxd8OT6
Content-Disposition: form-data; name="UploadFile"; filename="QQ截图20170925101502.png"
Content-Type: image/png


------WebKitFormBoundary63FiWN3UoYxd8OT6
Content-Disposition: form-data; name="sid"

sid
------WebKitFormBoundary63FiWN3UoYxd8OT6
Content-Disposition: form-data; name="fun"

add
------WebKitFormBoundary63FiWN3UoYxd8OT6
Content-Disposition: form-data; name="mode"

```

### 4. 总结
这几种方式功能上说没什么区别，都能实现数据的提交，大家选择自己喜欢的方式就行，最重要的是前后端协调好, 虽然这里后端是以PHP为例，但是其他语言也是大同小异。最后，再说一下数组提交，这个倒不是新的提交方式，我这里是指遇到那种一个字段提交多个数据的情况，比如说删除多个文章，一般前端需要传多个id，举例子字段名字叫ids，一般有这2种方案：
###### 1. 逗号相隔
![](http://upload-images.jianshu.io/upload_images/3571187-5ca0c75d95ef56bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样传参，后端获取到之后是一个字符串，在PHP里面可以用explode这样的函数去把字符串拆分成数组，非常方便，当然你也可以选择其他分隔符，比如说“-”，“+”等字符。
###### 2. JSON形式
这就是文中说的第二种方式，把id放在数组里面以json方式传到后台，这样后台可以直接获取到一个数组.
![](http://upload-images.jianshu.io/upload_images/3571187-8a85e131e56d62b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



