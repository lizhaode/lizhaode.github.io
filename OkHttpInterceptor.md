# 关于 OkHttp3 上 interceptor 的一些整理

### 背景

在 Spring Cloud 中一般会使用 Feign 来发送 http 请求，发现他可以自定义 client ，官方也提供了 OkHttp 作为 client

OkHttp 官方提供了非常好用的 log 打印工具 `logging-interceptor`

在爬一些网站的时候，也需要自定义一些 Request Header ，比如 `User-Agent`, `X-Forwarded-For`, `REMOTE_ADDR` 等等(这些头的作用等到爬虫的帖子再说)


### 使用官方提供的 log 打印请求

```groovy
@Slf4j
class CustomFeignConfig {

    @Bean
    OkHttpClient feignClientConfig() {
        HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor(new HttpLoggingInterceptor.Logger() {
            @Override
            void log(String s) {
                log.info(s)
            }
        })
        interceptor.level(HttpLoggingInterceptor.Level.BODY)
        okhttp3.OkHttpClient client = new okhttp3.OkHttpClient.Builder().addNetworkInterceptor(interceptor).build()
        return new OkHttpClient(client)
    }
}
```

代码还是使用的 Spring Cloud 的 Feign 的配置，重点看 OkHttp 的配置

- 直接 new 准备好的 HttpLoggingInterceptor ，实现他的 Logger 方法就行了。这里吹一波 Groovy ，打印log直接加 `@Slf4j` 注解就完了，Java 还得引 lombok 才能这么用，Kotlin 就不知道有没有这样简便的方法了

- 这里使用 `addNetworkInterceptor()` 方法添加拦截器，不用 `addInterceptor()` 是因为这两种拦截器的顺序不一样

- 差别简单来说就是 会先执行 `addInterceptor()` 的拦截器，然后执行 `addNetworkInterceptor()` 添加的拦截器，前者只执行一次，而后者每次网络请求都会执行一次，具体的讲解请看官方文档 [Interceptors](https://square.github.io/okhttp/interceptors/)

- OkHttp 除了自定义的拦截器之外，还有自带的几个拦截器，每次执行请求的时候都会添加到一个 List 中，用户添加的放在最前边

- OkHttp 自带的拦截器中有一个 `BridgeInterceptor` 他会做一个事情是：检查 Request Header 中是否有 `Accept-Encoding` ，没有的话会添加，并且设置成 `gzip` 

综上，打印log还是放到 Network Interceptors 比较合适


### 在 Header 中添加一些自定义的头

老规矩，先放代码

```groovy
class RandomIPAndUA {

    static String getIp() {
        Random r = new Random()
        return String.format("%d.%d.%d.%d",
                r.nextInt(255), r.nextInt(255), r.nextInt(255), r.nextInt(255))
    }

    static String getUA() {
        Random r = new Random()
        return UAList[r.nextInt(UAList.size())]
    }
    // 避免长度问题，只放上来几个
    private static List<String> UAList = [
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2227.1 Safari/537.36",
            "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2227.0 Safari/537.36",
    ]
}
```

先写一个动态生成 UA 和 IP 的方法

```groovy
class HeaderInterceptor implements Interceptor {

        @Override
        Response intercept(@NotNull Chain chain) throws IOException {
            Request request = chain.request()
            Request newRequest = request.newBuilder().removeHeader('Cookie').addHeader('Accept-Language', 'zh-CN,zh;q=0.9')
                    .addHeader('User-Agent', RandomIPAndUA.getUA())
                    .addHeader('REMOTE_ADDR', RandomIPAndUA.getIp()).build()
            return chain.proceed(newRequest)
        }
    }
```

然后写一个实现 OkHttp 的 interceptor 接口的类，实现他的 intercept 方法就行了

在这个方法里，可以修改 Request 也可以修改 Response

然后在 OkHttp Client 中加上自己的拦截器就生效了


### OkHttp 内置的 interceptor

##### BridgeInterceptor

这个就不上代码了，这是 OkHttp 自己的代码

这个拦截器大体上做的事情就是

- 检查 Request Body 是否存在，存在的话取到他的类型，放到 `Content-Type` header 里
- 检查 Request Body 的 `body.contentLength()`，有值就放到 `Content-Length` header 里，并删除 header 中的 `Transfer-Encoding`，不然就删除 `Content-Length` 设置 `Transfer-Encoding: chunked`
- 检查 Request Header 的 `Connection` ，没有的话设置成 `Keep-Alive`
- 重头戏来了，检查 Request Header 中 `Accept-Encoding` 和 `Range`，都没设置的话，就设置 `Accept-Encoding: gzip`
- 其他的不太重要就不一一叙述了，比如设置 `Cookie`, `User-Agent`
- 同样，对设置了 `Accept-Encoding: gzip` 的请求，用 gzip 处理 Response


##### BrotliInterceptor

[项目链接](https://github.com/square/okhttp/tree/master/okhttp-brotli)

介绍
```
This module is an implementation of Brotli compression.
It enables Brotli support in addition to tranparent Gzip support, provided Accept-Encoding is not set previously. Modern web servers must choose to return Brotli responses. n.b. It is not used for sending requests.
```

与 `BridgeInterceptor` 类似，也是需要用户不手动指定 `Accept-Encoding` 才会启用

这个项目只有一个文件 [BrotliInterceptor.kt](https://github.com/square/okhttp/blob/master/okhttp-brotli/src/main/java/okhttp3/brotli/BrotliInterceptor.kt)

实现的逻辑也很简单，就是在请求头中添加 `Accept-Encoding: br,gzip`，然后在 Response 时处理一下

关于 [Brotli](https://github.com/google/brotli)

```
Brotli is a generic-purpose lossless compression algorithm that compresses data using a combination of a modern variant of the LZ77 algorithm, Huffman coding and 2nd order context modeling, with a compression ratio comparable to the best currently available general-purpose compression methods. It is similar in speed with deflate but offers more dense compression.
```

又是谷歌搞得一个压缩算法，据说压缩比远远高于 gzip

[回到目录](README.md)