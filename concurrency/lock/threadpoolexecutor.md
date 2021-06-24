

一组线程消费队列中的任务

实现了任务执行和任务之间的解耦

1. 任务队列

   阻塞队列

2. 线程池

   1. 线程池的扩容和缩容
   2. 临界值的控制

3. 状态管理

   运行

   停止 

## 状态

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ThreadPoolExecutor线程池有5个状态，分别是：

1. RUNNING：可以接受新的任务，也可以处理阻塞队列里的任务
2. SHUTDOWN：不接受新的任务，但是可以处理阻塞队列里的任务
3. STOP：不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
4. TIDYING：过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
5. TERMINATED：终止状态，terminated方法调用完成以后的状态

状态跃迁

![state_switch](pics/state_switch.png)

添加任务

线程池要执行任务，那么必须先添加任务，execute()虽说是执行任务的意思，但里面也包含了添加任务的步骤在里面，下面源码：

```java
// java.util.concurrent.ThreadPoolExecutor
public void execute(Runnable command) {
  if (command == null)
    throw new NullPointerException();

  int c = ctl.get();
  // 1. 首先判断工作线程数量是否超过核心线程数量
  if (workerCountOf(c) < corePoolSize) {
    // 新建核心线程执行任务
    if (addWorker(command, true))
      return;
    c = ctl.get();
  }
  
  // 2. 线程池仍然是运行状态并且任务队列未满，将任务压入队列
  if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    // 若线程池不是运行状态，移除任务并拒绝
    if (! isRunning(recheck) && remove(command))
      reject(command);
    else if (workerCountOf(recheck) == 0)
      addWorker(null, false);
  }
  // 3. 任务队列满了，则创建非核心线程执行任务
  else if (!addWorker(command, false))
    reject(command);
}
```

画一下execute执行任务的流程图：

<img src="pics/add_worker.png" alt="add_worker" style="zoom:67%;" />

新建线程

1. 占位
2. 创建线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
	// 1. 占位
  retry:
  for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c);

    // 如果线程池的状态到了SHUTDOWN或者之上的状态时候，只有一种情况还需要继续添加线程，
    // 那就是线程池已经SHUTDOWN，但是队列中还有任务在排队,而且不接受新任务（所以firstTask必须为null）
    // 这里还继续添加线程的初衷是，加快执行等待队列中的任务，尽快让线程池关闭
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
      // 原子增加工作线程数量
      if (compareAndIncrementWorkerCount(c))
        break retry;
      c = ctl.get();
      if (runStateOf(c) != rs)
        continue retry;
      // 由于 workerCount 改变导致 CAS 失败，重试
    }
  }

  // 创建线程
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

        // 线程池处于 RUNNING 或者 （SHUTDOWN并且初始任务为null）时才创建线程
        if (rs < SHUTDOWN ||
            (rs == SHUTDOWN && firstTask == null)) {
          if (t.isAlive()) // precheck that t is startable
            throw new IllegalThreadStateException();
          workers.add(w);
          int s = workers.size();
          // largestPoolSize表示线程池中创建过的线程的最大数
          if (s > largestPoolSize)
            largestPoolSize = s;
          workerAdded = true;
        }
      } finally {
        mainLock.unlock();
      }
      if (workerAdded) {
        // 启动线程
        t.start();
        workerStarted = true;
      }
    }
  } finally {
    if (! workerStarted)
      // 回滚
      addWorkerFailed(w);
  }
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
  final ReentrantLock mainLock = this.mainLock;
  mainLock.lock();
  try {
    if (w != null)
      workers.remove(w);
    decrementWorkerCount();
    tryTerminate();
  } finally {
    mainLock.unlock();
  }
}
```

线程的封装

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {

        // 线程，工厂创建失败可能为null
        final Thread thread;
        // 初始任务，可能为null
        Runnable firstTask;
        // 每个线程完成任务的计数器
        volatile long completedTasks;
  
        Worker(Runnable firstTask) {
            setState(-1); // 直到调用runWorker，否则禁止中断
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            runWorker(this);
        }

  			// 0 - 未锁定状态
  			// 1 - 锁定状态
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

线程执行

先执行初始任务，再循环取得任务

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

取任务

```java
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

