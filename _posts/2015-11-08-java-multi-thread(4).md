---
layout:     post
title:      java 多线程 (4) - thread-groups
date:       2015-11-08 14:47
summary:    java 多线程基础 - 线程组
categories: java
---

先上图来的直观
![java-thread-group](/assets/img/java-threads-groups.gif)

``` java
class ThreadGroupDemo {
  public static void main(String [] args) {
    ThreadGroup tg = new ThreadGroup ("subgroup 1");
    Thread t1 = new Thread(tg, "thread 1");
    Thread t2 = new Thread(tg, "thread 2");
    Thread t3 = new Thread(tg, "thread 3");
    tg = new ThreadGroup("subgroup 2");
    Thread t4 = new Thread(tg, "my thread");
    tg = Thread.currentThread().getThreadGroup();
    int agc = tg.activeGroupCount();
    System.out.println ("Active thread groups in " + tg.getName() +
                        " thread group: " + agc);
    tg.list();
  }
}
```

上面这段代码就复原了图中的结构

输出如下
```
Active thread groups in main thread group: 2
java.lang.ThreadGroup[name=main,maxpri=10]
    Thread[main,5,main]
    Thread[Thread-0,5,main]
    java.lang.ThreadGroup[name=subgroup 1,maxpri=10]
        Thread[thread 1,5,subgroup 1]
        Thread[thread 2,5,subgroup 1]
        Thread[thread 3,5,subgroup 1]
    java.lang.ThreadGroup[name=subgroup 2,maxpri=10]
        Thread[my thread,5,subgroup 2]
```

分组的好处是可以对 `组` 进行批量操作。但是似乎实际情况用的比较少。所以有个概念之后如果遇到了再深入吧。

## volatility
这个概念很有意思。大概就是因为平台不同，有的时候不同的 thread 相互切换的时候不会从主的内从中去拿去 shared property, 而是在 cpu cache 里拿。所以这样一来 java 就引入了新的关键字 `volatility` 意思就是两个 thread 并行的时候就算是一个 property 实在cache 里， jvm 也保证两个 thread 拿到的是一样的东西。

## Thread local variables
没啥解释的。就是独立于外界的只有 thread 可见的 variables 有3个方法

1. Object get()
2. void set()
3. Object getInitialValue()

## Timer, TimerTask
这个对于 daemon thread 来说就比较重要了。

``` java
import java.util.*;
class Clock {
  public static void main (String [] args) {
    Timer t = new Timer ();
    t.schedule (new TimerTask() {
      public void run() {
        System.out.println (new Date().toString());
      }
    },0 /* delay */, 1000 /* interval */);
  }
}
```
 可以把这个想象成 javascript 里面的 `setInterval`

参考: 

* [java-word-thread-4](http://www.javaworld.com/article/2074481/java-concurrency/java-101--understanding-java-threads--part-4---thread-groups--volatility--and-threa.html)
