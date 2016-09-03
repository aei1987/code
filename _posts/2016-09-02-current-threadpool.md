---
layout: post
title:  "Java线程池详解"
date:  2016-09-03
categories: [并发编程]
---

Java线程池详解

**目录**

* TOC
{:toc}

---
### 1.概述
好的软件设计不建议手动创建和销毁线程。线程的创建和销毁是非常耗 CPU 和内存的，因为这需要 JVM 和操作系统的参与。64位 JVM 默认线程栈是大小1 MB。这就是为什么说在请求频繁时为每个小的请求创建线程是一种资源的浪费。线程池可以根据创建时选择的策略自动处理线程的生命周期。重点在于：在资源（如内存、CPU）充足的情况下，线程池没有明显的优势，否则没有线程池将导致服务器奔溃。有很多的理由可以解释为什么没有更多的资源。例如，在拒绝服务（denial-of-service）攻击时会引起的许多线程并行执行，从而导致线程饥饿（thread starvation）。除此之外，手动执行线程时，可能会因为异常导致线程死亡，程序员必须记得处理这种异常情况。

即使在你的应用中没有显式地使用线程池，但是像 Tomcat、Undertow这样的web服务器，都大量使用了线程池。所以了解线程池是如何工作的，怎样调整，对系统性能优化非常有帮助。

### 2线程池的基础知识
#### 2.1线程池的构造方法
Java提供了Executor框架提供了一套完善的多线程编程api，包括对线程生命周期，统计信息，程序管理机制，性能监控等功能。

``` java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

- `corePoolSize`：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者`prestartCoreThread`()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；

- `maximumPoolSize`：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；
- `unit`：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：
- `workQueue`：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：
```
ArrayBlockingQueue;//
LinkedBlockingQueue;//链式结构的阻塞队列，作为Executors.newFixThreadPool()生成线程池的默认队列，入列出列快，但是同时会产生Node对象
SynchronousQueue;//同步队列，没有存储能力，作为Executors.newCacheThreadPool()线程池的默认队列
```

- `threadFactory`：线程工厂，主要用来创建线程；
- `handler`：表示当拒绝处理任务时的策略，有以下四种取值：

```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```
####2.2如何创建一个线程池
我们可以通过构造函数初始化一个自己想要的线程池，但是一般不推荐这么做，除非Executors工厂方法构造的线程池不能满足我们的要求。
下面介绍一下Executos的api创建的线程池的特点：
``` java

public class Executors {

   /**
   *创建一个固定大小的线程池，每当提交一个任务就创建一个线程，知道达到线程池的最大数量，这时线程规模不在变化
   （如果某个线程发生了未知Exception，那么线程池将补充一个线程）
   **/
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    /**
    *创建一个可缓存的线程池，如果线程当前规模超过处理的需求，那么回收空闲线程（默认60s回收一次），
    当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制
    **/
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }


    /**
    *是一个单线程的Executors，如果这个线程意外结束，则会创建另一个线程替代，newSingleThreadExecutor能保证任
    务串行执行，例如（FIFO，LIFO，优先级）
    *Netty中的NioEventLoop就是一个单线程的Executors，保证业务逻辑串行无锁执行
    **/
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    /**
    *创建定时执行的线程池，功能类似Timer，建议使用newSingleThreadScheduledExecutor来取代Timer（
    1.Timer异常处理糟糕；2.Timer只会创建一个线程，不能保证定时的精确性）
    **/
    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }

}
```
#### 2.3对比newFixedThreadPool&newCachedThreadPool
了解newFixedThreadPool&newCachedThreadPool的区别有助于我们更好的理解和使用线程池，那么这两者到底有什么不同呢？
默认newCachedThreadPool会创建一个corePoolSize为0，maximumPoolSize为Integer.maxValue，队列为SynchronousQueue的线程池，默认60s回收一次空闲线程。这意味着newCachedThreadPool创建的线程池能够自动适应，有效的避免了由于任务依赖而造成的饥饿问题。线程编程实战书上推荐为默认的选择。
当然，newCachedThreadPool也会有缺点，当程序处理速度跟不上任务进入队列的速度时候会造成大量的线程，可能会耗完程序CPU。
这个时候newFixedThreadPool使我们另外一个选择，默认会创建一个corePoolSize为nThread，maximumPoolSize也为nThread，线程队列为LinkedBlockingQueue的线程池，`队列大小为Integer.maxValue，也就是说是一个无界队列`，这个很危险，当程序处理速度跟不上任务进入队列的速度时候会造成大量的线程，会有大量的对象积压在任务队列里面，直到耗完程序内存。
所以，如果我们的线程池有可能会面临高并发访问的场景，我们在创建的时候一定要小心，一定要用构造好合适的初始化值。

这里的合适的初始化值主要指，线程的corePoolSize，maximumPoolSize，workerQueue&handler，这里的workerQueue和handler尤为重要，`最好不要使用无界队列。使用有界队列最坏的情况可能会造成部分任务无法响应，但至少能保证大部分的任务正常执行。而且我们还可以定制自己的handler（饱和策略）来处理这些处理不了的任务。`

#### 2.4线程池的饱和策略

### 3.线程池大小


### 4.高能预警
### 5.线程池的应用与扩展

### 6.一些问题
#### 6.1什么时候线程池能够最有效的提升性能？
>当大量任务相互独立且同构时才能体现出程序的工作负载分配到多个任务带来的真正性能提升。所以如果任务队列里面的任务有相互依赖，或者存在不同的任务，我们需要谨慎的处理。
>netty中的线程池就会同时处理IO任务和少量的定时任务，我们可以通过分配适当的执行比例ratio来优化。
>当线程中的任务出现依赖的情况，我们尤其要适当的增大线程池的大小，以防止任务阻塞，造成的线程饥饿。

#### 6.2 如何处理线程池中部分运行时间较长的任务？
>如果线程池中出现运行时间较长的任务，即使不出现死锁，线程的响应速度也会变得非常糟糕。执行时间较长的任务不仅会造成线程池的阻塞，甚至还会增加执行时间较短的任务的服务时间。
>有一项技术可以缓存执行时间较长的任务的影响，即限定任务等待资源的时间，而不要无限制的等待。如果等待时间超时，那么可以把任务标志为失败，然后终止任务或者将任务重新放入队列。
>当然如果线程池中总是充满被阻塞的任务，那么也很有可能说明线程池的太小。
### 7.总结

延伸阅读：