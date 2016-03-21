---
layout: post
title:  "InterruptedException处理"
date:   2014-03-23 23:00:13
categories: java
permalink: /blogs/interruptedException
---

java 中的受检查异常 InterruptedException 如何处理是令人头痛的问题，下面是我对处理这个问题的理解。
Java 中的 InterruptedException 一直是一个令人头疼的问题，对初级开发者来说尤其如此。但实际上不应如此，这其实是一个很容易理解的问题。我会尽可能简单地描述这个问题。
我们从这段代码开始：

```
while (true) {
  // Nothing
}
```
它做了什么？什么都没做，只是无止境的消耗 CPU。我们能终止它吗？在 Java 中是不行的。只有当你按下 Ctrl-C 来终止整个 JVM 时这段程序才会停止。在 Java 中没有方式来终止一个线程，除非该线程自动退出。请务必牢记的这一原则，其它东西就显而易见了。
我们将这个死循环放在一个线程里：

```
Thread loop = new Thread(
  new Runnable() {
    @Override
    public void run() {
      while (true) {
      }
    }
  }
);
loop.start();
// Now how do we stop it?
```
所以，怎样才能停止一个需要停止的线程？
下面是 Java 中设计终止一个线程的方法。在线程的外部，设置一个标识变量（flag），然后在线程内部检查改标识变量，从而实现线程的终止。过程如下：

```
Thread loop = new Thread(
  new Runnable() {
    @Override
    public void run() {
      while (true) {
        if (Thread.interrupted()) {
          break;
        }
        // Continue to do nothing
      }
    }
  }
);
loop.start();
loop.interrupt();
```
这是终止线程的唯一方式，在这个例子里使用了两个方法。当调用 loop.interrupt() ，线程内部将标志位设置为 true。当调用 interrupted() 时，立即返回，并将标识变量设置为 false。确实，这个方法就是这样设计的。检查标识变量、返回、设置为 false。我知道这很丑陋。
因此，我从来没有在线程内调用 Thread.interrupted() 方法，因此标识变量为 true 时线程不会退出，没有人能停止这个线程。准确地说，我会忽略他们对 interrupt() 方法的调用。虽然它们会要求终止线程，但是我会忽略它们。它们不能让线程中断。
因此，总结一下我们现在理解的内容，一种合理的设计是通过检查标识变量来优雅地终止线程。如果代码中不检测标识变量，也不调用 Thread.interrupted()，那么终止线程的方式就只能按下 Ctrl-C 了。
现在你听明白这个逻辑了吗？我希望是。
现在，JDK 中有一些方法来检测标识变量，如果设置该标识变量，则会抛出 InterruptedException。例如，Thread.sleep() 方法的设计（一种最基本的方法）：

```
public static void sleep(long millis)
  throws InterruptedException {
  while (/* You still need to wait */) {
    if (Thread.interrupted()) {
      throw new InterruptedException();
    }
    // Keep waiting
  }
}
```
为什么要这么做？为什么不能等待并且不用去检查标识变量？我相信一定有一个非常好的理由。理由如下（如果我说错了，请修正我的错误）：为了让代码变快或是中断准备，没有其他理由。
如果你的代码足够快，你从来不会检测中断标识变量，因为你不想处理任何中断。如果你代码很慢，可能需要执行数秒，这时你就有可能需要处理中断了。
这就是为什么 InterruptedException 是受检查异常。这种设计告诉你，如果你想在几毫秒内停止线程，确定你已经做好中断准备。实践中一般做如下处理：

```
try {
  Thread.sleep(100);
} catch (InterruptedException ex) {
  // Stop immediately and go home
}
```
现在，你可以将它抛给负责捕获该异常的上级程序去处理。这种观点是有人在使用线程，并且会捕获该异常。理想情况下，会终止线程，因为这就是标识变量的功能。如果抛出 InterruptedException，就意味着有人在检查标识变量，线程需要尽可能快地终止。
线程的拥有者不想再等待线程执行，我们应该尊重拥有者的决定。
因此，当捕获到 InterruptedException 时，你应该完成相关的操作再退出线程。
现在，我们再看一下 Thread.sleep() 的代码：

```
public static void sleep(long millis)
  throws InterruptedException {
  while (/* ... */) {
    if (Thread.interrupted()) {
      throw new InterruptedException();
    }
  }
}
```
请记住，Thread.interrupted() 不仅仅是返回标识变量的值，而且会将标识变量的值设置为 false。因此，一旦抛出 InterruptedException 异常，标志变量将会重置。线程不再收到任何拥有者发送的中断请求。
线程的所有者要求停止线程，Thread.sleep() 监测到该请求并将其删除，再抛出 InterruptedException。如果你再次调用 Thread.sleep()，就不会响应任何中断请求，也不会抛出任何异常。
知道我想要说的是什么吗？不要丢失 InterruptedException，这一点非常重要。我们不能吞噬该异常并继续运行。这严重违背了 Java 多线程原则。所有者（线程的所有者）要求停止线程，而我们却将其忽略，这是非常不好的想法。
下面是大多数人对 InterruptedException 的处理：

```
try {
  Thread.sleep(100);
} catch (InterruptedException ex) {
  throw new RuntimeException(ex);
}
```
这看起来是符合逻辑的，但是这不能保证上层程序真正停止并退出。上层可能捕获了运行时异常，所以这个线程还是存活的。线程所有者将会非常失望。
我们必须通知上层捕获了一个中断请求。我们不能只抛出运行时异常，这种行为太不负责了。当一个线程接收一个中断请求时，我们不能只是将其转换成为一个 RuntimeException。我们不能将这种严峻的情况如此轻松地对待。
这是我们应该做的：

```
try {
  Thread.sleep(100);
} catch (InterruptedException ex) {
  Thread.currentThread().interrupt(); // Here!
  throw new RuntimeException(ex);
}
```
我们需要将标识变量重新设置为 true。
由于抛出InterruptedException时当前线程的状态interrupted已经变为false，上层很可能判断不出请求中断的实事，所以这样做就可以通知上层可以通过线程的状态清理掉当前线程
现在，没有人会谴责我们以不负责的态度来处理标识变量。我们发现其状态为 true，将其清理，重新设置为 true，最后抛出运行时异常。接下来会发生什么？我们已经不关心了。


