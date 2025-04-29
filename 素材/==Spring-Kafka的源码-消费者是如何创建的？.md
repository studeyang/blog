# Spring-Kafka消费者是如何创建的？

> 源码版本：spring-kafka v2.6.6

## 如何使用Spring-Kafka消费消息？

### 1、引入依赖

```xml
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
  <version>2.6.6</version>
</dependency>
```

### 2、消费者配置

配置类上添加 `@EnableKafka` 注解，并初始化 `ConcurrentKafkaListenerContainerFactory` Bean。

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

### 3、消息监听器

```java
@Service
public class Listener {
    @KafkaListener(id = "listen1", topics = "topic1")
    public void listen1(String in) {
        System.out.println(in);
    }
}
```

## 初始化过程

以 `@EnableKafka` 注解为切入点。

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
        registry.registerBeanDefinition(KafkaListenerConfigUtils.KAFKA_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME,
                    new RootBeanDefinition(KafkaListenerAnnotationBeanPostProcessor.class));
        // beanName: org.springframework.kafka.config.internalKafkaListenerEndpointRegistry
        registry.registerBeanDefinition(KafkaListenerConfigUtils.KAFKA_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,
                    new RootBeanDefinition(KafkaListenerEndpointRegistry.class));
    }
}
```

总结代码的链路如下：

```
@EnableKafka
-> KafkaListenerConfigurationSelector
-> KafkaBootstrapConfiguration
    [注册Bean: KafkaListenerAnnotationBeanPostProcessor]
    [注册Bean: KafkaListenerEndpointRegistry]
```

### 消费者注册

核心角色：

- KafkaListenerAnnotationBeanPostProcessor：扫描 `@KafkaListener`；
- KafkaListenerEndpointRegistrar：`@KafkaListener` 注册器；
- KafkaListenerEndpointRegistry：`@KafkaListener` 注册表；

KafkaListenerAnnotationBeanPostProcessor

```java
public class KafkaListenerAnnotationBeanPostProcessor<K, V>
        implements BeanPostProcessor, Ordered, BeanFactoryAware, SmartInitializingSingleton {
    @Override
    public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {
        // 省略无关代码
        // 查找被 @KafkaListener 标记的方法 (自定义注解上带 @KafkaListener 也能识别)
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



被 `@KafkaListener` 标记的方法最终会被实例化成 `MethodKafkaListenerEndpoint` 对象。

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
        endpoint.setTopicPartitions(resolveTopicPartitions(kafkaListener));
        endpoint.setTopics(resolveTopics(kafkaListener));
        endpoint.setTopicPattern(resolvePattern(kafkaListener));
        endpoint.setClientIdPrefix(resolveExpressionAsString(kafkaListener.clientIdPrefix(), "clientIdPrefix"));
        // 执行注册
        this.registrar.registerEndpoint(endpoint, factory);
    }
}
```

`KafkaListener` 会被注册到 List 集合中。

```java
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
    // 容器
    private final List<KafkaListenerEndpointDescriptor> endpointDescriptors = new ArrayList<>();
    // 省略其他方法
    
    public void registerEndpoint(KafkaListenerEndpoint endpoint, KafkaListenerContainerFactory<?> factory) {
        // 省略无关代码
        KafkaListenerEndpointDescriptor descriptor = new KafkaListenerEndpointDescriptor(endpoint, factory);
        this.endpointDescriptors.add(descriptor);
    }
}
```

总结代码的链路如下：

```
-> KafkaListenerAnnotationBeanPostProcessor#postProcessAfterInitialization()
-> processKafkaListener()
-> processListener()
-> KafkaListenerEndpointRegistrar#registerEndpoint()
-> endpointDescriptors [注册到容器里List]
```

此时，程序里已经完成了 `KafkaListener` 的注册并存储至 endpointDescriptor 中。



同样是 KafkaListenerAnnotationBeanPostProcessor，

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



```java
public class KafkaListenerEndpointRegistrar implements BeanFactoryAware, InitializingBean {
    
    private KafkaListenerEndpointRegistry endpointRegistry;
    
    @Override
    public void afterPropertiesSet() {
        registerAllEndpoints();
    }

    protected void registerAllEndpoints() {
		for (KafkaListenerEndpointDescriptor descriptor : this.endpointDescriptors) {
			this.endpointRegistry.registerListenerContainer(
					descriptor.endpoint, resolveContainerFactory(descriptor));
		}
    }
}
```



```java
public class KafkaListenerEndpointRegistry implements DisposableBean, SmartLifecycle, ApplicationContextAware,
        ApplicationListener<ContextRefreshedEvent> {
            
    private final Map<String, MessageListenerContainer> listenerContainers = new ConcurrentHashMap<>();
    
    public void registerListenerContainer(KafkaListenerEndpoint endpoint, KafkaListenerContainerFactory<?> factory) {
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



### 启动监听

```java
public class KafkaListenerEndpointRegistry implements DisposableBean, SmartLifecycle, ApplicationContextAware,
        ApplicationListener<ContextRefreshedEvent> {
            
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

`ConcurrentMessageListenerContainer` 委托给一个或多个`KafkaMessageListenerContainer`实例以提供多线程消费。

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

总结代码的链路如下：

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


