---
title: Threads
date: 2018-08-04 14:01:22
tags:
    - EssentialClasses
---

{% meting "537854742" "netease" "song" %}

## 创建线程
Java创建线程有两种方式
* 实现Runnable接口<br>
```angularjs
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Runnable");
    }
});
```
* 继承Thread类<br>
```angularjs
class MyThread extends Thread{
        @Override
        public void run() {
            System.out.println("Extends Thread");
        }
}
Thead t = new MyThread();
```

## 启动线程
启动线程需要调用Thread的start方法，这样才能真正的启动一个新的线程，
直接调用run方法的话相当于在主线程调用一个方法，并不能启动一个新的线程。
```angularjs
t.start(); //这样就能启动了一个新的线程
```

## 暂停执行线程
要使线程暂停执行，可以使用Thread类的sleep方法，这个方法是静态方法.
* Thread.sleep(long millis)
使当前线程暂停执行millis毫秒
* Thread.sleep(long millis, int nanos)
使当前线程暂停执行millis毫秒+nanos微秒

时间精度取决于操作系统和底层硬件。
执行了这个方法后，当前线程会被挂起（暂停执行）。
在暂停执行的这段时间内它占用的CPU和其他资源可以让给其他线程使用。

## Interrupts
>An interrupt is an indication to a thread that it should stop what it is doing and do something else. It's up to the programmer to decide exactly how a thread responds to an interrupt, but it is very common for the thread to terminate. This is the usage emphasized in this lesson.

中断表明线程应该停止当前正在执行的任务并转去执行其他任务，这取决于开发者的实现，通常的做法是在线程收到中断信号时终止该线程。
```angularjs
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        for(int i=0;i<999999999;i++){
            doHeavyTask(); /*执行某项任务*/
             if(Thread.interrupted()){
                 return ;  
                 //一般的做法，收到中断信号，直接终止线程
             }
        }
    }
});
```

>A thread sends an interrupt by invoking interrupt on the Thread object for the thread to be interrupted. For the interrupt mechanism to work correctly, the interrupted thread must support its own interruption.

可以通过调用线程的interrupt方法向该线程发送中断信号
```angularjs
public static void main(String[] args){
    Thread t = new Thread(new Runnable(){  
    @Override
    public void run() {
      /*do something*/
    }
    });
    t.start();
    t.interrupt(); // 这样就向线程t发送了中断信号
}
```

中断机制能正常运作的前提是该线程能正确地处理中断信号，比如：
```angularjs
for (int i = 0; i < importantInfo.length; i++) {
    // Pause for 4 seconds
    try {
        Thread.sleep(4000);
    } catch (InterruptedException e) {
        // We've been interrupted: no more messages.
        return;
    }
    // Print a message
    System.out.println(importantInfo[i]);
}
```
Thread.sleep()有可能会抛出中断异常，当捕获到该异常的时候，说明该线程收到了中断信号。
收到中断信号后，需要对中断信号作出正确的处理才能使中断机制正常运作，如果收到中断信号后
继续当无事发生，中断机制就不能正常运作了（类似于遥控器失灵了）。

在上面的代码中，捕获到中断异常（接受到中断信号）后，直接从方法中返回。

当没有调用sleep这些会抛出中断异常的方法的时候，需要反复调用Thread.interrupted()
方法使得线程能够响应中断。
```angularjs
for (int i = 0; i < inputs.length; i++) {
    heavyCrunch(inputs[i]);
    if (Thread.interrupted()) {
        // We've been interrupted: no more crunching.
        return;
    }
}
```

## The Interrupt Status Flag
中断机制的内部实现使用的是一个内部标志，这个标志叫做interrupt status(中断状态)。
调用Thread对象的interrupt方法会将这个标志设置为true，表示线程被中断。
下面讲解一下几个检查这个标志的方法。
* ### Thread.interrupted<br>
该方法为Thread类的静态方法，先来看看这个方法的实现：<br>
```angularjs
public static boolean interrupted() {
     return currentThread().isInterrupted(true);
}
```
currentThread()将返回当前正在运行的线程，所以这个isInterrupted方法是Thread类的方法。

接下来看isInterrupted方法的实现：<br>
```angularjs
private native boolean isInterrupted(boolean ClearInterrupted);
```
这个方法是个本地方法，看具体的实现比较麻烦，我们直接看它的文档:
>/**
       * Tests if some Thread has been interrupted.  The interrupted state
       * is reset or not based on the value of ClearInterrupted that is
       * passed.
  */

这个方法可以获取线程的中断状态，然后根据参数ClearInterrupted的值决定是否重置中断状态，
重置中断状态就是将中断状态设置为false。

回到上面的interrupted方法，这个方法调用了isInterrupted(true)，可以看到它将重置中断状态。
所以调用Thread.interrupted()将获取当前线程的中断状态，接着重置中断状态。

```angularjs
Thread.interrupted(); //将得到false, 因为线程没有被中断
Thread.currentThread().interrupt(); //中断线程
Thread.interrupted(); //将得到true，因为线程被中断了，然后重置中断状态
Thread.interrupted(); //将得到false，因为上面重置了中断状态
Thread.interrupted()； //同上
Thread.currentThread().interrupt(); //中断线程
Thread.interrupted(); //将得到true，因为线程被中断了，然后重置中断状态
Thread.currentThread().interrupt(); //中断线程
Thread.interrupted(); //将得到true，因为线程又被中断了，然后重置中断状态
```

* ### Thread.isInterrupted<br>
该方法为Thread类的实例方法，看该方法的实现：<br>
```angularjs
public boolean isInterrupted() {
    return isInterrupted(false);
}
```
可以看到和上面的静态方法Thread.interrupted差不多，就是ClearInterrupted为false,
即不重置中断状态。

因此，调用这个方法，只返回线程的中断状态，而不修改中断状态，因此调用这个方法不会改变线程的中断状态

```angularjs
Thread.currentThread().isInterrupted(); //将得到false, 因为线程没有被中断
Thread.currentThread().interrupt(); //中断线程
Thread.currentThread().isInterrupted()//将得到true，因为线程被中断了
Thread.currentThread().isInterrupted() //将得到true，因为线程被中断了，然后中断状态没有被重置
Thread.currentThread().isInterrupted() //同上
```

>The non-static isInterrupted method, which is used by one thread to query the interrupt status of another, does not change the interrupt status flag

文档上说这个方法是用来在一个线程里面查询另一个线程是否被中断的，因此该方法不重置线程的中断状态<br>
```angularjs
Thread t1 = ...;
if((new Date()).getMonth() == 2){
   t1.interrupt();
}
if(t1.isInterrupted()){
  //不会重置线程t1的中断状态
  /*domeSOmeghing...*/
}
```

## Joins
join方法是Thread类的实例方法，用于等待某个线程执行完成。
* join()<br>
直接使用Thread实例调用即可，如`t.join()`
该方法可能抛出InterruptedException，当该线程被其它线程中断时，就会抛出这个异常。
当抛出这个异常时，该线程的中断状态会被重置。

* join(long millis)
等待线程执行完成，最多等待millis毫秒，millis为0时表示无限期等待。
* join(long millis, int nanos)
等待线程执行完成，最多等待millis毫秒+nanos微秒

## yield
这个方法一般在测试并发程序运行效果的时候用到，作用是让出该线程的CPU资源。

文档原话：
>A hint to the scheduler that the current thread is willing to yield its current use of a processor. The scheduler is free to ignore this hint.<p>
Yield is a heuristic attempt to improve relative progression between threads that would otherwise over-utilise a CPU. Its use should be combined with detailed profiling and benchmarking to ensure that it actually has the desired effect.<p>
It is rarely appropriate to use this method. It may be useful for debugging or testing purposes, where it may help to reproduce bugs due to race conditions. It may also be useful when designing concurrency control constructs such as the ones in the java.util.concurrent.locks package.

笔记来源： [Threads](https://docs.oracle.com/javase/tutorial/essential/concurrency/threads.html)


