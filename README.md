#### 常见的交换机类型：

- **Direct Exchange**：直连交换机。消息会被路由到所有`路由键`与`绑定队列`。
- **Fanout Exchange**：广播交换机。消息会被路由到所有`绑定队列`，忽略路由键。
- **Topic Exchange**：主题交换机。消息会被路由到所有`绑定队列`，支持通配符。

最简单的消息队列只需要`生产者类`和`消费者类`，如下：

```java
@SpringBootTest
public class RabbitProducer {

    @Test
    public void testSimpleQueue() {
        // 创建连接工厂
        CachingConnectionFactory connectionFactory = new CachingConnectionFactory("192.168.20.80");
        connectionFactory.setUsername("leke");
        connectionFactory.setPassword("leke@@@");
        // 创建 RabbitTemplate
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        // 发送消息
        rabbitTemplate.convertAndSend("e.test", "q.test", "hello, spring amqp!");

    }
}


@Component
public class RabbitConsumer {
    // 表明监听的队列名称
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = "q.test", durable = "true"),
            exchange = @Exchange(value = "e.test")))
    public void listenSimpleQueueMessage(String msg) {
        //3、打印消息，模拟处理消息
        System.out.println("spring 消费者接收到消息：【" + msg + "】");
    }

}
```



**但从项目设计角度出发，我们需要这样使用消息队列，如下：**

`RabbitMqConstant`常量类：存放交换机名称和队列名称

```java
public class RabbitMqConstant {

    public static final String TEST_EXCHANGE = "e.test";
    public static final String TEST_QUEUE = "q.test";

}
```

`RabbitConfig`配置类：注册交换机类和队列类，绑定交换机和队列（如果采用注解绑定，可以不写）

```java
public class RabbitConfig {
    @Bean
    public DirectExchange testExchange() {
        return new DirectExchange(RabbitMqConstant.TEST_EXCHANGE);
    }

    @Bean
    public Queue testQueue() {
        return new Queue(RabbitMqConstant.TEST_QUEUE);
    }

    @Bean
    public Binding testBinding() {
        return BindingBuilder.bind(testQueue())
                .to(testExchange())
                .with(RabbitMqConstant.TEST_QUEUE);
    }
    
    // 其他的配置，例如配置消息监听容器工厂。以下代码定义了如何处理消息消费过程中的异常情况，特别是通过自定义的异常策略来控制消息的重试逻辑，增强了消息处理的健壮性和灵活性。
        @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            SimpleRabbitListenerContainerFactoryConfigurer configurer, ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        configurer.configure(factory, connectionFactory);
        @SuppressWarnings("unchecked")
        FatalExceptionStrategy strategy = LekeFatalExceptionStrategy.retryOn(DataAccessException.class,
                RemoteAccessException.class, Exception.class);
        factory.setErrorHandler(new ConditionalRejectingErrorHandler(strategy));
        return factory;
    }
}
class FatalExceptionStrategy extends DefaultExceptionStrategy {
    
	// 构建策略
    @SuppressWarnings("unchecked")
    public static FatalExceptionStrategy retryOn(Class<? extends Throwable>... exClasses) {
        Set<Class<? extends Throwable>> set = new HashSet<>();
        for (Class<? extends Throwable> clazz : exClasses) {
            set.add(clazz);
        }
        return new FatalExceptionStrategy(set);
    }

    final Set<Class<? extends Throwable>> retryables;

    public FatalExceptionStrategy(Set<Class<? extends Throwable>> exceptions) {
        this.retryables = exceptions == null ? Collections.emptySet() : Collections.unmodifiableSet(exceptions);
    }

    @Override
    protected boolean isUserCauseFatal(Throwable e) {
        boolean needRetry = retryables.stream().anyMatch(clazz -> clazz.isInstance(e));
        return !needRetry;
    }
}
```

`RabbitConsumer`消费者类

```java
@Component
public class RabbitConsumer {
//    @RabbitListener(queues = RabbitMqConstant.TEST_QUEUE)
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = RabbitMqConstant.TEST_QUEUE, durable = "true"),
            exchange = @Exchange(value = RabbitMqConstant.TEST_EXCHANGE)))
    public void listenSimpleQueueMessage(String msg) {
        System.out.println("spring 消费者接收到消息：【" + msg + "】");
    }

}
```

`RabbitProducer`生产者类

```java
@SpringBootTest
public class RabbitProducer {

    @Test
    public void testSimpleQueue() {
        // 创建连接工厂
        CachingConnectionFactory connectionFactory = new CachingConnectionFactory("192.168.20.80");
        connectionFactory.setUsername("leke");
        connectionFactory.setPassword("leke@@@");
        // 创建 RabbitTemplate
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        // 配置交换机和队列
//        DirectExchange testExchange = new DirectExchange(RabbitMqConstant.TEST_EXCHANGE);
//        Queue testQueue = new Queue(RabbitMqConstant.SUBMIT_TASK_QUEUE);
//        Binding binding = BindingBuilder.bind(testQueue).to(testExchange).with(RabbitMqConstant.TEST_QUEUE);
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMqConstant.TEST_EXCHANGE, RabbitMqConstant.TEST_QUEUE, "hello, spring amqp!");

    }
}
```

