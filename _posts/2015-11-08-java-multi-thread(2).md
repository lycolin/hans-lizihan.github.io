---
layout:     post
title:      java 多线程 (2) - synchronizated
date:       2015-11-08 1:07
summary:    java 多线程基础 - 同步
categories: java
---

## synchronizated

这是一个关键字 意思就是当前这个方法在执行的时候 jvm 会给当前的类上一个 `object lock`, 所以在执行该方法的时候其他的 `thread` 不可以对于该 object 的任何属性进行更改，知道这个方法执行完毕。

一个简单的例子就是

``` java
class HttpRequest implements Runnable {
  public static String url;
  public static int size;
  
  public void setUrl(String url) {
    this.url = url;
  }
  @Override
  public void run() {
    System.out.println("Requesting via http...");
    
    transferRequest();

    System.out.println("Reuest finished!");
  }

  private void transferRequest() {
    for (int i = 0; i < 5000; i++) {
      size ++;
    }
  }
}
class Main {
  public static void main(String[] args) {
    HttpRequest imageRequest = new HttpRequest();
    
    imageRequest.setUrl("image1.png");
    Thread image1RequestThread = new Thread(imageRequest);

    imageRequest.setUrl("image2.png");
    Thread image2RequestThread = new Thread(imageRequest);
    
    image1RequestThread.start();
    image2RequestThread.start();
    
    try {
      image1RequestThread.join();
      image2RequestThread.join();
    } catch (InterruptedException e) {
      System.err.println("Error");
    }
    System.out.println(HttpRequest.size);
  }
}
```

最后输出的结果是 `8189` 而不是想象中的 `10000`

### race condition 

简单讲就是 `run` block 中的代码想要更改 `object` 上面的某一个属性。然而我们知道如今的电脑都是多核的，所以就让 concurrency 变得更加 名副其实 了。因为两个 thread 可能 分别在 `cpu1` 和 `cpu2` 上面运行。而当两个 `thread` 同时运行的时候可能有一瞬间，两个 `thread` 都从内存中取得了 `size` 属性为 `500` 于是同时对其 `++`，所以本来`500` 应该在这个过程中变成 `502` 但是由于两个 `thread` 都去读取并且更新了这个属性，导致最后属性的真实值并不统一。

结局方案不难 有两种方案

### synchronized method
简单讲就是 加一个 synchronized 关键字就大功告成了

``` java
synchronized private void transferRequest() {
  for (int i = 0; i < 5000; i++) {
    size ++;
  }
}
```
这个方法的意义在于，执行这段代码的时候，当前的**类** 会被加上一个 `object lock` 使得其他 `thread` 无法执行该实例中的所有方法 jvm 对其的实现是通过 `monitor` 检视对象。如果进入了 `synchronized` 块则将对象上锁。

注意如果当这个方法是 static 方法时 这个锁就会进化成更加恐怖的 `class lock`

### synchronized block

也可以对于相应的对象上锁。例如

``` java
synchronized (Object o) {
  // ...
}
```
这一段的含义就是在执行 `synchronized` 关键字中的代码块的时候 `o` 对象将会被锁住。

### Thread.holdsLock(Object o)
这个静态方法可以判断某一个对象有没有被锁住。 其实实际上似乎用的比较少。

## synchronized 关键字使用可能造成的问题

有时滥用 synchronized 关键字会有非常意想不到的效果。有以下两种情况。此外，对象锁和类锁的开锁解锁都是需要时间的, 所以过分依赖 `synchronized` 关键字会有潜在的性能问题。

### No synchronization

简单讲就是由于思路不清晰导致 synchronized 关键字上锁的对象并不是两个 thread 共享的资源，所以导致了虽然写了 synchronized 关键字，而程序的执行并没有像想象中那样执行。还是以 image 为例子

``` java
class HttpRequest extends Thread {
  public static String url;
  public static int size;
  
  public HttpRequest(String url) {
    this.url = url;
  }
  @Override
  public void run() {
    System.out.println("Requesting via http...");
    
    transferRequest();

    System.out.println("Reuest finished!");
  }

  void transferRequest() {
    if (this.url.equals("image1")) {
      synchronized(this) {
        for (int i = 0; i < 5000; i++) {
          size ++;
        }
        System.out.println(this.url + " " + size);
      }
    } else if (this.url.equals("image2")) {
      synchronized(this) {
        for (int i = 0; i < 5000; i++) {
          size ++;
        }
        System.out.println(this.url + " " + size);
      }
    }
  }
}
class Main {
  public static void main(String[] args) {
    HttpRequest image1Request = new HttpRequest("image1");
    HttpRequest image2Request = new HttpRequest("image2");

    image1Request.start();
    image2Request.start(); 

    try {
      image1Request.join();
      image2Request.join();
    } catch (InterruptedException e) {
      System.err.println("Error");
    }
  }
}
```

输出结果如下
```
Requesting via http...
Requesting via http...
image1 6887
Reuest finished!
image2 7614
Reuest finished!
```

可以看到虽然写了 synchronized 关键字，但是由于 `this` 是一个实例，虽然都在一个代码块中，但是 `this` 缺代表了两个不同的实例。这就是传说中的 `no synchroniztion`

解决方案就是 `synchronized (Object o) {}` 中尽量使得 `o` 是两个 thread 共享的实例。

### Dead lock

这个课本里面说的挺多的。形象说就是 object `a` 需要 object `b` 解锁的时候运行。 而 object `b` 需要在 object `a` 解锁的时候运行。 出现的结果就是 `a` `b` 互相依赖， `b` 在等待 `a` 解锁的时候 `a` 也在等待 `b` 解锁，二者会永远互相等下去。

``` java
class Deadlock {
  static class Friend {
    private final String name;
    public Friend(String name) {
      this.name = name;
    }
    public String getName() {
      return this.name;
    }
    public synchronized void bow(Friend bower) {
      System.out.format("%s: %s has bowed to me!%n", this.name, bower.getName());
      bower.bowBack(this);
    }
    public synchronized void bowBack(Friend bower) {
      System.out.format("%s: %s has bowed back to me!%n", this.name, bower.getName());
    }
  }

  public static void main(String[] args) {
    final Friend alphonse = new Friend("Alphonse");
    final Friend gaston = new Friend("Gaston");
    new Thread(() -> alphonse.bow(gaston)).start();
    new Thread(() -> gaston.bow(alphonse)).start();
  }
}
```

Oracle 官方文档的这例子就很形象， `gaston` 和 `alphonse` 要在 `bow` 之后 `bowback`, 然而在 `bow` method 启动的时候两个人都做不了 `bowback` 这个动作

1. alphonse start a process to bow to gaston, during this time, alphonse cannot do bowback action
2. alphonse bow to gaston
3. alphonse wait for gaston to bow back
4. gaston start a process to bow to alphonse, during this time, gaston cannot do bowback action
5. gaston bow to alphonse
6. gaston wait for alphonse to bow back
7. because 1). and 4). situation due to `synchronized` keyword, neither gaston nor alphonse could end the process. They will wait forever.

结局方案挺坑爹的。建议的解决方案是写程序之前东小脑好好想想。尤其是一个 `synchronized` method 调用另一个 `synchronized` method 的时候，一定画个图什么的保证两者没有互相的依赖。

参考: 

* [java-word-thread-2](http://www.javaworld.com/article/2074318/java-concurrency/java-101--understanding-java-threads--part-2--thread-synchronization.html)
