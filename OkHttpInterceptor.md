# 关于 OkHttp3 上 interceptor 的一些整理

## 背景

在 Spring Cloud 中一般会使用 Feign 来发送 http 请求，发现他可以自定义 client ，官方也提供了 OkHttp 作为 client

OkHttp 官方提供了非常好用的 log 打印工具 `logging-interceptor`

在爬一些网站的时候，也需要自定义一些 Request Header ，比如 `User-Agent`, `X-Forwarded-For`, `REMOTE_ADDR` 等等(这些头的作用等到爬虫的帖子再说)


## 使用官方提供的 log 打印请求

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

- OkHttp 自带的拦截器中有一个 `BridgeInterceptor` 他会做一个事情是：检查 Request Header 中是否有 `Accept-Encoding` ，没有的话会添加，并且设置成 `gzip` (**但是在官方文档中看到有 `BrotliInterceptor` 替代了，需要详细查一下[BrotliInterceptor](https://square.github.io/okhttp/4.x/okhttp-brotli/okhttp3.brotli/-brotli-interceptor/)**)

综上，打印log还是放到 Network Interceptors 比较合适


## 在 Header 中添加一些自定义的头

老规矩，先放代码

```groovy

```