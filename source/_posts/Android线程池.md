---
title: Android线程池
date: 2018-07-16 14:09:26
tags: [Android,线程]
categories: Android
---

## 为什么使用线程池
线程是操作系统能进行运算调度的最小单元，在Java 中直接使用线程，给我们带来了很多便利，但是线程的使用同时也存在一些问题
* 线程生命周期的开销非常高，即在线程的创建和销毁过程都会消耗较大的cpu资源
* 资源消耗，线程的存在期间会消耗系统资源，尤其是内存（短时间内高并发任务尤其需要注意）

## 线程池
线程池就是一种线程复用的手段，它通过缓存已有线程，来减小线程创建过程的消耗，它通过控制线程数量来控制线程存在的系统消耗，同时他把任务和任务的执行进行了解耦，把任务本身和任务的执行过程分离。这一点从Executor就可以看出

    public interface Executor {
    
        void execute(Runnable command);
    }
任务封装成Runnable，通过execute在内部通过缓存的线程对任务进行处理，对于Runnable这个任务而言，具体的执行策略它毫不知晓，同时我们可以有极大的空间来制定执行策略。
## 构造线程池
线程池给我们创造了极大扩展空间用于管理线程资源和定义任务的执行策略，同时为了更加方便我们使用，Java定义了一些配置成型的线程池供我们使用，这些线程池可以通过Executors这样一个工厂类获取
* newFixedThreadPool。newFixedThreadPool会创建一个固定长度的线程池，每当提交一个任务时，就会创建一个线程（即使有空闲的线程），直到达到线程最大值。
* newCachedThreadPool。newCachedThreadPool会创建一个可缓存的线程池，任务提交时没有空闲的线程就会创建新的线程，且这个过程没有上线，当线程执行完成后60s内没有被其他任务复用就会被销毁。
* newSingleThreadExecutor。newSingleThreadExecutor是一个单线程的Executor，一般按照FIFO的顺序执行（不同于我们自己创建的单线程Thread，如果Thread不慎挂掉，会创建一个新的线程保证运行过程）
* newSingleThreadExecutor。创建一个固定长度线程池，会使用延迟或者定时的方式来执行任务。

上面列举了几个Java中提供好的线程策略，类似的在Executors中还有很多，这里不一一列举，本质上来说，这些方法是对现有的几个线程池实现类的一种配置方式，具体线程池的复用以及调度的方式是怎么样的呢？
## 线程池调度策略
### 调度内容
线程池是对线程的一个管理，它到底管理什么东西
* 使用什么线程执行任务
* 任务执行顺序
* 有多少任务可以同时处理
* 能够有多少任务等待，如果需要执行的任务超过这个上限如何处理
* 任务运行时出现异常怎么处理
* 任务前后有哪些动作

### ThreadPoolExecutor
我们使用的绝大部分线程池最终的实现类都是ThreadPoolExecutor，我们可以通过这个对象来真正了解线程池，首先是ThreadPoolExecutor的构造方法

     public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
从构造方法可以看出很多内容，ThreadPoolExecutor中的线程分为普通线程和核心线程，有线程存活时间，任务等待队列，线程工厂，任务拒绝的handler。
### 核心线程数和最大线程数
关于核心线程和最大线程可以看一下execute方法的一段代码

    //线程数小于核心线程直接添加新线程执行
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
    }
    //将任务放入等待队列，同时检测线程池运行状态并处理
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //线程池状态改变（不是运行状态）直接打回任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    //等待队列满了，试图添加新线程，如果失败打回任务
    //（当线程数以达到最大线程数则会失败）
    }else if (!addWorker(command, false))
        reject(command);
ctl是一个AtomicInteger类型，它用前3位标示线程池运行状态，后面位数表示已有的线程数量，当线程池新增一个任务时，会有四种情况
1. 当前线程数小于核心线程数，直接新建线程执行任务
2. 大于核心线程数，等待队列未满，直接进入队列
3. 等待队列已满，且小于最大线程数，直接新建线程执行任务
4. 都不满足，打回任务

可以看出即使等待队列时FIFO，任务也不一定会完成按照我们添加的顺序执行，当等待队列满的时候，一部分任务会优先执行。

### 线程复用与回收
线程池最大的特点就是线程的复用，同时还有上一节中的非核心线程的回收，本质上来说核心线程和非核心线程没有去吧，他们都存储在一个集合中

    private final HashSet<Worker> workers = new HashSet<>();
线程池中的线程都使用Worker进行封装，线程复用的过程也是借助Workder来进行的，Worker的主要代码如下

     private final class Worker extends AbstractQueuedSynchronizer
        implements Runnable{
       
        ......
        final Thread thread;
        Runnable firstTask;
        ......
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
    
        /** Delegates main run loop to outer runWorker. */
        public void run() {
            runWorker(this);
        }
        ......
    }
我们可以看到Worker本身就是Runnable，他在创建内部线程的同时只是把自己作为参数传递进去，最终thread.start运行时执行的就是Worker的run方法，同时这里会调用ThreadPoolExecutor的runWorker方法以完成最终的线程调度。

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        ......
        try {
            //getTask从等待队列中获取下一个task
            while (task != null || (task = getTask()) != null) {
                ......
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
可以看到在runWorker方法里，Worker内的task为空，会从等待队列中获取一个新的task并执行，同时会调用beforeExecute和afterExecute这两个生命周期方法，同时当任务队列没有清空或者没有异常发生时，这是一个死循环，如果跳出循环，则说明要么任务队列被清空，要么线程异常结束，processWorkerExit会处理这些情况，如果是异常结束会起一个新的线程，否则则会移除大于核心线程数的线程，到此线程的复用过程基本清楚了

我们可以看到ThreadPoolExecutor还有一个keepAliveTime的参数用于空闲时回收非核心线程的时机，这个过程事实上是在getTask获取下一个任务时进行的


    ​private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?
        for (;;) {
            int c = ctl.get();
            ......
    
            int wc = workerCountOf(c);
            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            ......
      
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
可以看到当目前的线程数大于核心线程数时，获取任务时会执行workQueue.poll方法，这个方法在队列空时会等待给定的时间然后才会返回，如果在规定的时间仍然没有新的任务，则会在上面的processWorkerExit回收线程。

除此之外线程池还有等待队列、线程工厂、任务拒绝的handler等要点，由于时间有限，这里暂时不一一说明，以后有时间再做补充，大家也可自行探索。

## 需要注意的问题
线程池十分好用，但是使用中也有些需要注意的点，这里我稍稍列举一些，希望大家使用中可以注意

* 不要使用线程池执行可能会相互依赖的任务，因为线程池不一定能保证执行顺序，很有可能会发生线程死锁
* 任务需要考虑可能出现的同步问题，因为任务不是在单线程环境下运行
* 对时间敏感的任务，不要使用线程池，因为线程池可以保证执行但是无法保证执行的时间
* 运行时间较长的任务尽量不要和时间较短的任务一起执行，这会严重影响短耗时任务的执行效率