---
title: Java线程池原理
date: 2019-01-06 15:49:54
tags:
    - JavaMedium
---

Java线程池主要是为了复用宝贵的线程资源，其相关接口和具体实现在java.util.concurrent包下。

## 主要接口
### Executor
[Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html)是Java线程池的顶层接口，它只有一个execute方法
```java
void execute(Runnable command);
```
`Executor`的作用是让开发者能够专注于任务的提交，而不用关心任务具体怎样被执行（比如创建线程，配置线程，然后在这个新创建的线程里面执行，或者直接在主线程执行，或者每隔一段时间执行一次）。

### ExecutorService
[ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html)拓展了`Executor`接口，它提供了更多的方法来管理线程池和执行任务
```java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
* shutdown方法用于关闭线程池，调用该方法后，已经提交的任务还会被执行，但不会等待任务执行完成，并且不再接受新提交的任务。

* shutdownNow方法用于关闭线程池，调用该方法后，已经提交的任务（保持在队列中，还没有被执行）不会被执行，这些任务会被从队列中清除，然后作为返回值返回。该方法会停止线程池里面的任务的执行，但不保证一定能停止（如果任务不能响应中断信号的话任务还是会一直执行下去，不会停止，主要还是看任务能不能响应中断信号和怎样处理中断信号，毕竟调用该方法之后所做的只是把线程的中断标志设置为true）

* isShutdown方法用于检测线程池是否已经被关闭

* isTerminated方法用于检测在调用关闭线程池的方法后任务是否都已经执行完成了，如果没有调用关闭线程池的方法（shutdown，shutdownNow），该方法永远不会返回true

* awaitTermination方法用于在调用关闭线程池的方法后等待线程池里面的任务执行完成，该方法会一直阻塞，直到超过指定的时间（将返回false），或者当前线程被中断（将抛出`InterruptedException`异常，需要捕获异常并进行处理），或者线程池的任务全部执行完毕(将返回true)

* submit用于执行一个有返回值的异步任务，调用该方法将会立即返回一个`Future`对象，调用`Future`的get方法后将会阻塞，直到任务执行完成并返回一个值

* invokeAll方法用于批量执行有返回值的异步任务，调用该方法将立即返回一个`List<Future>`对象

* invokeAny方法用于批量执行有返回值的异步任务，调用该方法将返回最早执行完成的任务的返回值，该方法返回后其它任务将会被取消执行，所以它和invokeAll方法是不同的，如果没有异常情况，invokeAll方法执行的全部任务都能执行完成，而invokeAny方法只有一个任务能执行完成，其它未完成的任务将会被取消


### Future
[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)表示一个异步运算的执行结果，它提供了检测运算是否完成的方法，获取运算结果的方法，获取运算的方法等
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;            
}
```
* cancel方法用于取消运算

* isCanceelled方法用于检测运算是否被取消了

* isDone方法用于检测运算是否已经完成

* get方法用于获取运算结果，调用该方法将一直阻塞到运算完成（将返回运算的结果），或者超时(将抛出`TimeoutException`)，或者发生异常（将抛出`InterruptedException`或者`ExecutionException`）

### ThreadFactory
[ThreadFactory](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadFactory.html)是一个用于创建线程的Factory接口，它提供了newThread方法用于创建线程
```java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```
调用newThread方法将创建一个线程，并设置好线程的各种信息（比如线程的前缀名称，所属线程组，是否为守护线程等），然后返回创建的线程

`Callable`和`Runnable`接口应该很熟悉了，这里就不再复读了


## ThreadPoolExecutor
[ThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)是JDK线程池的一个实现类，这个类提供了很多的参数能够让开发者自行设置，也提供了一些默认的参数，基本上能满足大部分的需求。要掌握线程池，首先要知道它的每一个参数的作用（文档上有详细的讲解）。

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
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
这个构造方法是参数最多的一个构造方法，上面的参数就是开发者能够自定义的所有参数（还有一个allowCoreThreadTimeOut参数不能在构造器直接设置，不过可以通过allowCoreThreadTimeOut方法设置），

### corePoolSize
表示线程池中的核心线程数的最大值，核心线程是被创建之后就一直存在的线程，它不会由于空闲时间超过keepAliveTime就被停止（除非设置allowCoreThreadTimeOut为ture，默认是为false的，所以核心线程会一直存在）。如果当前的核心线程数小于corePoolSize，则每次有任务到来的时候都会创建一个新的线程来执行任务，直到当前的核心线程数等于corePoolSize才会复用线程。


### maximumPoolSize
表示线程池中的最大线程数，当线程池中的线程数等于这个值的时候（包括核心线程和非核心线程），将不能再创建更多的线程，合理地设置maximumPoolSize能够防止程序不断地创建新线程导致机器资源被耗尽。根据一般的经验，IO密集型的任务可以设置成CPU数×2 + 1，CPU密集型的任务可以设置成CPU数 + 1。当corePoolSize和maximumPoolSize相等的时候，表示这是一个固定线程池

### keepAliveTime
表示线程的最大空闲时间，线程的空闲时间（没有任务执行的时间）超过keepAliveTime后，线程会被停止。如果allowCoreThreadTimeOut为false，则只有非核心线程会由于超时被停止，否则线程池中的所有线程都会由于超时被停止。

### workQueue
它表示存放任务的队列，类型为`BlockingQueue<Runnable>`，当核心线程无法处理提交的任务的时候，任务会存放到workQueue，然后当核心线程空闲的时候，就会从workQueue中取出任务来执行。如果队列是有界队列，当队列满了之后（队列满了，核心线程也在执行其它任务），会创建新的线程（非核心线程）从队列中取出任务执行，如果线程池中的线程也达到了maximumPoolSize，则使用`RejectedExecutionHandler`处理后面提交的任务（队列已满，且线程池的线程数达到maximumPoolSize的情况下提交的任务将使用`RejectedExecutionHandler`进行处理），默认的hadlre为`AbortPolicy`（丢弃任务，抛出`RejectedExecutionException`）。

### threadFactory
创建线程的工厂类，负责创建线程，并设置好线程的属性，比如线程名称前缀，所属线程组，是否为守护线程等，默认的threadFactory为`DefaultThreadFactory`

### handler
`RejectedExecutionHandler`，用来处理被线程池拒绝执行的任务（一般都是因为任务堆积导致线程池处理不过来），JDK实现了4种拒绝策略，`CallerRunsPolicy`，`AbortPolicy`，`DiscardPolicy`和`DiscardOldestPolicy`。其中`AbortPolicy`是默认的策略。

* __CallerRunsPolicy__ 将把任务交给调用者处理，这会在调用者的线程直接执行该任务，如果该没有执行执行完成，则无法继续向线程池提交新的任务，这能有效的防止任务继续堆积

* __AbortPolicy__ 将把任务丢弃，然后抛出`RejectedExecutionException`

* __DiscardPolicy__ 将直接丢弃任务，不抛出任何异常（任务被悄悄丢弃）

* __DiscardOldestPolicy__ 将把队列中最老的（最早进入队列的）任务从队列取出，然后执行被拒绝执行的任务（不是刚刚从队列取出的，而是被拒绝执行的）

`RejectedExecutionHandler`在任务大量堆积导致线程池无法处理的时候起到很重要的作用，需要根据不同的场景设置不同的拒绝策略，比如任务不能被丢弃的时候，可以选择`CallerRunsPolicy`，在任务超过一定的时间没有执行可以被丢弃的时候，可以选择`DiscardOldestPolicy`

### 线程池的状态
```java
    private static final int RUNNING    = -1 << COUNT_BITS;  // 10000000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;  // 00000000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;  // 00100000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;  // 01000000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;  // 01100000000000000000000000000000
```
线程池有以上的5种状态

* __RUNNING：__ 该状态可以接受新的任务和执行已提交的任务，是线程池创建之后的初始状态

* __SHUTDOWN：__ 不接受新的任务，但可以执行已提交的（保持在队列中的）任务，调用shutdown方法后会切换到该状态

* __STOP：__ 不接受新的任务，不执行已提交（存放在队列中）的任务，调用shutdownNow方法后会切换到该状态

* __TIDYIN：__ 表示没有任务在执行，工作线程也为0了（没有线程在执行任务），准备执行terminated方法，调用了shutdown或者shutdownNow方法之后都有可能会转换到该状态

* __TERMINATED：__ terminated方法已经执行完成，在成功转换到TIDYING状态之后一定会继续转换到TERMINATED状态

几种状态直接可以这样转换：
```java
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
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
```
1. RUNNING -> SHUTDOWN
调用shutdown方法之后（或者显式调用finalize方法）会由RUNNING变成SHUTDOWN

2. (RUNNING or SHUTDOWN) -> STOP
调用shutdownNow方法之后可能会由RUNNING或者SHUTDOWN变成STOP

3. STOP -> TIDYING
当线程池为空（没有线程）的时候可能会由STOP变成TIDYING

4. TIDYING -> TERMINATED
当terminated方法执行完成的时候会由TIDYING变成TERMINATED

如果调用了awaitTermination方法，则这个方法会在线程池状态变成TERMINATED的时候返回


该线程池通过ctl变量来表示线程当前的状态和当前的工作线程数量
```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```
ctl变量为`AtomicInteger`类型，它是一个能够进行原子性自增或自减操作的`Integer`类型，COUNT_BITS为Integer.SIZE - 3，即32 -3 = 29，表示使用ctl低29位来存放线程池中工作线程的数量，高3位用来表示线程池的状态。

所以`ThreadPoolExecutor`目前能存放的线程数最多为2^29 - 1 = 536870911，二进制为00011111111111111111111111111111，所以即使设置maximumPoolSize为大于大于该值的数值，线程池最多可以创建的线程数也是536870911


```java
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

* runStateOf方法根据ctl变量的值计算出当前线程池的状态，具体是通过把ctl的值和CAPACITY按位去反后的值进行与运算
```
假设
ctl    =          10000000000000000000000000000000
~CAPACITY =       11100000000000000000000000000000
ctl & ~CAPACITY = 10000000000000000000000000000000 ---> RUNNING
```

* workerCountOf方法根据ctl变量的值计算出线程池当前的线程数量，具体是通过把ctl的值和CAPACITY直接进行与运算
```
假设
ctl    =          10000000000000000000000000000001
CAPACITY =        00011111111111111111111111111111
ctl & CAPACITY =  00000000000000000000000000000001 ---> 10进制值为1，表示有1个线程
```

* ctlOf方法根据当前线程池的状态和线程池的线程数量计算出ctl的值，具体是通过把线程池的状态对应的值和线程池的线程数进行或运算
```
假设
线程状态为RUNNING，即
rs      =            10000000000000000000000000000000
线程数量为1，即
wc      =            00000000000000000000000000000001
ctl =  rs | wc  =    10000000000000000000000000000001
```

### execute方法
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
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
![Imgur](https://i.imgur.com/9y8uyVA.png)

execute方法接收一个`Runnable`参数，其主要处理步骤如下：
1. 如果线程池当前线程数（workerCount）小于corePoolSize，则添加核心线程处理任务（这里将调用addWorker方法）

2. 如果线程池当前线程数大于等于corePoolSize，检测线程池的状态是否为RUNNING，如果线程池状态为RUNNING，则尝试将任务存放到队列中，如果成功将任务放入队列，则再次检测线程池的状态，如果线程池状态不为RUNNING，则调用remove方法从队列中移除任务，如果成功移除任务，则使用`RejectedExecutionHandler`处理任务，否则检查线程池的工作线程数是否为0，如果是，则添加线程非核心线程（只是添加一个非核心线程，任务为`null`）

3. 如果线程池的状态RUNNING或者状态为RUNNING但任务无法放入队列（可能是因为调用了shutdown或者shutdownNow方法，或者队列满了，队列满了以后将添加非核心线程处理新提交的任务），则添加非核心线程执行任务，如果添加线程失败了（可能由于调用了shutdown方法，线程数已达到maximumPoolSize等），则使用`RejectedExecutionHandler`处理任务

### addWorker方法
```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        // 这个循环将不断地从ctl变量中获取线程池的状态和线程数
        // 然后判断能否创建新的线程，如果可以，则更新ctl变量(线程数+1)
        // 更新失败就不断重试
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 这个if条件比较难理解，需要深入了解shutdown和shutdownNow方法
            // 和线程池的状态才能更好地理解这个if条件
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
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
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
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
addWorker是比较核心的一个方法，它会创建一个工作线程并启动该线程，接着工作线程会不断地从队列从取出任务（任务存放在阻塞队列，所以该操作会发生阻塞）并执行。主要步骤如下：
1. 获取线程数（wc）和线程池状态（rs）

2. 如果线程池状态为不为RUNNING，则判断线程池状态是否为SHUTDOWN，不为SHTUDOWN则直接返回`false`（因为这时候可能调用了shutdownNow方法，调用了shutdownNow方法后新提交的任务或者队列中的任务都不应该在执行，所以此时就不用再创建线程执行任务了，直接返回`false`即可），如果线程池状态为SHUTDOWN，则检查firstTask参数是否为`null`，不为`null`则直接返回`false`（因此这时候可能调用了shutdown方法，可以执行队列中的任务，但是不接受新提交的任务，firstTask不为`null`的话说明这是新提交的任务，为什么？因为只有线程数小于核心线程数或者队列满了以后才会将新提交的任务传给addWorker方法的firstTask参数），如果firstTask为`null`，则查看队列是否为空，如果队列为空则直接返回`false`（这时候线程池状态为SHUTDOWN，firstTask为`null`，说明调还要执行队列中剩余的任务，队列为空的话，说明已经没有任务了，所以也不再需要创建线程了，因此可以直接返回`false`）

3. 如果在第2部没有直接返回，则判断创建的线程是否为核心线程，如果为核心线程，则以corePoolSize为边界值，如果不是核心线程，则以maximumPoolSize为边界值，接着判断当前线程数（wc）是否超过边界值，超过了则直接返回`false`，没超过的话说明可以创建新的工作线程，接着调用compareAndIncrementWorkerCount方法原子性地更新ctl变量（线程数增加1），如果更新失败，说明ctl的值被其它线程并发修改了，所以则获取ctl的最新值，然后再次查看线程池的状态，如果线程池的状态改变了，则回到步骤1，如果没有，则再次执行步骤3的操作

4. 创建一个工作线程（类型为`Worker`），然后调用mainlock的lock方法获取锁，防止其它线程并发修改`workers`变量（类型为`HashSet<Worker>`，这是一个非线程安全的集合类，所以必须加锁来保证线程安全），成功获取锁后，再次检查线程池的状态，如果线程池状态为RUNNING或者状态为SHUTDOWN且firstTask为`null`(状态为SHUTDOWN且firstTask为`null`的时候说明调用了shutdown方法并且要执行的不是新提交的任务，此时可以执行队列中的任务，但是不接受新提交的任务，这时是可以创建线程来处理队列中剩余的任务的)，则继续检查线程线程是否已经启动，如果已经启动，则抛出异常，如果没有启动，则把创建的工作线程添加到workers集合中，然后更新largestPoolSize变量，调用mainlock的unlock方法释放锁

5. 如果创建的工作线程成功添加到workers集合中，则启动工作线程，工作线程启动后实际上会调用runWorker方法来执行任务

6. 如果创建的线程没能成功启动（可能由于`ThreadFactory`没能成功创建线程实例），则调用个addWorkerFailed方法回退步骤345的操作，比如把线程数减少1，把添加到workers集合的`Worker`实例移除

### shutdown方法
```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 将线程池状态设置为SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断空闲线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        // 尝试将线程池状态设置为TERMINATED
        tryTerminate();
    }
```
shutdown方法的逻辑比较简单的，主要步骤如下：
1. 调用mainLock的lock方法获取锁（主要是防止workers变量被其它线程同时修改，因为它不是线程安全的）

2. 调用checkShutdownAccess检查调用者是否有权限关闭线程池的线程

3. 调用advanceRunState方法将线程池状态设置为SHUTDOWN

4. 调用interruptIdleWorkers中断空闲线程，空闲线程即没有正在执行的任务的线程（一般空闲线程都是阻塞在队列的获取元素的方法），所谓的中断线程，实际上就是调用`Thread`的interrupt方法设置线程的中断标志为true，相当于向线程发送了一个中断信号，而不是直接把线程停止，如果任务里面没有相应中断信号的逻辑，则对任务不会有任何的影响，任务会一直执行到结束

5. 调用onShutdown方法，该方法需要子类实现（默认什么也不做），一般用来做监控

6. 释放锁（这时候就释放锁是因为后面已经不需要访问workers变量了）

7. 调用tryTerminate尝试将线程池状态设置为TERMINATED


shutdown方法调用返回后，线程池的状态可能为SHUTDOWN（当调用了shutdown后，队列中还有任务为执行完成的话就会是这个状态），也可能为TERMINATED


### shutdownNow方法
```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
shutdownNow方法和shutdown方法有几个不同的地方，首先，shutdownNow方法是有返回值的，它会返回队列中的任务；其次，shutdownNow方法先将线程池状态设置为STOP，而不是SHUTDOWN；最后，shutdownNow方法会中断线程池的所有线程，而不仅仅是空闲线程，这说明shutdownNow方法调用之后，会停止正在执行的任务。该方法的主要步骤如下：
1. 调用mainLock的lock方法获取锁

2. 调用checkShutdownAccess检查调用者是否有权限关闭线程池的线程

3. 调用advanceRunState方法将线程池状态设置为STOP

4. 调用interruptWorkers方法中断线程池的所有线程，是所有线程，而不仅仅是空闲线程，正在执行的任务也会被中断（前提是任务能响应中断），这里的中断也是调用`Thread`的interrupt方法设置中断标志为ture，所以对不能响应中断的任务不会产生任何影响

5. 调用drainQueue方法，移除队列中的所有任务，然后把这些任务保持到tsks变量

6. 释放锁

7. 调用tryTerminate尝试将线程池状态设置为TERMINATED

8. 返回tasks

shutdownNow方法调用返回后，线程池的状态可能为STOP，也可能为TERMINATED

### tryTerminate方法
```java
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
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
tryTerminate会尝试将线程池状态设置为TERMINATED，该状态表示线程数为0并且队列中没有任务，一般有两情况能成功设置，第一种情况是当线程池状态为SHUTDOWN且线程数为0且队列中没有任务的时候，第二种情况是线程池状态为STOP且线程数为0的时候。该方法的主要步骤如下：
1. 获取ctl变量的值

2. 判断线程池的状态，如果状态为RUNNING(没有直接由RUNNING到TERMINATED的转换)，或者状态为TIDYING以上（包括TIDYING和TERMINATED），或者状态为SHUTDOWN并且队列不为空（队列中的任务还需要执行，所以还不能设置为TERMINATED），则直接返回

3. 如果线程数不为0，则调用interruptIdleWorkers停止一个空闲的线程，然后返回

4. 调用mainLock的lock方法获取锁（因为后面调用了mainLock的`Condition`实例的signalAll方法，这个方法需要在获取锁后才能调用，不然会抛出异常）

5. 设置线程池的状态为TIDYING，线程数设置为0，设置失败则释放锁然后回到第1步

6. 如果第5步设置成功，则调用terminated方法，然后再设置线程的状态为TERMINATED，线程数设置为0，然后调用`Condition`实例的signalAll方法唤醒正在等待的线程（通常等待的线程为调用了awaitTermination方法的线程，调用了awaitTermination方法后线程会一直阻塞直到线程池的状态变成TERMINATED）

7. 释放锁然后返回（因为释放锁的操作在finally块里面，所以释放锁的操作一定会被执行）

### interruptWorkers方法
```java
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 遍历所有工作线程，然后调用线程的interrupt方法中断线程
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
```
这个方法会中断所有的工作线程，将会遍历workers集合，然后调用集合的线程的interrupt中断线程，实际上只是把线程的中断标志设置为true，并不会是直接停止线程

### interruptIdleWorkers方法
```java
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                    	// 调用Thread的interrupt方法中断线程
                    	// 实际上只是设置线程的中断标志为true
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                // 如果仅中断一个空闲线程，则在遍历完第一个工作线程的时候就会返回
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
这个方法只会中断空闲的线程（没有在执行任务的线程），当对应的线程在执行任务的时候，因为无法获取到对应的工作线程的锁，所以不会中断正在执行任务的线程，只会中断空闲的线程（一般空闲线程阻塞在队列获取元素的方法上）


### runWorker方法
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
```
创建工作线程后，会启动对应的线程，然后线程会执行runWorker方法，这个方法会调用提交的任务（`Runnable`）的run方法，在调用任务的run方法的前后还会执行两个方法，beforeExecute和afterExecute，这两个方法需要子类实现，默认是什么也不做的。这个方法的主要步骤如下：
1. 如果firstTask不为空，则将firstTask作为第一个task

2. 将工作线程的firstTask变量设置为`null`，因为firstTask应该只被执行一次，如果这里不设置为空的话，当前线程后面将会一直执行firstTask，而无法执行其它的任务

3. 释放工作线程占用的锁，使得该线程能接收中断信号（因为在中断空闲线程的时候，会先获取工作线程的锁，然后再中断工作线程里面的线程，如果这里不释放锁，将无法中断空闲线程）

4. 如果task为空（现在的task为第一步的firstTask），调用getTask方法从队列中获取任务，如果获取到的任务为`null`，则跳转到最后一步

5. task不为空，则先获取锁，现在获取锁主要是为了避免线程在执行任务的时候被当做空闲线程中断，这也是中断空闲线程的时候无法中断正在执行任务的线程的原因（中断空闲线程要获取对应的工作线程的锁，而工作线程在执行任务的时候占用了锁）

6. 如果线程池正在关闭（调用了shutdownNow方法），则需要确保线程被中断（将线程的中断标注设置为true，将调用`Thread`的interrupt方法），因为判断的时候调用了Thread.interrupted方法，该方法会清除线程的中断标志（设置为false），所以需要再次调用`Thread`的interrupt方法将中断标志设置为ture

7. 调用beforeExecute方法，这个方法可能抛出异常，如果抛出了异常，最终会导致工作线程停止运行（线程停止前会调用processWorkerExit方法，因为该方法在finally块中被调用了）

8. 执行任务（`Runnable`）的run方法，这里也可能抛出异常，抛出异常后和第7步一样

9. 执行afterExecute方法，这个方法可能抛出异常，抛出异常后也和第7步一样

10. 将task设置为`null`，completedTasks自增1，释放锁

11. 执行processWorkerExit方法，如果上面的7,8,9的任何一步抛出了异常，都会将任务视为异常结束，如果任务是异常结束的，则该方法进行一些额外的处理


### processWorkerExit方法
```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 因为如果任务不是正常完成的话，一般是发生了异常
        // 如果是线程是正常地被当作空闲线程中断的话，则会自动地减少线程数
        // 但是如果线程是在执行任务的过程中被中断的，
        // 应该手动调用decrementWorkerCount减少工作线程数
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        // 要访问线程安全的workers变量，需要加锁
        mainLock.lock();
        try {
        	// 统计该工作线程完成的任务数
            completedTaskCount += w.completedTasks;
            // 将当前工作线程从workers集合中移除
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
        // 尝试将线程状态设置为TERMINATED
        // 因为线程终止可能是因为线程池被关闭了
        tryTerminate();

        int c = ctl.get();
        // 检测线程池的状态，如果状态为RUNNING或者SHUTDOWN
        // 则说明可能还有任务需要执行
        if (runStateLessThan(c, STOP)) {
        	// 如果线程是正常退出的，比如被当作空闲线程中断
        	// 则判断是否有必要添加一个工作线程取代它
            if (!completedAbruptly) {
            	// 如果核心线程运行超时，则最小线程数可以为0，否则至少为核心线程数
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // 如果最小线程数可以为0，但是队列中还有任务，则至少需要一个工作线程来执行任务
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果当前工作线程数大于等于min，则没有必要添加工作线程取代退出的线程
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 如果线程是异常退出的，则一定会添加一个工作线程取代它
            addWorker(null, false);
        }
    }
```
这个方法在工作线程停止的执行一些清理和记录操作，清理操作是把该工作线程从workers集合里面移除，记录工作是记录该工作线程执行完成的任务数。如果线程是异常终止的，则检测当前线程池的状态是否为RUNNING或者SHUTDOWN状态，如果是，并且队列中还有任务，则检查当前线程数是否大于等于核心或者大于等于1（究竟是检测大于等于核心线程数还是大于等于1，需要看allowCoreThreadTimeOut的取值），如果不是，则需要添加一个非核心线程来取代这个异常退出的工作线程（因为还有任务需要执行，而当前线程异常终止了，所以需要创建一个线程取代它继续执行任务）

## AbstractExecutorService
[AbstractExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/AbstractExecutorService.html)是`ThreadPoolExecutor`的父类，它实现了submit，invokeAny和invokeAll等方法（这些方法是由`AbstractExecutorService`类实现的）

### submit方法
```java
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        // 将task包装成一个RunnableFuture
        // RunnableFuture继承了Runnable和Future接口
        RunnableFuture<T> ftask = newTaskFor(task, result);
        // 将调用ThreadPoolExecutor实现的execute方法
        execute(ftask);
        return ftask;
    }
```
submit方法一共有3个，上面的只是其中的1个，它接收一个`Runnable`类型的参数和一个任意类型的参数，`Runnable`类型的参数就是要执行的任务，任意类型的参数用于确定任务执行完成的返回值的类型。这个方法的主要步骤如下：
1. 检测任务是否为`null`，如果是则抛出异常

2. 调用newTaskFor方法将task封装成一个`RunnableFuture`对象，并根据result确定任务执行结果的类型

3. 调用`ThreadPoolExecutor`实现的execute方法执行上面的`RunnableFuture`（实际上不一定是调用`ThreadPoolExecutor`的execute方法，如果子类重写了execute方法，则调用子类重写的execute方法）

4. 返回第2步创建的`RunnableFuture`对象

### newTaskFor方法
```java
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```
newTaskFor方法有2个，上面的是其中的1个，它接收一个`Runnable`类型的参数和一个任意类型的参数，`Runnable`类型的参数就是要执行的任务，任意类型的参数用于确定任务执行完成的返回值的类型。newTaskFor方法直接创建一个`FutureTask`对象并返回，`FutureTask`类实现了`RunnableFuture`接口，`RunnableFuture`接口继承了`Runnable`和`Future`接口，具体类图如下：

![Imgur](https://i.imgur.com/wOfolGV.png)

### FutureTask.run()
```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
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
这个方法是`FutureTask`实现的run方法，最终会在线程池的线程里面执行这个方法，可以看到它将调用`Callable`的call方法执行任务并获得返回值（`Callable`实例是调用`Executors`的callable方法创建的），然后调用set方法把结果存储到outcome变量里面，然后当我们调用get方法的时候就能获取到任务的执行结果了。

```java
    // FutureTask的其中一个构造器
    public FutureTask(Runnable runnable, V result) {
        // 当传进来的是一个Runnable对象的时候，会调用Executors.callable方法
        // 将Runnable对象封装成Callable对象
        this.callable = Executors.callable(runnable, result);
        // FutureTask创建完成的初始状态为NEW
        this.state = NEW;       // ensure visibility of callable
    }
    
    // set方法，用于把任务执行结果保持到outcome变量中
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        	// 这里把任务执行结果保持到了outcome变量
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

    // get方法，用于获取任务执行结果，如果任务没有执行完成会一直阻塞到任务执行完成
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
        	// 等待任务执行完成，将一直阻塞到任务执行完成
            s = awaitDone(false, 0L);
        // 调用report方法获取结果
        return report(s);
    }
    
    // report方法，用于从outcome变量中取出结果
    private V report(int s) throws ExecutionException {
        // 从outcome变量取出结果，outcome变量为Object类型
        Object x = outcome;
        if (s == NORMAL)
        	// 将结果转换成对象类型
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }    
```

### FutureTask.cancel()
```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                    	// 发送中断信号，如果任务能响应中断信号，则有可能会成功取消
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```
`Future`是可以取消执行的，具体是怎么取消的可以看上面的代码。实际上是通过调用`Thread`的interrupt方法设置正在执行任务的线程的中断标志为true，具体能不能取消，需要看这个任务能否响应中断信号（比如调用会抛出`InterruptedException`的方法，或者把代码包裹在Thread.interrupted方法中），如果不能，那么这个calcel方法对任务不会产生任何影响。

一个例子：
```java
public class ThreadPoolTest {
    public static void main(String[] args) throws InterruptedException {
        // 新建一个线程池执行任务
        ThreadPoolExecutor exec = new ThreadPoolExecutor(1, 1, 0, TimeUnit.MICROSECONDS, new ArrayBlockingQueue<Runnable>(1));
        // 新建一个任务
        Runnable r = new Runnable() {
            public void run() {
            	// 死循环，每隔2秒输出一次task is running...
                while (true){
                    System.out.println("task is running...");
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                    	// 这里虽然能响应中断限号，但是接收到中断信号后没有退出循环
                    	// 所以任务将无法被取消，将会一直执行先去
                    	// 所以任务能不能够取消取决于任务的具体实现
                        System.out.println("interrupted!");
                    }
                }
            }
        };
        // 执行任务
        Future<StringBuffer> result = exec.submit(r, null);
        // 等待3秒，确保任务开始执行之后在取消任务
        // 如果任务还没有开始执行，是能够成功取消的
        // 因为任务开始之前会检测相应的状态位，状态为不对的话直接返回，不执行任务
        Thread.sleep(3000);
        // 取消正在执行的任务
        result.cancel(true);
        System.out.println("task was canceled.");
    }
}
```
程序运行结果如下图：

![Imgur](https://i.imgur.com/I7NIw0f.png)

可以看到即使执行了cancel方法，任务仍然在运行。原因是任务收到中断信号后没有退出循环。

## Executors
[Executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)是`Executor`的工厂类和工具类，它提供了相应的方法创建如下的几种对象：
1. `ExecutorService`，具体类型有[ThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)，[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)，`FinalizableDelegatedExecutorService`，`DelegatedExecutorService`

2. `ScheduledExecutorService`，具体类型有`DelegatedScheduledExecutorService`，[ScheduledThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)

3. `ThreadFactory`，具体类型有`DefaultThreadFactory`，`PrivilegedThreadFactory`

4. `Callable` 具体类型有`RunnableAdapter`和匿名`Callable`类

### ExecutorService
`Executors`可以创建的`ExecutorService`有以下几种：

1. newFixedThreadPool
固定线程池，它可以创建的线程数是固定的，它的核心线程数（corePoolSize）和最大线程数（maximumPoolSize）相同，需要在参数里面指定，使用的是无边界队列`LinkedBlockingQueue`，因为线程数是固定的，可以自己配置，所以合理的配置线程数可以有限地避免因为大量线程创建导致系统资源耗尽，但是它的队列是无边界的，所以当大量任务到达的时候，如果线程处理不过来，也会使得大量任务堆积在队列中，最终导致内存溢出。

2. newSingleThreadExecutor
只有一个线程的线程池，它的核心线程数（corePoolSize）和最大线程数（maximumPoolSize）都是1，使用的是无边界队列`LinkedBlockingQueue`，它只有一个线程用于执行任务，所以效率不高，如果有大量的任务到达的时候，这些任务将会堆积在队列中，由于队列是无边界的，所以任务的大量堆积最终可能导致内存溢出。可以看到固定线程池和单线程池都可能会由于大量任务堆积在队列导致内存溢出，所以应尽量避免使用这些线程池，直接使用`ThreadPoolExecutor`，根据不同的使用场景配置线程数和队列。

3. newCachedThreadPool
缓存线程池，它可以创建的线程数是没有限制的，核心线程数为0，说明所有的工作线程都不是核心线程，线程超时时间默认为60秒，使用的是同步队列`SynchronousQueue`，这个队列的特点是入队列操作和出队列操作必须在不同的线程同时进行，比如要往队列插入一个元素的时候，必须有另一个线程在队列中取出元素，要从队列中取出一个元素的时候，必须有另一个线程在队列中插入元素，所以这个队列的入队列和出队列操作是必须一一对应的，把一个元素入队列之后必须取出元素之后才能再次入队列，这个队列是没有容量的，也就是说这个队列不存储元素，元素入队列之后，必须被取出才能继续进行入队列操作。因为可以创建线程数没有限制（Integer.MAX_VALUE，相当于没有限制），所以当任务多的时候，已经创建的线程无法处理这么多的任务，同时队列是同步队列，无法存储任务，所以有大量新任务提交的时候会不断创建新的线程来处理任务（已经创建的线程执行完当前的任务后也可以处理新提交的任务，所以线程是可以复用的，但是任务太多的时候，所以线程都在执行任务，这时候只能创建新的线程来执行任务），最终导致系统资源耗尽，所以这个线程池应该谨慎使用。这个线程池和前面线程池相比，不会由于任务堆积在队列导致内存溢出，但是会由于创建大量的线程导致系统资源耗尽。

4. newWorkStealingPool
这个线程池能充分利用CPU的多个核心来提升性能，不是很懂，先看看[介绍](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)吧（上面的几个线程池都是基于`ThreadPoolExecutor`创建的，而这个线程池有它自己的想法）




