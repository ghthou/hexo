---
title: Apache Common Lang 学习之 concurrent 包
tags:
  - Java
  - Common Lang
  - 源码研究
categories:
  - Java
date: 2018-01-14 21:14:27
---

> 标准的 Java 库不能提供足够的方法来操纵其核心类，所以 [Apache Commons Lang](http://commons.apache.org/proper/commons-lang/) 为我们提供了这些额外的方法
> 本文便介绍 [Apache Commons Lang](http://commons.apache.org/proper/commons-lang/) 中 concurrent 包的使用说明

# 引入 jar 包

JDK 版本需要大于等于 1.7

## Maven

``` html
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.7</version>
</dependency>
```

## Gradle

```groovy
compile group: 'org.apache.commons', name: 'commons-lang3', version: '3.7'
```

# 包结构说明

## 接口摘要

| Interface                                | Description                              |
| :--------------------------------------- | ---------------------------------------- |
| [CircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CircuitBreaker.html)<T> | 描述断路器 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html) 的接口 |
| [Computable](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Computable.html)<I,O> | 为单个参数的计算提供定义一个包装接口,并返回一个结果               |
| [ConcurrentInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentInitializer.html)<T> | 定义线程安全的初始化接口                             |

## 类摘要

| Class                                    | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| [AbstractCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AbstractCircuitBreaker.html)<T> | 断路器基础类                                   |
| [AtomicInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicInitializer.html)<T> | `ConcurrentInitializer` 接口基于 `AtomicReference` 变量的实现类 |
| [AtomicSafeInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicSafeInitializer.html)<T> | `ConcurrentInitializer` 接口基于 `AtomicReference` 变量的实现类,但是初始化方法 `AtomicSafeInitializer.initialize()`只会执行一次 |
| [BackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BackgroundInitializer.html)<T> | `ConcurrentInitializer` 接口实现类,通过后台线程完成初始化 |
| [BasicThreadFactory](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BasicThreadFactory.html) | `ThreadFactory` 接口实现类,提供一些配置获取方法         |
| [BasicThreadFactory.Builder](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BasicThreadFactory.Builder.html) | `BasicThreadFactory` 的 `Builder` 模式实例    |
| [CallableBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CallableBackgroundInitializer.html)<T> | `BackgroundInitializer` 实现类,通过传入 `Callable`进行实现 |
| [ConcurrentUtils](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentUtils.html) | 提供 `java.util.concurrent` 包相关功能的工具类      |
| [ConstantInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConstantInitializer.html)<T> | `ConcurrentInitializer` 接口实现类,直接返回构造器中传入的参数 |
| [EventCountCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/EventCountCircuitBreaker.html) | 计算特定事件的 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html) 断路器模式的简单实现 |
| [LazyInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/LazyInitializer.html)<T> | `ConcurrentInitializer` 接口实现类,通过双重检查锁进行实现 |
| [Memoizer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Memoizer.html)<I,O> | 为单个参数的计算定义一个包装的接口,并返回一个结果                |
| [MultiBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/MultiBackgroundInitializer.html) | `BackgroundInitializer`实现类,可以在后台进行多个初始化  |
| [MultiBackgroundInitializer.MultiBackgroundInitializerResults](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/MultiBackgroundInitializer.MultiBackgroundInitializerResults.html) | 一个数据类,用于存储`MultiBackgroundInitializer` 初始化后的结果 |
| [ThresholdCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ThresholdCircuitBreaker.html) | 通过给定阈值开启 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html) 断路器模式的简单实现 |
| [TimedSemaphore](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/TimedSemaphore.html) | 一个专门的*信号量*实现,在给定的时间内提供许可                 |

## 枚举摘要

| Enum                                     | Description    |
| ---------------------------------------- | -------------- |
| [AbstractCircuitBreaker.State](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AbstractCircuitBreaker.State.html) | 表示断路器不同状态的内部枚举 |

## 异常摘要

| Exception                                | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| [CircuitBreakingException](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CircuitBreakingException.html) | 用于报告与 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html) 断路器相关的运行时错误条件的异常类 |
| [ConcurrentException](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentException.html) | 用于报告与访问后台任务数据相关的错误条件的异常类                 |
| [ConcurrentRuntimeException](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentRuntimeException.html) | 用于报告与访问后台任务数据有关的**运行时**错误条件的异常类          |

# 使用说明

## [CircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CircuitBreaker.html)<T> 及其子类

### 类层次结构

- [CircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CircuitBreaker.html)<T>
  - [AbstractCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AbstractCircuitBreaker.html)<T>
    - [EventCountCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/EventCountCircuitBreaker.html)
    - [ThresholdCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ThresholdCircuitBreaker.html)

### 类详细介绍

#### [CircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CircuitBreaker.html)<T>

描述断路器的接口

一个**断路器**可用来防止不可靠的服务或意外负载的应用程序。 它通常监视特定的资源。 只要这个资源按照预期工作，它就处于**关闭**状态，这意味着资源可以被使用。 如果使用该资源时遇到问题时，断路器可以切换到**打开**状态 。 那么访问这个资源是被禁止的。 根据具体的实施方式，当资源再次可用时，断路器可能切换到**关闭**状态。 

该接口定义了断路器组件的通用协议

#### [AbstractCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AbstractCircuitBreaker.html)<T>

断路器的基础类，实现了 `CircuitBreaker` 大部分接口

#### [EventCountCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/EventCountCircuitBreaker.html)

计算特定事件的断路器模式的简单实现

一个**断路器**可用来防止不稳定的服务或遭遇意外高峰负载的应用程序。 新创建的`EventCountCircuitBreaker`对象最初处于**关闭**状态，意味着没有检测到任何问题。 当应用程序遇到特定事件（如错误或服务超时）时，它会通知断路器增加一个内部计数器。 如果在一个特定的时间间隔中报告的事件的数量超过配置的阈值时，断路器将被**打开**。 这意味着相关的子系统有问题； 应用程序不应该再使用它，而是给它一点时间让它安顿下来。 如果接收的事件数量低于阈值，断路器可以配置为在一定的时间之后切换回**关闭**状态。 

当一个`EventCountCircuitBreaker`对象被构造时，可以提供下列参数：

- 导致状态转换为打开的事件数量的阈值 。 如果在检查间隔内收到事件的数量等于阈值，则断路器将切换到打开状态。 
- 检查断路器是否打开的时间间隔。 所以可以指定 「如果在一分钟内遇到 10 个以上的错误，断路器应该打开」。
- 自动关闭断路器的阈值。 如 「如果请求数量降低到每分钟 100 次，断路器应该再次自动关闭」。根据使用情况，关闭断路器的门槛略低于打开门槛，以避免在收到的事件数量接近门槛时连续转换。

构造方法如下

```java
// threshold                 改变断路器状态的阈值; 如果在检查间隔内收到的事件数量大于此值,则断路器打开; 如果它低于这个值,它会再次关闭
// checkInterval             打开或关闭断路器的检查间隔
// checkUnit                 checkInterval 的时间单位
public EventCountCircuitBreaker(final int threshold, final long checkInterval, final TimeUnit checkUnit) {
  this(threshold, checkInterval, checkUnit, threshold);
}

// openingThreshold        改变断路器状态的阈值; 如果在检查间隔内收到的事件数量大于此值,则断路器打开
// checkInterval        打开或关闭断路器的检查间隔
// checkUnit            checkInterval 的时间单位
// closingThreshold        关闭断路器的阈值; 如果检查间隔内接收的事件数量低于该阈值,则断路器再次关闭
public EventCountCircuitBreaker(final int openingThreshold, final long checkInterval, final TimeUnit checkUnit, final int closingThreshold) {
  this(openingThreshold, checkInterval, checkUnit, closingThreshold, checkInterval, checkUnit);
}

// openingThreshold        改变断路器状态的阈值; 如果在检查间隔内收到的事件数量大于此值,则断路器打开
// openingInterval         打开断路器的检查间隔
// openingUnit            checkInterval 的时间单位
// closingThreshold        关闭断路器的阈值; 如果检查间隔内接收的事件数量低于该阈值,则断路器再次关闭
// closingInterval        关闭断路器的时间间隔
// closingUnit            closingInterval 的时间单位
public EventCountCircuitBreaker(final int openingThreshold, final long openingInterval, final TimeUnit openingUnit, final int closingThreshold, final long closingInterval, final TimeUnit closingUnit){
    // 忽略
}                                    
```

这个类支持以下典型用例：

**防止负载高峰**

想象一下你有一个服务器可以每分钟处理一定数量的请求。 突然之间，请求数量显著增加，可能是遭到 DDoS（拒绝服务攻击）。 `EventCountCircuitBreaker` 可以配置为在检测到突然的高峰负载时停止应用程序处理请求，并在事情平静时再开始请求处理。 以下代码片段显示了这种情况的典型示例。 这里`EventCountCircuitBreaker`在断路器打开之前允许每分钟最多 1000 个请求。 当负载再次下降到每秒 800 个请求时，它会切换回关闭状态：

```java
 EventCountCircuitBreaker breaker = new EventCountCircuitBreaker(1000, 1, TimeUnit.MINUTE, 800);
 // something
 public void handleRequest(Request request) {
       // 将监测值增加 1, 并检查该断路器的当前状态是否关闭
     if (breaker.incrementAndCheckState()) {
         //实际上处理这个请求
     } else {
         //做其他事, 例如发送一个错误代码
     }
 }
```

**处理不稳定的服务**

假如应用程序是一个不稳定的第三方服务。 如果错误太多，服务应被视为关闭，停止服务一段时间。 可以使用以下方式来实现，在这个具体的例子中，我们在 2 分钟内允许 5 个错误； 如果达到这个限制，服务会有 10 分钟的休息时间：

```java
 EventCountCircuitBreaker breaker = new EventCountCircuitBreaker(5, 2, TimeUnit.MINUTE, 5, 10, TimeUnit.MINUTE);
 // something
 public void handleRequest(Request request) {
       // 检查断路器是否关闭
     if (breaker.checkState()) {
         try {
             service.doSomething();
         } catch (ServiceException ex) {
           // 将监测值增加 1
             breaker.incrementAndCheckState();
         }
     } else {
         // 返回错误代码,或使用替代服务
     }
 }
```

除了自动状态转换，断路器的状态可以使用 `open()`和 `close()`方法进行手动更改。 同时也可以使用 `addChangeListener(final PropertyChangeListener listener) ` 方法注册监听器，当发生状态转换时，可以得到事件改变对象 `PropertyChangeEvent`，此时可以根据状态更改的情况做出反应

*实现说明：*

- 该实现使用非阻塞算法来更新内部计数器和状态。 如果没有太多的竞争，这应该是非常有效的。 
- 这个实现并不打算在非常短的检查间隔中作为高精度定时器来运行。 故意保持简单，以避免复杂和耗时的状态检查。 它应该在几秒到几分钟甚至更长的时间内运行良好。 如果时间间隔太短，可能会因竞争状态导致虚假的状态转换。 
- 检查间隔的处理有点简单。 因此，不能保证断路器在特定的时间点被触发； 可能会有一些延迟（小于检查间隔）。 

#### [ThresholdCircuitBreaker](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ThresholdCircuitBreaker.html)

如果请求的次数大于给定阈值，则会打开 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html) 模式的简单实现。 

它包含一个从零开始的内部计数器，每次调用将计数器增加一个给定的数量。 如果阈值是零，断路器将永远处于打开状态。 

一个内存断路器示例

```java
 long threshold = 10L;
 ThresholdCircuitBreaker breaker = new ThresholdCircuitBreaker(10L);
 ...
 public void handleRequest(Request request) {
     long memoryUsed = estimateMemoryUsage(request);
     if (breaker.incrementAndCheckState(memoryUsed)) {
         // 实际上处理这个请求
     } else {
         // 做其他事, 例如发送一个错误代码
     }
 }
```
## [Computable](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Computable.html)<I,O> 及其子类

### 类层次结构

- [Computable](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Computable.html)<I,O> 
  - [Memoizer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Memoizer.html)<I,O>

### 类详细介绍

#### [Computable](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Computable.html)<I,O>

为单个参数的计算定义一个包装的接口，并返回一个结果。 

#### [Memoizer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/Memoizer.html)<I,O>

为单个参数的计算定义一个包装的接口，并返回一个结果。 计算结果将被缓存

这不是一个全功能的缓存，一旦生成结果，就无法限制或删除结果。 但是，如果在上一次计算过程中抛出错误，则可以通过在构造方法中设置一个选项来实现重新生成给定参数的结果。 如果没有设置或设置为 false，将抛出缓存的异常

##### 源码分析

```java
public class Memoizer<I, O> implements Computable<I, O> {

      // 计算结果缓存
    private final ConcurrentMap<I, Future<O>> cache = new ConcurrentHashMap<>();
      // 构造参数中的 Computable
    private final Computable<I, O> computable;
      // 计算发生异常时是否重算
    private final boolean recalculate;

    // 根据 Computable 创建对象
      // 如果在计算过程中发生异常,再次计算时会直接抛出异常
    public Memoizer(final Computable<I, O> computable) {
        this(computable, false);
    }

    // recalculate 是否重算
      // 如果为 true, 上次计算发生异常时下次计算时重新计算,否则抛出异常
    public Memoizer(final Computable<I, O> computable, final boolean recalculate) {
        this.computable = computable;
        this.recalculate = recalculate;
    }

      // 计算,并根据参数缓存结果
    @Override
    public O compute(final I arg) throws InterruptedException {
        while (true) {
              // 获取缓存结果
            Future<O> future = cache.get(arg);
              // 如果缓存为 null
            if (future == null) {
                  // 创建一个 Callable
                final Callable<O> eval = new Callable<O>() {

                    @Override
                    public O call() throws InterruptedException {
                          // 调用构造参数中的 computable 对 arg 进行计算
                        return computable.compute(arg);
                    }
                };
                  // 创建 FutureTask
                final FutureTask<O> futureTask = new FutureTask<>(eval);
                  // 如果 arg 不存在,添加缓存并返回 null,否则返回缓存中的 Future 对象
                  // 此操作是防止 compute 被并发执行
                future = cache.putIfAbsent(arg, futureTask);
                  // 如果缓存添加成功
                if (future == null) {
                    future = futureTask;
                      // 运行
                    futureTask.run();
                }
            }
            try {
                  // 获取缓存的结果
                return future.get();
            } catch (final CancellationException e) {
                cache.remove(arg, future);
            } catch (final ExecutionException e) {
                  // 如果设置了重新计算
                if (recalculate) {
                      // 删除缓存
                    cache.remove(arg, future);
                }
                // 抛出异常
                throw launderException(e.getCause());
            }
        }
    }

       // 异常处理
    private RuntimeException launderException(final Throwable throwable) {
        if (throwable instanceof RuntimeException) {
            return (RuntimeException) throwable;
        } else if (throwable instanceof Error) {
            throw (Error) throwable;
        } else {
            throw new IllegalStateException("Unchecked exception", throwable);
        }
    }
}
```
##  [ConcurrentInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentInitializer.html)<T> 及其子类

###  类层次结构

- [ConcurrentInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentInitializer.html)<T>
  - [AtomicInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicInitializer.html)<T>
  - [AtomicSafeInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicSafeInitializer.html)<T>
  - [ConstantInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConstantInitializer.html)<T>
  - [LazyInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/LazyInitializer.html)<T>
  - [BackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BackgroundInitializer.html)<T>
    - [CallableBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CallableBackgroundInitializer.html)<T>
    - [MultiBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/MultiBackgroundInitializer.html)

###  类详细介绍

ConcurrentInitializer 及其子类主要是用来进行数据安全初始化的，开头的 Concurrent 表示支持并发操作

它只定义了一个接口

```java
 public interface ConcurrentInitializer<T> {
    T get() throws ConcurrentException;
 }
```

调用 `get()` 方法即可获取完成初始化的对象.同时提供了多种实现，可根据需要进行选择

#### [ConstantInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConstantInitializer.html)<T>

##### 功能描述

ConstantInitializer 是最简单的 ConcurrentInitializer 实现

在构造器中传入一个对象，在它的 `get()` 方法中直接返回这个对象.适用于单元测试及替换其他实现类

##### 源码分析

```java
public class ConstantInitializer<T> implements ConcurrentInitializer<T> {

    private final T object;

    public ConstantInitializer(final T obj) {
        object = obj;
    }

    public final T getObject() {
        return object;
    }

    @Override
    public T get() throws ConcurrentException {
        return getObject();
    }
}
```

##### 代码示例

```java
ConcurrentInitializer<String> initializer = new ConstantInitializer<>("test");
initializer.get();    //返回 test
```

#### [LazyInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/LazyInitializer.html)<T>

##### 功能描述

顾名思义，使用懒加载的方式进行初始化，只有在第一次调用时才进行初始化操作，通过实现 `initialize` 方法返回初始化后的对象

##### 源码分析

使用**双重检查锁**进行实现

```java
public abstract class LazyInitializer<T> implements ConcurrentInitializer<T> {

    private static final Object NO_INIT = new Object();

      // 使用 volatile 修饰符,保持 object 在多线程情况下的可见性
    private volatile T object = (T) NO_INIT;

    @Override
    public T get() throws ConcurrentException {

          // 创建返回对象,并赋值为 object
        T result = object;
        // 如果 result 等于 NO_INIT, 表明 object 对象未被更改过
        if (result == NO_INIT) {
              // 加锁
            synchronized (this) {
                  // 重新将 object 赋值给 result
                result = object;
                  // 如果 result 依然等于 NO_INIT,表明当前线程是第一个获取到锁的线程
                  // 否者表明在等待获取锁的时间已经有其他线程对 result 进行了更改
                if (result == NO_INIT) {
                      // 调用子类的 initialize 方法并赋值给 result,object
                    object = result = initialize();
                }
            }
        }

        return result;
    }

    protected abstract T initialize() throws ConcurrentException;
}

```

##### 代码示例

```java
ConcurrentInitializer<Properties> initializer = new LazyInitializer<Properties>() {
  @Override
  protected Properties initialize() throws ConcurrentException {
    return loadProperties();
  }
};
initializer.get();
```

#### [AtomicInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicInitializer.html)<T>

##### 功能描述

与 LazyInitializer 功能，用法一致，不过实现方式不同，同时其 `initialize` 方法会存在调用多次的问题

##### 源码分析

使用 **CAS** 方式进行实现

```java
public abstract class AtomicInitializer<T> implements ConcurrentInitializer<T> {

      // 初始化一个 AtomicReference 对象,可以以原子方式更新对象引用
      // 默认引用值为 null
    private final AtomicReference<T> reference = new AtomicReference<>();

    @Override
    public T get() throws ConcurrentException {
          // 获取引用值
        T result = reference.get();
        // 如果为 null,表明对象引用未被设置
          // 否者表明该对象引用已经被设置,即被初始化
        if (result == null) {
              // 调用 initialize 进行初始化
              // 假如多个线程同时执行到这一步,会导致 initialize 方法被调用多次
            result = initialize();
              // 使用 CAS 的方式修改引用值,该应用旧值为 null,新值为初始化后返回的 result
              // 如果更新成功,直接返回 result 对象
            if (!reference.compareAndSet(null, result)) {
                // 如果更新失败,表示有其他线程更新成功,通过 reference.get() 方法可获取其他线程设置的值
                result = reference.get();
            }
        }

        return result;
    }


    protected abstract T initialize() throws ConcurrentException;
}

```

##### 代码示例

```java
ConcurrentInitializer<Properties> initializer = new AtomicInitializer<Properties>() {
  @Override
  protected Properties initialize() throws ConcurrentException {
    return loadProperties();
  }
};
initializer.get();
```

#### [AtomicSafeInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/AtomicSafeInitializer.html)<T>

##### 功能描述

AtomicInitializer 存在一个 `initialize` 执行多次的问题.而 AtomicSafeInitializer 则是为了解决这个问题而诞生，使用它，`initialize` 只会被执行一次

##### 源码分析

```java
public abstract class AtomicSafeInitializer<T> implements
        ConcurrentInitializer<T> {

      // 一个 AtomicReference 变量,用于保证 initialize 只会被调用一次
    private final AtomicReference<AtomicSafeInitializer<T>> factory = new AtomicReference<>();

    private final AtomicReference<T> reference = new AtomicReference<>();

    @Override
    public final T get() throws ConcurrentException {
        T result;
        // 获取引用
          // 如果引用为 null,进入初始化步骤
        while ((result = reference.get()) == null) {
              // 操作 factory 更新引用,在多线程的情况下,只有一个线程能执行成功
              // 如果更新成功,当前线程进行初始化,并将返回结果设置为 reference 的引用
              // 如果更新失败,在 while 中重新获取 reference 的引用,直到更新成功的那个线程为 reference 设置新值
            if (factory.compareAndSet(null, this)) {
                reference.set(initialize());
            }
        }

        return result;
    }


    protected abstract T initialize() throws ConcurrentException;
}

```

##### 代码示例

```java
ConcurrentInitializer<Properties> initializer = new AtomicSafeInitializer<Properties>() {
  @Override
  protected Properties initialize() throws ConcurrentException {
    return loadProperties();
  }
};
initializer.get();
```

#### [BackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BackgroundInitializer.html) <T>

##### 功能描述

使用后台线程进行初始化，在创建对象后需要先执行 `start()` 保证后台线程运行，再调用 `get()` 方法

默认`Executors.newFixedThreadPool(1)` 创建一个线程数量为 1 的线程池，在 `initialize` 方法执行完成之后对线程池进行关闭

同时提供一个 `protected BackgroundInitializer(final ExecutorService exec) {}` 构造器方法，此时使用传入的 `ExecutorService` 对象执行 `initialize` 方法，执行完成后不会该关闭线程池

##### 源码分析

```java
public abstract class BackgroundInitializer<T> implements
        ConcurrentInitializer<T> {
    // 构造器传入的 ExecutorService 对象
    private ExecutorService externalExecutor; 
    // 默认生成的 ExecutorService 对象
    private ExecutorService executor; 
    // 异步计算的结果
    private Future<T> future;

    protected BackgroundInitializer() {
        this(null);
    }
    // ExecutorService 参数构造器,使用传入的 ExecutorService 执行 initialize
    protected BackgroundInitializer(final ExecutorService exec) {
        setExternalExecutor(exec);
    }
    // 判断后台线程是否运行
    public synchronized boolean isStarted() {
        return future != null;
    }

    public synchronized boolean start() {
        // 后台线程是否启动
        if (!isStarted()) {

            // 定义一个临时线程池变量
            ExecutorService tempExec;
              // 获取构造器传入的 ExecutorService 对象
            executor = getExternalExecutor();
              // 如果 executor 为 null,使用无参构造器创建 BackgroundInitializer
            if (executor == null) {
                  // 创建一个线程数量为1的线程池对象,在使用完成后进行关闭
                executor = tempExec = createExecutor();
            } else {
                tempExec = null;
            }
            // 后台执行初始化
            future = executor.submit(createTask(tempExec));

            return true;
        }
        return false;
    }

      // 获取初始化结果
    @Override
    public T get() throws ConcurrentException {
        try {
            return getFuture().get();
        } catch (final ExecutionException execex) {
            ConcurrentUtils.handleCause(execex);
            return null;
        } catch (final InterruptedException iex) {
            Thread.currentThread().interrupt();
            throw new ConcurrentException(iex);
        }
    }

    public synchronized Future<T> getFuture() {
        if (future == null) {
            throw new IllegalStateException("start() must be called first!");
        }
        return future;
    }

    protected abstract T initialize() throws Exception;


    private Callable<T> createTask(final ExecutorService execDestroy) {
        return new InitializationTask(execDestroy);
    }

    private ExecutorService createExecutor() {
        return Executors.newFixedThreadPool(getTaskCount());
    }

    private class InitializationTask implements Callable<T> {

        private final ExecutorService execFinally;

          // ExecutorService 参数构造器,用来关闭默认创建的 ExecutorService 对象
        InitializationTask(final ExecutorService exec) {
            execFinally = exec;
        }

        @Override
        public T call() throws Exception {
            try {
                  // 执行子类初始化方法
                return initialize();
            } finally {
                  // 如果构造参数中的 ExecutorService 不为 null,进行关闭
                  // 只有使用 BackgroundInitializer() 创建对象才存在这种情况
                if (execFinally != null) {
                    execFinally.shutdown();
                }
            }
        }
    }
}
```

##### 代码示例

``` java
BackgroundInitializer<Properties> initializer = new BackgroundInitializer<Properties>() {
  @Override
  protected Properties initialize() throws ConcurrentException {
    return loadProperties();
  }
};
initializer.start();
initializer.get();
```

#### [CallableBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/CallableBackgroundInitializer.html) <T>

##### 功能描述

BackgroundInitializer 的子类，使用方式为在构造器中传入一个 `Callable` 对象，在初始化时调用该参数的 `call()` 方法

##### 源码分析

```java
public class CallableBackgroundInitializer<T> extends BackgroundInitializer<T> {
    // 构造参数中的 Callable
    private final Callable<T> callable;

    public CallableBackgroundInitializer(final Callable<T> call) {
        checkCallable(call);
        callable = call;
    }

    public CallableBackgroundInitializer(final Callable<T> call, final ExecutorService exec) {
        super(exec);
        checkCallable(call);
        callable = call;
    }

    @Override
    protected T initialize() throws Exception {
        return callable.call();
    }

    private void checkCallable(final Callable<T> call) {
        Validate.isTrue(call != null, "Callable must not be null!");
    }
}

```

##### 代码示例

```java
BackgroundInitializer<Properties> initializer = new CallableBackgroundInitializer<>(new Callable<Properties>() {
  @Override
  public Properties call() throws Exception {
    return loadProperties();
  }
});
initializer.start();
initializer.get();
```

#### [MultiBackgroundInitializer](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/MultiBackgroundInitializer.html)

##### 功能描述

BackgroundInitializer 多后台任务的实现，通过该类，可以并行执行多个 BackgroundInitializer

##### 源码分析

```java
public class MultiBackgroundInitializer
        extends
        BackgroundInitializer<MultiBackgroundInitializer.MultiBackgroundInitializerResults> {
    // 需要执行的多个 BackgroundInitializer Map, String 为 BackgroundInitializer 的名称
    private final Map<String, BackgroundInitializer<?>> childInitializers =
        new HashMap<>();

    public MultiBackgroundInitializer() {
        super();
    }

    public MultiBackgroundInitializer(final ExecutorService exec) {
        super(exec);
    }

    // 添加一个需要执行的 BackgroundInitializer
    public void addInitializer(final String name, final BackgroundInitializer<?> init) {
        Validate.isTrue(name != null, "Name of child initializer must not be null!");
        Validate.isTrue(init != null, "Child initializer must not be null!");

        synchronized (this) {
            if (isStarted()) {
                throw new IllegalStateException(
                        "addInitializer() must not be called after start()!");
            }
            childInitializers.put(name, init);
        }
    }

    // 返回默认线程池的构建数量
      // 默认情况下所有 BackgroundInitializer 与 MultiBackgroundInitializer 共享同一个线程池
      // 数量为 1 + 所有 BackgroundInitializer 的数量
      // 1 为 MultiBackgroundInitializer 后台执行的线程数
      // 理论上所有的 BackgroundInitializer 都是并行执行,不需要等待其他对象初始化完毕
      // 但是在构造参数中传入 ExecutorService 例外
    @Override
    protected int getTaskCount() {
        int result = 1;
        for (final BackgroundInitializer<?> bi : childInitializers.values()) {
            result += bi.getTaskCount();
        }
        return result;
    }

    // 多任务的初始化方法
    @Override
    protected MultiBackgroundInitializerResults initialize() throws Exception {
        Map<String, BackgroundInitializer<?>> inits;
        synchronized (this) {
            // 创建一个 childInitializers 对象快照
            inits = new HashMap<>(
                    childInitializers);
        }

        // 启动所有的 BackgroundInitializer
        final ExecutorService exec = getActiveExecutor();
        for (final BackgroundInitializer<?> bi : inits.values()) {
              // 如果 BackgroundInitializer 没有外部 ExecutorService
            if (bi.getExternalExecutor() == null) {
                // 使用 MultiBackgroundInitializer 默认的 ExecutorService
                bi.setExternalExecutor(exec);
            }
            bi.start();
        }

        // BackgroundInitializer 名称与初始化值 Map
        final Map<String, Object> results = new HashMap<>();
          // BackgroundInitializer 名称与异常 Map
        final Map<String, ConcurrentException> excepts = new HashMap<>();
        for (final Map.Entry<String, BackgroundInitializer<?>> e : inits.entrySet()) {
            try {
                results.put(e.getKey(), e.getValue().get());
            } catch (final ConcurrentException cex) {
                excepts.put(e.getKey(), cex);
            }
        }
        // 构建多 BackgroundInitializer 结果对象
        return new MultiBackgroundInitializerResults(inits, results, excepts);
    }

    // 多 BackgroundInitializer 结果对象
    public static class MultiBackgroundInitializerResults {

        private final Map<String, BackgroundInitializer<?>> initializers;

        private final Map<String, Object> resultObjects;

        private final Map<String, ConcurrentException> exceptions;

        private MultiBackgroundInitializerResults(
                final Map<String, BackgroundInitializer<?>> inits,
                final Map<String, Object> results,
                final Map<String, ConcurrentException> excepts) {
            initializers = inits;
            resultObjects = results;
            exceptions = excepts;
        }

        public BackgroundInitializer<?> getInitializer(final String name) {
            return checkName(name);
        }

        public Object getResultObject(final String name) {
            checkName(name);
            return resultObjects.get(name);
        }

        public boolean isException(final String name) {
            checkName(name);
            return exceptions.containsKey(name);
        }

        public ConcurrentException getException(final String name) {
            checkName(name);
            return exceptions.get(name);
        }

        public Set<String> initializerNames() {
            return Collections.unmodifiableSet(initializers.keySet());
        }

          // 是否全部成功
        public boolean isSuccessful() {
            return exceptions.isEmpty();
        }

        private BackgroundInitializer<?> checkName(final String name) {
            final BackgroundInitializer<?> init = initializers.get(name);
            if (init == null) {
                throw new NoSuchElementException(
                        "No child initializer with name " + name);
            }

            return init;
        }
    }
}

```

##### 代码示例

```java
MultiBackgroundInitializer initializer = new MultiBackgroundInitializer();
initializer.addInitializer("loadProperties1", new BackgroundInitializer<Properties>() {
  @Override
  protected Properties initialize() throws Exception {
    return loadProperties1();
  }
});
initializer.addInitializer("loadProperties2", new CallableBackgroundInitializer<Properties>(new Callable<Properties>() {
  @Override
  public Properties call() throws Exception {
    return loadProperties2();
  }
}));
initializer.start();
MultiBackgroundInitializer.MultiBackgroundInitializerResults results = initializer.get();
if (results.isSuccessful()) {
  results.getResultObject("loadProperties1");
  results.getResultObject("loadProperties2");
}
```

### 使用选择

如果初始化程序执行时间过长且不希望在第一次调用时等待太久请选择 `BackgroundInitializer` 与 `CallableBackgroundInitializer`

否则使用 `LazyInitializer` 与 `AtomicSafeInitializer`

## [BasicThreadFactory](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/BasicThreadFactory.html)

`ThreadFactory` 接口的实现，，提供一些配置获取方法

`ThreadFactory` 用于 `ExecutorService` 创建执行任务的线程。 在许多情况下，用户并不关心，因为 `ExecutorService` 会使用一个默认的 `ThreadFactory`  。 但是，如果对线程有特殊要求，则必须创建一个自定义的`ThreadFactory`

这个类为它创建的线程提供了一些经常需要的配置选项，这些是：

- `namingPattern` 线程的名称模式。 如果希望日志输出或异常跟踪更容易阅读，可以为线程起一个有意义的名称。 在线程命名时使用 `thread.setName(String.format(namingPattern, count));` 方式，`count` 即线程池当前创建的线程数量，从 1 开始。  `namingPattern` 中的 `%d` 会被替换为 `count` 。 如 ： `namingPattern` 为 `"My %d. worker thread"`，那么生成的线程名称为 `"My 1. worker thread"`，`"My 2. worker thread"` 
- `daemonFlag` 守护线程标志。 该工厂创建的线程是否是守护线程。 这会影响当前 Java 应用程序的退出行为，当正在运行的线程都是守护线程时，Java 虚拟机将退出，默认为 false ，即用户线程
- `priority` 线程的优先级.  `Integer` 类型,值区间为 **[1,10]**
- `uncaughtExceptionHandler` 线程由于未捕获到异常而突然终止时调用的处理程序。 如果线程内发生未捕获的异常，则调用此处理程序

`BasicThreadFactory`不是直接创建实例，而是使用内部类 `Builder`类实现此目的，使用 `Builder` 只需要设置应用程序感兴趣的配置选项。 以下示例显示了 `BasicThreadFactory`是如何创建并配置在在`ExecutorService`中：

```java
 // 创建一个线程工厂,配置名称模式,守护线程标志,优先级
 BasicThreadFactory factory = new BasicThreadFactory.Builder()
     .namingPattern("workerthread-%d")
     .daemon(true)
     .priority(Thread.MAX_PRIORITY)
     .build();
 // 创建一个单线程的线程池,指定 factory 为线程工厂
 ExecutorService exec = Executors.newSingleThreadExecutor(factory);
```
## [ConcurrentUtils](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/ConcurrentUtils.html)

提供 `java.util.concurrent` 包相关功能的工具方法

该类涉及到 `ConcurrentException` 的方法存在一个 `原方法名+Unchecked` 的相同功能方法，用于将受检异常 `ConcurrentException` 转换为运行时异常 `ConcurrentRuntimeException`

用法如下

```java
 Future<Object> future = ...;
 try {
   Object result = future.get();
   ...
 } catch (ExecutionException eex) {
   ConcurrentUtils.handleCause(eex);
   // ConcurrentUtils.handleCauseUnchecked(eex);
 }
```

### 方法列表

| Modifier and Type                   | Method and Description                   |
| ----------------------------------- | ---------------------------------------- |
| `static <T> Future<T>`              | `constantFuture(T value)`<br>创建一个已完成的 `Future` ,该  `Future` 的返回值及参数中的 `T value` |
| ` static <K,V> V`                   | `createIfAbsent(ConcurrentMap<K,V> map, K key, ConcurrentInitializer<V> init)`<br>检查 `ConcurrentMap` 是否包含 `key`,如果未包含,为 `ConcurrentMap` 设置 `key` : `init.get()` |
| ` static <K,V> V`                   | `createIfAbsentUnchecked(ConcurrentMap<K,V> map, K key, ConcurrentInitializer<V> init)`<br>`createIfAbsent` 的非受检异常版本 |
| `static ConcurrentException`        | `extractCause(ExecutionException ex)`<br>`ExecutionException` 异常原因转换, `RuntimeException` 与 `Error` 进行抛出,否则转换为受检异常 `ConcurrentException` 并返回 |
| `static ConcurrentRuntimeException` | `extractCauseUnchecked(ExecutionException ex)`<br>`extractCause` 的非受检异常版本 |
| `static void`                       | `handleCause(ExecutionException ex)`<br>`ExecutionException` 异常处理,`RuntimeException` 与 `Error` 进行抛出,否则转换为 `ConcurrentException` 在进行抛出 |
| `static void`                       | `handleCauseUnchecked(ExecutionException ex)`<br>`handleCauseUnchecked` 的非受检异常版本 |
| `static <T> T`                      | `initialize(ConcurrentInitializer<T> initializer)`<br>获取 `initializer` 的初始化返回值 |
| `static <T> T`                      | `initializeUnchecked(ConcurrentInitializer<T> initializer)`<br>`initialize` 的非受检异常版本 |
| `static <K,V> V`                    | `putIfAbsent(ConcurrentMap<K,V> map, K key, V value)`<br>检查 `ConcurrentMap` 是否包含 `key`,如果未包含,为 `ConcurrentMap` 设置 `key` : `value` |



## [TimedSemaphore](http://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/concurrent/TimedSemaphore.html)

一个专门的信号量实现，在给定的时间内提供许可，到期自动释放

该类的功能与 `java.util.concurrent.Semaphore` 有点类似，通过调用 `acquire()` 方法获取许可，但是没有提供 `release()` 方法进行许可释放，因为它会在时间到期后释放所有的许可

如果在许可已经用尽的情况下调用`acquire()`，那么当前线程会一直阻塞，直到时间到期，所有许可被释放.此时再重新获取许可.这意味着可以在**规定的时间范围内，限制给定操作的次数**

用法示例

假如存在一个通过后台线程查询数据库以进行收集统计信息的应用程序.这种后台处理不应该对数据库产生太多负载，防止影响系统的功能和性能.所以可以使用 `TimedSemaphore` 来限制该线程每秒只能发出一定数量的数据库查询

执行查询的线程类伪代码可能如下

```java
 public class StatisticsThread extends Thread {
     // 限制数据库查询次数的 semaphore 对象
     private final TimedSemaphore semaphore;
     // 创建一个线程实例并设置 semaphore 对象
     public StatisticsThread(TimedSemaphore timedSemaphore) {
         semaphore = timedSemaphore;
     }
     // 收集统计信息
     public void run() {
         try {
             while(true) {
                 semaphore.acquire();   // 限制数据库查询
                 performQuery();        // 发出查询
             }
         } catch(InterruptedException) {
             // fall through
         }
     }
     ...
 }
```

下面的代码片段显示了如何创建一个每秒只允许 10 次的 `TimedSemaphore` 并传递给统计线程 `StatisticsThread`

```java
TimedSemaphore sem = new TimedSemaphore(1, TimeUnit.SECONDS, 10);
StatisticsThread thread = new StatisticsThread(sem);
thread.start();
```

构造方法如下

```java
// timePeriod             时间段
// timeUnit        时间单位,如 SECONDS 秒,MILLISECONDS 毫秒,MINUTES 分
// limit         许可数量
public TimedSemaphore(final long timePeriod, final TimeUnit timeUnit, final int limit)

// service        延迟线程池,会使用该线程池创建释放许可的周期任务
// timePeriod             时间段
// timeUnit        时间单位,如 SECONDS 秒,MILLISECONDS 毫秒,MINUTES 分
// limit         许可数量
public TimedSemaphore(final ScheduledExecutorService service, final long timePeriod,final TimeUnit timeUnit, final int limit)
```

所以 `new TimedSemaphore(1, TimeUnit.SECOND, 10);`  含义是在 `timePeriod(1) * timeUnit(TimeUnit.SECOND) = 1秒` 的时间内只发放 `limit(10)` 个许可

在使用时需要在限制操作前调用 `acquire()`方法。 `TimedSemaphore` 会统计调用 `acquire()` 的次数，并在许可达到上限后阻塞当前线程，直到时间周期结束后释放所有许可，此时再进行许可获取

另外提供 `tryAcquire()` 方法，该方法尝试获取许可，如果获取成功，返回`true`，否者返回`false` ，使用这种方法不会造成当前线程的阻塞

同时在运行过程中你可以随时调用 `setLimit(final int limit)` 修改许可数量。 如果一个操作的次数限制需要动态调整，比如白天减少许可，晚上增加许可。 如果设置的许可小于原本许可，那么会立即生效，但是设置的许可大于原来的许可，阻塞的线程依然阻塞，直到所有许可释放，阻塞的线程被唤醒.此时按照新设置的许可进行处理.如果许可数量设置小于等于 0，那么 `acquire()` 操作不会阻塞，直接放行

当`TimedSemaphore` 不需要时，可以调用 `shutdown()` 方法，该方法会取消释放所有许可的周期任务，如果执行周期任务的 `ScheduledExecutorService` 不是外部传入，同时也会结束该线程池

# 参考资料
- [Apache Commons Lang](http://commons.apache.org/proper/commons-lang/)
- [Apache Commons Lang Javadoc ](http://commons.apache.org/proper/commons-lang/javadocs/api-release/index.html)