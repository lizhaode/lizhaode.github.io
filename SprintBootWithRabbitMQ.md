# Sprint Boot 连接 RabbitMQ 发布消息

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

- 添加配置代码

  - 其实也可以用 `@SpringBootConfiguration`
  - 每个都要加上 `@Bean` 代表由 Spring 进行管理，这样才能生效

```groovy
@Configuration
class CustomRabbitConfig {

    @Bean
    Queue directQueue(){
        return new Queue('q名字')
    }

    @Bean
    DirectExchange topicExchange(){
        return new DirectExchange('exchange名字')
    }

    @Bean
    Binding binding(Queue q, DirectExchange d){
        return BindingBuilder.bind(q).to(d).with('routingKey名字')
    }
}
```


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

**感觉这样有点问题，首先是具体send的时候还需要重复填写exchange和routingKey不太合理，另外，如果是多个队列这样就不合适了**

留个坑，以后填