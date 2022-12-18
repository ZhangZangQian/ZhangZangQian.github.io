---
title: ThreadPoolExecutor 源码阅读笔记一（JDK 1.8）
author: zhangzangqian
date: 2018-11-11 21:00:00 +0800
categories: [技术]
tags: [Java, 并发, 源码]
math: true
mermaid: true
---

线程是个很好的工具，我们知道可以通过 `new Thread().start()` 的方式去开启一个线程，但是如果纯粹的使用这种方式去创建线程，每来一个任务，创建一个线程，这种方式是存在缺陷的，尤其是在无限制的创建大量线程的时候：

- 线程生命周期的开销非常高。线程的创建和销毁不是没有代价的。根据平台的不同，实际的开销开销也有所不同，因为线程的创建过程都会需要时间，延迟请求的处理，并且需要 JVM 和操作系统提供一些辅助操作。如果请求的到达率非常高且请求的处理过程是轻量级的，例如大多数服务器应用程序就是这种情况，那么为每个请求创建一个新线程将消耗大量的系统资源。
- 资源消耗。活跃的线程会消耗系统资源，尤其是内存。如果可运行的线程数量超过可用处理器的数量，那么有些线程将闲置。大量空闲的线程会占用许多内存，给垃圾收集器带来压力，而且大量线程在竞争 CPU 资源时还会产生其他的性能开销。如果你已经拥有足够多的线程使 CPU 保持忙碌状态，那么再创建更多的线程反而会降低性能。
- 稳定性。在可创建线程数量上存在一个限制。这个限制值将随着平台的不同而不同，并且受多个因素制约，包括 JVM 的启动参数、Thread 构造函数中请求栈的大小，以及底层操作系统对线程的限制等。如果破坏了这些限制，那么很可能会抛出 OutOfMemoryError 异常，要想从这种错误中恢复过来是非常危险的，更简单的办法是通过构造程序来避免超出这种限制。
因此看来，使用线程池管理线程是多么的有必要。Java 中是通过 ThreadPoolExecutor 来实现线程池的，接下来就通过阅读源码来看下线程池从初始化到执行到结束的具体实现。

ThreadPoolExecutor 继承自 AbstractExecutorService 抽象类，提供了所有方法的具体实现；其中 AbstractExecutorService 实现自 ExecutorService 接口，它实现的是一些结构相同个的模板方法；而 ExecutorService 继承自 Executor 接口，新增了一些操作线程池状态、判断线程池状态和执行带有返回值的线程的方法；而 Executor 仅有一个 execute 方法，入参是一个 Runnable 类型的对象。

ThreadPoolExecutor 有由四个构造函数，如下图：

![ThreadPoolExecutor Constructors](/assets/img/tpe-constructors.png)

其中前三个方法都是入参中删减了一些非必须由用户指定的参数，即 ThreadFactory 和 RejectedExceptionHandler。这两个参数可以直接使用线程池默认提供的类。前三个方法都是又调用了最后一个方法，我们直接来看最后一个构造函数的实现就 OK。代码如下：

```java
/**
* 使用传入的初始化参数创建一个新的 ThreadPoolExecutor 对象。
* @param corePoolSize 线程池的核心大小，即使线程都处于空闲状态线程池也会保持这个大小，除非设置了 allowCoreThreadTimeOut
* @param maxmumPoolSize 线程在线程池中的最大数量
* @param keepAliveTime 当线程池中的线程数大于核心大小时，在终止之前等待执行新任务的最大时间
* @param unit 参数 keepAliveTime 的时间单位 
* @param workQueue 用于在任务执行之前接收任务的队列，只用于接收 execute 方法提交的 Runnable 类型的任务
* @param threadFactory 在执行器在创建线程时用到的线程工厂
* @param handler 因为队列达到容量上限时需要使用 handler 阻止执行
*/
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {
    // 参数校验
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    // 初始化相关属性
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

线程池初始化完毕之后，接下来就是使用线程池，一般情况下我们都是调用 `public void execute(Runnable command)` 方法执行一个没有返回值得线程。具体的代码实现如下：

## execute

```java
/**
* 在未来的某个时间执行传入的任务，这个任务可能会被一个新创建的线程执行，也可能被线程池中已存在的线程执行。
* 
* 如果线程任务不能被提交只执行，要么是因为执行器已经关闭，或者就是因为他的容量达到上限。这个任务会被当前的 
* RejectedExecutionHandler 处理。
*
* @param command 要被执行的任务
* @throws RejectedExecutionException 有 RejectedExecutionHandler 自行处理，如果任务不能被加入执行。
* @throws NullPointerException 如果 command 为空，抛出该异常
*/
public void execute(Runnable command) {
    // 参数校验
    if (command == null) {
        throw new NullPointerException(); 
    }
    
    /*
    * ctl 详解点击左侧大纲跳转查看解析
    */
    int c = ctl.get();
    /*
    * 如果 worker 的数量小于 corePoolSize，则创建一个核心 worker。
    */ 
    if (workerCountOf(c) < corePoolSize) {
        /*
        * 创建一个核心 worker。如果创建成功则直接返回。
        */
        if (addWorker(command, ture)) {
            return;
        }
        /* 
        * 创建失败，且再 command 不为空的情况下原因只能是如下其中之一：
        * 1. 线程池不是 RUNNING 状态
        * 2. 线程池大小超过了设置的线程池核心大小。
        */
        c = ctl.get();
    }
    /*
    * 走到这里，会出现两种情况：
    * 1. 线程池 Worker 数量小于线程池线程的核心大小，但是添加 Worker 失败。
    * 2. 线程池 Worker 数量超过线程池线程的核心大小
    * 
    * 重新校验线程池状态是否为 RUNNING 状态并且是否可以将任务入队（队列未满）
    */
    if (isRunning(c) && workQueue.offer(command)) {
        // 更新 ctl 
        int recheck = ctl.get();

        /* 
        * 在成功入队的情况下，再次校验线程池状态，如果是 RUNNING 状态则不做任何处理，否则将任务移除队列
        * 确定任务可以移除而不会被已创建 worker 消费吗？答案是不一定。从任务队列消费任务时，会判断线程池
        * 状态是否是 RUNNING 状态。如果此期间有 worker 执行完成，线程池状态会变回 RUNNING，此时任务
        * 有可能会被已存在的 worker 消费掉，不过这样逻辑也是正确的。如果线程池状态仍然是非 RUNNING 状态
        * 那就一定不会被消费掉。
        */ 
        if (!isRunning(recheck) && remove(command)) {
            /*
            * 拒绝策略看这
            */
            reject(command);
        } 
        /*
        * 如果线程池是空的，则创建线程处理任务。
        */
        else if (workerCountOf(recheck) == 0) {
            addWorker(null, false);
        }
    } 
    /* 
    * 在线程池非 RUNNING 的状态下，或者队列已满，重新尝试创建 worker，创建失败，则执行拒绝策略
    * 1. 如果是非 RUNNING 状态，在结束判断后，此时可能已有任务处理完成，可能又会变回 RUNNING 状态，
    *    因此再次交由 addWorker 去判断，如果仍然创建失败，执行拒绝策略
    * 2. 如果是队列已满，如果添加失败（原因仍然只能是非 RUNNING 状态或者线程池大小超过了线程池最大容量），
    *    非 RUNNING 的可能性，很小，执行拒绝策略也没错，如果是在任务队列已满且线程池大小超过了线程池容量
    *    最大值，也同样执行拒绝策略。
    */
    else if (!addWorker(command, false)) {
        reject(command);
    }
}
```

# addWorker

```java
private boolean addWorker(Runnable firstTask, core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        /*
        * 校验线程池状态：
        * 1. STOP、TIDYING、TERMINATED 状态不再新增 Worker
        * 2. SHUTDOWN 状态下 firstTask 不为空或者任务队列为空不再新增 Worker
        */
        if (rs >= SHUTDOWN &&
                !(rs == SHUTDOWN &&
                    firstTask == null &&
                    ! workQueue.isEmpty())) {
            return false;
        }

        // 通过乐观锁的方式去更新 workerCount
        for (;;) {
            int wc = workerCountOf(c);
            /*
            * 判断当前线程池大小，如果满足其中一条，便不再创建 worker：
            * 1. 如果当前大小大于等于线程池自身最大承受的线程数（CAPACITY）时
            * 2. 如果是核心 worker，当前 worker 数大于等于线程池核心大小时；
            *    如果是非核心 worker，当前 worker 数大于等于设置的线程池最大容量是。
            */
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize)) {
                return false;
            }
            // 通过 CAS 设置 workerCount = workerCount + 1
            if (compareAndIncrementWorkerCount(c)) {
                break retry;
            }
            // 设置失败，产生并发，更新 workerCount
            c = ctl.get();
            // 校验 runState 是否发生变化，runState 发生变化则重新校验 runState，
            // 否则重新校验并设置 workerCount
            if (runStateOf(c) != rs) {
                continue retry;
            }
        }
        
    }

    boolean workerStarted = false;
    boolean workerAdded = false;

    Worker w = null;
    try {
        // 创建 Worker 对象
        w = new Worker(firstTask);
        // 通过 Worker 创建的线程对象
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁同步
            mainLock.lock();
            try {
                // 重新校验 runState，必须在 RUNNING 状态下或者 SHUTDOWN 状态且任务为空的情况下
                // 允许创建 Worker
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    
                    // 校验线程还没有被启动
                    if (t.isAlive()) {
                        throw new IllegalThreadStateException();
                    }

                    // 将 worker 对象加入已 worker 集合
                    workers.add(w);

                    // 更新线程池 worker 的实际最大值
                    int s = worker.size();
                    if (s > largestPoolSize) {
                        largestPoolSize = s;
                    }

                    workerAdded = true;
                
                }
            } finally {
                // 解锁
                mainLock.unlock();
            }
            /*
            * 启动 worker 中的 thread。在 Worker 的构造函数中会调用线程工厂创建一个 Thread 对象，
            * 并把自己传进去（Worker 实现了 Runnable 接口）。因此其实这里启动线程就是调用 Worker 
            * 的 run 方法，run 方法里面只是调用了线程池的 runWorker 方法，然后把 worker 本身当
            * 做参数传进了 runWorker 方法。因此直接看 runWorker 的实现逻辑就可以了。
            */
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 如果 worker 启动(创建)失败，则调用启动失败的回调方法
        if (!workerStarted) {
            // 启动失败回调逻辑 
            addWorkerFailed(w);
        }
    }
    // 返回结果
    return workerStarted;
}
```

## ctl

```java
/**
* 这是线程池中最为重要且精妙的一个变量，使用这一个变量，直接控制了线程池的两个状态：
* workerCount： 线程的有效数量
* runState：线程池的运行状态，例如 RUNNING、SHUTDOWN、STOP、TINDYING 或者 TERMINATED，
*           为了允许进行有序比较，这几项值之间是有序的。
* 
* 通过调用 ctlOf 方法初始化 ctl，初始化时线程池是 RUNNING 状态的且 workerCount 为 0，最后
* 得出的值也就等于 RUNNING 的值。
*/
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
/**
* 线程池线程数上限的长度
*/
private static final int COUNT_BITS = Integer.SIZE - 3;
/**
* 线程池线程数上限，二进制值等于 11111111111111111111111111111 （29个1）
*/
private static final int CAPACITY = (1 << COUNT_BITS) - 1;

/**
* runState 的存储顺序
* 每次创建线程都会 runState 都会自增，线程执行完成之后自减一。但是不会真的触达每个状态。转换时机如下：
* RUNNING -> SHUTDOWN
*   在调用 shutdown() 方法后，也可能是隐式的调用 finalize() 时。
* (RUNNING or SHUTDOWN) -> STOP
*   在调用 shutdownNow() 后。
* SHUTDOWN -> TINDYING
*   当队列和线程池都为空时。
* STOP -> TINDYING
*   队列为空的时候。
* TINDYING -> TERMINATED
*  当 terminated() 的钩子方法执行完成后
* 
* 为什么要这么存储呢？因为我们原来说过了，ctl 一个值便控制了 workerCount 和 runState 两个状态。
* 其实就是把 ctl 的二进制值拆成了两部分，高3位表示状态，低29位表示线程池的线程数。怎么计算得出的，
* 我们可以看下 runStateOf、workerCountOf 的实现。
*/
private static final int RUNNING = -1 << CAPACITY;
private static final int SHUTDOWN = 0 << CAPACITY;
private static final int STOP = 1 << CAPACITY;
private static final int TINDYING = 2 << CAPACITY;
private static final int TERMINATED = 3 << CAPACITY;

/**
* 查询线程池当前状态。参数是 ctl 的值，举个例子来看下计算方式，假设初始化后线程池中有 10 个线程。
* 每次创建一个线程 ctl 都会自增一，那么此时 ctl 的二进制值应该是 11100000000000000000000000001010。
* 然后去跟 CAPACITY 进行位非运算后的值去进行比较。CAPACITY 值等于 11111111111111111111111111111。
* 因为 CAPACITY 的长度为 29 位，所以现将其补位，然后进行位非运算得到的值为 11100000000000000000000000000000。
* 然后进行位于运算。
* ctl       11100000000000000000000000001010
* CAPACITY  11100000000000000000000000000000
* runState  11100000000000000000000000000000 = RUNNING
* 
* 因为 CAPACITY 进行位非运算后低 29 位都为 0，所以进行位于运算也就想到与把 ctl 的低 29 位都舍去了，
* 直接取高 3 位。
*/
private static int runStateOf(int c) {
    return c & ~CAPACITY;
}

/**
* workerCountOf 直接 ctl 的值跟 CAPACITY 进行位与运算，因为 CAPACITY 补位后的值位 00011111111111111111111111111111。
* 然后进行位与也就相当于把 ctl 的高 3 位舍去，直接取后 29 位。
*/
private static int workerCouontOf(int c) {
    return c & CAPACITY;
}

/**
* 通过 runState、workerCount 进行位或运算得出 ctl 的值
*/
private int ctlOf (int rs, int wc) {
    return rs | wc;
}
```

## Worker

Worker 是 ThreadPoolExecutor 中的一个内部类，每次在新建线程池时，都是创建一个 Worker，然后 Runable 的任务传给 Worker，通过 Worker 启动一个线程，我们来看下源码，了解一下具体实现。
```java
/**
* 继承了 AbstractQueuedSynchronizer 并实现了 Runnable 。通过继承 AbstractQueuedSynchronizer 实现了一个不可重入锁。
*
*/
private final class Worker extends AbstractQueuedSynchronizer
    implements Runnable {
    
    /**
    * Worker 正在运行的线程，如果工厂创建失败，该值为空
    */
    final Thread thread;

    /**
    * 初始化时运行的任务，有可能为空。
    */
    Runnable firstTask;

    /**
    * 完成任务数的计数器
    */
    volatile long completedTasks;

    /**
    * 构造函数
    */
    Worker (Runnable firstTask) {
        setState(-1); // 禁止中断直到执行 runWorker
        this.firstTask = firstTask;
        this.thread = getThraedFactory().newThread(this);
    }

    /**
    * 直接调用线程池的 runWorker 方法
    */
    public void run() {
        runWOrker(this);
    }    

    /**
    * 判断当前锁状态
    * 0 代表解锁状态
    * 1 代表上锁状态
    */
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0,1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease() {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock() {
        acquire(1);
    }

    
    public boolean tryLock() {
        return tryAcquire(1);
    }

    public boolean unloock() {
        release(1);
    }

    public boolean isLocked() {
        return isHeldExclusively();
    }

    void intrruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t == thread) != null && !t.isInterrupted()) {
            try {
                t.intrrupt();
            } catch (SecrityException ignore) {

            }
        }
    }

}
```

## runWorker

```java
/**
* 执行 worker ，先从参数 worker 的 firstTask 中获取任务去执行，如果 firstTask 为空，则从队列中获取任务
* 然后判断线程池和当前线程状态，根据结果判断是否需要中断执行，如果不需要中断则开始执行前置回调函数，之后执行
* 任务，执行完成后无论是否任务无论是否正常结束，都执行后置回调函数，最后进行相关执行信息的记录，
* 再去循环获取任务进行执行，直到从队列中获取任务的方法 getTask() 返回 null，然后执行 worker 退出的回调函数，
* 最后退出 runWorker 方法。（一个 Worker 线程便退出了）
*/
final void runWorker (Worker w) {
    // 当前执行任务的线程对象
    Thread wt = Thread.currentThread();
    // 获取 firstTask
    Runnable task = w.firstTask;
    // 清空 firstTask
    w.fistTask = null;
    // 将当前线程设置为允许被中断
    w.unlock();
    // 异常退出标识符
    boolean completedAbruptly = true;
    try {
        /*
        * getTask() 是如何执行的？左侧大纲跳转
        */
        while (task != null && (task = getTask()) != null) {
            // 加锁当前 worker
            w.lock();
            /*
            * 如果线程池的 runState 大于等于 STOP，则确认当前线程已经被打断；
            * 如果没有，确认线程线程没有被中断。在第二种情况下需要重新校验来解决
            * 在清除中断状态时 shutdownNow 的竞争。
            */
            if ((runStateAtLeast(ctl.get(), STOP) 
                    || (Thread.interrupted() && 
                        runStateAtLeast(ctl.get(), STOP))) && 
                    !wt.isInterrupted()) {
                wt.interrupt();
            }
            try {
                // 执行前置回调函数，ThreadPoolExecutor 中的函数体是空的，ScheduledThreadPoolExecutor 中有实现
                beforeExecute(wt, task);
                
                /*
                * 开始执行任务，执行任务时无论是否有异常抛出，都将执行后置回调函数(ThreadPoolExecutor 中的函数体是空的，
                * ScheduledThreadPoolExecutor 中有实现)，然后任务完成数自增，解锁 worker，如果有异常抛出则退出循环，
                * 否则进行下一次循环。
                */
                Throwable thrown = null;
                try {
                    task.run();
                } catch(RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedCounter++;
                w.unlcok();
            }
        }
        // 设置为正常退出
        completedAbruptly = false;
    } finally {
        /*
        * 执行 worker 退出的回调函数，
        */
        processWorkerExit(w, completedAbruptly);        
    }
}
```

## addWorkerFailed

```java
/**
* Worker 线程创建回滚
*/
private void addWOrkerFailed(Worker w) {
    final ReentarntLock mainLock = this.mainLock;
    mainLock.lock()
    try {
        // 如果 worker 的 set 集合中存在 worker，则移除
        if (w != null) {
            workers.remove(w);
        }
        // workerCount 自减
        decrementWorkerCount();
        /*
        * 尝试将线程池状态设置为 TERMINATED，点这看实现：
        * http://localhost:4000/java/concurrency/source-code/2018/11/11/thread-pool-executor.html#tryterminate
        */
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

## getTask

```java
/**
* 阻塞运行或者定时等待一个任务，依赖于当前配置设定。或者在如下任意情况下必须返回 null:
* 1. worker 的数量超过了 maximumPoolSize 
* 2. 线程池的 runState 为 STOP 或者为 SHUTDOWN 并且队列是空的
* 3. worker 等待一个任务超时，超时的 worker （也就是说 allowCoreThreadTimeout 为 true 或者
*    workerCount 大于 corePoolSize）无论是在等待之前还是之后都要终止，如果队列不为空，
*    那么这个 worker 不会是线程池中的最后一个线程。
*/
private Runnable getTask() {
    // 最后一个 poll() 是否超时了的标识符
    boolean timedOut = false;

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        /*
        * 校验状态。如果满足如下讲下，workerCount 自减并返回 null:
        * 1. runState == SHUTDOWN && workerQueue.isEmpty()
        * 2. runState > SHUTDOWN
        */
        if (rs >= SHUTDOWN && (rs >= STOP || workerQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 是否定时等待
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        /*
        * 在 workerCount 大于等于一或者任务队列为空的情况下，如果如果 workerCount 大于 maximumPoolSize
        *，或者等待超时了，则尝试 clt 自减，如果是成功了则返回 null，否则表示产生了并发，需要重新校验 runState。
        */
        if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workerQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) {
                return null;
            }
            continue;
        }

        try {
            // 根据 timed 去获取任务
            Runnable r = timed ? 
                workerQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workerQueue.take();
            
            // 获取到不为空则返回任务，否则表示超时了，标记 timedOut，进入下次循环
            if (r != null) {
                return r;
            }
            timedOut = true;
        } catch (InterruptedException retry) {
            // 被打断，重试。
            timedOut = false;
        }
    }
}
```

## tryTerminate

```java
/**
* 尝试将线程转换为 TERMINATED 状态
* 
*/
final void tryTernimate() {
    for (;;) {
        int c = ctl.get();
        /*
        * 校验线程池状态，如下情况不会转换线程池状态：
        * 1. 线程池在 RUNNING || TINDYING || TERMINATED 状态 
        * 2. SHUTDOWN 状态 && 任务队列不为空
        */
        if (isRunning(c) ||
            runStateLastAt(c, TINDYING) ||
            (runStateOf(c) == SHUTDOWN && !workerQueue.isEmptry())) {
            return;
        }
        // 如果线程池不为空，打断一个空闲的（等待获取任务中的）线程，然后结束方法。
        if (workerCountOf(c) != 0) {
            /*
            * 打断空闲中的 worker 
            */
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        /*
        * 走到这一步，只能是两种情况：
        * 1. SHUTDOWN 状态 && 线程池为空 && 任务队列为空
        * 2. STOP 状态 && 线程池为空
        */
        final ReentrantLock mainLock = this.mainLock;
        // 变更 ctl 加锁
        mainLock.lock();
        try {
            // 将线程池状态设置为过渡状态 TINDYING ，然后执行 terminated 的钩子方法。
            if (ctl.compareAndSet(c, ctlOf(c, ctlOf(TINDYING, 0)))) {
                try {
                    // 执行钩子方法，该线程池只是声明了该方法，没有实现任务逻辑。
                    terminated();
                } finally {
                    // 设置为 TERMINATE 状态
                    ctl.set(ctlOf(TERMINATED, 0));
                    // 通知所有线程池为 TERMINATE 状态才执行的方法。
                    termination.singalAll();
                }
                return;
            }
        } finally {
            // 解锁 
            mainLock.unlock();
        }

        // 如果 CAS 失败，重新循环。
    }
}
```

## processWorkerExit

```java
/**
* 为临终的线程进行清理和相关簿记，只会在 worker 线程结束运行时被调用，除非是直接结束的 worker 线程。
*/
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 在 worker 异常退出的情况下，workerCount 没有调整，在这里调整 workerCount
    if (completedAbruptly) {
        decrementWorkerCount();
    }

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 记录完成任务数
        completedTaskCount += w.completedTasks;
        // 从线程集合中移除线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    /*
    * 尝试将线程池转换到 TERMIANTE 状态，实现逻辑看这
    * http://localhost:4000/java/concurrency/source-code/2018/11/11/thread-pool-executor.html#tryterminate
    */
    tryTerminate();
    
    int c = ctl.get();
    // 如果状态处于 STOP 之前（RUNNING 或者 SHUTDOWN）
    if (runStateLessThan(c, STOP)) {
        /*
        * worker 异常退出，即使正常退出的情况下，如果 workerCount 小于 corePoolSize 或者线程池为空但是
        * 任务队列不为空的情况下，需要重新创建一个线程，继续处理任务。
        */
        if (!completdAbruptly) {
            int min = allCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && !workerQueue.isEmpty()) {
                min = 1;
            }
            if (workerCountOf(c) >= min) {
                return;
            }
        }
        /*
        * 创建线程，实现逻辑看这
        *  http://localhost:4000/java/concurrency/source-code/2018/11/11/thread-pool-executor.html#addworker
        */
        addWorker(null, false);
    }
}
```

## reject

```java
/**
* 拒绝任务逻辑，直接调用 RejectedExecutionHandler 的拒绝方法。
*/
final void reject(Runnable command) {
    handler.rejectExecption(command, this);
}
```

## interruptIdleWorkers

```java
/**
* 中断空闲线程，通过尝试获取锁来判断线程是否空闲。thr
*/
private void interruptIdleWorkers(boolean onlyOne) {
    // 获取主锁并加锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 循环线程集合
        for (Worker w : workers) {
            Thread t = w.thread;
            // 如果线程没有被中断且可以获得锁，则中断线程，并释放锁
            if (!t.isInterruptd() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch(SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            // 如果 onlyOne 为 true 则退出循环
            if (onlyOne) {
                break;
            }
        }
    } finally {
        // 最后释放主锁
        mainLock.unlock();
    }
}
```