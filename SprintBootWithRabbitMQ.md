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

### 接收消息

- 配置文件

```yml
spring:
  rabbitmq:
```