

> 本篇主要记录和总结工作中不同场景需要的线程池实现

## 简介

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后再创建线程后自动启动这些任务。

## 说明

但不同的业务场景，线程池配置需要不一致。合理配置线程池，可以如虎添翼，更好的提高性能。

本篇中主要实现两种线程池，用于运用不同的场景需求；

- 适合于业务处理需要远程资源的场景
- 比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部）

## 原理

### 适合于CPU密集型应用

ThreadPoolExecutor: coreThread -> queue -> maxThread -> reject

优先offer到queue，queue满后再扩充线程到maxThread，如果已经到了maxThread就reject

场景优势：比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部）

### 适合于业务处理需要远程资源

StandardThreadExecutor：coreThread -> maxThread -> queue -> reject

优先扩充线程到maxThread，再offer到queue，如果满了就reject

场景优势：比较适合于业务处理需要远程资源的场景

### LinkedTransferQueue

LinkedTransferQueue 能保证更高性能，相比与LinkedBlockingQueue有明显提升。

采用一种预占模式。意思就是消费者线程取元素时，如果队列不为空，则直接取走数据，若队列为空，那就生成一个节点（节点元素为null）入队，然后消费者线程被等待在这个节点上，后面生产者线程入队时发现有一个元素为null的节点，生产者线程就不入队了，直接就将元素填充到该节点，并唤醒该节点等待的线程，被唤醒的消费者线程取走元素，从调用的方法返回。我们称这种节点操作为“匹配”方式。

缺点:没有队列长度控制，需要在外层协助控制

## 实现

### 定义properties配置属性类

```
package com.thread.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * Description:
 * <br/>
 * ThreadPoolProperties
 *
 * @author feiliang
 */
@ConfigurationProperties(prefix = ThreadPoolProperties.THREAD_PREFIX)
public class ThreadPoolProperties {
    public final static String THREAD_PREFIX = "thread.config";

    /**
     * 是否开启标准线程池
     */
    private boolean standardEnabled = Boolean.FALSE;

    /**
     * 默认最小核心线程数
     */
    private int minMumPoolSize = 20;
    /**
     * 默认最大核心线程数
     */
    private int maxiMumPoolSize = 200;
    /**
     * 队列大小
     */
    private int queueCapacity = 200;

    /**
     * keepAliveTime时间
     */
    private int keepAliveTime = 60 * 1000;

    public static String getThreadPrefix() {
        return THREAD_PREFIX;
    }

    public boolean isStandardEnabled() {
        return standardEnabled;
    }

    public void setStandardEnabled(boolean standardEnabled) {
        this.standardEnabled = standardEnabled;
    }

    public int getMinMumPoolSize() {
        return minMumPoolSize;
    }

    public void setMinMumPoolSize(int minMumPoolSize) {
        this.minMumPoolSize = minMumPoolSize;
    }

    public int getMaxiMumPoolSize() {
        return maxiMumPoolSize;
    }

    public void setMaxiMumPoolSize(int maxiMumPoolSize) {
        this.maxiMumPoolSize = maxiMumPoolSize;
    }

    public int getQueueCapacity() {
        return queueCapacity;
    }

    public void setQueueCapacity(int queueCapacity) {
        this.queueCapacity = queueCapacity;
    }

    public int getKeepAliveTime() {
        return keepAliveTime;
    }

    public void setKeepAliveTime(int keepAliveTime) {
        this.keepAliveTime = keepAliveTime;
    }
}

```

### 线程工厂命名声明

```
package com.thread.config;

import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Description: 线程工厂命名声明
 * <br/>
 * NamedThreadFactory
 *
 * @author feiliang
 */
public class NamedThreadFactory implements ThreadFactory {
    private static final AtomicInteger POOL_SEQ = new AtomicInteger(1);
    private final AtomicInteger mThreadNum = new AtomicInteger(1);
    private final String mPrefix;
    private final boolean daemon;
    private final ThreadGroup group;

    public NamedThreadFactory() {
        this("standard-pool-" + POOL_SEQ.getAndIncrement(), false);
    }

    public NamedThreadFactory(String prefix) {
        this(prefix, false);
    }

    public NamedThreadFactory(String prefix, boolean daemon) {
        this.mPrefix = prefix + "-thread-";
        this.daemon = daemon;
        SecurityManager s = System.getSecurityManager();
        this.group = (s == null) ? Thread.currentThread().getThreadGroup() : s.getThreadGroup();
    }

    @Override
    public Thread newThread(Runnable r) {
        String name = mPrefix + mThreadNum.getAndIncrement();
        Thread t = new Thread(group, r, name, 0);
        t.setDaemon(daemon);
        return t;
    }
}

```


### 线程池配置

```
package com.thread;

import com.thread.config.NamedThreadFactory;
import com.thread.config.ThreadPoolProperties;

import java.util.concurrent.LinkedTransferQueue;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Description:
 * * The Standard Thread Executor
 * * <p>
 * * ThreadPoolExecutor: coreThread -> queue -> maxThread -> reject:
 * * 优先offer到queue，queue满后再扩充线程到maxThread，如果已经到了maxThread就reject
 * * 场景优势：比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部: memory copy or compute）
 * * <p>
 * * StandardThreadExecutor：coreThread -> maxThread -> queue -> reject
 * * 优先扩充线程到maxThread，再offer到queue，如果满了就reject
 * * 场景优势：比较适合于业务处理需要远程资源的场景
 * @author feiliang
 */
public class StandardThreadExecutor extends ThreadPoolExecutor {
    /**
     * 提交处理中的任务数
     */
    private AtomicInteger submittedTasksCount;
    /**
     * 最大并发任务限制
     */
    private int maxSubmittedTaskCount;

    public StandardThreadExecutor(ThreadPoolProperties threadPoolProperties) {
        // CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
        super(threadPoolProperties.getMinMumPoolSize(), threadPoolProperties.getMaxiMumPoolSize(), threadPoolProperties.getKeepAliveTime(),
                TimeUnit.MILLISECONDS, new StandardExecutorQueue(),
                new NamedThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy());
        ((StandardExecutorQueue) getQueue()).setStandardThreadExecutor(this);
        submittedTasksCount = new AtomicInteger(0);

        // 最大并发任务限制： 队列buffer数 + 最大线程数
        maxSubmittedTaskCount = threadPoolProperties.getQueueCapacity() + threadPoolProperties.getMaxiMumPoolSize();
    }

    @Override
    public void execute(Runnable command) {
        int count = submittedTasksCount.incrementAndGet();

        // 超过最大的并发任务限制，进行 reject
        // 依赖的LinkedTransferQueue没有长度限制，因此这里进行控制
        if (count > maxSubmittedTaskCount) {
            submittedTasksCount.decrementAndGet();
            getRejectedExecutionHandler().rejectedExecution(command, this);
        }

        try {
            super.execute(command);
        } catch (Exception rx) {
            if (!((StandardExecutorQueue) getQueue()).force(command)) {
                submittedTasksCount.decrementAndGet();
                getRejectedExecutionHandler().rejectedExecution(command, this);
            }
        }
    }

    public int getSubmittedTasksCount() {
        return this.submittedTasksCount.get();
    }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        submittedTasksCount.decrementAndGet();
    }
}


/**
 * Description:
 * LinkedTransferQueue 能保证更高性能，相比与LinkedBlockingQueue有明显提升。
 * 采用一种预占模式。意思就是消费者线程取元素时，如果队列不为空，则直接取走数据，若队列为空，那就生成一个节点（节点元素为null）入队，然后消费者线程被等待在这个节点上，
 * 后面生产者线程入队时发现有一个元素为null的节点，生产者线程就不入队了，直接就将元素填充到该节点，并唤醒该节点等待的线程，被唤醒的消费者线程取走元素，从调用的方法返回。我们称这种节点操作为“匹配”方式。
 * <p>
 * 缺点:没有队列长度控制，需要在外层协助控制
 */
class StandardExecutorQueue extends LinkedTransferQueue<Runnable> {

    private StandardThreadExecutor threadPoolExecutor;

    StandardExecutorQueue() {
        super();
    }

    void setStandardThreadExecutor(StandardThreadExecutor threadPoolExecutor) {
        this.threadPoolExecutor = threadPoolExecutor;
    }

    boolean force(Runnable o) {
        if (threadPoolExecutor.isShutdown()) {
            throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
        }

        return super.offer(o);
    }

    @Override
    public boolean offer(Runnable o) {
        int poolSize = threadPoolExecutor.getPoolSize();

        // we are maxed out on threads, simply queue the object
        if (poolSize == threadPoolExecutor.getMaximumPoolSize()) {
            return super.offer(o);
        }

        // 任务少于当前线程数，不需要新开线程，任务直接放入队列
        if (threadPoolExecutor.getSubmittedTasksCount() <= poolSize) {
            return super.offer(o);
        }

        // 任务大于当前线程数，且当前线程数少于最大线程数，新开线程处理任务，不放入队列了
        if (poolSize < threadPoolExecutor.getMaximumPoolSize()) {
            return false;
        }

        // 任务放到队列中
        return super.offer(o);
    }
}
```

### spring boot配置类

```
package com.thread.config;

import com.thread.StandardThreadExecutor;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * Description:
 * <br/>
 * ThreadPoolAutoConfig
 *
 * @author feiliang
 */
@Configuration
@EnableConfigurationProperties(value = {ThreadPoolProperties.class})
public class ThreadPoolAutoConfig {

    /**
     * 比较适合于业务处理需要远程资源的场景
     *
     * @param threadPoolProperties
     * @return
     */
    @Bean
    @ConditionalOnProperty(prefix = ThreadPoolProperties.THREAD_PREFIX, name = "standard-enabled", havingValue = "true")
    public StandardThreadExecutor standardThreadExecutor(ThreadPoolProperties threadPoolProperties) {
        return new StandardThreadExecutor(threadPoolProperties);
    }

    /**
     * 比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部: memory copy or compute）
     *
     * @param threadPoolProperties 线程池配置
     * @return
     */
    @Bean
    @ConditionalOnMissingBean(ThreadPoolExecutor.class)
    public ThreadPoolExecutor threadPoolExecutor(ThreadPoolProperties threadPoolProperties) {
        return new ThreadPoolExecutor(threadPoolProperties.getMinMumPoolSize(), threadPoolProperties.getMaxiMumPoolSize(), threadPoolProperties.getKeepAliveTime(), TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(threadPoolProperties.getQueueCapacity()), Executors.defaultThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy());
    }
}

```

### 配置在resources/META-INF/spring.factories

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.thread.config.ThreadPoolAutoConfig
```

## 运用

### 引用依赖

```
<dependency>
    <groupId>org.example</groupId>
    <artifactId>threadpool-starter</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### 配置参数

```
thread:
  config:
    standard-enabled: true
    min-mum-pool-size: 20
    maxi-mum-pool-size: 200
    queue-capacity: 200
```

### 场景应用

```
@Autowired
private StandardThreadExecutor standardThreadExecutor;
@Autowired
private ThreadPoolExecutor threadPoolExecutor;

standardThreadExecutor.execute(() -> {
    String name = Thread.currentThread().getName();
    System.out.println(name);
    if (!name.startsWith("standard")) {
        System.out.println("-----------------" + name);
    }
});
```