# 爬虫初步 -- PornHub


## 前言

爬虫是一个大坑，特别最近关于爬虫出事的新闻出来，这个东西变得敏感了起来

传言说得好，`爬虫写的好，牢饭吃到饱` 

所以写爬虫一定要注意这方面的问题

- 不能爬取非公开且用户协议中明确标明不能获取的信息

- 爬虫不要一味求快而造成被爬取的服务器异常


黑五的时候，世界知名网站 PornHub 搞了一个永久会员的活动，所以正好有兴趣爬一下传说中的高清视频，顺便看看流量最大(存疑)的网站有没有什么特别的反爬措施

所以本文就围绕爬取 PornHub 说一下在爬取过程中常见的一些问题


## 什么是爬虫与框架

像 百度 Google 这些搜索引擎就是一个爬虫

爬虫的逻辑简单归纳，就是 请求网页 -> 提取数据 -> 清洗存储

所以只要你的代码实现了这些功能，就是一个爬虫

比如：

Python: requests + beautifulsoup

Java: retrofit(okhttp) + jsoup

如果是轻量的爬取一些东西，直接用这些库上手撸代码足够了

但是在实际生产中，会产生多种多样的业务情况，比如：

- 网络请求需要伪装不同的 Http Headers
- 网络请求需要随机选取一个代理服务
- 多层级，多数量网页获取，类似下一页、通过链接进入下一个网页
- 根据不同类型的数据选择不同的清洗逻辑，入不同的数据库

在写不同的爬虫的时候，上边列出的问题都可能会遇到，使用框架会大大减少重复工作的情况

Python 语言下有个超级好用的框架 [Scrapy](https://scrapy.org/)

此次爬取 PornHub 就是使用 Scrapy 框架来编写，具体示例代码可以看 [pornhub-spider](https://github.com/lizhaode/pornhub)

下面只说一下这次爬取过程中遇到的问题

## 遇到的问题

### 1. 获取视频地址

爬取 PornHub 当然是冲着视频去的，不然只是要收集一下都有哪些片商，哪些model，片子的名字多没意思

所以在分析视频播放页面时，遇到了问题

视频的链接不是直接放在 Html 里边的，也不是通过 XHR 等形式去动态请求视频链接，Html 中有一段 js ，这段 js 会拼接出来类似 `quality_2160p` ，`quality_1080p` 格式的变量，链接就是这些变量的值

所以遇到的第一个问题就是通过 js 动态生成的页面怎么处理

思路一般就是使用类似 Selenium 这种前端自动化工具，让浏览器来打开这个页面，这样肯定能解决问题

但是这样带来的缺陷也是很明显的，由于需要真的调用浏览器打开网页，非常消耗资源，Selenium 速度也很慢，会拖累爬虫的速度

那么 Python 怎么能执行 js 语句呢

用 [js2py](https://github.com/PiotrDabkowski/Js2Py)

范例代码

```python
exec_js = 'function f(){' + prepare_js + 'if (typeof (quality_2160p) != "undefined") { return quality_2160p; } else if (typeof (quality_1440p) != "undefined") { return quality_1440p;} else if (typeof (quality_1080p) != "undefined") {return quality_1080p;} else if (typeof (quality_720p) != "undefined") { return quality_720p; } else { return ""; }} '
f = js2py.eval_js(exec_js)
video_url = f()
```

`prepare_js` 变量是从 Html 中提取的 js语句，然后想提取出来最高的清晰度，所以自己在外边增加了一个方法

这样 `js2py.eval_js()` 返回的就是一个 func， 需要手动执行一下才能获取结果

### 2. 下载视频

Scrapy 框架自带了 `FilesPipeline` 来处理爬取到数据后，将数据下载成文件这种情况

我们这次爬虫，主要目标就是为了爬取相应的视频，所以获取到视频地址当然就扔给`FilesPipeline`去下载

然而实际运行爬虫时，会发现遇到一个问题：**内存占用巨大**

翻看 Scrapy 相关的代码，发现 `FilesPipeline` 每次接到 Twisted 的 Response 之后，都是用 `BytesIO` 将接到的 Response 完全读取到内存中

请求视频链接的 Http response body 就是二进制的视频流文件，如果是 1080P 甚至往上的高清视频，都是几百M甚至上G大小的

这种情况下，不能直接将 response body 完整读取到内存，而是作为数据流来处理，每次读取一定大小，处理完成后继续读取，直到 response body 读取完毕

给 Scrapy 提了 issue 询问之后，告知目前没有人力来解决这个问题，在官方修复之前，只能用一些替代的方法了

1. 在 pipeline 中用 requests 自己做请求，使用`stream=True`参数。不交给 Scrapy 来处理

但是这样你需要自己处理用 requests 多线程下载，不然默认的单线程下载视频比较慢

2. 请求 Aria2 rpc 接口，交给 Aria2 下载

使用 Aria2 虽然不需要考虑下载过程的处理，但是需要处理 Aria2 下载结果，比如下载失败要重试

这两种方法都会带来一个问题，pipeline 处理完一个 item 才会处理下一个，所以实际上你只能同时下载一个视频

除非用 Aria2 下载的时候，你不关心下载结果，只将下载任务扔给 Aria2

为什么使用`FilesPipeline`就可以实现同时下载多个视频呢？

因为 Scrapy 框架对所有 Http 请求是使用 Twisted 来进行管理的，Scrapy 的并发实际上是 Twisted 对 Http 请求的并发，所以自定义的 Pipeline 中没有并发

[回到目录](README.md)