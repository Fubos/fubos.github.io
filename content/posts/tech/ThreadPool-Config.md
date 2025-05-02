---
title: "[Tech]IM项目中的统一线程池管理"
date: 2025-02-13T16:42:25+08:00
lastmod: 2025-02-13T16:42:25+08:00
author: ["Fubos"]

categories:
- tech

tags:
- 线程池管理

keywords:
- 线程池管理

description: "" # 文章描述，与搜索优化相关
summary: "" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: false # 该页面可以被搜索到
showbreadcrumbs: true #顶部显示当前路径
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

在项目实践中，会用到很多池化的技术，比如线程池、数据库连接池、HTTP连接池等，这种池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。

其中，线程池的定义是管理一系列的资源池，并提供管理和限制线程资源的方式。每个线程池还维护一些基本统计信息，比如已完成的任务数量。

在《Java并发编程的艺术》中，总结了线程池的好处：
- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。 
- 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。 
- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

当频繁创建、销毁线程和线程池，会给系统带来额外的开销，未经池化及统一管理的线程，会导致系统内线程数上限不可控。因此，为解决上述问题，可增加统一线程池配置，替换掉自建线程和线程池。

在线程池实现中，很多都基于Executor框架，Executor 框架是 Java5 之后引进的，在 Java 5 之后，通过 Executor 来启动线程比使用 Thread 的 start 方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免 this 逃逸问题(this 逃逸是指在构造函数返回之前其他线程就持有该对象的引用，调用尚未构造完全的对象的方法可能引发令人疑惑的错误)。

Executor 框架不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略等，Executor 框架让并发编程变得更加简单。Executor 框架结构主要由三大部分组成：
- 1、任务(Runnable /Callable),执行任务需要实现的 Runnable 接口 或 Callable接口。Runnable 接口或 Callable 接口实现类都可以被 ThreadPoolExecutor 或 ScheduledThreadPoolExecutor 执行。
- 2、任务的执行(Executor),包括任务执行机制的核心接口 Executor ，以及继承自 Executor 接口的 ExecutorService 接口。ThreadPoolExecutor 和 ScheduledThreadPoolExecutor 这两个关键类实现了 ExecutorService 接口。
- 3、异步计算的结果(Future),Future 接口以及 Future 接口的实现类 FutureTask 类都可以代表异步计算的结果。

因此得到Executor框架流程图：

![Executor框架流程图](/posts_imgs/Executor-flow.png)

1.主线程首先要创建实现 Runnable 或者 Callable 接口的任务对象。

2.把创建完成的实现 Runnable/Callable接口的 对象直接交给 ExecutorService 执行: ExecutorService.execute（Runnable command））或者也可以把 Runnable 对象或Callable 对象提交给 ExecutorService 执行（ExecutorService.submit（Runnable task）或 ExecutorService.submit（Callable <T> task））。

3.如果执行 ExecutorService.submit（…），ExecutorService 将返回一个实现Future接口的对象（submit()会返回一个 FutureTask 对象）。由于 FutureTask 实现了 Runnable，我们也可以创建 FutureTask，然后直接交给 ExecutorService 执行。

4.最后，主线程可以执行 FutureTask.get()方法来等待任务执行完成。主线程也可以执行 FutureTask.cancel（boolean mayInterruptIfRunning）来取消此任务的执行。


## 自建线程池

{{< collapse "自建线程池" >}}
```java
@Configuration
@EnableAsync
public class ThreadPoolConfig implement AsyncConfigurer {
    /**
     * 项目共用线程池
     */
    public static final String MALLCHAT_EXECUTOR = "mallchatExecutor";
    /**
     * websocket通信线程池
     */
    public static final String WS_EXECUTOR = "websocketExecutor";

    @Override
    public Executor getAsyncExecutor() {
        //返回项目主线程池
        return mallchatExecutor();
    }

    @Bean(MALLCHAT_EXECUTOR)
    @Primary
    public ThreadPoolTaskExecutor mallchatExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //线程池优雅停机的关键，等任务执行完再停机，保证任务不丢失。
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(200);
        //线程前缀
        executor.setThreadNamePrefix("mallchat-executor-");
        //拒绝策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());//满了调用线程执行，认为重要任务
        //使用自己修改后的线程工厂
        executor.setThreadFactory(new MyThreadFactory(executor));
        executor.initialize();
        return executor;
    }
}
```
{{< /collapse >}}
上面的代码创建了一个统一线程池，并通过实现AsyncConfigurer设置了@Async注解也使用我们统一的线程池，方便统一管理。

我们的线程池没有使用Executors快速创建，这是因为Executors创建的线程池用的是无界队列，有oom的风险。

通过设置线程前缀便于排查cpu占用，死锁问题或其他bug时，可以根据线程名看出是业务问题还是底层框架问题。

## 优雅停机

当项目关闭时，需要通过jvm的shutdownHook回调线程池，等队列中的任务执行完后再停机，保证任务不丢失。

{{< collapse "通过shutdownHook回调实现优雅停机" >}}
```java
import java.util.concurrent.Callable;

//省略import
public class ThreadUtil {
    //懒汉式单例创建线程池：用于执行定时、顺序任务
    static class SeqOrScheduledTargetThreadPoolLazyHolder {
        //线程池：用于定时任务、顺序排队执行任务
        static final ScheduledThreadPoolExecutor EXECUTOR =
                new ScheduledThreadPoolExecutor(1,
                        new CustomThreadFactory("seq"));

        static {
            //注册JVM关闭时的钩子函数
            Runtime.getRuntime().addShutdownHook(
                    new ShutdownHookThread("定时和顺序任务线程池",
                            new Callable<Void>() {
                                @Override
                                public Void call() throws Exception {
                                    //优雅地关闭线程池
                                    shutdownThreadPoolGracefully(EXECUTOR);
                                    return null;
                                }
                            }));
        }
    }
    //省略不相关代码
}
```
{{< /collapse >}}


由于shutdownHook会回调spring容器，因此我们实现spring的DisposableBean的destory方法可以达到同样的效果，在里面调用executor.shutdown()并等待线程池执行完毕。


{{< collapse "调用executor.shutdown()实现优雅停机" >}}
```java
public class TestService{
    public static void main(String[] args) {
        //创建固定 3 个线程的线程池，测试使用，工作中推荐ThreadPoolExecutor
        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        //向线程池提交 10 个任务
        for (int i = 1; i <= 10; i++) {
            final int index = i;
            threadPool.submit(() -> {
                System.out.println("正在执行任务 " + index);
                //休眠 3 秒,模拟任务执行
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        //休眠 4 秒
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        //关闭线程池
        threadPool.shutdown();
    }
}
```
{{< /collapse >}}

在上述代码中，构造了一个包含固定3线程数的线程池，循环提交10个任务，每个任务休眠3秒，但主程序休眠4秒后，会调用shutdown方法。理论上在第二个线程时间循环中，线程池就已经被停止了，3s+4s最多执行完6个任务，但是我们在程序运行过程中丝毫感受不到线程何时被停止了，这个过程就是优雅停机的作用。



{{< collapse "调用shutdownNow()实现停机" >}}
```java
/**
 * 尝试停止所有正在执行的任务，停止处理等待的任务，
 * 并返回等待处理的任务列表。
 *
 * @return 从未开始执行的任务列表
 */
public List<Runnable> shutdownNow() {
    List<Runnable> tasks; // 用于存储未执行的任务的列表
    final ReentrantLock mainLock = this.mainLock; // ThreadPoolExecutor的主锁
    mainLock.lock(); // 加锁以确保独占访问
    try {
        checkShutdownAccess(); // 检查是否有关闭的权限
        advanceRunState(STOP); // 将执行器的状态更新为STOP
        interruptWorkers(); // 中断所有工作线程
        tasks = drainQueue(); // 清空队列并将结果放入任务列表中
    } finally {
        mainLock.unlock(); // 无论try块如何退出都要释放锁
    }
    tryTerminate(); // 如果条件允许，尝试终止执行器
    
    return tasks; // 返回队列中未被执行的任务列表
}
```
{{< /collapse >}}

与shutdown不同的是shutdownNow会尝试终止所有的正在执行的任务，清空队列，停止失败会抛出异常，并且返回未被执行的任务列表。

结合**shutdown()+awaitTermination(long timeout, TimeUnit unit)**实现优雅停机：

awaitTermination(long timeout, TimeUnit unit)是可以允许我们在调用shutdown方法后，再设置一个等待时间，如设置为5秒，则表示shutdown后5秒内线程池彻底终止，返回true，否则返回false;

这种方式里，我们将shutdown()结合awaitTermination(long timeout, TimeUnit unit)方法去使用，注意在调用 awaitTermination() 方法时，应该设置合理的超时时间，以避免程序长时间阻塞而导致性能问题，而且由于这个方法在超时后也会抛出异常，因此，我们在使用的时候要捕获并处理异常。

## 线程池使用
对于不同的线程池，通过设置beanName区分，

{{< collapse "设置beanName区分不同线程池" >}}
```java
    /**
     * 项目共用线程池
     */
    public static final String MALLCHAT_EXECUTOR = "mallchatExecutor";
    /**
     * websocket通信线程池
     */
    public static final String WS_EXECUTOR = "websocketExecutor";
    
    @Bean(MALLCHAT_EXECUTOR)
    @Primary
    public ThreadPoolTaskExecutor mallchatExecutor() {
        //具体的设置
    }
    
    @Bean(WS_EXECUTOR)
    public ThreadPoolTaskExecutor websocketExecutor() {
        //具体的设置
    }
```
{{< /collapse >}}


当业务需要用时，可以通过    

```java
@Qualifier(ThreadPoolConfig.MALLCHAT_EXECUTOR)
//指定特定的线程执行器
private ThreadPoolTaskExecutor threadPoolTaskExecutor;
threadPoolTaskExecutor.execute()->webSocketService.scanSuccess(eventKey);
```

或者直接在方法上加异步注解@Async实现线程池使用。

```java
    @Async
    @TransactionalEventListener(classes = GroupMemberAddEvent.class,fallbackExecution = true)
    public void sendAddMsg(GroupMemberAddEvent event){
        List<GroupMember> memberList = event.getMemberList();
        RoomGroup roomGroup = event.getRoomGroup();
        Long inviteUid = event.getInviteUid();
        User user = userInfoCache.get(inviteUid);
        List<Long> uidList = memberList.stream().map(GroupMember::getUid).collect(Collectors.toList());
        //组装系统消息，用于通知群成员
        ChatMessageReq chatMessageReq = RoomAdapter.buildGroupAddMessage(roomGroup, user, userInfoCache.getBatch(uidList));
        chatService.sendMsg(chatMessageReq,User.UID_SYSTEM);
    }
```
## 异常捕获
普通的异常被我们捕获之后，日志会记录相关的异常提示，但是没有控制台相关的异常信息。如果一个异常未被捕获，从线程中抛出来了，JVM会回调一个dispatchUncaughtException。

```java
 /**
 * Dispatch an uncaught exception to the handler. This method is
 * intended to be called only by the JVM.
 */
private void dispatchUncaughtException(Throwable e) {
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

这个方法在Thread类中，会进行默认的异常处理，也就是获取一个默认的异常处理器，这个异常处理器是ThreadGroup实现的异常捕获方法。

为了捕获线程异常，我们给线程添加一个异常捕获处理器，当出现异常时，将异常转化为error日志。

Thread有两个属性，一个是实例对象，一个是类静态变量，都可以设置异常捕获，区别在于一个生效的范围是单个thread对象，一个生效的范围是全局的thread。

```java
// null unless explicitly set-实例对象-单个线程对象
private volatile UncaughtExceptionHandler uncaughtExceptionHandler;

// null unless explicitly set-类静态变量-全局所有线程
private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
```

一般选择为每个Thread实例都加一个异常捕获，也就是只对自己开发的线程负责。

```java
Thread thread = new Thread(() -> {
    log.info("111");
    throw new RuntimeException("运行时异常了");
});
Thread.UncaughtExceptionHandler uncaughtExceptionHandler =(t,e)->{
    log.error("Exception in thread ",e);
};
thread.setUncaughtExceptionHandler(uncaughtExceptionHandler);
thread.start();
```

## 线程池异常捕获
使用线程池的ThreadFactory，创建线程的工厂，创建线程的时候给线程添加异常捕获。

比如在ip解析线程池中，直接在工厂中添加一个异常捕获处理器，在创建thread时，将这个异常捕获处理器赋值给thread。

```java
private static ExecutorService executor = new ThreadPoolExecutor(1, 1,
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>(500), 
    new NamedThreadFactory("refresh-ipDetail",null, false,
                           new MyUncaughtExceptionHandler()));

public class MyUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler{
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        log.error("Exception in thread:" , e);
    }
}
```

由于我们还有两个线程池，用到了Spring线程池，由于Spring的封装，想给线程工厂设置一个捕获器很困难。

比如在websocket线程池中，

```java
 @Bean(WS_EXECUTOR)
public ThreadPoolTaskExecutor websocketExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(16);
    executor.setMaxPoolSize(16);
    executor.setQueueCapacity(1000);//支持同时推送1000人
    executor.setThreadNamePrefix("websocket-executor-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());//满了直接丢弃，默认为不重要消息推送
    executor.initialize();
    return executor;
}
```

查看源码可知，Spring的线程池自己实现了ThreadFactory，我们并没有机会去设置一个自定义的线程捕获器。

而它的抽象类ExecutorConfigurationSupport将自己的值赋值给了线程工厂，提供了一个解耦机会。

```java
package org.springframework.scheduling.concurrent;
public abstract class ExecutorConfigurationSupport extends CustomizableThreadFactory implements BeanNameAware, InitializingBean, DisposableBean {
    protected final Log logger = LogFactory.getLog(this.getClass());
    private ThreadFactory threadFactory = this;
    private boolean threadNamePrefixSet = false;
    ...
```

但是如果我们将这个线程工厂换了，那么它的线程创建方法都会失效，线程名，优先级都需要我们一起设置了，这加大很多无关的工作，而我们只想扩展一个线程捕获。

因此，我们想到了装饰器模式，装饰器模式不会改变原有的功能，只是在功能的前后做一个扩展点，因此适合我们的改动。

首先写一个自己的线程工厂，再把spring的线程工厂传进来，调用它的线程创建后，再扩展设置我们自己的异常捕获。

```java
public class MyThreadFactory implements ThreadFactory {

    private ThreadFactory original;

    @Override
    public Thread newThread(Runnable r) {
        //original是组合方式创建线程
        Thread thread = original.newThread(r);//执行spring线程自己的创建逻辑
        //额外装饰我们需要的创建逻辑
        thread.setUncaughtExceptionHandler(new MyUncaughtExceptionHandler());//异常捕获
        return thread;
    }
}
```

然后替换spring的线程池的线程工厂

```java
    @Bean(WS_EXECUTOR)
    public ThreadPoolTaskExecutor websocketExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //线程池优雅停机的关键，等任务执行完再停机，保证任务不丢失。
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setCorePoolSize(16);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(1000);
        //线程前缀
        executor.setThreadNamePrefix("websocket-executor-");
        //拒绝策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());//满了直接丢弃
        //使用自己修改后的线程工厂
        executor.setThreadFactory(new MyThreadFactory(executor));
        executor.initialize();
        return executor;
    }
```

通过上述配置，我们就能够实现统一的线程池管理和提高线程池效率。

---END---

> 参考文献
> https://www.yuque.com/snab/mallchat/oo87m9gndo4yx2sw
> https://javaguide.cn/java/concurrent/java-thread-pool-summary.html
