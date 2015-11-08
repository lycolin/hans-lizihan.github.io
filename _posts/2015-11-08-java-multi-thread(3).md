---
layout:     post
title:      java 多线程 (3) - schedule
date:       2015-11-08 12:07
summary:    java 多线程基础 - 流程控制
categories: java
---

暴力点说 thread scheduling 的原始出发点是将 thread 归类。因为默认而言 jvm 会给所有的 thread 安排恰当的的 `context switch` 然而对于一些重 cpu 运算的 thread 而言，他们占用更长的 cpu 时间。而且 thread 并不是 `input -> output` 这么简单，很可能有些 thread 的 intermidiate result 是另一个 thread 的 `dependency`。所以如何保证大家的调用顺序就尤其重要了。

## Thread States
1. initial state (一个 Thread 实例被创建出来但是并没有被 `start` 调用)
2. runnable state (`start` 调用了之后 jvm 会默认安排各个 thread 的顺序)
3. blocked state (`sleep`, `wait`, `join`, 或者等待 `synchronized` 解锁, 使得一个 runnable state 的 Thread 没有办法 `run`)
4. terminating state (`run block` 结束了， Thread 实例等待被销毁)

## Thread scheduling categories
简单说就是 java 为了优化 thread 能力 会根据平台的不同而实行不同的 scheduling 策略。 对于没有 OS 级别的 thread scheduling 的系统来说，使用 jvm 内置的 thread scheduling system。对于 OS 有 thread scheduling 的平台，使用默认的 OS 级别接口以提高效率。

### Green thread Scheduling
Green thread / User thread 就是 JVM 自己的 thread scheduler

两张 javaworld 的图来说话

![java-thread-lower-priority](/assets/img/java-threads-scheduling-lower-priority.gif)

上面这副图中 `reading thread` 有最高的优先级 `calculation thread` 优先级最低。

green scheduling 中永远是首先满足 top priority thread，当其blocking 的时候来满足 second top priority ... 以此类推。

![java-thread-lower-priority](/assets/img/java-threads-scheduling-equal-priority.gif)

上面这副图表示 `main` 和 `calculation` thread 优先级相等。此时 jvm 会分别给两个 thread 公平的机会执行。

#### priority inversion
就是说低等级的 thread 可能会优先高等级thread 来跑。

先来普及下 java priority 的等级制度。 

> 1 - 10 10 最高 1 最低。

想象一下下面的情景

1. Three threads with priority 6 5 4, respectively
2. priority 5 requires a lock holds in priority 4. 
3. priority 6 blocks
4. priority 5 cannot be excecuted
5. priority 4 excecutes 

#### priority inheritance
java 的解决方案是
如果一个 低级的 thread 有一个 高级的 thread 的锁，那么 jvm 会默认将此 thread 提升优先级

用上面的例子就是 虽然 priority 4 真实的优先级只有 4, 但是实际上在执行的它的时候 jvm 静默地将其的优先级变成了 9

### Navitve thread scheduling
这个才是大众的 scheduler. 2015 年了，没有 thread scheduler 的操作系统会被打死吧。

其实打的方向上面 Native 和 Green 是基本相同的。唯一不同的是对待同等 priority 的 thread 时 native 会想多一层

#### time-slicing
简单讲就是设定一个时钟，对于 thread1 和 thread2, 如果他们的 priority 一样得话就给他们上一个倒计时。如果变成了 0 就交给他的兄弟去跑。

#### starvation problem
除此之外还有 有些 低级的 thread 总是得不到运行的机会，这就是当时课本里的 windows 32 priority 中 后 16 个 priority 不断动态变化而跑的例子了。

## setPriority()
很简单的设定 优先级的方法 

## yeild()
简单讲就是告诉 jvm, hi 我是个 thread，能不能赏脸让我爽一把？ 然后 scheduler 就会看看现在的进程池然后相应地安排下。如果说有比你更高优先级的 thread 在前面那就没戏了。如果你正好有个兄弟在跑，那么这时候你发出的信号可能会有回报了。

Javadoc 中对这个方法很不推崇。说只有 debug 的时候才应该用，因为这个后面的黑魔法太多了，而且会因为系统有没有 `time-slicing` 而输出不同的结果。所以统一最后都用 setPriority() 来调度各个 thread 的优先级。

## wait/notify

这部分是非常重要的, wait/notify 避免死锁的一个很关键的点。

首先明确一点 `wait` 和 `notify` 方法 直接定义在了 `object` 这个类上。它并不是 Thread 中的方法。

但是考虑下面的情景。

1. A tests condition and discovers it must wait
2. B set the condition and calls notify() to inform a to resume
3. A is not yet waiting, so nothing happens
4. A waits and because A raced the `notify` call, so a waits forever

所以使用 `wait` 和 `notify` 的场景中一点要在一段 `synchronized` 的环境里才行

``` java
// Condition variable initialized to false to indicate condition has not occurred.
boolean conditionVar = false;
// Object whose lock threads synchronize on.
Object lockObject = new Object ();
// Thread A waiting for condition to occur...
synchronized (lockObject) {
  while (!conditionVar) {
    try {
       lockObject.wait ();
    } catch (InterruptedException e) {}
  }
}
// ... some other method
// Thread B notifying waiting thread that condition has now occurred...
synchronized (lockObject) {
  conditionVar = true;
  lockObject.notify ();
}
```

这里应该还是比较有道理的。

### Producer-consumer relationship

``` java
class Main {
  public static void main (String [] args)  {
    Shared s = new Shared ();
    new Producer(s).start ();
    new Consumer(s).start ();
  }
}
class Shared {
  private char c = '\u0000';
  void setSharedChar (char c) { this.c = c; }
  char getSharedChar () { return c; }
}
class Producer extends Thread {
  private Shared s;
  Producer (Shared s)   {
    this.s = s;
  }
  public void run ()  {
    for (char ch = 'A'; ch <= 'Z'; ch++) {
      try {
        Thread.sleep ((int) (Math.random () * 4000));
      }catch (InterruptedException e) {}
      s.setSharedChar (ch);
      System.out.println (ch + " produced by producer.");
    }
  }
}
class Consumer extends Thread {
  private Shared s;
  Consumer (Shared s)   {
    this.s = s;
  }
  public void run ()  {
    char ch;
    do{
      try{
        Thread.sleep ((int) (Math.random () * 4000));
      } catch (InterruptedException e) {}
      ch = s.getSharedChar ();
      System.out.println (ch + " consumed by consumer.");
    } while (ch != 'Z');
  }
}
```

上面的例子非常好地阐明了 `producer-consumer` 的关系。

然而问题是 
```
consumed by consumer.
A produced by producer.
```

当 `producer` 还没有生产第一条 character 的时候 `consumer` 就亟不可待地去 `consume` 了。 这并不是 synchronized 的问题，而是调度问题。

``` java
class Main {
  public static void main (String [] args) {
    Shared s = new Shared ();
    new Producer (s).start ();
    new Consumer (s).start ();
  }
}

class Shared {
  private char c = '\u0000';
  private boolean writeable = true;

  synchronized void setSharedChar (char c) {
    while (!writeable) {
      try{
        wait ();
      } catch (InterruptedException e) {}
    }
    this.c = c;
    writeable = false;
    notify ();
  }

  synchronized char getSharedChar ()  {
    while (writeable) {
      try {
        wait ();
      } catch (InterruptedException e) { }
    }
    writeable = true;
    notify ();

    return c;
  }
}

class Producer extends Thread {
  private Shared s;

  Producer (Shared s)   {
    this.s = s;
  }

  public void run ()  {
    for (char ch = 'A'; ch <= 'Z'; ch++)  {
      try {
        Thread.sleep ((int) (Math.random () * 4000));
      } catch (InterruptedException e) {}

      s.setSharedChar (ch);
      System.out.println (ch + " produced by producer.");
    }
  }
}

class Consumer extends Thread {
  private Shared s;

  Consumer (Shared s)   {
    this.s = s;
  }

  public void run ()  {
    char ch;

    do {
      try{
        Thread.sleep ((int) (Math.random () * 4000));
      }catch (InterruptedException e) {}

      ch = s.getSharedChar ();
      System.out.println (ch + " consumed by consumer.");
    } while (ch != 'Z');
  }
}
```

可以看到其中的重点在 `shared` 这个类上。

每一次更新状态的之前 `shared`, shared 会让 producer `wait()`，知道 `consumer` 拿到了 (`getSharedChar`)。 所以这样可以实现一个多线程的 `p-c` 的架构。

## Thread interruption
每一次调用 `thread` 相关的方法总是会捕获一个 `InterruptedException` 的玩意儿。为啥？

因为 thread 是可能被打断的。

下面就是一个例子

``` java
class ThreadInterruptionDemo {
   public static void main (String [] args) {
      ThreadB threadB = new ThreadB ();
      threadB.setName ("B");
      ThreadA threadA = new ThreadA (threadB);
      threadA.setName ("A");
      threadB.start ();
      threadA.start ();
   }
}
class ThreadA extends Thread {
  private Thread threadB;
  ThreadA (Thread threadB) {
    this.threadB = threadB;
  }
  public void run () {
    int sleepTime = (int) (Math.random () * 10000);
    System.out.println (getName () + " sleeping for " + sleepTime + " milliseconds.");
    try {
      Thread.sleep (sleepTime);
    } catch (InterruptedException e) {}
    System.out.println (getName () + " waking up, interrupting other thread and terminating.");
    threadB.interrupt ();
  }
}
class ThreadB extends Thread {
  int count = 0;
  public void run () {
    while (!isInterrupted ()) {
      try {
        Thread.sleep ((int) (Math.random () * 10));
      } catch (InterruptedException e) {
        System.out.println (getName () + " about to terminate...");
        // Because the Boolean flag in the consumer thread's thread
        // object is clear, we call interrupt() to set that flag.
        // As a result, the next consumer thread call to isInterrupted()
        // retrieves a true value, which causes the while loop statement
        // to terminate.
        interrupt ();
      }
      System.out.println (getName () + " " + count++);
    }
  }
}
```

### interrupt
其中要注意的是 interrupt() 这个方法 这个方法十分复杂。 

1. 当 thread 在 `join sleep wait` 这三种状态时，interrupt 会抛出 `InterruptedException` 异常并且不去改变 `interrupted flag`
2. 当 thread 在 正常运行时它会改变 `interrupted flag` 使得 `Thread.isInterrupted()` 返回 true

参考: 

* [java-word-thread-3](http://www.javaworld.com/article/2071214/java-concurrency/java-101--understanding-java-threads--part-3--thread-scheduling-and-wait-notify.html)
