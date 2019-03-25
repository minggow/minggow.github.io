---
layout: post
title: Java线程池
category: other
tags: [other]
excerpt:  Java线程池
---

## 前言
在Java服务器开发中，服务器接受并处理请求，是通过线程创建线程的方式来处理的。如果每次请求都新创建一个线程的话实现起来非常简便，
但是存在一个问题：
如果并发的请求数量非常多，但每个线程执行的时间很短，这样就会频繁的创建和销毁线程，如此一来会大大降低系统的效率。可能出现服务器在为每个请求
创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。
线程池为线程生命周期的开销和资源不足问题提供了解决方案。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。
什么场景下适合使用线程池？
- 单个任务处理时间比较短
- 需要处理的任务数量很大
使用线程池的好处有哪些？
- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

问题：
通过上面线程池的介绍，线程池避免了线程的频繁销毁和创建，这个是如何做到的呢？

## ThreadPoolExecutor说明

### 重要概念
ThreadPoolExecutor继承自AbstractExecutorService，也是实现了ExecutorService接口。
```java
/***
    *   RUNNING:  Accept new tasks and process queued tasks
    *   SHUTDOWN: Don't accept new tasks, but process queued tasks
    *   STOP:     Don't accept new tasks, don't process queued tasks,
    *             and interrupt in-progress tasks
    *   TIDYING:  All tasks have terminated, workerCount is zero,
    *             the thread transitioning to state TIDYING
    *             will run the terminated() hook method
    *   TERMINATED: terminated() has completed
    * RUNNING -> SHUTDOWN
    *    On invocation of shutdown(), perhaps implicitly in finalize()
    * (RUNNING or SHUTDOWN) -> STOP
    *    On invocation of shutdownNow()
    * SHUTDOWN -> TIDYING
    *    When both queue and pool are empty
    * STOP -> TIDYING
    *    When pool is empty
    * TIDYING -> TERMINATED
    *    When the terminated() hook method has completed
*/
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```
说明：
ctl是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它包含两部分的信息: 
线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，这里可以看到，使用了Integer类型来保存，高3位保存runState，
低29位保存workerCount。COUNT_BITS 就是29，CAPACITY就是1左移29位减1（29个1），这个常量表示workerCount的上限值，大约是5亿。

线程有5种状态：新建状态，就绪状态，运行状态，阻塞状态，死亡状态。

```java
RUNNING    -- 对应的高3位值是111。
SHUTDOWN   -- 对应的高3位值是000。
STOP       -- 对应的高3位值是001。
TIDYING    -- 对应的高3位值是010。
TERMINATED -- 对应的高3位值是011。
```

状态变化如下：
![](/assets/images/2019/03/java_concurrent_threadpool_status.jpg)

ctl相关计算方法：
```java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
- runStateOf：获取运行状态；
- workerCountOf：获取活动线程数；
- ctlOf：获取运行状态和活动线程数的值。



### 构造方法
主要实现类：``ThreadPoolExecutor``，核心构造方法如下：
```java
    /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler);

```
构造方法中的字段含义如下：
- corePoolSize：核心线程数量，当有新任务在execute()方法提交时，会执行以下判断：
    - 如果运行的线程少于 corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的；
    - 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，则只有当workQueue满时才创建新的线程去处理任务；
    - 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放
    入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
    - 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；
所以，任务提交时，判断的顺序为 corePoolSize –> workQueue –> maximumPoolSize。
- maximumPoolSize：最大线程数量；
- workQueue：等待队列，当任务提交时，如果线程池中的线程数量大于等于corePoolSize的时候，把该任务封装成一个Worker对象放入等待队列；队列主
要有以下几种处理方式:
    - 直接切换：这种方式常用的队列是SynchronousQueue
    - 使用无界队列：一般使用基于链表的阻塞队列LinkedBlockingQueue。如果使用这种方式，那么线程池中能够创建的最大线程数就是corePoolSize
    ，而maximumPoolSize就不会起作用了（后面也会说到）。当线程池中所有的核心线程都是RUNNING状态时，这时一个新的任务提交就会放入等待队列中。
    - 使用有界队列：一般使用ArrayBlockingQueue。使用该方式可以将线程池的最大线程数量限制为maximumPoolSize，这样能够降低资源的消耗，
    但同时这种方式也使得线程池对线程的调度变得更困难，因为线程池和队列的容量都是有限的值，所以要想使线程池处理任务的吞吐率达到一个相对合理
    的范围，又想使线程调度相对简单，并且还要尽可能的降低线程池对资源的消耗，就需要合理的设置这两个数量。
        - 如果要想降低系统资源的消耗（包括CPU的使用率，操作系统资源的消耗，上下文环境切换的开销等）, 可以设置较大的队列容量和较小的线程池
        容量, 但这样也会降低线程处理任务的吞吐量。
        - 如果提交的任务经常发生阻塞，那么可以考虑通过调用 setMaximumPoolSize() 方法来重新设定线程池的容量。
        - 如果队列的容量设置的较小，通常需要将线程池的容量设置大一点，这样CPU的使用率会相对的高一些。但如果线程池的容量设置的过大，则在提
        交的任务数量太多的情况下，并发量会增加，那么线程之间的调度就是一个要考虑的问题，因为这样反而有可能降低处理任务的吞吐量。
- keepAliveTime：线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不
会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime；
- threadFactory：它是ThreadFactory类型的变量，用来创建新线程。默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的
ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。
handler：它是RejectedExecutionHandler类型的变量，表示线程池的饱和策略。如果阻塞队列满了并且没有空闲的线程，这时如果继续提交任务，就需要
采取一种策略处理该任务。线程池提供了4种策略：
    -AbortPolicy：直接抛出异常，这是默认策略；
    - CallerRunsPolicy：用调用者所在的线程来执行任务；
    - DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
    - DiscardPolicy：直接丢弃任务；


## 代码细节走读
> 注：JDK版本：jdk1.8.0_111，带中文数字编号的中文注释是我添加的

### execute方法实现

```
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

        /*
         * clt记录着runState和workerCount
         */
        int c = ctl.get();
        
        /*
         * workerCountOf方法取出低29位的值，表示当前活动的线程数；
         * 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中；
         * 并把任务添加到该线程中。
         */
        if (workerCountOf(c) < corePoolSize) {
        /*
         * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断；
         * 如果为true，根据corePoolSize来判断；
         * 如果为false，则根据maximumPoolSize来判断
         */
            if (addWorker(command, true))
                return;
             /*
             * 如果添加失败，则重新获取ctl值
             */
            c = ctl.get();
        }

        /*
         * 如果当前线程池是运行状态并且任务添加到队列成功
         */
        if (isRunning(c) && workQueue.offer(command)) {

            /**
             * 重新获取ctl值
            **/
            int recheck = ctl.get();

            /*
             * 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
        	 * 这时需要移除该command
        	 * 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
        	 */
            if (! isRunning(recheck) && remove(command))
                reject(command);
             /*
             * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
             * 这里传入的参数表示：
             * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
             * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
             * 如果判断workerCount大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。
             */
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }

        /*
         * 如果执行到这里，有两种情况：
         * 1. 线程池已经不是RUNNING状态；
         * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
         * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
         * 如果失败则拒绝该任务
         */
        else if (!addWorker(command, false))
            reject(command);
    }
```
简单来说，在执行execute()方法时如果状态一直是RUNNING时，的执行过程如下：
- 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务；
- 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
- 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
- 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。
这里要注意一下addWorker(null, false);，也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workQueue中获取任务。所以，在workerCountOf(recheck) == 0时执行addWorker(null, false);也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。

execute方法执行流程如下：
![](/assets/images/2019/03/java_concurrent_threadpool_process.png)

### addWorker的方法
addWorker方法的主要工作是在线程池中创建一个新的线程并执行，firstTask参数 用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize，代码如下addWorker方法的主要工作是在线程池中创建一个新的线程并执行，firstTask参数 用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize，代码如下

```java
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

	         /* 
	         * 获取运行状态
	         */
            int c = ctl.get();
            int rs = runStateOf(c);

	        /*
	         * 这个if判断
	         * 如果rs >= SHUTDOWN，则表示此时不再接收新任务；
	         * 接着判断以下3个条件，只要有1个不满足，则返回false：
	         * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
	         * 2. firsTask为空
	         * 3. 阻塞队列不为空
	         * 
	         * 首先考虑rs == SHUTDOWN的情况
	         * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false；
	         * 然后，如果firstTask为空，并且workQueue也为空，则返回false，
	         * 因为队列中已经没有任务了，不需要再添加线程了
	         */
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                /* 
                 * 获取线程数
                 */
                int wc = workerCountOf(c);
	            /*
	             * 如果wc超过CAPACITY，也就是ctl的低29位的最大值（二进制是29个1），返回false；
	             * 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，
	             * 如果为false则根据maximumPoolSize来比较。
	             */
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                /*
                 * 尝试增加workerCount，如果成功，则跳出第一个for循环
                 */
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                /*
                 * 如果增加workerCount失败，则重新获取ctl的值
                 */ 
                c = ctl.get();  // Re-read ctl
                /*
                 * 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
                 */
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {

            /*
             * 根据firstTask来创建Worker对象
             * 每一个Worker对象都会创建一个线程
             */
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

                    /*
                     * rs < SHUTDOWN表示是RUNNING状态；
                     * 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                     * 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                     */
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        /*
                         * workers是一个HashSet，用于存储执行的任务
                         */    
                        workers.add(w);
                        int s = workers.size();
                        /*
                         * largestPoolSize记录着线程池中出现过的最大线程数量
                         */
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
            		/*
            		 * 任务添加成功，启动线程
            		 */
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
注意：
t.start()这个语句，启动时会调用Worker类中的run方法，Worker本身实现了Runnable接口，所以一个Worker类型的对象也是一个线程。

### Worker类
线程池中的每一个线程被封装成一个Worker对象，ThreadPool维护的其实就是一组Worker对象，下面是Worker类定义：
```java
    /**
     * Class Worker mainly maintains interrupt control state for
     * threads running tasks, along with other minor bookkeeping.
     * This class opportunistically extends AbstractQueuedSynchronizer
     * to simplify acquiring and releasing a lock surrounding each
     * task execution.  This protects against interrupts that are
     * intended to wake up a worker thread waiting for a task from
     * instead interrupting a task being run.  We implement a simple
     * non-reentrant mutual exclusion lock rather than use
     * ReentrantLock because we do not want worker tasks to be able to
     * reacquire the lock when they invoke pool control methods like
     * setCorePoolSize.  Additionally, to suppress interrupts until
     * the thread actually starts running tasks, we initialize lock
     * state to a negative value, and clear it upon start (in
     * runWorker).
     */
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
            // state AQS的状态，设置禁止中断，直到运行工作程序
            setState(-1); // inhibit interrupts until runWorker
            // 执行的Runnable任务，然后通过线程工厂创建线程 
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
Worker类继承了AQS，并实现了Runnable接口，注意其中的firstTask和thread属性：firstTask用它来保存传入的任务；thread是在调用构造方法时通过ThreadFactory来创建的线程，是用来处理任务的线程。

在调用构造方法时，需要把任务传入，这里通过getThreadFactory().newThread(this);来新建一个线程，newThread方法传入的参数是this，因为Worker本身继承了Runnable接口，也就是一个线程，所以一个Worker对象在启动的时候会调用Worker类中的run方法。

Worker继承了AQS，使用AQS来实现独占锁的功能。为什么不使用ReentrantLock来实现呢？可以看到tryAcquire方法，它是不允许重入的，而ReentrantLock是允许重入的：

- lock方法一旦获取了独占锁，表示当前线程正在执行任务中；
- 如果正在执行任务，则不应该中断线程；
- 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断；
- 线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；
之所以设置为不可重入，是因为我们不希望任务在调用像setCorePoolSize这样的线程池控制方法时重新获取锁。如果使用ReentrantLock，它是可重入的，这样如果在任务中调用了如setCorePoolSize这类线程池控制的方法，会中断正在运行的线程。
所以，Worker继承自AQS，用于判断线程是否空闲以及是否可以被中断。

此外，在构造方法中执行了setState(-1);，把state变量设置为-1，为什么这么做呢？是因为AQS中默认的state是0，如果刚创建了一个Worker对象，还没有执行任务时，这时就不应该被中断，看一下tryAquire方法：
```java
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```
tryAcquire方法是根据state是否是0来判断的，所以，setState(-1);将state设置为-1是为了禁止在执行任务前对线程进行中断。

正因为如此，在runWorker方法中会先调用Worker对象的unlock方法将state设置为0.

### runWorker方法
在Worker类中的run方法调用了runWorker方法来执行任务，runWorker方法的代码如下：
```java
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
        Thread wt = Thread.currentThread();
        /*
         * 获取第一个任务
         */
        Runnable task = w.firstTask;
        w.firstTask = null;
        /*
         * 允许中断
         */ 
        w.unlock(); // allow interrupts
        /*
         * 是否因为异常退出循环
         */
        boolean completedAbruptly = true;
        try {
            
    	   /*
    	    * 如果task为空，则通过getTask来获取任务。
            */
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
                        /*
                         * 任务执行
                         */
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
这里说明一下第一个if判断，目的是：

- 如果线程池正在停止，那么要保证当前线程是中断状态；
- 如果不是的话，则要保证当前线程不是中断状态；
这里要考虑在执行该if语句期间可能也执行了shutdownNow方法，shutdownNow方法会把状态设置为STOP，回顾一下STOP状态：

> 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态。

STOP状态要中断线程池中的所有线程，而这里使用Thread.interrupted()来判断是否中断是为了确保在RUNNING或者SHUTDOWN状态时线程是非中断状态的，因为Thread.interrupted()方法会复位中断的状态。

总结一下runWorker方法的执行过程：

- while循环不断地通过getTask()方法获取任务；
- getTask()方法从阻塞队列中取任务；
- 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态；
- 调用task.run()执行任务；
- 如果task为null则跳出循环，执行processWorkerExit()方法；
- runWorker方法执行完毕，也代表着Worker中的run方法执行完毕，销毁线程。
- 这里的beforeExecute方法和afterExecute方法在ThreadPoolExecutor类中是空的，留给子类来实现。

completedAbruptly变量来表示在执行任务过程中是否出现了异常，在processWorkerExit方法中会对该变量的值进行判断。

### getTask方法实现
getTask方法用来从阻塞队列中取任务，代码如下：
```java
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
	        /*
	         * 如果线程池状态rs >= SHUTDOWN，也就是非RUNNING状态，再进行以下判断：
	         * 1. rs >= STOP，线程池是否正在stop；
	         * 2. 阻塞队列是否为空。
	         * 如果以上条件满足，则将workerCount减1并返回null。
	         * 因为如果当前线程池状态的值是SHUTDOWN或以上时，不允许再向阻塞队列中添加任务。
	         */
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            /* 
             * timed变量用于判断是否需要进行超时控制。
             * allowCoreThreadTimeOut默认是false，也就是核心线程不允许进行超时；
             * wc > corePoolSize，表示当前线程池中的线程数量大于核心线程数量；
             * 对于超过核心线程数量的这些线程，需要进行超时控制
             */
            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                /*
                 * wc > maximumPoolSize的情况是因为可能在此方法执行阶段同时执行了setMaximumPoolSize方法；
                 * timed && timedOut 如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取任务发生了超时
                 * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1；如果减1失败，则返回重试。
                 * 如果wc == 1时，也就说明当前线程是线程池中唯一的一个线程了。
                 * 注意：如果time=false，执行workQueue.take方法，因为take方法是阻塞的，这样核心线程数可以不销毁的原因
                 */
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                /*
                 * r == null, 获取任务超时，标记超时
                 */
                timedOut = true;
            } catch (InterruptedException retry) {
                /*
                 * 如果获取任务时当前线程发生了中断，则设置timedOut为false并返回循环重试
                 */
                timedOut = false;
            }
        }
    }

```
如何保持核心线程池的数量?
- 第二个if判断，在执行execute方法时，如果当前线程池的线程数量超过了corePoolSize且小于maximumPoolSize，并且workQueue已满时，则可以增加工作线程，但这时如果超时没有获取到任务，也就是timedOut为true的情况，说明workQueue已经为空了，也就说明了当前线程池中不需要那么多线程来执行任务了，可以把多于corePoolSize数量的线程销毁掉，保持线程数量在corePoolSize即可。
- timed变量用于判断是否需要进行超时控制。allowCoreThreadTimeOut默认是false，也就是核心线程不允许进行超时；wc > corePoolSize，表示当前线程池中的线程数量大于核心线程数量；对于超过核心线程数量的这些线程，需要进行超时控制。队列为空了，当前线程数不大于核心线程数，通过take阻塞线程，保持核心线程数数量。

什么时候销毁线程呢？
- 当然是runWorker方法执行完之后，也就是Worker中的run方法执行完，由JVM自动回收。
- getTask方法返回null时，在runWorker方法中会跳出while循环，然后会执行processWorkerExit方法。

### processWorkerExit方法
```java
    /**
     * Performs cleanup and bookkeeping for a dying worker. Called
     * only from worker threads. Unless completedAbruptly is set,
     * assumes that workerCount has already been adjusted to account
     * for exit.  This method removes thread from worker set, and
     * possibly terminates the pool or replaces the worker if either
     * it exited due to user task exception or if fewer than
     * corePoolSize workers are running or queue is non-empty but
     * there are no workers.
     *
     * @param w the worker
     * @param completedAbruptly if the worker died due to user exception
     */
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        /*
         * 如果completedAbruptly值为true，则说明线程执行时出现了异常，需要将workerCount减1；
         * 如果线程执行时没有出现异常，说明在getTask()方法中已经已经对workerCount进行了减1操作，这里就不必再减了。 
         */
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            /*
             * 统计完成的任务数
             */
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        /*
         * 根据线程池状态进行判断是否结束线程池
         */
        tryTerminate();

        /*
         * 当线程池是RUNNING或SHUTDOWN状态时，如果worker是异常结束，那么会直接addWorker；
         * 如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker；
         * 如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize。
         */
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }

```
至此，processWorkerExit执行完之后，工作线程被销毁，以上就是整个工作线程的生命周期，从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取任务，然后执行任务，如果getTask返回null，进入processWorkerExit方法，整个线程结束，如图所示:
![threadpool-lifecycle](/assets/images/2019/03/java_concurrent_threadpool-lifecycle.png)

### tryTerminate方法
tryTerminate方法根据线程池状态进行判断是否结束线程池，代码如下：
```java
    /**
     * Transitions to TERMINATED state if either (SHUTDOWN and pool
     * and queue empty) or (STOP and pool empty).  If otherwise
     * eligible to terminate but workerCount is nonzero, interrupts an
     * idle worker to ensure that shutdown signals propagate. This
     * method must be called following any action that might make
     * termination possible -- reducing worker count or removing tasks
     * from the queue during shutdown. The method is non-private to
     * allow access from ScheduledThreadPoolExecutor.
     */
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            /*
             * 当前线程池的状态为以下几种情况时，直接返回：
             * 1. RUNNING，因为还在运行中，不能停止；
             * 2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了；
             * 3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task；
             */
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            /*
             * 如果线程数量不为0，则中断一个空闲的工作线程，并返回
             */
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                /*
                 * 这里尝试设置状态为TIDYING，如果设置成功，则调用terminated方法
                 */
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        /*
                         * terminated方法默认什么都不做，留给子类实现
                         */
                        terminated();
                    } finally {
                        /*
                         * 设置状态为TERMINATED
                         */
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
interruptIdleWorkers(ONLY_ONE);的作用是因为在getTask方法中执行workQueue.take()时，如果不执行中断会一直阻塞。在下面介绍的shutdown方法中，会中断所有空闲的工作线程，如果在执行shutdown时工作线程没有空闲，然后又去调用了getTask方法，这时如果workQueue中没有任务了，调用workQueue.take()时就会一直阻塞。所以每次在工作线程结束时调用tryTerminate方法来尝试中断一个空闲工作线程，避免在队列为空时取任务一直阻塞的情况。

### shutdown方法
shutdown方法要将线程池切换到SHUTDOWN状态，并调用interruptIdleWorkers方法请求中断所有空闲的worker，最后调用tryTerminate尝试结束线程池。
```java
    /**
     * Initiates an orderly shutdown in which previously submitted
     * tasks are executed, but no new tasks will be accepted.
     * Invocation has no additional effect if already shut down.
     *
     * <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            /*
             * 安全策略判断
             */
            checkShutdownAccess();
            /*
             * 切换状态为SHUTDOWN
             */ 
            advanceRunState(SHUTDOWN);
            /*
             * 中断空闲线程
             */
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

```
下面是[深入理解Java线程池](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/)中的问题探究，非常深入，细节非常到位，👍

这里思考一个问题：在runWorker方法中，执行任务时对Worker对象w进行了lock操作，为什么要在执行任务的时候对每个工作线程都加锁呢？分析过程：
- 在getTask方法中，如果这时线程池的状态是SHUTDOWN并且workQueue为空，那么就应该返回null来结束这个工作线程，而使线程池进入SHUTDOWN状态需要调用shutdown方法；
- shutdown方法会调用interruptIdleWorkers来中断空闲的线程，interruptIdleWorkers持有mainLock，会遍历workers来逐个判断工作线程是否空闲。但getTask方法中没有mainLock；
- 在getTask中，如果判断当前线程池状态是RUNNING，并且阻塞队列为空，那么会调用workQueue.take()进行阻塞；
- 如果在判断当前线程池状态是RUNNING后，这时调用了shutdown方法把状态改为了SHUTDOWN，这时如果不进行中断，那么当前的工作线程在调用了workQueue.take()后会一直阻塞而不会被销毁，因为在SHUTDOWN状态下不允许再有新的任务添加到workQueue中，这样一来线程池永远都关闭不了了；
- 由上可知，shutdown方法与getTask方法（从队列中获取任务时）存在竞态条件；
- 解决这一问题就需要用到线程的中断，也就是为什么要用interruptIdleWorkers方法。在调用workQueue.take()时，如果发现当前线程在执行之前或者执行期间是中断状态，则会抛出InterruptedException，解除阻塞的状态；
- 但是要中断工作线程，还要判断工作线程是否是空闲的，如果工作线程正在处理任务，就不应该发生中断；
- 所以Worker继承自AQS，在工作线程处理任务时会进行lock，interruptIdleWorkers在进行中断时会使用tryLock来判断该工作线程是否正在处理任务，如果tryLock返回true，说明该工作线程当前未执行任务，这时才可以被中断。

下面就来分析一下interruptIdleWorkers方法。
### interruptIdleWorkers方法
```java
    /**
     * Common form of interruptIdleWorkers, to avoid having to
     * remember what the boolean argument means.
     */
    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    /**
     * Interrupts threads that might be waiting for tasks (as
     * indicated by not being locked) so they can check for
     * termination or configuration changes. Ignores
     * SecurityExceptions (in which case some threads may remain
     * uninterrupted).
     *
     * @param onlyOne If true, interrupt at most one worker. This is
     * called only from tryTerminate when termination is otherwise
     * enabled but there are still other workers.  In this case, at
     * most one waiting worker is interrupted to propagate shutdown
     * signals in case all threads are currently waiting.
     * Interrupting any arbitrary thread ensures that newly arriving
     * workers since shutdown began will also eventually exit.
     * To guarantee eventual termination, it suffices to always
     * interrupt only one idle worker, but shutdown() interrupts all
     * idle workers so that redundant workers exit promptly, not
     * waiting for a straggler task to finish.
     */
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

```
interruptIdleWorkers遍历workers中所有的工作线程，若线程没有被中断tryLock成功，就中断该线程。

为什么需要持有mainLock？因为workers是HashSet类型的，不能保证线程安全。

### shutdownNow方法
```java
    /**
     * Attempts to stop all actively executing tasks, halts the
     * processing of waiting tasks, and returns a list of the tasks
     * that were awaiting execution. These tasks are drained (removed)
     * from the task queue upon return from this method.
     *
     * <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     *
     * <p>There are no guarantees beyond best-effort attempts to stop
     * processing actively executing tasks.  This implementation
     * cancels tasks via {@link Thread#interrupt}, so any task that
     * fails to respond to interrupts may never terminate.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            /*
             * 中断所有工作线程，无论是否空闲
             */
            interruptWorkers();
            /*
             * 取出队列中没有被执行的任务
             */
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
shutdownNow方法与shutdown方法类似，不同的地方在于：
- 设置状态为STOP；
- 中断所有工作线程，无论是否是空闲的；
- 取出阻塞队列中没有被执行的任务并返回。
shutdownNow方法执行完之后调用tryTerminate方法，该方法在上文已经分析过了，目的就是使线程池的状态设置为TERMINATED。

## 怎么创建线程池
### Executors
- newSingleThreadExecutor
创建一个线程的线程池，在这个线程池中始终只有一个线程存在。如果线程池中的线程因为异常问题退出，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

- newFixedThreadPool
创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

- newCachedThreadPool
可根据实际情况，调整线程数量的线程池，线程池中的线程数量不确定，如果有空闲线程会优先选择空闲线程，如果没有空闲线程并且此时有任务提交会创建新的线程。在正常开发中并不推荐这个线程池，因为在极端情况下，会因为 newCachedThreadPool 创建过多线程而耗尽 CPU 和内存资源。

- newScheduledThreadPool
此线程池可以指定固定数量的线程来周期性的去执行。比如通过 scheduleAtFixedRate 或者 scheduleWithFixedDelay 来指定周期时间。

另外在写定时任务时（如果不用 Quartz 框架），最好采用这种线程池来做，因为它可以保证里面始终是存在活的线程的。

### 推荐使用 ThreadPoolExecutor 方式
在阿里的 Java 开发手册时有一条是不推荐使用 Executors 去创建，而是推荐去使用 ThreadPoolExecutor 来创建线程池。

这样做的目的主要原因是：使用 Executors 创建线程池不会传入核心参数，而是采用的默认值，这样的话我们往往会忽略掉里面参数的含义，如果业务场景要求比较苛刻的话，存在资源耗尽的风险；另外采用 ThreadPoolExecutor 的方式可以让我们更加清楚地了解线程池的运行规则，不管是面试还是对技术成长都有莫大的好处。

改了变量，其他线程可以立即知道。保证可见性的方法有以下几种：

volatile
加入 volatile 关键字的变量在进行汇编时会多出一个 lock 前缀指令，这个前缀指令相当于一个内存屏障，内存屏障可以保证内存操作的顺序。当声明为 volatile 的变量进行写操作时，那么这个变量需要将数据写到主内存中。

由于处理器会实现缓存一致性协议，所以写到主内存后会导致其他处理器的缓存无效，也就是线程工作内存无效，需要从主内存中重新刷新数据。

## 线程池监控
线程池的监控
通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

- getTaskCount：线程池已经执行的和未执行的任务总数；
- getCompletedTaskCount：线程池已完成的任务数量，该值小于等于taskCount；
- getLargestPoolSize：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize；
- getPoolSize：线程池当前的线程数量；
- getActiveCount：当前线程池中正在执行任务的线程数量。
通过这些方法，可以对线程池进行监控，在ThreadPoolExecutor类中提供了几个空方法，如beforeExecute方法，afterExecute方法和terminated方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自ThreadPoolExecutor来进行扩展。



## 总结
本文比较详细的分析了线程池的工作流程，总体来说有如下几个内容：

- 分析了线程的创建，任务的提交，状态的转换以及线程池的关闭；
- 这里通过execute方法来展开线程池的工作流程，execute方法通过corePoolSize，maximumPoolSize以及阻塞队列的大小来判断决定传入的任务应该被立即执行，还是应该添加到阻塞队列中，还是应该拒绝任务。
- 介绍了线程池关闭时的过程，也分析了shutdown方法与getTask方法存在竞态条件；
- 在获取任务时，要通过线程池的状态来判断应该结束工作线程还是阻塞线程等待新的任务，也解释了为什么关闭线程池时要中断工作线程以及为什么每一个worker都需要lock。


Q1:为什么要用线程池，有哪些优点？
- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。 
- 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。 
- 提高线程的可管理性

Q2:核心线程是如何一直保持的？能否让核心线程销毁呢？
getTask方法中
```java
    //从阻塞任务队列中取任务，如果设置了allowCoreThreadTimeOut(true)
    // 或者
    // 当前运行的任务数大于设置的核心线程数，那么timed =true
    boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

    //根据 timed true 或 false 来判断从哪里取任务，是否超时取任务
    Runnable r = timed ?
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            // take()会一直阻塞，等待任务的添加。
            workQueue.take();
```
如果当前线程数小于或者等于核心线程数（allowCoreThreadTimeOut默认false）,则timed为false。
timed为false表示取任务不超时（阻塞去任务），调用workQueue.take执行
设置allowCoreThreadTimeOut(true) 就能使核心线程销毁的呢，前提是必须设置setKeepAliveTime，否则线程超时时间没有，无法设置

```java
    /**
     * Sets the policy governing whether core threads may time out and
     * terminate if no tasks arrive within the keep-alive time, being
     * replaced if needed when new tasks arrive. When false, core
     * threads are never terminated due to lack of incoming
     * tasks. When true, the same keep-alive policy applying to
     * non-core threads applies also to core threads. To avoid
     * continual thread replacement, the keep-alive time must be
     * greater than zero when setting {@code true}. This method
     * should in general be called before the pool is actively used.
     *
     * @param value {@code true} if should time out, else {@code false}
     * @throws IllegalArgumentException if value is {@code true}
     *         and the current keep-alive time is not greater than zero
     *
     * @since 1.6
     */
    public void allowCoreThreadTimeOut(boolean value) {
        /*
         * value和keepAliveTime必须设置
         */
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
   
```

通过上面的分析，猜测一下这个测试方法的结果：
```java
    @Test
    public void coreThreadTimeOutTest() {
        /*
         * 设置核心线程和最大线程都是5个的线程池
         */
        ThreadPoolExecutor threadPoolExecutor =
                new ThreadPoolExecutor(5, 5,
                        0L, TimeUnit.MILLISECONDS,
                        new LinkedBlockingQueue<>(),
                        Executors.defaultThreadFactory());
        /*
         * 设置核心线程超时时间是10秒，可以销毁
         */
        threadPoolExecutor.setKeepAliveTime(10, TimeUnit.SECONDS);
        threadPoolExecutor.allowCoreThreadTimeOut(true);

        System.out.println("main thread start");

        /*
         * 不断执行线程，线程内部sleep 2秒
         */
        for (int i = 0; i < 15; i++) {
            threadPoolExecutor.execute(() -> {
                try {
                    System.out.println("线程：" + Thread.currentThread().getName() + "开始sleep");
                    Thread.sleep(2000);
                    System.out.println("线程：" + Thread.currentThread().getName() + "结束sleep");
                } catch (Exception e) {
                    System.out.println("执行异常");
                }
            });
        }

        /*
         * 主线程sleep 40秒
         */
        try {
            System.out.println("主线程开始 sleep");
            Thread.sleep(1000 * 40);
        } catch (Exception e) {
            System.out.println("主线程结束 sleep");
        }
    }
```
通过这篇线程池博客的梳理，将自己之前的很多疑惑解决了，感觉自己有了一些进步，同时参考了很多大牛的博客，向大牛致谢，感谢他们的指引。

## 参考
- [线程池的工作原理与源码解读](https://www.cnblogs.com/qingquanzi/p/8146638.html)
- [Java并发编程总结5——ThreadPoolExecutor](https://www.cnblogs.com/everSeeker/p/5632450.html)
- [深入理解Java多线程核心知识知识](https://www.cnblogs.com/lfs2640666960/p/10497806.html)
- [线程池---如何保证核心线程不被销毁的](https://blog.csdn.net/m0_37039331/article/details/87734270)
- [深入理解Java线程池：ThreadPoolExecutor](http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/)
