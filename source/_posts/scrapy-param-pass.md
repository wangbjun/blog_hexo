---
title: 关于Scrapy爬虫数据传递问题
date: 2016-01-07 12:08:45
tags: scrapy
category: 编程开发
---

## 问题：
这两天研究爬虫掉进一个大坑，爬了好久才爬出去，这里说几句，我写的爬图片的爬虫很简单，从一个图片列表进二级图片详情页，然后爬取二级详情页的所有图片，但是有个需求就是需要以二级详情页的标题为目录分类存放图片！思路很简单，就是在item里面增加一个字段title存放标题：
```
class PicscrapyItem(scrapy.Item):
    image_urls = scrapy.Field() # 图片地址
    images = scrapy.Field()
    title = scrapy.Field() # 图片标题（目录）
```
然后在pipelines里面获取item里面数据，保存的时候做一下处理：
```
class PicscrapyPipeline(ImagesPipeline):
    item = []
    def get_media_requests(self, item, info):
        self.item = item
        return [Request(x) for x in item.get(self.images_urls_field, [])]

    # 重写函数，修改了下载图片名称的生成规则
    def file_path(self, request, response=None, info=None):
        if not isinstance(request, Request):
            url = request
        else:
            url = request.url
        url = urlparse(url)
        img_name = url.path.split('/')[5].split('.')[0]
        return self.item['title'] + '/%s.jpg' % img_name
```
上面的代码看上去没毛病，重写了Scrapy框架ImagesPipeline的方法,根据title字段分目录存放，但是当我跑起来的时候看上去也没毛病，但是查看数据的时候却不对了，目录是出来了，但是牛头不对马嘴！

<!--more-->

---
## 分析：
研究了好久我才发现问题就在于多线程，Scrapy框架默认是开启多线程的，在settings里面有个字段可以定义开启的线程数，默认是开启16个线程同时爬取：
```
# Configure maximum concurrent requests performed by Scrapy (default: 16)
# CONCURRENT_REQUESTS = 32
```
我上面的代码如果是单线程运行没毛病，但是多线程的话，数据是共享的，就会错乱，导致图片保存的位置根本不是我想要的结果，怎么解决呢？
#### 1. settings
在settings里面设置线程数为1，釜底抽薪，不过即使这样设置，偶尔也会出现错乱，这种方法牺牲了爬取效率，不可取
#### 2. meta
我之所以采取类共享的方式传递item是因为在file_path函数内部我无法获取到item的值，后来，网上查了好久发现有一种方式可以在函数间安全传递数据，就是request的meta属性，所以正确的做法如下：
```
class PicscrapyPipeline(ImagesPipeline):
    def get_media_requests(self, item, info):
        return [Request(x, meta={'title': item['title']}) for x in item.get(self.images_urls_field, [])]

    # 重写函数，修改了下载图片名称的生成规则
    def file_path(self, request, response=None, info=None):
        if not isinstance(request, Request):
            url = request
        else:
            url = request.url
        url = urlparse(url)
        img_name = url.path.split('/')[5].split('.')[0]
        return request.meta['title'] + '/%s.jpg' % img_name
```
然后问题解决，新技能get！


