### 线程模型的 ULT 和 KLT









### 线程池的几种状态

```java
// runState is stored in the high-order bits
// int 前三个字节代表线程的状态，对应的 ASCII 码
private static final int RUNNING    = -1 << COUNT_BITS;// 运行
private static final int SHUTDOWN   =  0 << COUNT_BITS;// 关闭
private static final int STOP       =  1 << COUNT_BITS;// 停止
private static final int TIDYING    =  2 << COUNT_BITS;// 整理
private static final int TERMINATED =  3 << COUNT_BITS;// 已终止
```





### 线程池的四种拒绝策略

指的是线程池的最大线程数量达到最大值，任务等待队列为有空闲位置。这时线程池对新进来的请求做出什么样的操作。JDK默认有4种实现，他们都有自己独特的处理方式。

1. **AbortPolicy**

    线程池无暇处理，会直接拒绝并抛出异常。

2. **CallerRunsPolicy**

    线程池无暇处理，如果线程池未关闭，则由调用线程自己处理。

3. **DiscardPolicy**

    线程池无暇处理，直接丢弃任务，无响应。

4. **DiscardOldestPolicy**

    从任务等待队列里选择一个最老的任务丢弃，新的任务放进去。





### 线程池已经忙不过来，拒绝策略就有新进来的线程执行。





### 核心线程数量时候会过期？？？

答：核心线程数量默认的不会过期，在1.6之后可以通过参数设置核心线程在空闲超过一定时间后被关闭。







-   为什么Worker继承AQS？
-   runWoker都干了哪些事？
-   我想在run方法真正执行自己业务代码的前后各自打印log怎么做？
-   非核心线程是怎么结束的？核心线程又是怎么长期存活不会销毁的？









### 工作线程如何获取任务

答：正常情况下，可以直接从任务对列获取。当任务队列已空，没有新任务到达，为了避免工作线程空旋，判断是否有新任务，造成`CPU`资源的浪费，会将其封装成`Node`对象，放置于`AQS抽象同步等待队列`中。等待新任务添加到任务队列后唤醒头节点处理。

扩展点1：调用`LockSupport.park`函数阻塞线程指定时间。到期自动苏醒，结束阻塞状态，在其判断任务队列是否存在任务，没有的情况下会减少工作线程数量，从工作线程数组中删除，结束循环。







### 核心线程与非核心线程的区别

答：正常情况核心线程不会死亡，非核心线程闲置指定时间会结束执行。想要核心线程也采取闲置结束，需要设置`allowCoreThreadTimeOut：true`。

扩展点1：首先`Worker`没有属性区分核心与非核心，同时线程池也只在乎有没有指定数量的线程处理任务。举个栗子：核心线程数`4`，最大线程数`8`，现在有`7`个工作线程。那个当任务队列为空时，把他们都当作非核心处理，指定阻塞时间。阻塞结束后会将`timedOut`设置为`true`，因为工作线程数大于核心线程数，`timed`也是`true`，两者都是`true`会结束非核心数量之外的线程。









### 线程池任务处理完毕，核心线程进入阻塞状态，这时放进新任务，如何同步核心线程处理呢？

描述：线程池任务清空后，工作线程获取任务时会进入阻塞状态。

答：任务进入等待队列后，调用`signal`函数唤醒







### 线程池有个`ReentrantLock`，都用在了哪里？









### `ShutDown`怎么做只中断空闲线程

线程













```
This requires a recheck in second case to deal with shutdownNow race while clearing interrupt
```

```java
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    // 获取ctl（32位）
    int c = ctl.get();
    // 工作线程数 < 核心线程数
    if (workerCountOf(c) < corePoolSize) {
        // 创建工作线程执行任务
        if (addWorker(command, true))
            // 终止
            return;
        // 走到这里说明创建工作线程失败
        // 1、创建工作线程时，其他线程调用了shutdown() 函数，导致创建失败。
        // 2、多个异步请求同时创建仅剩的一个工作线程名额，必然有一个失败。
        c = ctl.get();
    }
    // 线程池状态为 RUNNING 且 任务添加阻塞队列成功
    if (isRunning(c) && workQueue.offer(command)) {
        // 重新获取线程池状态
        int recheck = ctl.get();
        // 线程池非 RUNNING 且 删除删除任务成功
        if (! isRunning(recheck) && remove(command))
            // 线程池关闭，拒绝处理任务
            reject(command);
        // 任务拒绝不成功，判断当前工作线程是否 == 0，
        // == 0 表示有任务，但是没有工作线程去处理
        else if (workerCountOf(recheck) == 0)
            // 创建工作线程
            addWorker(null, false);
    }
    // 任务添加阻塞队列失败，尝试创建非核心线程执行任务
    else if (!addWorker(command, false))
        // 非核心线程创建失败，走拒绝策略
        reject(command);
}


/**
* Checks if a new worker can be added with respect to current
* pool state and the given bound (either core or maximum). If so,
* the worker count is adjusted accordingly, and, if possible, a
* new worker is created and started, running firstTask as its
* first task. This method returns false if the pool is stopped or
* eligible to shut down. It also returns false if the thread
* factory fails to create a thread when asked.  If the thread
* creation fails, either due to the thread factory returning
* null, or due to an exception (typically OutOfMemoryError in
* Thread.start()), we roll back cleanly.
*
* @param firstTask the task the new thread should run first (or
* null if none). Workers are created with an initial first task
* (in method execute()) to bypass queuing when there are fewer
* than corePoolSize threads (in which case we always start one),
* or when the queue is full (in which case we must bypass queue).
* Initially idle threads are usually created via
* prestartCoreThread or to replace other dying workers.
*
* @param core if true use corePoolSize as bound, else
* maximumPoolSize. (A boolean indicator is used here rather than a
* value to ensure reads of fresh values after checking other pool
* state).
* @return true if successful
*/
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        // 获取ctl（32位）
        int c = ctl.get();
        // 获取线程池状态
        int rs = runStateOf(c);

        /**
         * 拦截非RUNNING、SHUTDOWN状态下的任务请求
         * Check if queue empty only if necessary.
         */
        if (rs >= SHUTDOWN && // 线程池状态非 RUNNING 
            ! (rs == SHUTDOWN && // 线程池状态是 SHUTDOWN 
               firstTask == null && // 提交的任务是 null，基于上一步判断，SHUTDOWN 不接收新的任务
               ! workQueue.isEmpty())) // 任务队列 workQueue 里的任务不为零
            return false;

        /**
         * 由于外层方法为做同步处理，此处通过 CAS 来规避工作线程数超过预设值
         */
        for (;;) {
            // 获取当前工作线程数
            int wc = workerCountOf(c);
            if (wc >= CAPACITY || // 工作线程数 >= 线程池最大值
                wc >= (core ? corePoolSize : maximumPoolSize)) // 工作线程数 >= 设置的工作线程规定阈值
                // 工作线程数达到最大值，不允许继续创建返回 false
                return false;
            // CAS 递增工作线程数量
            if (compareAndIncrementWorkerCount(c))
                // 终止外层循环
                break retry;
            /
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 工作线程启动标识
    boolean workerStarted = false;
    // 工作线程添加集合成功标识
    boolean workerAdded = false;
    // 工作线程
    Worker w = null;
    try {
        // 创建工作线程
        w = new Worker(firstTask);
        // 获取工作线程的线程对象
        final Thread t = w.thread;
        // 非空判断，这一步不可能为空
        if (t != null) {
        	// 获取线程池内部锁对象（非公平锁）
            final ReentrantLock mainLock = this.mainLock;
        	// 阻塞等待加锁成功
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 获取线程池状态
                int rs = runStateOf(ctl.get());

                // 线程池状态 < SHUTDOWN || SHUTDOWN 状态下任务队列不为 null
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // TODO 检查工作线程是否已经启动，防止后续调用 start 函数报错
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 工作线程加入，工作线程 set 集合
                    workers.add(w);
                    // 添加后的集合大小
                    int s = workers.size();
                    // 集合大小 > 历史最大值
                    if (s > largestPoolSize)
                        // 同步历史最大工作线程数
                        largestPoolSize = s;
                    // 同步工作线程添加集合成功标识
                    workerAdded = true;
                }
            } finally {
                // 释放线程池同步锁
                mainLock.unlock();
            }
            // 工作线程添加集合成功
            if (workerAdded) {
                // 启动工作线程处理任务
                t.start();
                // 同步工作线程启动标识
                workerStarted = true;
            }
        }
    } finally {
        // 工作线程启动失败
        if (! workerStarted)
            // 删除工作线程
            addWorkerFailed(w);
    }
    // 工作线程启动标识
    return workerStarted;
}


/**
 * Rolls back the worker thread creation.
 * - removes worker from workers, if present
 * - decrements worker count
 * - rechecks for termination, in case the existence of this
 *   worker was holding up termination
 */
private void addWorkerFailed(Worker w) {
    // 操作工作线程及衍生品，需要获取线程池的内部锁
    final ReentrantLock mainLock = this.mainLock;
    // 阻塞获取锁
    mainLock.lock();
    try {
        // 待撤销线程不为 null
        if (w != null)
            // 移除待撤销工作线程
            workers.remove(w);
        // CAS 递减工作线程数
        decrementWorkerCount();
        // TODO
        tryTerminate();
    } finally {
        // 释放锁
        mainLock.unlock();
    }
}



/**
 * Main worker run loop.  Repeatedly gets tasks from queue and
 * executes them, while coping with a number of issues:
 *
 * 1. We may start out with an initial task, in which case we
 * don't need to get the first one. Otherwise, as long as pool is
 * running, we get tasks from getTask. If it returns null then the
 * worker exits due to changed pool state or configuration
 * parameters.  Other exits result from exception throws in
 * external code, in which case completedAbruptly holds, which
 * usually leads processWorkerExit to replace this thread.
 *
 * 2. Before running any task, the lock is acquired to prevent
 * other pool interrupts while the task is executing, and then we
 * ensure that unless pool is stopping, this thread does not have
 * its interrupt set.
 *
 * 3. Each task run is preceded by a call to beforeExecute, which
 * might throw an exception, in which case we cause thread to die
 * (breaking loop with completedAbruptly true) without processing
 * the task.
 *
 * 4. Assuming beforeExecute completes normally, we run the task,
 * gathering any of its thrown exceptions to send to afterExecute.
 * We separately handle RuntimeException, Error (both of which the
 * specs guarantee that we trap) and arbitrary Throwables.
 * Because we cannot rethrow Throwables within Runnable.run, we
 * wrap them within Errors on the way out (to the thread's
 * UncaughtExceptionHandler).  Any thrown exception also
 * conservatively causes thread to die.
 *
 * 5. After task.run completes, we call afterExecute, which may
 * also throw an exception, which will also cause thread to
 * die. According to JLS Sec 14.20, this exception is the one that
 * will be in effect even if task.run throws.
 *
 * The net effect of the exception mechanics is that afterExecute
 * and the thread's UncaughtExceptionHandler have as accurate
 * information as we can provide about any problems encountered by
 * user code.
 *
 * @param w the worker
 */
final void runWorker(Worker w) {
    // 获取当前线程
    Thread wt = Thread.currentThread();
    // 获取工作线程任务
    Runnable task = w.firstTask;
    // 
    w.firstTask = null;
    // 
    w.unlock(); // allow interrupts
    // 
    boolean completedAbruptly = true;
    try {
        // 
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
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


/**
 * Transitions to TERMINATED state if either (SHUTDOWN and pool
 * and queue empty) or (STOP and pool empty).  If otherwise
 * eligible to terminate but workerCount is nonzero, interrupts an
 * idle worker to ensure that shutdown signals propagate. This
 * method must be called following any action that might make
 * termination possible -- reducing worker count or removing tasks
 * from the queue during shutdown. The method is non-private to
 * allow access from ScheduledThreadPoolExecutor.
 * 此方法必须在任何可能导致终止的操作之后调用
 * 在关闭期间减少工人数量或从队列中删除任务
 */
final void tryTerminate() {
    for (;;) {
        
        int c = ctl.get();
        /**
         * 线程池是否需要终止
         * 如果以下3中情况任一为true，return，不进行终止
         * 1、还在运行状态
         * 2、状态是TIDYING、或 TERMINATED，已经终止过了
         * 3、SHUTDOWN 且 workQueue不为空
         */
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        
        /**
         * 只有shutdown状态 且 workQueue为空，或者 stop状态能执行到这一步
         * 如果此时线程池还有线程（正在运行任务，正在等待任务）
         * 中断唤醒一个正在等任务的空闲worker
         * 唤醒后再次判断线程池状态，会return null，进入processWorkerExit()流程
         */
        if (workerCountOf(c) != 0) { // Eligible to terminate 资格终止
            interruptIdleWorkers(ONLY_ONE); // 中断workers集合中的空闲任务，参数为true，只中断一个
            return;
        }

        /**
         * 如果状态是SHUTDOWN，workQueue也为空了，正在运行的worker也没有了，开始terminated
         */
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // CAS：将线程池的ctl变成TIDYING
            //（所有的任务被终止，workCount为0，为此状态时将会调用terminated()方法），期间ctl有变化就会失败，会再次for循环
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated(); // 需子类实现
                } 
                finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // 将线程池的ctl变成TERMINATED
                    termination.signalAll(); // 唤醒调用了 等待线程池终止的线程 awaitTermination() 
                }
                return;
            }
        }
        finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
        // 如果上面的CAS判断false，再次循环
    }
}


/**
* Performs blocking or timed wait for a task, depending on
* current configuration settings, or returns null if this worker
* must exit because of any of:
* 1. There are more than maximumPoolSize workers (due to
*    a call to setMaximumPoolSize).
* 2. The pool is stopped.
* 3. The pool is shutdown and the queue is empty.
* 4. This worker timed out waiting for a task, and timed-out
*    workers are subject to termination (that is,
*    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
*    both before and after the timed wait, and if the queue is
*    non-empty, this worker is not the last thread in the pool.
*
* @return task, or null if the worker must exit, in which case
*         workerCount is decremented
*/
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        // 获取 worker 工作数量
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // allowCoreThreadTimeOut 是否允许核心线程被销毁
       	// wc > corePoolSize 判断 worker 工作线程数量是否大于核心线程数量
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
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







```
0001 1111 1111 1111 1111 1111 1111 0100
0010 0000 0000 0000 0000 0000 0000 0000
```















