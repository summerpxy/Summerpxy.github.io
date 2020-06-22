---
layout: post
title: "ThreadPoolExecutor的源码分析"
date:  2020-6-21 22:34:00  
categories: java
---

`ThreadPoolExecutor`是我们在使用线程池的时候必须要了解的一个类，也是我们面试的时候经常被问到的一个问题，那么我们今天就来彻底的剖析一下线程的原理。首先我们从线程池的使用上说起，线程的使用首先是它的构造方法，我们以它最经常使用的一个构造方法来说。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler)
```

这里有6个参数，代表了`ThreadPoolExecutor`的核心参数。

- corePoolSize：核心线程数量
- maximumPoolSize：最大线程数量
- keepAliveTime：线程存活的时间
- unit：存活时间的单位
- workQueue：阻塞队列，用于存储暂时不能获取线程资源的任务
- handler：拒绝策略，当任务超越线程池的负载的时候所采取的一种策略。

`ThreadPoolExecutor`的工作原理如下：

1. 如果没有空闲的线程执行该任务且当前运行的线程数少于corePoolSize，则添加新的线程执行该任务。

2. 如果没有空闲的线程执行该任务且当前的线程数等于corePoolSize同时阻塞队列未满，则将任务入队列，而不添加新的线程。

3. 如果没有空闲的线程执行该任务且阻塞队列已满同时池中的线程数小于maximumPoolSize，则创建新的线程执行任务。

4. 如果没有空闲的线程执行该任务且阻塞队列已满同时池中的线程数等于maximumPoolSize，则根据构造函数中的handler指定的策略来拒绝新的任务。



下面就从源码的角度来看看`ThreadPoolExecutor`是如何做到的。

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
      
       //获取当前正在运行的线程数量
        int c = ctl.get();
       //满足少于核心线程的数量
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

首先通过`ctl`这个`AtomicInteger`类，获取正在运行的线程的数量。如果少于核心线程的数量，那么调用addWorker方法。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //1.检查线程池的状态，如果关闭了或者是下面的任意一种情况，返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //2.检查当前线程的数量
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //3.线程数量+1
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
           //4.新建一个worker实例  
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //5.启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

代码不长，首先是两个for循环，在第一个循环中，检查线程池的状态，是否是关闭或者是其他不符合的情况。在第二个for循环中，检查当前的线程数量是否符合规定的数量内，如果都满足的话，使用CAS让线程数量加1。如果上面的两个for循环都满足的话，那么接下来使用传递进来的Runnable也就是`firstTask`创建一个Worker对象，在这个对象中有一个thread对象，并且在一系列的检查之后，启动这个thread对象。所以关键就是看这个worker到底干了啥

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

从类的继承上来看，`Worker`首先是一个AQS类，并且实现了`runnable`接口，所以在调用start方法之后，实际上是调用到了`runWorker`这个方法。

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
               //如果线程池正处于停止状态，需要中断线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //正在运行的地方在这里
                        task.run();
                    } catch (RuntimeException x) {
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
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

对于`runWorker`方法，这里使用了一个while循环，第一次的task不为null，第二次的时候就要通过`getTask`这个方法去获取，`getTask`方法我们后面去说，**总结一下`addWorker`这个方法，在运行线程数少于核心线程数的情况下，会为每个任务创建一个`Worker`对象，并且立刻运行。**

回到`executor`这个方法

```java
int c = ctl.get();
if (workerCountOf(c) < corePoolSize)
{
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
//新加线程加入到阻塞队列中
if (isRunning(c) && workQueue.offer(command))
{
    int recheck = ctl.get();
    //二次校验
    if (! isRunning(recheck) && remove(command))
        reject(command);
    //核心线程都阵亡了，重新新建线程
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false))
    reject(command);
```

当核心线程数满了的时候，`addWorker`方法返回false。这个时候会将新加入的线程放到阻塞线程中，同时这里做了二次校验。如果阻塞线程也满了，会再次走到`addWorker`方法。

```java
for (;;)
{
    int wc = workerCountOf(c);
    if (wc >= CAPACITY ||
            wc >= (core ? corePoolSize : maximumPoolSize))
        return false;
    if (compareAndIncrementWorkerCount(c))
        break retry;
    c = ctl.get();  // Re-read ctl
    if (runStateOf(c) != rs)
        continue retry;
    // else CAS failed due to workerCount change; retry inner loop
}
```

这次`addWorker`方法传递的第二个参数是false，表示是进行`maximumPoolSize`的比较，如果线程数量少于这个数的，那么会新建`Worker`对象，启动线程执行。

如果最后线程数量超过了最大线程数量，执行reject方法。

> **线程池是如何做到缓存线程的**

缓存的实现是依赖于`runWorker`方法中的这个while循环

```java
while (task != null || (task = getTask()) != null)
```

第一次task不为null，后面获取task则依赖于`getTask`这个方法。

```java
private Runnable getTask()
{
    boolean timedOut = false; // Did the last poll() time out?

    for (;;)
    {
        int c = ctl.get();
        int rs = runStateOf(c);

        //线程池已经关闭或者正在关闭过程中
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty()))
        {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty()))
        {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try
        {
            Runnable r = timed ?
                         workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                         workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        }
        catch (InterruptedException retry)
        {
            timedOut = false;
        }
    }
}
```

这里只看核心部分，当线程池数量大于核心线程数或者允许核心线程超时的情况下，`timed`为true，poll会计时等待，`timed`为false的情况下，take会阻塞等待。所以在不允许核心线程关闭的情况下，当线程池的数量达到核心线程数的时候，`getTask`方法会永远阻塞下去不会消亡。

> **线程池的关闭shutdown与shutdownNow的区别**

`shutdown`的操作如下：

- 停止接收外部submit的任务
- 内部正在跑的任务和队列里等待的任务，会执行完
- 等到第二步完成后，才真正停止

`shutdownNow`的操作如下：

- 跟shutdown()一样，先停止接收外部提交的任务
- 忽略队列里等待的任务
- 尝试将正在跑的任务`interrupt`中断
- 返回未执行的任务列表

```java
public void shutdown()
{
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try
    {
        checkShutdownAccess();
        //设置状态为shutdown
        advanceRunState(SHUTDOWN);
        //中断闲置的线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    }
    finally
    {
        mainLock.unlock();
    }
    tryTerminate();
}


public List<Runnable> shutdownNow()
{
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try
    {
        checkShutdownAccess();
        //设置状态为stop
        advanceRunState(STOP);
        //中断工作的线程
        interruptWorkers();
        tasks = drainQueue();
    }
    finally
    {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

`shutdown`中断闲置的线程，让线程从take阻塞中跳出，重新检测状态。`shutdownNow`则直接终端工作中的线程(其实也没啥用，中断的逻辑需要自己实现方法才行，比如`beforeExecutor`方法等等)。

