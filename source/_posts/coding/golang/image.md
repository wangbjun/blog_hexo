---
title: Golang之图片处理
date: 2020-07-19 16:01:00
tags: Golang
category: Golang
---
之前做项目有一个图片裁剪需求，没想到Golang标准库对图片处理的支持还挺强大的，在此记录一下，以备后用。

在Golang里面图片处理主要依赖一个image库，支持常见的png、jpeg、gif等格式，使用这个库我们可以做生成图片、读取图片、裁剪图片等操作。

这篇文章只介绍某几个常用的操作，详细API和使用的技巧大家可以参考Go官方文档和博客，比如：https://blog.golang.org/image-draw

<!--more-->

## 1.生成图片
```go
func genImage() {
    m := image.NewRGBA(image.Rect(0, 0, 640, 480))

    draw.Draw(m, m.Bounds(), &image.Uniform{C: color.White}, image.Point{}, draw.Src)

    f, err := os.Create("demo.jpeg")
    if err != nil {
        panic(err)
    }
    err = jpeg.Encode(f, m, nil)
    if err != nil {
        panic(err)
    }
}
```
以上代码可以生成一个长宽640x480，内容白色的图片，虽然没什么用，但是咱可以通过这些代码简单了解一下这个库的一些API。

## 2.读取图片
图片也是一个文件，读取图片文件和一般文件是一样的，但是如果我们把这个文件当成图片去处理，必须经过一个decode操作，decode操作就是去校验这个文件是否是图片，是否是支持的格式。

不同的图片格式的编码存储方式都不一样，这些信息存放在文件头部分，而当我们需要把处理完的图片保存到文件里面的时候，还需经过一个encode操作。

<img src="/images/2020-07-19.png" width="25%"/>

本文以这个文件为例，展示一些操作：
```go
func readImage()  {
    f, err := os.Open("ubuntu.png")
    if err != nil {
        panic(err)
    }
    // decode图片
    m, err := png.Decode(f)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%v\n", m.Bounds())     // 图片长宽
    fmt.Printf("%v\n", m.ColorModel()) // 图片颜色模型
    fmt.Printf("%v\n", m.At(100,100))  // 该像素点的颜色
}
```
通过decode读取的只是一个image.Image类型，其实现的方法并不多，只有3个，如果我们要做一些操作，则需把其转换成相应的颜色模型，比如RGBA、CMYK等。
```go
// Image is a finite rectangular grid of color.Color values taken from a color
// model.
type Image interface {
    // ColorModel returns the Image's color model.
    ColorModel() color.Model
    // Bounds returns the domain for which At can return non-zero color.
    // The bounds do not necessarily contain the point (0, 0).
    Bounds() Rectangle
    // At returns the color of the pixel at (x, y).
    // At(Bounds().Min.X, Bounds().Min.Y) returns the upper-left pixel of the grid.
    // At(Bounds().Max.X-1, Bounds().Max.Y-1) returns the lower-right one.
    At(x, y int) color.Color
}
```

在源码里面大家可以看到很多下面类似代码，有很多种不同的颜色模型，他们都实现了这个3个方法。
```go
// RGBA is an in-memory image whose At method returns color.RGBA values.
type RGBA struct {
    // Pix holds the image's pixels, in R, G, B, A order. The pixel at
    // (x, y) starts at Pix[(y-Rect.Min.Y)*Stride + (x-Rect.Min.X)*4].
    Pix []uint8
    // Stride is the Pix stride (in bytes) between vertically adjacent pixels.
    Stride int
    // Rect is the image's bounds.
    Rect Rectangle
}

func (p *RGBA) ColorModel() color.Model { return color.RGBAModel }

func (p *RGBA) Bounds() Rectangle { return p.Rect }

func (p *RGBA) At(x, y int) color.Color {
    return p.RGBAAt(x, y)
}
```

## 3.图片裁剪
下面这个操作就可以把整个图片截取上半部分，而且是无损的裁剪，其主要依靠SubImage方法：
```go
func readImage() {
    f, err := os.Open("ubuntu.png")
    if err != nil {
        panic(err)
    }
    // decode图片
    m, err := png.Decode(f)
    if err != nil {
        panic(err)
    }
    rgba := m.(*image.RGBA)
    subImage := rgba.SubImage(image.Rect(0, 0, 266, 133)).(*image.RGBA)

    // 保存图片
    create, _ := os.Create("new.png")
    err = png.Encode(create, subImage)
    if err != nil {
        panic(err)
    }
}
```

## 4.图片转Base64
base64适合一些小图片的存储，经常也会用到，操作很简单：
```go
func b64() {
    f, err := os.Open("ubuntu.png")
    if err != nil {
        panic(err)
    }
    all, _ := ioutil.ReadAll(f)

    str := base64.StdEncoding.EncodeToString(all)
    
    fmt.Printf("%s\n", str)
}
```
实际上，这块和image库没关系，但是需要注意一点，上述的结果并不包含头部分，所以得自己加一个，比如: **data:image/png;base64,**

## 5.图片大小压缩
为了节省流量和存储空间，一般都会对图片做有损压缩，比如手机拍一张照片大概有7-8MB，但是网页上一般100-200k大小就可以了，太大的话影响用户体验不说，服务器带宽压力也大。

Go的话标准库支持还不够，如果自己去实现也比较麻烦，这里推荐一个第三方库：https://github.com/nfnt/resize

这个库支持多种采样算法，以满足各种需求，支持等比或者固定比缩放，有需要的人可以尝试一下。

```go
package main

import (
    "github.com/nfnt/resize"
    "image/jpeg"
    "log"
    "os"
)

func main() {
    // open "test.jpg"
    file, err := os.Open("test.jpg")
    if err != nil {
        log.Fatal(err)
    }
    // decode jpeg into image.Image
    img, err := jpeg.Decode(file)
    if err != nil {
        log.Fatal(err)
    }
    file.Close()

    // resize to width 1000 using Lanczos resampling
    // and preserve aspect ratio
    m := resize.Resize(1000, 0, img, resize.Lanczos3)

    out, err := os.Create("test_resized.jpg")
    if err != nil {
        log.Fatal(err)
    }
    defer out.Close()

    // write new image to file
    jpeg.Encode(out, m, nil)
}
```

总体来说，这个库的功能还是比较简单的，能够满足一些基本需求，再深入的话可能就得自己去实现了，毕竟Go只是通用语言，不可能把一些图片处理算法都给你实现好，但是框架已经给你搭好了。
