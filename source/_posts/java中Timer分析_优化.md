---
title: Java SDK中Timer的分析
date: 2020-06-08
categories: Java
tags: [Timer, JDK源码]
---

Java SDK的Timer是一个比较方便的定时器工具类，用来定时或周期性地执行用户提交的任务。其主要解决了简单的任务调度问题，让用户可以简单地提交和取消定时任务。同时Timer的线程安全性也保证了多个线程可以并发地对同一个任务进行调度和取消。

本博文尝试分析一下SDK中的Timer实现。大致从几个方面去介绍：

1. What : Timer是什么，解决什么问题，怎么使用。
2. How&Why : 内部是怎么实现的，为什么要那么实现。有哪些优点和局限。
3. When : 什么场景下适合使用Timer。

## 一、Timer是什么

> A facility for threads to schedule tasks for future execution in a background thread. Tasks may be scheduled for one-time execution, or for repeated execution at regular intervals.

Timer是JDK1.3引入的定时任务执行器。JDK文档说明了Timer是一个任务执行器，它可以在后台执行一次性或者周期性的任务。比如，我们在日常编码中，可能经常会碰到一个场景，需要定时清理缓存、日志，又或者有些场景需要周期性更新某个状态。那么Timer就是替我们做这些事情的。然而由于Timer的局限性，JDK1.5又引入了ScheduledThreadPoolExecutor，提供更稳定更精确且功能更多的定时执行器。

> Java 5.0 introduced the `java.util.concurrent` package and one of the concurrency utilities therein is the [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html) which is a thread pool for repeatedly executing tasks at a given rate or delay. It is effectively a more versatile replacement for the `Timer`/`TimerTask` combination, as it allows multiple service threads, accepts various time units, and doesn't require subclassing `TimerTask` (just implement `Runnable`). Configuring `ScheduledThreadPoolExecutor` with one thread makes it equivalent to `Timer`.

<!--more-->

### Timer的使用场景案例

tomcat源码example中有个snake的示例程序。功能是使用websocket实现一个贪吃蛇的游戏。每隔100ms，程序需要更新贪吃蛇在屏幕中的位置。那么更新位置信息就可以设置成一个周期性任务。下面是snake程序的代码片段。

```java
public class SnakeTimer {
	private static Timer gameTimer = null;
	private static final long TICK_DELAY = 100;
	
	// 启动定时执行器
  public static void startTimer() {
    // 创建一个定时执行器
    gameTimer = new Timer(SnakeTimer.class.getSimpleName() + " Timer");
    
    // 向定时执行器提交一个周期性任务。
    gameTimer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                tick();  // 更新位置信息
            } catch (RuntimeException e) {
                log.error("Caught to prevent timer from shutting down", e);
            }
        }
    }, TICK_DELAY, TICK_DELAY);
  }
  
  protected static void tick() {
      StringBuilder sb = new StringBuilder();
      for (Iterator<Snake> iterator = SnakeTimer.getSnakes().iterator();
              iterator.hasNext();) {
          Snake snake = iterator.next();
          snake.update(SnakeTimer.getSnakes());
          sb.append(snake.getLocationsJson());
          if (iterator.hasNext()) {
              sb.append(',');
          }
      }
      broadcast(String.format("{\"type\": \"update\", \"data\" : [%s]}",
              sb.toString()));
  }

  public static void stopTimer() {
      if (gameTimer != null) {
          gameTimer.cancel();
      }
  }
}使用Timer比较简单，创建一个Timer，然后向Timer提交一个任务。Timer后台线程会在指定的时间自动执行任务。向Timer提交任务主要有两个方法。```schedule```和```scheduleAtFixedRate```，一个是提交一次性任务，一个是提交周期性任务。
```

## 二、Timer内部的实现

### 2.1 几个内部组件

根据Timer所要实现的功能需求，大致可以分解出几个组件。

1. Task，用户提交的任务，想要Timer替我们执行任务，用户得创建一个Task，也就是这个Task得implements Runnable。与Timer协同的任务是TimerTask抽象类。

   ```java
   public abstract class TimerTask implements Runnable {
       public abstract void run();
   }
   ```

   用户任务UserTask需要extends TimerTask，实现run函数，将任务所需要做的事情和访问的资源统一在run函数里面实现。TimerTask需要有个任务状态的标识，标识这个任务处于什么状态，待运行、正在运行、被取消、执行完成。另外，TimerTask还需要有个标识来记录这个Task是一次性的还是周期执行的。

   ```java
   public abstract class TimerTask implements Runnable {
     int state = VIRGIN; // 任务状态。没有用enum,为什么
     static final int VIRGIN = 0;
     static final int SCHEDULED = 1;
     static final int EXECUTED = 2;
     static final int CANCELLED = 3;
     
     long nextExecutionTime; // 下一次执行的时间戳，Timer中小顶堆队列是按照task的这个属性进行比较大小
     long period = 0; // 任务执行周期，0表示一次性任务
     
     public abstract void run();
   }
   ```

   

2. Timer, 任务执行器。任务执行器执行用户提交的任务，显然需要一个队列来保存任务，同时需要一个内部线程来执行任务。队列中存放的是待运行的任务，这些任务都有个属性，nextExecutionTime下一次运行的时间戳。内部线程在while(true)循环里面执行逻辑：从队列中找出时间戳最近的任务，如果当前时间戳已经达到或超过任务运行时间戳，就执行task.run()来执行任务，对于周期性任务还会将下一次执行的任务实例再次插入队列。这里涉及到几个问题：a. 任务队列用什么数据结构，由于内部线程每次只需要取一个最近时间的任务，容易想到堆或者优先队列。SDK Timer内部使用数组实现的小顶堆。b. 内部线程在没有任务执行或者堆顶任务运行时间还没有到的时候，不能耗费CPU，需要阻塞wait，等待其它用户线程来唤醒或者自己等到一段时间再run。c. 用户任务，Timer内部队列需要线程安全。用户任务为什么需要线程安全？可能用户线程A提交后，用户线程B由于某种原因需要提前终止它。而对于Timer内部队列，可能会有多个用户线程往Timer提交任务，所以也得是线程安全的。

   ```java
   public class Timer {
       private final TaskQueue queue = new TaskQueue(); // 内部任务队列
       private final TimerThread thread = new TimerThread(queue); // 内部执行线程
       private static final AtomicInteger nextSerialNumber = new AtoicInteger(0);
   }
   ```

   从上面代码可以看出，每个Timer的实例都有一个内部线程和队列。Timer实例对其内部线程进行了编号。

3. TaskQueue，任务队列，是Timer的内部类，就是一个数组实现的小顶堆。TaskQueue内部没有锁，它的线程安全性是由Timer在访问TaskQueue的时候加锁。

4. TaskThread，任务线程，也是Timer的内部类。由于TaskThread需要访问TaskQueue，所以在TaskThread构造的时候将Timer的TaskQueue注入进去了。

   ```java
   class TimerThread extends Thread {
     public void run() {
        try {
          mainLoop();
        } finally {
          synchronized(queue) {
            newTaskMayBeScheduled = false;
            queue.clear();
          }
        }
     }
     
     private void mainLoop() {
       while (true) {
           try {
               TimerTask task;
               boolean taskFired;
               synchronized(queue) {
                   // Wait for queue to become non-empty
                   while (queue.isEmpty() && newTasksMayBeScheduled)
                       queue.wait();
                   if (queue.isEmpty())
                       break; // Queue is empty and will forever remain; die
   
                   // Queue nonempty; look at first evt and do the right thing
                   long currentTime, executionTime;
                   task = queue.getMin();
                   synchronized(task.lock) {
                       if (task.state == TimerTask.CANCELLED) {
                           queue.removeMin();
                           continue;  // No action required, poll queue again
                       }
                       currentTime = System.currentTimeMillis();
                       executionTime = task.nextExecutionTime;
                       if (taskFired = (executionTime<=currentTime)) {
                           if (task.period == 0) { // Non-repeating, remove
                               queue.removeMin();
                               task.state = TimerTask.EXECUTED;
                           } else { // Repeating task, reschedule
                               queue.rescheduleMin(
                                 task.period<0 ? currentTime   - task.period
                                               : executionTime + task.period);
                           }
                       }
                   }
                   if (!taskFired) // Task hasn't yet fired; wait
                       queue.wait(executionTime - currentTime);
               }
               if (taskFired)  // Task fired; run it, holding no locks
                   task.run();
           } catch(InterruptedException e) {
           }
       }
     }
   }
   ```

   我们来看看```TimerThread.run()```的实现。就一个mainLoop循环。外面用try包裹一下，有个finally块负责清理queue。我们看mainLoop是不会抛异常的(queue.wait可能会抛的InterruptedException被mainLoop内部catch住了，task.run也不会抛异常)，为什么TimerThread.run里面要将mainLoop包在try finally块里面呢？原因在于：虽然TimerThread本身在执行mainLoop时不会抛InterruptedExecption，但是外部用户线程可以向TimerThread发送中断信号，于是mainLoop得用try finally块包裹一下。

   下面是一个概要图。

   ![Timer](http://image.dzmiba.com/Timer.jpg)

   

### 2.2 优点和局限

Timer的优点：

1. 实现简单，使用起来也很简单。
2. 轻量级，不需要很多系统资源，只有一个后台线程和一个内部任务数组。支持取消任务。
3. 线程安全。Timer内部执行任务是线程安全的。多个线程可以同时使用同一个Timer实例。用户任务自身的线程安全性由用户保证。

Timer的局限：

1. 无法保证一个任务在精确的时间执行。由于Timer内部只有一个线程去执行任务，假如线程在执行某个任务过程中耗费了太多时间，或者卡住了，那么Timer内部队列里面的任务都会受到影响，因为从内部队列中取任务的也是这个内部线程。它卡住了，就不能保证队列中的任务在预期的时间被取出来执行。所以执行时间比较短的轻量级定时task比较适合使用Timer。



## 三、Timer内部实现的一些分析

### 3.1 Timer内部线程安全性

#### 3.1.1 内部queue的线程安全性

外部用户向Timer提交任务时，Timer需要将任务加到内部queue中，同时Timer内部thread需要从queue中取出任务执行，这里有race condition，queue必须要加锁。所以内部线程while循环里面，对queue的访问，使用了```synchronized(queue)```。外部线程调用schedule或者scheduleAtFixedRate时，同样也使用了```synchronized(queue)```。外部线程调用cancel时，需要清空queue，也一样使用了同步锁。

#### 3.1.2 Timer内部循环线程对Task也加锁

从上面Timer内部mainLoop代码中看到，内部线程从queue中取出**最小任务**后，使用了```synchronized(task.lock)```来修改task的数据，包括修改task.state, 将task重新放回queue。为什么要对task锁一下呢。主要是因为，在task放入Timer之后，可能有用户线程对task执行cancel操作，修改task.state。看下面TimerTask.cancel方法，也加锁了。

```java
    public boolean cancel() {
        synchronized(lock) {
            boolean result = (state == SCHEDULED);
            state = CANCELLED;
            return result;
        }
    }
```

 #### 3.1.3 TimerThread.newTaskMayBeScheduled起什么作用

```TimerThread```内部有个```newTaskMayBeScheduled```有什么作用呢，这个变量标识Timer不会接收新任务了，Timer执行完队列里面的任务可能会终止。初始时，这个标识是true，外部对调用Timer.cancel时，会将该标识置false，同时清空队列。如果之后还有外部线程再提交任务，会触发异常，但是不会影响当前正在执行的任务。看下面Timer.cancel的代码。

```java
/**
Terminates this timer, discarding any currently scheduled tasks. Does not interfere with a currently executing task (if it exists). Once a timer has been terminated, its execution thread terminates gracefully, and no more tasks may be scheduled on it.
Note that calling this method from within the run method of a timer task that was invoked by this timer absolutely guarantees that the ongoing task execution is the last task execution that will ever be performed by this timer.
This method may be called repeatedly; the second and subsequent calls have no effect.
*/
public void cancel() {
    synchronized(queue) {
        thread.newTasksMayBeScheduled = false;
        queue.clear();
        queue.notify();  // In case queue was already empty.
    }
}
```

最后一行代码，queue.notify()起什么作用？注释说"In case queue was already empty"是什么意思？这是为了确保TimerThread停止下来。再回到TimerThread.mainLoop代码，可以看到，如果外部线程调用Timer.cancel时，TimerThread正在执行任务，当任务执行完成后，mainLoop会终止下来。但是如果当外部Timer.cancel时，queue已经空了，TimerThread阻塞在```queue.wait()```上，这个时候Timer.cancel不调用```queue.notify()```，TimerThread将永远阻塞在那里，不能终止。

## 四、Timer使用场景

Timer是Java SDK1.3引入的，SDK1.5时引入了ScheduledThreadPoolExecutor更能更强大，也更稳定，更精确。大多数时候，应该使用ScheduledThreadPoolExecutor。Timer优势在于使用简单，轻量级，比较适合轻量级任务且精度要求不高的场景，比如周期性发送心跳包，周期性清理缓存，周期性更新状态。
