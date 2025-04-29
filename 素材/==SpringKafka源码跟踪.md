# Spring-Kafka消费过程中，提供了哪些扩展点？

> 源码版本：spring-kafka v2.6.6

```
Q: @KafkaListener(containerFactory = "sendKafkaListenerContainerFactory") 未配工厂，会使用哪个？
A: 会使用默认的 Bean: kafkaListenerContainerFactory
相关源码:
KafkaAnnotationDrivenConfiguration#kafkaListenerContainerFactory()
```



```
Q: kafkaListenerContainerFactory 是如何创建实例的？
A: 未使用反射，设置了messageConverter,recordInterceptor,containerCustomizer
messageConverter可进行Java对象反序列；
【可基于这个实现集群方式的 MessageListenerContainer】
相关源码: 
KafkaListenerEndpointRegistry#createListenerContainer();
MessageListenerContainer listenerContainer = factory.createListenerContainer(endpoint);
```



```
Q: @KafkaListener和工厂配置的GroupId，哪个优先？
A: @KafkaListener 优先于工厂配置。
相关源码：
KafkaMessageListenerContainer#doStart()
AbstractMessageListenerContainer#getGroupId()
```



```
Q: org.apache.kafka.clients.consumer.KafkaConsumer 在哪里实例化的？
A: DefaultKafkaConsumerFactory#createRawConsumer()
```



```
Q: KafkaConsumer 监听到 kafka 消息，消费流程是怎么样的？
A: KafkaMessageListenerContainer.ListenerConsumer#pollAndInvoke()
-> invokeListener()
-> invokeRecordListener()
-> doInvokeWithRecords() [earlyRecordInterceptor]
-> doInvokeRecordListener() [monitor]
   [捕获RuntimeException]
   [执行ErrorHandler, 例如SeekToCurrentErrorHandler]
-> invokeOnMessage()
-> doInvokeOnMessage() [recordInterceptor]

-> RecordMessagingMessageListenerAdapter#onMessage() [@KafkaListener最终会是这个类]
   [捕获异常ListenerExecutionFailedException]
   [执行KafkaListenerErrorHandler]
-> MessagingMessageListenerAdapter#toMessagingMessage()      [反序列化]
-> JsonMessageConverter#extractAndConvertValue()             [反序列化]
-> MessagingMessageListenerAdapter#invokeHandler()           [反射调用]
-> HandlerAdapter#invoke()                                   [反射调用]
-> InvocableHandlerMethod#doInvoke [spring-message.jar]      [反射调用]
-> MessagingMessageListenerAdapter#handleResult              [处理方法返回结果]
-> KafkaMessageListenerContainer.ListenerConsumer#ackCurrent [ACK]
```

## ErrorHandler

> org.apache.kafka.clients.consumer.ConsumerConfig#REQUEST_TIMEOUT_MS_CONFIG 默认30s

```
被 @KafkaListener 标记的方法如果抛出 Exception, 会进行 ErrorHandler#handle 处理。SeekToCurrentErrorHandler 是 Kafka 默认的 ErrorHandler, 核心作用是：当消息处理失败时，重置当前偏移量，而不是提交下一偏移量处理下一条消息。
```

假设我们有一个Kafka主题`test-topic`，当前有3条消息：

| 偏移量(offset) |    消息内容    |
| :------------: | :------------: |
|       0        |  "消息1-正常"  |
|       1        | "消息2-会失败" |
|       2        |  "消息3-正常"  |

```java
@KafkaListener(topics = "test-topic", groupId = "test-group")
public void consume(ConsumerRecord<String, String> record) {
    System.out.printf("收到消息[offset=%d]: %s%n", record.offset(), record.value());
    
    if (record.value().contains("失败")) {
        throw new RuntimeException("抛出异常");
    }
    
    System.out.println("处理成功");
}
```

1、第一次消费（正常）:

- 读取 offset=0 的消息："消息1-正常"
- 处理成功
- 提交 offset=1（表示下次从 offset=1 开始读）

2、第二次消费（失败）:

- 读取 offset=1 的消息："消息2-会失败"
- 抛出异常
- SeekToCurrentErrorHandler 介入：
  - 关键动作：执行`consumer.seek(partition, 1)`，将消费者偏移量重置回当前消息的起始位置(offset=1)
  - 效果：下次 poll 时还会再次获取这条消息

3、重试消费:

- 再次读取 offset=1 的消息
- 如果继续失败，根据配置决定是否继续重试或转移到死信队列

4、后续消费:

- 只有当这条消息最终处理成功(或转移到 DLT 后)
- 才会提交 offset=2，继续处理下一条消息



## 待办的问题

```
Q: 如何过滤掉未监听的Event?
A: RecordInterceptor

Q: 如何过滤掉灰度环境的消息?
A: 根据环境进行初始化

Q: KafkaListenerContainerFactory 能不能动态的获取？根据cluster来获取？
A: 已实现，参考: KafkaListenerContainerFactoryRegistrar.java

Q: 考虑将 @EventHandler 的其它配置设置一个默认值。
A: cluster 默认从 Event 中取
topics 默认从 Event 中取
groupId 默认 spring.application.name
containerFactory 默认 cluster + "KafkaListenerContainerFactory"

Q: 消费者的Kafka配置再考虑一下
A： Done
```

## 思路

```
另一种写法
@Component
@KafkaListener(id = "multiGroup", topics = "multitype")
public class MultiTypeKafkaListener {

    @KafkaHandler
    public void handleGreeting(Greeting greeting) {
        System.out.println("Greeting received: " + greeting);
    }

    @KafkaHandler
    public void handleF(Farewell farewell) {
        System.out.println("Farewell received: " + farewell);
    }

    @KafkaHandler(isDefault = true)
    public void unknown(Object object) {
        System.out.println("Unkown type received: " + object);
    }
}
```





