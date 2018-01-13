title: Java 源码研究之线程池
author: ghthou
tags:
  - Java
  - 线程池
categories:
  - Java
date: 2018-01-13 13:09:00
---
本文是在观看 [深入分析java线程池的实现原理](http://www.jianshu.com/p/87bff5cc8d8c) 后,对其中讲述的方法虽然了解其功能及大致步骤,但是对其中具体实现依然不太明白,所以查看其中的源码,并对源码的操作步骤进行说明.至于方法功能,使用等等.请参考上面的文章

主要研究的类为

- java.util.concurrent.ThreadPoolExecutor
- java.util.concurrent.FutureTask

源码版本为`jdk1.8.0_91`

### ThreadPoolExecutor
首先需要了解 `AtomicInteger` 类型的 `ctl` 变量,这个变量以32位二进制的方式描述俩种信息

1. 前三位表示线程池的状态
    1. RUNNING(111) 接受新任务并处理排队的任务
    2. SHUTDOWN(000) 不接受新任务，而是处理排队的任务
    3. STOP(011) 不接受新任务，不处理排队的任务和中断进行中的任务
    4. TIDYING(100) 所有任务已终止，workerCount 为零， 线程过渡到状态 TIDYING 将运行 terminate() 钩子方法
    5. TERMINATED(110) terminated() 已完成
2. 后二十九位表示当前工作的线程数(`[0,(2^29)-1]`)

任务执行有俩种方法,其中 `execute` 方法由 `ThreadPoolExecutor` 提供
`submit` 方法继承自 `AbstractExecutorService` 的实现

`java.util.concurrent.ThreadPoolExecutor#execute(Runnable command)`
```java
public void execute(Runnable command) {
    // 参数校验
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // workerCountOf 获取当前工作的线程数与设置的核心线程数进行判断
    if (workerCountOf(c) < corePoolSize) {
        // 创建新的线程(核心线程)执行任务
        if (addWorker(command, true))// 如果成功,结束该方法
            return;
        // 如果创建失败(可能存在多线程同时创建线程执行任务),往下继续执行
        c = ctl.get();//重新获取ctl的值,因为执行 addWorker 方法后该值可能变更
    }
    //如果处于 RUNNING 状态,并且成功将任务添加到队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 继续检查线程状态,因为可能存在上面操作完成后,线程池被关闭
        // 如果线程状态变更(不能接受新任务)且该任务没有被执行(依旧在队列中),丢弃该任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)// 如果工作线程数为0,创建新线程并启动(此处这样做的原因暂时尚未研究清除)
            addWorker(null, false);
    } else if (!addWorker(command, false))
    // 如果不是 RUNNING 状态或者任务列队已满无法再添加新任务
    // 创建新的线程进行处理(非核心线程),如果处理失败(线程池被关闭或者已达最大线程数量),进行丢弃处理
        reject(command);//丢弃处理,默认为抛出异常,可通过改变参数列表中的 RejectedExecutionHandler 对象来进行其他操作
}            
```

`java.util.concurrent.ThreadPoolExecutor#addWorker(Runnable firstTask, boolean core)`
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        //根据 ctl 获取运行状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 判断线程池是否能够接受新任务
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            // 获取工作的线程数
            int wc = workerCountOf(c);
            // 判断是否能够新建线程,如果不能结束操作
            // 如果线程数大于等于 536870911 或者大于等于设置的线程数直接返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 如果能创建线程,线程数加1
            // 如果增加成功(CAS操作),跳出最外层循环,执行线程启动逻辑
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 如果增加失败,表明 ctl 已被修改,更新ctl的值
            c = ctl.get();  // Re-read ctl
            // 如果线程池的状态变更,重新执行 retry 循环
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;// worker 是否启动标识符
    boolean workerAdded = false;// worker 是否被添加标识符
    Worker w = null;
    try {
        w = new Worker(firstTask);// 根据 firstTask 构建 Worker 对象
        final Thread t = w.thread;// 该线程对象根据 Worker 创建而来
        if (t != null) {// 如果无法创建线程,返回false
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();// 开启锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());// 获取状态

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {// 如果是RUNNING状态 或者是 SHUTDOWN 状态但是任务为空
                    if (t.isAlive()) // precheck that t is startable 如果任务已经运行 抛出异常
                        throw new IllegalThreadStateException();
                    workers.add(w);// 将 Worker 添加到 Set 集合中
                    int s = workers.size();// 获取 Set 中的数量
                    if (s > largestPoolSize)// 如果数量大于 largestPoolSize 更新 largestPoolSize 的值
                        largestPoolSize = s;// largestPoolSize 是同时存在的最大线程数
                    
                    workerAdded = true;// 标志 Worker 已经被添加到 Set 中
                }
            } finally {
                mainLock.unlock(); // 无论如何,释放锁资源
            }
            if (workerAdded) {// 如果 Worker 已经被添加,启动 Worker 中的线程
                t.start(); // 实际上是调用 ThreadPoolExecutor.runWorker(Worker w) 方法
                workerStarted = true;// 标志 Worker 已经启动
            }
        }
    } finally {
        if (! workerStarted) // 如果 Worker 没有启动,将该 Worker 从 Set 中移除,同时当前工作线程数减1,尝试结束线程池
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

`java.util.concurrent.ThreadPoolExecutor#runWorker(Worker w)`

```java
final void runWorker(Worker w) {
    // 获取当前的线程
    Thread wt = Thread.currentThread();
    // 获取 Worker 中的具体执行任务
    Runnable task = w.firstTask;
    w.firstTask = null;// 将 firstTask 对象设置为null
    w.unlock(); // allow interrupts 释放锁资源,允许中断
    boolean completedAbruptly = true;// 是否为未执行任务就结束线程
    try {
        // 循环判断当前任务 或者 阻塞获取任务
        // 如果 task 且 getTask() 不为 null,执行该任务,否则表明线程中断,超时,需要结束该线程
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 对线程中断进行判定
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 线程运行之前方法,在 ThreadPoolExecutor 中为空实现(可通过继承进行一些自己的处理)
                beforeExecute(wt, task);
                Throwable thrown = null;// 获取异常信息,传递给 afterExecute 方法
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 线程运行之后方法,在 ThreadPoolExecutor 中为空实现(可通过继承进行一些自己的处理)
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;// 将 task 设置为 null,防止重复执行
                w.completedTasks++;// 线程执行次数加1
                w.unlock();// 释放锁
            }
        }
        completedAbruptly = false;
    } finally {
        // 退出线程,统计执行任务总数,从 Set 中删除 Worker,尝试结束线程池
        processWorkerExit(w, completedAbruptly);
    }
}

```
`java.util.concurrent.ThreadPoolExecutor#getTask()`
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out? 
    // workQueue.poll 操作是否超时

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 检查线程池状态及列队是否有值
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 减少线程数量并返回 null,返回 null 会结束调用该方法的线程
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 判断是否开启线程超时清理操作
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 如果工作线程大于设定的最大线程数 或者 应该清理线程且该线程超时
        // 且 工作线程数大于1 或 列队已空
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 减少工作线程数量,如果成功返回 null 结束调用该方法的线程
            if (compareAndDecrementWorkerCount(c))
                return null;
            // 否则重新执行循环
            continue;
        }

        try {
            // 如果该结束超时线程,在设定的超时时间内获取列队中的元素,如果获取为空
            // 则表明该线程已经超时,在下次循环式如果条件允许,结束该线程
            // 否则一直阻塞获取列队中的元素,然后返回
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

`java.util.concurrent.AbstractExecutorService#submit(Callable<T> task)`
```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // 依靠FutureTask的构造方法,根据对应参数新建一个FutureTask对象
    // FutureTask(Callable<V> callable) ,FutureTask(Runnable runnable, V result) 
    // FutureTask 实现了 RunnableFuture 接口,而 RunnableFuture 继承了 Future,Runnable 接口
    RunnableFuture<T> ftask = newTaskFor(task);
    // 执行 execute 方法,该方法的实现由子类提供,此处参考 java.util.concurrent.ThreadPoolExecutor#execute
    execute(ftask);
    // 返回 ftask 对象
    return ftask;
}
```

### FutureTask

存在一个 `int` 类型的 `state` 变量,改变量的至可能为以下六种
1. NEW           初始状态
2. COMPLETING    任务执行完成,即将更改为NORMAL或EXCEPTIONAL状态
3. NORMAL        任务执行完成(无异常)
4. EXCEPTIONAL   任务执行完成(有异常)
5. CANCELLED     任务取消
6. INTERRUPTING  中断进行中
7. INTERRUPTED   中断完成

状态变更存在以下几种顺序
> 
NEW -> COMPLETING -> NORMAL
NEW -> COMPLETING -> EXCEPTIONAL
NEW -> CANCELLED
NEW -> INTERRUPTING -> INTERRUPTED


`java.util.concurrent.FutureTask#get()`
```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 如果当前状态为 NEW 或 COMPLETING 证明该线程还没有执行完成,等待该线程完成
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);// 否则获取返回的状态值,进行判断后返回
}
```
`java.util.concurrent.FutureTask#report(int s)`
```java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL) // 如果状态为 NORMAL 代表线程执行正常返回,其他都属于异常情况(执行过程异常,线程中断,任务取消)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

`java.util.concurrent.FutureTask#awaitDone(boolean timed, long nanos)`
```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    // 超时到期时间,如果不进行超时判断,返回0
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null; // 等待任务结果消息的节点
    boolean queued = false;// 是否成功将当前 WaitNode 放入到 waiters 链表中
    for (;;) {
        // 如果当前线程中断,取消链接超时或中止的等待节点
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 如果状态大于 COMPLETING 证明已经执行完毕,返回状态值
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 如果状态等于 COMPLETING,暂停线程,等待状态变更,然后继续进入循环
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)// 如果 状态等于 NEW 并且 WaitNode 为null
            q = new WaitNode();// 创建一个新的 WaitNode 对象,然后重新循环后进入下一步
        else if (!queued)// 如果 queued 为false
            // 将 waiters 对象设置为当前的 WaitNode 的下一个节点,然后将 waiters 引用指向当前的 WaitNode 
            // 如果操作失败,循环执行,直到成功或在之前的条件判断中提前结束
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
            // 修改成功过后,判断获取结果是否需要进行超时判定
        else if (timed) {
            nanos = deadline - System.nanoTime();
            // 如果已经超时,取消链接超时或中止的等待节点,返回状态
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 在到期时间内禁用当前线程,任务完成后会被唤醒
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 禁用当前线程,任务完成后会被唤醒
            LockSupport.park(this);
    }
}
```
在 `java.util.concurrent.ThreadPoolExecutor#runWorker(Worker w)` 方法中任务执行是直接调用 `run` 方法,因为`FutureTask`需要获取任务运行结果及收集异常,所以对 `run` 方法进行了包装
在构造 FutureTask 时参数允许接受 Callable 与 Runnable 类型,实际上他会将 Runnable 类型转为一个 Callable 类型,然后使用 `call()` 方法进行调用
`java.util.concurrent.FutureTask#run()`
```java
public void run() {
    // 检查线程状态,如果不是 NEW 状态或 runner 变量不为 null 结束该方法
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        // 如果 Callable 可以运行
        if (c != null && state == NEW) {
            V result;
            boolean ran;// 是否执行过程中无异常
            try {
                // 运行 Callable 中的任务
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                // 异常处理
                result = null;
                ran = false;
                // 更新状态,并将异常设置为返回值,以方便异常传递,通知所有等待节点恢复运行
                setException(ex);
            }
            if (ran)
                // 更新状态,设置返回值,通知所有等待节点恢复运行
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
`java.util.concurrent.FutureTask#set(V v)`
```java
protected void set(V v) {
    // 更新状态为 COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 设置返回值
        outcome = v;
        // 更新状态为 NORMAL
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        // 通知其他等待线程,如 get() 让他恢复运行
        finishCompletion();
    }
}
```
`setException(Throwable t)` 方法请参考 `set(V v)`
`java.util.concurrent.FutureTask#finishCompletion()`
```java
private void finishCompletion() {
    // assert state > COMPLETING;
    // 如果等待对象不为空
    for (WaitNode q; (q = waiters) != null;) {
        // 将 waiters 设置为 null,然后循环唤醒等待结果的线程
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                // 获取等待的线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    // 唤醒对应的线程
                    LockSupport.unpark(t);
                }
                // WaitNode 为链表结构,获取下一个WaitNode
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    // 执行 done 方法,在 FutureTask 中为空实现,子类可以覆写该方法用于其他处理
    done();

    callable = null;        // to reduce footprint
}
```
