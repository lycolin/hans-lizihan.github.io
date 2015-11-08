---
layout:     post
title:      java 多线程 (1)
date:       2015-11-07 18:40
summary:    java 多线程基础
categories: java
---

java 中多线程的能力让给人又爱又恨。相比较 `Node.js` 里面开心的 eventloop，java 的 thread 简直就是灾难。

先来总结下关键词来享受一下世界的恶意

## Keywords

1. synchronized
2. volatile
3. race condition
4. dead lock
5. mutual exclusion
6. monitor
7. object lock
8. Runnable
9. isAlive()
10. join()
11. wait()
12. notify()
13. notifyAll()

## 创建 Thread
好吧 一个个来... 首先呢创建一个 Thread 有两种方法

### 继承 Thread 类
``` java
class HttpRequest extends Thread {
  @Override
  public void run() {
    System.out.println("Requesting via http...");
    try {
      sleep(1000); // sleep 是使当前进程持续执行 *** ms
    } catch (InterruptedException e) {
       System.out.println("The thread is interupted");
    }
    System.out.println("Reuest finished!");
  }
}
class Main {
  public static void main(String[] args) {
    (new HttpRequest()).start();
  }
}
```

这种方式利用继承的关系来使用 Thread 凑近了来看看，那么发现 `run` 方法是罪魁祸首。 这个方法实际上是一个 callback， 当外部的 `main thread` 调用了 `Thread` 的 `start()` 实例方法的时候那么 `run` 这个 callback 就会被 "唤醒"

这种方式的好处是这个类将可以调用所有的 Thread 父类方法。但是坏处也是显而易见，如果我们要实现 Thread 的类已经继承了一个父类，那么这种继承的方式就不适用了。所以 java 发明了一个 interface 叫做 `Runnable`

### 实现 Runnable 接口

``` java
class HttpRequest implements Runnable {
  @Override
  public void run() {
    System.out.println("Requesting via http...");
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      System.out.println("The thread is interupted");
    }
    System.out.println("Reuest finished!");
  }
}
class Main {
  public static void main(String[] args) {
    (new Thread(new HttpRequest())).start();
  }
}
```

可以看到两个东西说的是同一回事儿。

## Thread 执行过程

下面是从 javaworld 拿来的一张图解

![java-thread-execution](/assets/img/java-threads.gif)

这里面唯一需要注意的就是在新的 thread 中的 `start` 方法被叫出来的时候我们的 `main` 方法开始等待 thread 运行的结束。 这里假设的是上面的两个例子中的情况，即在执行了 `start()`方法之后 `main` 中没要执行的语句了。假设我们在 `start` 后面加入一个 loop

``` java
class Main {
  public static void main(String[] args) {
    (new Thread(new HttpRequest())).start();
    for (int i = 0; i < 10000; i++) {
      System.out.println(i); 
    }
  }
}
```

结果大概是这样的 (每一次输出结果都不一样)

```text
0
...
46
Requesting via http...
47
...
9999
Request finished!
```

这正好论证了 `Thread` 的 concurrency 的特性。即两个线程中的任务可能同时进行。

## isAlive() 
`isAlive()` 方法探查 thread 的 run method 是否正在执行中

> The JVM considers the thread to be alive immediately prior to the thread's call to run(), during the thread's execution of run(), and immediately after run() returns

也就是说如果说某一个 `Thread` 的 `run` method 没有执行完成，那么 jvm 就会认为这个 thread 是 `isAlive` 这个状态的。

``` java
class Main {
  public static void main (String [] args) {
    PiCalculatorThread piCalculatorThread = new PiCalculatorThread ();
    piCalculatorThread.start ();
    while (piCalculatorThread.isAlive ()) {
      try {
        Thread.sleep (10); // Sleep for 10 milliseconds
      } catch (InterruptedException e) {
        System.err.println("I am interrupted");
      }
    }
    System.out.println ("pi = " + piCalculatorThread.pi);
  }
}
class PiCalculatorThread extends Thread {
  boolean negative = true;
  double pi; // Initializes to 0.0, by default
  public void run () {
    for (int i = 3; i < 100000; i += 2) {
      if (negative) pi -= (1.0 / i);
      else pi += (1.0 / i);
      negative = !negative;
    }
    pi += 1.0;
    pi *= 4.0;
    System.out.println ("Finished calculating PI");
  }
}
```

## join() 

可以从上面的例子里面看到手动地写 `while(thread.isAlive()) Thread.sleep(time)` 这样的用例太多了。所以 jdk 直接提供了了这样的工具。

``` java
try {
  piCalculatorThread.join(10);
} catch(InterruptedException e) {
  System.err.println("I am interupted");
}
System.out.println ("pi = " + piCalculatorThread.pi);
```

上述代码完成了与 `isAlive()` 同样的功能。其中 `join` 方法的参数就是每一个 loop 中 `sleep` 等待的时长。

### daemon
简单讲就是 thread 分两种。第一种是 `user thread`, 第二种是 `daemon thread`。 举例来说 java runtime 中的 `garbage collection` 的 thread 就是一个 `daemon thread`

二者的区别很简单。就是 `user thread` 负责主要的逻辑处理等等， `daemon` 是在背后跑的 主要负责 `house keeping`。 真正的区别是 
1. **user thread的终止是程序终止的充分条件**
2. **daemon thread是无限循环的一个过程，程序可以在daemon thread 仍然活跃的时候被终止**

手工使 thread 成为 `daemon thread` 只需要 `thread.setDaemon(true)` 就行了

参考: 

* [java-word-thread-1](http://www.javaworld.com/article/2074217/java-concurrency/java-101--understanding-java-threads--part-1--introducing-threads-and-runnables.html)
