@Async 使用的是什么样的线程池?

```
ProxyAsyncConfiguration # asyncAdvisor()
AsyncAnnotationBeanPostProcessor # setBeanFactory() -> new AsyncAnnotationAdvisor()
AsyncAnnotationAdvisor -> buildAdvice() -> new AnnotationAsyncExecutionInterceptor()
AnnotationAsyncExecutionInterceptor
// 容器启动后，默认线程池依然是 null
```

第一次调用，会根据 Bean 名称去查找线程池。

AsyncExecutionAspectSupport：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230419125736337.png)

如果没有指定线程池，则会调用`getDefaultExecutor(this.beanFactory)`获取默认的线程池。

AsyncExecutionInterceptor：

![image-20230419130548985](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230419130548985.png)

SimpleAsyncTaskExecutor 是一个简单的异步执行器，通过 new Thread() 来执行任务。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230419131543136.png)