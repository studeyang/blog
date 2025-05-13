# 源码跟踪

## RetryTemplate

```
首次调用
1. RetryTemplate#doExecute()
2. CompositeRetryPolicy#canRetry()
    - MaxAttemptsRetryPolicy
    - BinaryExceptionClassifierRetryPolicy
3. RetryCallback#doWithRetry()
    - 执行业务逻辑 - 抛出异常
4. RetryTemplate#registerThrowable()
5. CompositeRetryPolicy#canRetry()
    - MaxAttemptsRetryPolicy
    - BinaryExceptionClassifierRetryPolicy
6. ExponentialBackOffPolicy#backOff()
    - ThreadWaitSleeper#sleep() 休眠
7. 下一次循环
```



2个重试Policy

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2024/202504181523713.png)

# 使用方式

## 使用 RetryTemplate

```java
RetryTemplate retryTemplate = RetryTemplate.builder()
                .maxAttempts(3)
                .exponentialBackoff(100, 2, 5000)
                .retryOn(ProducerException.class)
                .traversingCauses()
                .build();
```

解释一下这段代码：`exponentialBackoff(100, 2, 5000)`

```
第1次重试会在 100ms 之后
第2次重试会在 2*100ms 之后
第3次重试会在 3*100ms 之后
...
第50次重试会在 50*100ms 之后
第51次重试会在 5000ms 之后
第52次重试会在 5000ms 之后
```

解释一下这段代码： `traversingCauses()`

假设现在抛出了这样一个异常：`new MyLogicException(new IOException());`

这个 template 将不会重试：

```java
RetryTemplate.builder()
    .retryOn(IOException.class)
    .build();
```

而这个 template 会重试：

```java
RetryTemplate.builder()
    .retryOn(IOException.class)
    .traversingCauses()
    .build()
```

