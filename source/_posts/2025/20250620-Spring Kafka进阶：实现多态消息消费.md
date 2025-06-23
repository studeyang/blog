---
permalink: 2025/0620.html
title: Spring Kafka进阶：实现多态消息消费
date: 2025-06-20 23:00:00
tags: Kafka
cover: https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png
thumbnail: https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png
categories: technotes
toc: true
description: Spring Kafka 官方提供了一种基于消息类型的消费模式，实现多态消息的自动路由处理。开发者可以基于此特性，轻松实现从简单文本消息到复杂领域事件的各种消息处理场景。
---

Spring Kafka 官方提供了一种基于消息类型的消费模式，通过类上的 `@KafkaListener` 注解配合方法上的 `@KafkaHandler`，实现多态消息的自动路由处理。其典型实现方式如下：

```java
@KafkaListener(id = "multi", topics = "myTopic")
static class MultiListenerBean {
    
    @KafkaHandler
    public void listen(String foo) {
        ...
    }
    
    @KafkaHandler
    public void listen(Integer bar) {
        ...
    }
    
    @KafkaHandler(isDefault = true)
    public void listenDefault(Object object) {
        ...
    }
}
```

根据 Spring Kafka 官方文档 [@KafkaListener on a Class](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/class-level-kafkalistener.html) 的说明：

> When messages are delivered, the converted message payload type is used to determine which method to call.
> 接收到消息后，会使用转换后的消息类型来决定调用哪个方法。

开发者可以基于此特性，轻松实现从简单文本消息到复杂领域事件的各种消息处理场景。

<!-- more -->

## 一、实现多态消息消费

### 第一步：环境准备，引入必要依赖

首先在项目的 pom.xml 中添加 Spring Kafka 的依赖。

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>${spring-kafka-version}</version> <!-- 使用最新稳定版本 -->
</dependency>

<!-- 可选：若使用Fastjson进行JSON处理 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>${fastjson-version}</version>
</dependency>
```

### 第二步：消息模型设计，建立统一消息基类

设计一个抽象消息基类作为所有消息类型的父类，包含公共字段和基础方法：

```java
/**
 * 消息抽象基类，所有具体消息类型都应继承此类
 */
public abstract class AbstractMessage {
    /** 消息id */
    private String messageId;
    
    public String getMessageId() {
        return messageId;
    }
    
    public void setMessageId(String messageId) {
        this.messageId = messageId;
    }
}
```

### 第三步：实现反序列化器

```java
import org.apache.kafka.common.serialization.Deserializer;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class MessageDeserializer implements Deserializer<AbstractMessage> {

    @Override
    public AbstractMessage deserialize(String topic, byte[] data) {
        // 1. 字节数组转字符串
        String jsonData = new String(data);
        // 2. 使用Fastjson的自动类型识别功能
        Object object = JSON.parse(jsonData, Feature.SupportAutoType);
        if (object instanceof AbstractMessage) {
            return (AbstractMessage) object;
        }
        return null;
    }
}
```

### 第四步：配置消费者工厂

创建 Spring Kafka 配置类，设置消费者相关参数：

```java
@Configuration
@EnableKafka
public class Config {

   /** 消费者工厂 */
    @Bean
    public ConsumerFactory<String, AbstractMessage> consumerFactory() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        // 自定义的反序列化实现类
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, MessageDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(configs);
    }
    
    /** 监听容器工厂 */
    @Bean
    ConcurrentKafkaListenerContainerFactory<String, AbstractMessage>
                        kafkaListenerContainerFactory(ConsumerFactory<String, AbstractMessage> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, AbstractMessage> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(1); // 设置消费者线程数
        return factory;
    }
}
```

### 第五步：实现消息监听

创建消息监听服务，利用 `@KafkaHandler` 实现多态消息处理：

```java
@Service
@KafkaListener(id = "orderService", topics = "order.topic")
public class OrderMessageListener {
    
    @KafkaHandler
    public void handle(OrderCreateEvent event) {
        // 业务处理逻辑
        System.out.println("收到 OrderCreateEvent 消息");
    }
    
    @KafkaHandler
    public void handle(OrderCancelEvent event) {
        // 业务处理逻辑
        System.out.println("收到 OrderCancelEvent 消息");
    }
}
```

最后，我们往 `topic=order.topic` 中发一条消息：

```json
{
    "@type": "io.github.open.easykafka.event.OrderCreateEvent",
    "id": "67ff910f020000010001b064"
}
```

消费方控制台打印如下：

```
收到 OrderCreateEvent 消息
```

至此，我们实现了多态消息的消费模式。同时也引发了我们的思考：Spring Kafka 是如何精准找到正确的方法的？

## 二、实现原理

下面我们就来深入剖析 Spring Kafka 框架实现这一机制的核心原理。

> 为便于阅读和理解，以下 Spring Kafka 源码部分均省略和简化了无关代码。

### 2.1 注解扫描

Spring Kafka 会对所有被 `@KafkaListener` 标记的类和方法进行扫描，在本文示例中，`@KafkaListener` 标注在了类上，扫描逻辑如下：

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    
    @Override
    public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
        
        // 查找被 @KafkaListener 标记的类
        Collection<KafkaListener> classLevelListeners = findListenerAnnotations(targetClass);
        
        // 查找被 @KafkaHandler 标记的方法
        final List<Method> multiMethods = new ArrayList<>();
        Set<Method> methodsWithHandler = MethodIntrospector.selectMethods(targetClass,
                        (ReflectionUtils.MethodFilter) method ->
                                AnnotationUtils.findAnnotation(method, KafkaHandler.class) != null);
        multiMethods.addAll(methodsWithHandler);
        
        // 处理每个找到的监听方法
        processMultiMethodListeners(classLevelListeners, multiMethods, bean, beanName);
        return bean;
    }
}
```

### 2.2 端点注册

Spring Kafka 会根据 `@KafkaListener` 注解位置自动识别并创建不同类型的端点：

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    
    private void processMultiMethodListeners(Collection<KafkaListener> classLevelListeners, List<Method> multiMethods,
            Object bean, String beanName) {
        for (KafkaListener classLevelListener : classLevelListeners) {
            // @KafkaListener 标注在类上会被描述为 MultiMethodKafkaListenerEndpoint
            MultiMethodKafkaListenerEndpoint<K, V> endpoint =
                    new MultiMethodKafkaListenerEndpoint<>(checkedMethods, defaultMethod, bean);
            processListener(endpoint, classLevelListener, bean, bean.getClass(), beanName);
        }
    }
}
```

`@KafkaListener` 标注在类上会被 Spring Kafka 描述为 `MultiMethodKafkaListenerEndpoint`，它是 `MethodKafkaListenerEndpoint` 的子类。这些端点（`KafkaListenerEndpoint`）最终会被注册到注册表 `KafkaListenerEndpointRegistry` 的 List 集合中。

### 2.3 消息监听器实例化

Spring Kafka 会将注册表 `KafkaListenerEndpointRegistry` 中的端点 `KafkaListenerEndpoint` 实例化成消息监听器 `MessageListener` 对象。

```java
public abstract class AbstractKafkaListenerEndpoint<K, V>
        implements KafkaListenerEndpoint, BeanFactoryAware, InitializingBean {
    
    private void setupMessageListener(MessageListenerContainer container, MessageConverter messageConverter) {
        // 创建消息监听适配器
        MessagingMessageListenerAdapter<K, V> adapter = createMessageListener(container, messageConverter);
        Object messageListener = adapter;
        container.setupMessageListener(messageListener);
    }
}
```

创建消息监听适配器的过程如下。

```java
public class MethodKafkaListenerEndpoint<K, V> extends AbstractKafkaListenerEndpoint<K, V> {
    @Override
    protected MessagingMessageListenerAdapter<K, V> createMessageListener(MessageListenerContainer container,
            MessageConverter messageConverter) {
        // 创建消息监听器实例
        MessagingMessageListenerAdapter<K, V> messageListener = createMessageListenerInstance(messageConverter);
        return messageListener;
    }
    
    protected MessagingMessageListenerAdapter<K, V> createMessageListenerInstance(MessageConverter messageConverter) {
        // 最终的消息监听器是 RecordMessagingMessageListenerAdapter
        RecordMessagingMessageListenerAdapter<K, V> messageListener = new RecordMessagingMessageListenerAdapter<K, V>(
                this.bean, this.method, this.errorHandler);
        return messageListener;
    }
}
```

可以看到，最终的消息监听器是 `RecordMessagingMessageListenerAdapter`，它是接口 `MessageListener` 的实现类。

### 2.4 启动消息监听

在[《深入Spring Kafka：消费者是如何创建的？》](https://mp.weixin.qq.com/s/NYUVElVSw5K38M6MXqjHmA)这篇文章中，我们描述了启动消息监听的详细过程。一个 `@KafkaListener` 对应一个 `MessageListenerContainer`，最终初始化后的消息监听器容器如下图所示：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202506202023484.png)

在②中可见，`ContainerProperties` 中已经包含了处理消息的两个 `handle` 方法。

### 2.5 消息消费过程

基于前文分析，示例中的 `OrderMessageListener` 消费者最终会被实例化成 `RecordMessagingMessageListenerAdapter`。下面我们来看看该适配器收到消息后做了哪些处理。

```java
public class RecordMessagingMessageListenerAdapter<K, V> extends MessagingMessageListenerAdapter<K, V>
        implements AcknowledgingConsumerAwareMessageListener<K, V> {
    @Override
    public void onMessage(ConsumerRecord<K, V> record, Acknowledgment acknowledgment, Consumer<?, ?> consumer) {
        Message<?> message = toMessagingMessage(record, acknowledgment, consumer);
        // 反射调用业务方法
        invokeHandler(record, acknowledgment, message, consumer);
    }
}
```

反射调用业务方法过程如下。

```java
public class DelegatingInvocableHandler {
    public Object invoke(Message<?> message, Object... providedArgs) throws Exception {
        // 这里拿到的是 OrderCreateEvent.class
        Class<? extends Object> payloadClass = message.getPayload().getClass();
        // 通过 OrderCreateEvent.class 找 handle 方法
        Object result = InvocableHandlerMethod handler = getHandlerForPayload(payloadClass);
        // 反射调用 handle 方法
        handler.invoke(message, providedArgs)
        return new InvocationResult(result, replyTo, this.handlerReturnsMessage.get(handler));
    }
}
```

Spring Kafka 是如何通过 `OrderCreateEvent.class` 找到对应的 `handle` 方法的？

```java
public class DelegatingInvocableHandler {
    
    protected InvocableHandlerMethod getHandlerForPayload(Class<? extends Object> payloadClass) {
        // 通过 OrderCreateEvent.class 找 handle 方法
        InvocableHandlerMethod handler = findHandlerForPayload(payloadClass);
        return handler;
    }
    
    protected InvocableHandlerMethod findHandlerForPayload(Class<? extends Object> payloadClass) {
        InvocableHandlerMethod result = null;
        // 属性 handlers 里面有两个 Method 对象：
        // handle(OrderCreateEvent.class), handle(OrderCancelEvent.Class)
        for (InvocableHandlerMethod handler : this.handlers) {
            // 通过 OrderCreateEvent.class 找对应的 handle 方法
            if (matchHandlerMethod(payloadClass, handler)) {
                result = handler;
            }
        }
        return result != null ? result : this.defaultHandler;
    }
    
    protected boolean matchHandlerMethod(Class<? extends Object> payloadClass, InvocableHandlerMethod handler) {
        // 读取 Method handle(OrderCreateEvent.class) 对象的方法参数
        Annotation[][] parameterAnnotations = handler.getMethod().getParameterAnnotations();
        MethodParameter methodParameter = new MethodParameter(handler.getMethod(), 0);
        // 判断类型是否匹配
        if (methodParameter.getParameterType().isAssignableFrom(payloadClass)) {
            return true;
        }
        return foundCandidate != null;
    }
}
```

至此，Spring Kafka 便精准找到了 `handle(OrderCreateEvent.class)` 方法。

## 三、总结

上文所提到的代码调用链路如下：

```
RecordMessagingMessageListenerAdapter#onMessage()
   [捕获异常ListenerExecutionFailedException]
   [执行KafkaListenerErrorHandler]
-> MessagingMessageListenerAdapter#toMessagingMessage()      [反序列化]
-> JsonMessageConverter#extractAndConvertValue()             [反序列化]
-> MessagingMessageListenerAdapter#invokeHandler()           [反射调用]
-> HandlerAdapter#invoke()                                   [反射调用]
-> DelegatingInvocableHandler#invoke()                       [反射调用]
    -> getHandlerForPayload()                                [查找调用方法]
    -> findHandlerForPayload()                               [查找调用方法]
-> InvocableHandlerMethod#doInvoke [spring-message.jar]      [反射调用]
-> MessagingMessageListenerAdapter#handleResult              [处理方法返回结果]
-> KafkaMessageListenerContainer.ListenerConsumer#ackCurrent [ACK]
```

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png)

## 相关文章

- [深入Spring Kafka：消费者是如何创建的？](https://mp.weixin.qq.com/s/NYUVElVSw5K38M6MXqjHmA)

