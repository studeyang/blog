---
permalink: 2025/0507.html
title: 深入Spring Kafka：消费者是如何创建的？
date: 2025-05-07 23:00:00
tags: Kafka
cover: https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png
thumbnail: https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png
categories: technotes
toc: true
description: Spring Kafka 在原生客户端基础上进行了深度封装，通过声明式注解显著简化了开发流程。这种简洁的语法背后，Spring Kafka 实际上构建了一套完整的消费者（Consumer）管理机制。那么问题来了：Spring Kafka 是如何创建这些消费者的呢？
---

在 Java 生态中，Apache Kafka 通过 `kafka-clients.jar` 提供了原生客户端支持。开发者需要手动创建 `KafkaConsumer` 实例并订阅指定主题（Topic）来实现消息消费。典型实现如下：

```java
public void pollMessages() {
    // 1. 初始化消费者实例
    Consumer<String, String> consumer = new KafkaConsumer<>(getConsumerConfig());
    // 2. 订阅主题并设置重平衡监听器
    consumer.subscribe(Collections.singleton(topic), new RebalanceListener());
    // 3. 轮询获取消息（超时时间1秒）
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    // 4. 同步提交偏移量
    consumer.commitSync();
}
```

Spring Kafka 在原生客户端基础上进行了深度封装，通过声明式注解显著简化了开发流程。例如，只需使用 `@KafkaListener` 注解即可实现消息监听：

```java
@KafkaListener(id = "orderService", topics = "order.topic")
public void handleOrderEvent(ConsumerRecord<String, String> record) {
    // 业务处理逻辑
}
```

这种简洁的语法背后，Spring Kafka 实际上构建了一套完整的消费者（Consumer）管理机制。那么问题来了：Spring Kafka 是如何创建这些消费者的呢？

<!-- more -->

> 本文源码版本：spring-kafka v2.6.6

## 一、使用Spring Kafka消费消息

首先，我们通过一个完整的项目集成示例，具体说明其实现步骤。项目里要接入 Spring Kafka，通常需要经过以下几个步骤。

#### 第一步：引入依赖

需在项目中声明 Spring Kafka Starter 依赖。

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>2.6.6</version>
</dependency>
```

#### 第二步：消费者配置

配置类上添加 `@EnableKafka` 注解，并初始化 `ConcurrentKafkaListenerContainerFactory` Bean，这是最常见的使用方式。

```java
@Configuration
@EnableKafka
public class Config {

   /** 消费者工厂 */
    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(configs);
    }
    
    /** 监听容器工厂 */
    @Bean
    ConcurrentKafkaListenerContainerFactory<String, String>
                        kafkaListenerContainerFactory(ConsumerFactory<String, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3); // 设置消费者线程数
        return factory;
    }
}
```

#### 第三步：实现消息监听

在业务层方法上添加 `@KafkaListener` 注解，实现消息监听。

```java
@Service
public class OrderMessageListener {
    @KafkaListener(id = "orderService", topics = "order.topic")
    public void handleOrderEvent(ConsumerRecord<String, String> record) {
        // 业务处理逻辑
    }
}
```

至此，我们已经完成 Spring Kafka 的基础集成。接下来将深入分析`@KafkaListener`注解背后的消费者创建过程，揭示 Spring 是如何构建 KafkaConsumer 实例的。

## 二、消费者的初始化过程

基于上面示例，我们以 `@EnableKafka` 注解为切入点，源码如下：

```java
@Import(KafkaListenerConfigurationSelector.class)
public @interface EnableKafka {
}

@Order
public class KafkaListenerConfigurationSelector implements DeferredImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] { KafkaBootstrapConfiguration.class.getName() };
    }
}
```

该注解的核心作用是通过`KafkaBootstrapConfiguration`向 Spring 容器注册两个关键 Bean。注册的核心 Bean 如下所示：

```java
public class KafkaBootstrapConfiguration implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 省略无关代码
        
        // 注册注解处理器
        // beanName: org.springframework.kafka.config.internalKafkaListenerAnnotationProcessor
        registry.registerBeanDefinition(
            KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME,
            new RootBeanDefinition(KafkaListenerAnnotationBeanPostProcessor.class));
        
        // 注册监听器容器注册表
        // beanName: org.springframework.kafka.config.internalKafkaListenerEndpointRegistry
        registry.registerBeanDefinition(
            KafkaListenerConfigUtils.KAFKA_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,
            new RootBeanDefinition(KafkaListenerEndpointRegistry.class));
    }
}
```

注解处理器（`KafkaListenerAnnotationBeanPostProcessor`）负责扫描和解析`@KafkaListener`及其派生注解，并将监听方法转换为可执行的端点描述符（`KafkaListenerEndpointDescriptor`）。

容器注册表（`KafkaListenerEndpointRegistry`）作为所有消息监听容器的中央仓库，实现了生命周期管理（启动/停止容器）。

> <mark>代码阅读小记：</mark>
>
> ```
> 切入点: @EnableKafka
> -> KafkaListenerConfigurationSelector
> -> KafkaBootstrapConfiguration
>     [注册Bean: KafkaListenerAnnotationBeanPostProcessor]
>     [注册Bean: KafkaListenerEndpointRegistry]
> ```

接下来，我们就重点剖析一下这两个 Bean。

### 2.1 消费者注册流程剖析

#### 1、注解扫描阶段

首先来看第一个 Bean: `KafkaListenerAnnotationBeanPostProcessor`，它通过 Spring 后置处理器机制（`postProcessAfterInitialization`）实现了注解扫描：

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    @Override
    public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
        // 省略无关代码
        
        // 使用 MethodIntrospector 进行元数据查找
        // 查找被 @KafkaListener (及其派生注解) 标记的方法
        Map<Method, Set<KafkaListener>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
                (MethodIntrospector.MetadataLookup<Set<KafkaListener>>) method -> {
                    Set<KafkaListener> listenerMethods = findListenerAnnotations(method);
                    return (!listenerMethods.isEmpty() ? listenerMethods : null);
                });
        
        // 处理每个找到的监听方法
        for (Map.Entry<Method, Set<KafkaListener>> entry : annotatedMethods.entrySet()) {
            Method method = entry.getKey();
            for (KafkaListener listener : entry.getValue()) {
                processKafkaListener(listener, method, bean, beanName);
            }
        }
        return bean;
    }
}
```

上述代码有两个关键点，第一是通过`MetadataLookup`支持派生注解；第二是处理 `@KafkaListener` 监听方法。

什么是`MetadataLookup`呢？

举个例子，我们定义了一个新的注解 `@EventHandler` ，并在该注解上标记 `@KafkaListener`。

```java
@KafkaListener
public @interface EventHandler {
    @AliasFor(annotation = KafkaListener.class, attribute = "topics")
    String value();
    // 其他属性映射...
}
```

这种设计使得业务注解（如`@EventHandler`）可以透明地继承`@KafkaListener`的全部功能。

```java
@Service
public class OrderMessageListener {
    @EventHandler("order.topic")
    public void handleOrderEvent(ConsumerRecord<String, String> record) {
        // 业务处理逻辑
    }
}
```

#### 2、端点注册阶段

我们继续处理 KafkaListener 代码跟踪，现在来到了 `KafkaListenerAnnotationBeanPostProcessor` 的 `processListener()` 方法。

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    // KafkaListener 注册器
    private final KafkaListenerEndpointRegistrar registrar = new KafkaListenerEndpointRegistrar();
    
    protected void processListener(MethodKafkaListenerEndpoint<?, ?> endpoint, KafkaListener kafkaListener,
            Object bean, Object adminTarget, String beanName) {
        // 设置端点属性
        endpoint.setBean(bean);
        endpoint.setId(getEndpointId(kafkaListener));
        endpoint.setTopics(resolveTopics(kafkaListener));
        // 委托注册器进行注册
        this.registrar.registerEndpoint(endpoint, factory);
    }
}
```

被 `@KafkaListener` 标记的方法会被封装为 `MethodKafkaListenerEndpoint` ，并由注册器 `KafkaListenerEndpointRegistrar` 进行注册，注册器内部维护了一个端点描述符列表：

```java
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
    private final List<KafkaListenerEndpointDescriptor> endpointDescriptors = new ArrayList<>();
    
    public void registerEndpoint(KafkaListenerEndpoint endpoint, KafkaListenerContainerFactory<?> factory) {
        // 省略无关代码
        KafkaListenerEndpointDescriptor descriptor = new KafkaListenerEndpointDescriptor(endpoint, factory);
        this.endpointDescriptors.add(descriptor);
    }
}
```

由此可见，`KafkaListener` 会被注册到 List 集合中。

> <mark>代码阅读小记：</mark>
>
> ```
> -> KafkaListenerAnnotationBeanPostProcessor#postProcessAfterInitialization()
> -> processKafkaListener()
> -> processListener()
> -> KafkaListenerEndpointRegistrar#registerEndpoint()
> -> endpointDescriptors [注册到容器里List]
> ```
>

到这里，`BeanPostProcessor` 的 `postProcessAfterInitialization` 方法已经执行完了，程序完成了 `KafkaListener` 的注册并存储至 endpointDescriptors 中。

#### 3、容器实例化阶段

当所有 Bean 初始化完成后，接下来会通过`afterSingletonsInstantiated` 触发最终注册：

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    // KafkaListener 注册器
    private final KafkaListenerEndpointRegistrar registrar = new KafkaListenerEndpointRegistrar();
    
    @Override
    public void afterSingletonsInstantiated() {
        // 注册所有 KafkaListener
        this.registrar.afterPropertiesSet();
    }
}
```

注册器 `KafkaListenerEndpointRegistrar` 的注册逻辑如下。

```java
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
    
    private KafkaListenerEndpointRegistry endpointRegistry;
    
    @Override
    public void afterPropertiesSet() {
        registerAllEndpoints();
    }

    protected void registerAllEndpoints() {
        // 注册到注册表当中
        for (KafkaListenerEndpointDescriptor descriptor : this.endpointDescriptors) {
            this.endpointRegistry.registerListenerContainer(
                descriptor.endpoint, resolveContainerFactory(descriptor));
        }
    }
}
```

可见，注册器最终委托给了注册表处理，注册表中由一个 `ConcurrentHashMap` 进行保存。

```java
public class KafkaListenerEndpointRegistry implements DisposableBean, 
        SmartLifecycle, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent> {
            
    private final Map<String, MessageListenerContainer> listenerContainers = new ConcurrentHashMap<>();
    
    // 注册表的注册逻辑
    public void registerListenerContainer(KafkaListenerEndpoint endpoint, 
                                          KafkaListenerContainerFactory<?> factory) {
        registerListenerContainer(endpoint, factory, false);
    }
    
    public void registerListenerContainer(KafkaListenerEndpoint endpoint, KafkaListenerContainerFactory<?> factory,
            boolean startImmediately) {
        String id = endpoint.getId();
        // 通过工厂创建, 最终创建出来的 ConcurrentMessageListenerContainer
        MessageListenerContainer container = createListenerContainer(endpoint, factory);
        this.listenerContainers.put(id, container);
    }
}
```

在示例中，我们配置的是 `ConcurrentKafkaListenerContainerFactory` 来创建 `KafkaListener` 容器的，因此这里往注册表（KafkaListenerEndpointRegistry）里添加的是 `ConcurrentMessageListenerContainer` 对象实例。

> <mark>代码阅读小记：</mark>
>
> ```
> KafkaListenerAnnotationBeanPostProcessor#afterSingletonsInstantiated()
> -> KafkaListenerEndpointRegistrar#afterPropertiesSet()
> -> registerAllEndpoints()
> -> KafkaListenerEndpointRegistry#registerListenerContainer()
> -> [listenerContainers] [注册到容器里Map]
> ```

### 2.2 消费者启动机制

#### 1、并发监听容器

再来看第二个 Bean: `KafkaListenerEndpointRegistry`。它实现了 Spring 生命周期 SmartLifecycle 接口，在程序启动时，会调用它的 `start` 方法。

```java
public class KafkaListenerEndpointRegistry implements DisposableBean, 
        SmartLifecycle, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent> {
            
    @Override
    public void start() {
        // ConcurrentMessageListenerContainer 实例
        for (MessageListenerContainer listenerContainer : getListenerContainers()) {
            startIfNecessary(listenerContainer);
        }
    }
    
    private void startIfNecessary(MessageListenerContainer listenerContainer) {
        if (this.contextRefreshed || listenerContainer.isAutoStartup()) {
            listenerContainer.start();
        }
    }
}
```

注册表（`KafkaListenerEndpointRegistry`）维护的容器（`MessageListenerContainer`）实例分为两类：

- `ConcurrentMessageListenerContainer`：多线程容器
- `KafkaMessageListenerContainer`：单线程容器

`ConcurrentMessageListenerContainer`内部通过创建多个单线程容器实现并发：

```java
public class ConcurrentMessageListenerContainer<K, V> extends AbstractMessageListenerContainer<K, V> {

    @Override
    protected void doStart() {
        for (int i = 0; i < this.concurrency; i++) {
            KafkaMessageListenerContainer<K, V> container =
                    constructContainer(containerProperties, topicPartitions, i);
            // 启动每个子容器
            container.start();
        }
    }
}
```

可见，`ConcurrentMessageListenerContainer` 通过委托给多个`KafkaMessageListenerContainer`实例从而实现多线程消费。

#### 2、底层消费者创建

最终我们在 `KafkaMessageListenerContainer` 的内部类 `ListenerConsumer` 中发现了 kafka-clients.jar 中的 Consumer 接口类。它的创建过程是由 `ConsumerFactory` 代为创建，`ConsumerFactory` 是一个接口类，它只有一个实现：`DefaultKafkaConsumerFactory`。

```java
public class KafkaMessageListenerContainer<K, V>  extends AbstractMessageListenerContainer<K, V> {
    
    private volatile ListenerConsumer listenerConsumer;
    
    @Override
    protected void doStart() {
        this.listenerConsumer = new ListenerConsumer(listener, listenerType);
    }
    
    private final class ListenerConsumer implements SchedulingAwareRunnable, ConsumerSeekCallback {
        // Consumer 是 kafka-clients.jar 中的接口类
        private final Consumer<K, V> consumer;
        // ConsumerFactory 是一个接口，只有一个实现类 DefaultKafkaConsumerFactory
        protected final ConsumerFactory<K, V> consumerFactory;
        
        ListenerConsumer(GenericMessageListener<?> listener, ListenerType listenerType) {
            Properties consumerProperties = propertiesFromProperties();
            this.consumer = this.consumerFactory.createConsumer(
                            this.consumerGroupId,
                            this.containerProperties.getClientId(),
                            KafkaMessageListenerContainer.this.clientIdSuffix,
                            consumerProperties);
            // 监听 Topic
            subscribeOrAssignTopics(this.consumer);
        }
    }
}
```

Consumer 消费者的创建代码如下。

```java
public class DefaultKafkaConsumerFactory<K, V> extends KafkaResourceFactory
        implements ConsumerFactory<K, V>, BeanNameAware {

    protected Consumer<K, V> createKafkaConsumer(Map<String, Object> configProps) {
        return createRawConsumer(configProps);
    }
    protected Consumer<K, V> createRawConsumer(Map<String, Object> configProps) {
         // KafkaConsumer 是 kafka-clients.jar 中的类
        return new KafkaConsumer<>(configProps, this.keyDeserializerSupplier.get(),
            this.valueDeserializerSupplier.get());
    }
}
```

> <mark>代码阅读小记：</mark>
>
> ```
> KafkaListenerEndpointRegistry#start()
> -> AbstractMessageListenerContainer#start()
> -> ConcurrentMessageListenerContainer#doStart() (concurrency不能大于partitions)
> -> KafkaMessageListenerContainer#start() -> doStart()
> -> DefaultKafkaConsumerFactory#createRawConsumer()
> ```

## 三、总结

总结一下上文中各部分的代码阅读小记，得到如下代码链路：

```
切入点: @EnableKafka
-> KafkaListenerConfigurationSelector
-> KafkaBootstrapConfiguration
    [注册Bean:KafkaListenerAnnotationBeanPostProcessor]
    [注册Bean:KafkaListenerEndpointRegistry]
-> KafkaListenerAnnotationBeanPostProcessor#postProcessAfterInitialization()
-> processKafkaListener()
-> processListener()
-> KafkaListenerEndpointRegistrar#registerEndpoint()
-> endpointDescriptors [注册到容器里List]
(===== 此时，程序里已经有endpointDescriptor了 =====)

(===== 开始遍历endpointDescriptors =====)
KafkaListenerAnnotationBeanPostProcessor#afterSingletonsInstantiated()
-> KafkaListenerEndpointRegistrar#afterPropertiesSet()
-> registerAllEndpoints()
-> KafkaListenerEndpointRegistry#registerListenerContainer()
-> [listenerContainers] [注册到容器里Map]

(===== 开始启动监听 =====)
KafkaListenerEndpointRegistry#start()
-> AbstractMessageListenerContainer#start()
-> ConcurrentMessageListenerContainer#doStart() (concurrency不能大于partitions)
-> KafkaMessageListenerContainer#start() -> doStart()
-> DefaultKafkaConsumerFactory#createRawConsumer()
```

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202505071803032.png)

## 相关文章

- [Kafka 位移提交的正确姿势](https://mp.weixin.qq.com/s/iECpgIOaDBNIjfl3BtXIYQ)

