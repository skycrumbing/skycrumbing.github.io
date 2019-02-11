---
layout: post
title: JAVA线程池ThreadPoolExecutor
tags:
- thread
categories: java
description: JAVA线程池ThreadPoolExecutor
---
## ThreadPoolExecutor
ThreadPoolExecutor是java线程池的核心类，有三个关键属性：  

<!-- more -->

**corePoolSize**：核心线程池大小，在这个数量内的线程默认不可终止  
**maximumPoolSize**：最大线程数，包括核心线程数  
**keepAliveTime**：当线程池中的线程数大于corePoolSize时，这个属性生效。当线程空闲超过这个时间会被终止。核心线程池可通过allowCoreThreadTimeOut(boolean)让这个属性生效。  


## 执行流程
我们通过ThreadPoolExecutor的execute(Runnable command)方法提交一个线程。  
### execute(Runnable command)  
1，当线程池中的线程数小于corePoolSize 时，新提交的任务直接新建一个线程执行任务（不管是否有空闲线程）  
2，当线程池中的线程数等于corePoolSize 时，新提交的任务将会进入阻塞队列（workQueue）中，等待线程的调度  
3，当阻塞队列满了以后，如果corePoolSize < maximumPoolSize ,则新提交的任务会新建线程执行任务，直至线程数达到maximumPoolSize当线程数达到maximumPoolSize 时，新提交的任务会由(饱和策略)管理  
```
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
### addWorker(Runnable firstTask, boolean core)  
1，当线程池的状态大于shutdown或者状态等于shutdown并且任务为空以及阻塞队列为空时直接返回fase。  
2，无限循环，当任务的数量大于最大容量，或者无论核心线程大小还是最大线程大小超标后直接返回fasle，否则没问题直接增加任务数量并且进行下一步。  
3，创建worker，将线程放入worker进行装饰，并且执行同步代码块。  
4，在代码块中再次判断线程池状态，并且向workes（任务集合）中添加worker，记录历史最大的线程数，workerAdd置为true  
5，执行任务
由于线程被worker进行装饰，此时的start()方法指向的是worker对象的run()方法，worker的run()方法又指向了runWorker(Worker w)方法，重点就在这个方法里。  
```
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
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
### runWorker(Worker w)
1，主要看while循环。如果task不为空执行task，否则从getTask()中取任务。在执行完任务后会在finally块中设置task = null; 设置完之后继续循环执行getTask()。  
2，如果task和getTask()都为空，completedAbruptly置为false，表示是正常结束线程，而不是意外  
3，在finally块中执行processWorkerExit将当前worker移除  

```
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
### getTask()  
重点就在这个无限循环的try代码块中。**如果允许核心线程池失效或者当前运行的线程数大于核心线程池大小，就通过poll（如果在规定时间内内成功地移除了队列头元素，则立即返回头元素；否则在到达超时，返回null。）取阻塞队列里的线程，否则通过take（移除并返回队列头部的元素。如果队列为空，则阻塞）取阻塞队列里的线程。**
```
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

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

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
### processWorkerExit(Worker w, boolean completedAbruptly)
1，判断是不是意外结束的线程，如果是就workcount-1  
2，completedTaskCount进行增加，表示总共完成的任务数，并且从WorkerSet中将对应的Worker移除  
3，调用tryTemiate，进行判断当前的线程池是否处于SHUTDOWN状态，判断是否要终止线程池  
4，判断当前的线程池状态，如果当前线程池状态比STOP大的话，就不处理  
5，判断是否是意外退出，如果不是意外退出的话，那么就会判断最少要保留的核心线程数，如果allowCoreThreadTimeOut被设置为true的话，那么说明核心线程在设置的KeepAliveTime之后，也会被销毁。   
6，如果最少保留的Worker数为0的话，那么就会判断当前的任务队列是否为空，如果任务队列不为空的话而且线程池没有停止，那么说明至少还需要1个线程继续将任务完成。  
7，判断当前的Worker是否大于min，也就是说当前的Worker总数大于最少需要的Worker数的话，那么就直接返回，因为剩下的Worker会继续从WorkQueue中获取任务执行。  
8，如果当前运行的Worker数比当前所需要的Worker数少的话，那么就会调用addWorker，添加新的Worker，也就是新开启线程继续处理任务。  
```
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

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
## 总结
线程池能够一直等待任务执行而不被销毁的原因是当线程池没有线程时会进入阻塞状态直到线程池出现新的线程

