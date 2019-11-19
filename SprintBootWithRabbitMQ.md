# Sprint Boot 连接 RabbitMQ 发布/接收消息

### 背景

公司的测试项目中，开发提供了一个造单的方法，通过向指定的 Queue 发布指定的数据，可以构造想要的状态的订单

为了能够提供给测试同事使用，所以直接使用 Spring Boot 来做，可以方便打成 jar 包分发

### 详细要点

- 先搞定配置文件

```yaml
spring:
  rabbitmq:
    host: xxxx
    port: 5021
    username: xxx
    password: xxxx
    virtual-host: xxxx
```
`virtual-host` 一定要填，不然会找不到具体的 MQ 实例


- 最终发送消息的代码这样

```groovy
@Slf4j
@Component
class MQSender {

    @Autowired
    private RabbitTemplate rabbit

    void send(String message) {
        log.info("start send MQ message:$message")
        rabbit.convertAndSend('exchange名字', 'routingKey名字', message)
    }
}
```

发送消息的时候要注意，只能发 `String` 或者 `byte` 类型的数据，其他类型会出现类型转换的问题

### 接收消息

- 配置文件

```yml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 10
```

每个 Queue 的 listener 有 10 个

代码中这么写

```groovy
@Slf4j
class HouseDataLinstener {

    @RabbitListener(queues = 'house')
    void parseHouse(Message message) {
        log.info("parseHouse tag:${message.messageProperties.consumerTag}")
        HouseList house = new ObjectMapper().readValue(message.body, HouseList)
        log.info("receive house data:$house")
    }
}
```

##### 注意

`RabbitListener` 注解的 `queues` 用来设置监听哪个 Queue，他的类型是 `String[]` 所以可以配置多个

- 关于 RabbitMQ
  - `exchange` 和 `queue` 是通过 `routingKey` 绑定的
  - 通过相同的 `routingKey` 可以绑定多个 `queue` 到一个 `exchange` 上
  - 这样发消息的时候，指定一个 `exchange` 和 `routingKey` 就可以向多个 `queue` 发送消息