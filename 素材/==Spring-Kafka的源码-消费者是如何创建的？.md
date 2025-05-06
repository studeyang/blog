# Spring Kafka消费者是如何创建的？

对于 Java 应用，Apache Kafka 提供了访问 Kafka 服务的客户端包 `kafka-clients.jar`，要消费消息时，需要创建一个 `Consumer` 对象，并监听特定的 `Topic`。

下面这段代码想必我们都很熟悉了。

```java
public void poll() {
    Consumer<String, String> consumer = new KafkaConsumer<>(getProperties());
    consumer.subscribe(Collections.singleton(topic), new RebalanceListener());
    ConsumerRecords<String, String> records = consumer.poll(1000);
    consumer.commitSync();
}
```

而 Spring Kafka 基于 `kafka-clients.jar` 之上提供了更加便利的交互形式，在监听消息时只需通过 `@KafkaListener(id = "listen1", topics = "topic1")` 即可完成消息的监听。

便利之余，也引发了我们思考：Spring Kafka 消费者是如何创建的？

> 本文源码版本：spring-kafka v2.6.6

## 一、示例：使用Spring Kafka消费消息

项目里要接入 Spring Kafka，通常需要经过以下几个步骤。

**第一步：引入依赖**

pom.xml 文件引入如下依赖。

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
  <version>2.6.6</version>
</dependency>
```

**第二步：消费者配置**

配置类上添加 `@EnableKafka` 注解，并初始化 `ConcurrentKafkaListenerContainerFactory` Bean，这是最常见的使用方式。

```java
@Configuration
@EnableKafka
public class Config {

    @Bean
    ConcurrentKafkaListenerContainerFactory<Integer, String>
                        kafkaListenerContainerFactory(ConsumerFactory<Integer, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<Integer, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        return factory;
    }

    @Bean
    public ConsumerFactory<Integer, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerProps());
    }

    private Map<String, Object> consumerProps() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        // ...
        return props;
    }
}
```

**第三步：消息监听器**

在方法上添加 `@KafkaListener` 注解，这也是最常见的使用方式。

```java
@Service
public class Listener {
    @KafkaListener(id = "listen1", topics = "topic1")
    public void listen1(String in) {
        System.out.println(in);
    }
}
```

至此，项目就能轻松的实现消息监听了。我们基于此使用方式继续探讨 Spring Kafka 消费者的创建过程。

## 二、消费者的初始化过程

基于上面示例，我们以 `@EnableKafka` 注解为切入点，源码如下。

```java
@Import(KafkaListenerConfigurationSelector.class)
public @interface EnableKafka {
}
```

最终向 Spring 注册了两个 Bean：

```java
public class KafkaBootstrapConfiguration implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 省略无关代码
        // beanName: org.springframework.kafka.config.internalKafkaListenerAnnotationProcessor
        registry.registerBeanDefinition(
            KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME,
            new RootBeanDefinition(KafkaListenerAnnotationBeanPostProcessor.class));
        // beanName: org.springframework.kafka.config.internalKafkaListenerEndpointRegistry
        registry.registerBeanDefinition(
            KafkaListenerConfigUtils.KAFKA_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,
            new RootBeanDefinition(KafkaListenerEndpointRegistry.class));
    }
}
```

由此说明 `KafkaListenerAnnotationBeanPostProcessor` 和 `KafkaListenerEndpointRegistry` 在消费者的初始化过程中起到了重要的作用。

这是两个核心的角色：

- KafkaListenerAnnotationBeanPostProcessor：扫描 `@KafkaListener`；
- KafkaListenerEndpointRegistry：`@KafkaListener` 注册表；

> <mark>代码阅读小记：</mark>
>
> ```
> @EnableKafka
> -> KafkaListenerConfigurationSelector
> -> KafkaBootstrapConfiguration
>     [注册Bean: KafkaListenerAnnotationBeanPostProcessor]
>     [注册Bean: KafkaListenerEndpointRegistry]
> ```

### 2.1 消费者注册（第一个 Bean）

首先来看第一个 Bean: `KafkaListenerAnnotationBeanPostProcessor`。

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    @Override
    public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
        // 省略无关代码
        // 查找被 @KafkaListener 标记的方法 (或者自定义注解上带 @KafkaListener 标记的方法)
        Map<Method, Set<KafkaListener>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
                (MethodIntrospector.MetadataLookup<Set<KafkaListener>>) method -> {
                    Set<KafkaListener> listenerMethods = findListenerAnnotations(method);
                    return (!listenerMethods.isEmpty() ? listenerMethods : null);
                });
        // 处理 KafkaListener
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

上述代码有两个关键点，第一是查找 `@KafkaListener ` 标记的方法添加了特殊的 Lookup 特性；第二就是处理 `KafkaListener`。

什么是 Lookup 特性呢？

举个例子，我们定义了一个新的注解 `@EventHandler` ，并在该注解上标记 `@KafkaListener`。

```java
@KafkaListener
public @interface EventHandler {
    @AliasFor(annotation = KafkaListener.class, attribute = "topics")
    String topics() default "";

    @AliasFor(annotation = KafkaListener.class, attribute = "groupId")
    String groupId() default "";

    @AliasFor(annotation = KafkaListener.class, attribute = "concurrency")
    String concurrency() default "3";

    @AliasFor(annotation = KafkaListener.class, attribute = "containerFactory")
    String containerFactory() default "";
}
```

当我们使用新的注解 `@EventHandler` 标记方法时，能起到和 `@KafkaListener` 一样的效果。

```java
@Service
public class Listener {
    @EventHandler(groupId = "listen1", topics = "topic1")
    public void listen1(String in) {
        System.out.println(in);
    }
}
```

我们继续处理 KafkaListener 代码跟踪，现在来到了 `KafkaListenerAnnotationBeanPostProcessor` 的 `processListener()` 方法。

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    // KafkaListener 注册器
    private final KafkaListenerEndpointRegistrar registrar = new KafkaListenerEndpointRegistrar();
    // 省略其他方法
    
    protected void processListener(MethodKafkaListenerEndpoint<?, ?> endpoint, KafkaListener kafkaListener,
            Object bean, Object adminTarget, String beanName) {
        // 省略 endpoint 其他赋值代码
        endpoint.setBean(bean);
        endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);
        endpoint.setId(getEndpointId(kafkaListener));
        endpoint.setGroupId(getEndpointGroupId(kafkaListener, endpoint.getId()));
        endpoint.setTopics(resolveTopics(kafkaListener));
        // 执行注册
        this.registrar.registerEndpoint(endpoint, factory);
    }
}
```

被 `@KafkaListener` 标记的方法最终会被实例化成 `MethodKafkaListenerEndpoint` 对象，并由第三个角色--注册器 `KafkaListenerEndpointRegistrar` 进行注册。

```java
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
    private final List<KafkaListenerEndpointDescriptor> endpointDescriptors = new ArrayList<>();
    // 省略其他方法
    
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

接下来会执行 Bean 的下一个生命周期方法：`afterSingletonsInstantiated`。

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    // KafkaListener 注册器
    private final KafkaListenerEndpointRegistrar registrar = new KafkaListenerEndpointRegistrar();
    // 省略其他方法
    
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

### 2.2 启动监听（第二个 Bean）

再来看第二个 Bean: `KafkaListenerEndpointRegistry`。它实现了 Spring 生命周期 SmartLifecycle 接口，在程序启动时，会调用 Bean 的 `start` 方法。

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

注册表中包含了许多消息监听容器（MessageListenerContainer），具体来说是 `ConcurrentMessageListenerContainer`  ，这是一个多线程版本的消息监听容器（MessageListenerContainer），相对应的 `KafkaMessageListenerContainer` 是单线程版本的消息监听容器（MessageListenerContainer）。

```java
public class ConcurrentMessageListenerContainer<K, V> extends AbstractMessageListenerContainer<K, V> {

    @Override
    protected void doStart() {
        ContainerProperties containerProperties = getContainerProperties();
        TopicPartitionOffset[] topicPartitions = containerProperties.getTopicPartitions();
        for (int i = 0; i < this.concurrency; i++) {
            KafkaMessageListenerContainer<K, V> container =
                    constructContainer(containerProperties, topicPartitions, i);
            // 启动每一个 KafkaMessageListenerContainer
            container.start();
        }
    }
}
```

可见，`ConcurrentMessageListenerContainer` 通过委托给多个`KafkaMessageListenerContainer`实例从而实现多线程消费。

```java

public class KafkaMessageListenerContainer<K, V>  extends AbstractMessageListenerContainer<K, V> {
    
    private volatile ListenerConsumer listenerConsumer;
    
    @Override
    protected void doStart() {
        ContainerProperties containerProperties = getContainerProperties();
        Object messageListener = containerProperties.getMessageListener();
        GenericMessageListener<?> listener = (GenericMessageListener<?>) messageListener;
        ListenerType listenerType = determineListenerType(listener);
        
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

最终我们在 `KafkaMessageListenerContainer` 的内部类 `ListenerConsumer` 中发现了 kafka-clients.jar 中的 Consumer 接口类。它的创建过程是由 `ConsumerFactory` 代为创建，`ConsumerFactory` 是一个接口类，它只有一个实现：`DefaultKafkaConsumerFactory`。

最终，Consumer 消费者的创建代码如下。

```java
public class DefaultKafkaConsumerFactory<K, V> extends KafkaResourceFactory
		implements ConsumerFactory<K, V>, BeanNameAware {

    protected Consumer<K, V> createKafkaConsumer(Map<String, Object> configProps) {
		Consumer<K, V> kafkaConsumer = createRawConsumer(configProps);
		return kafkaConsumer;
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
@EnableKafka
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

