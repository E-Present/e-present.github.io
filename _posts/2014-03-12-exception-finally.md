---
layout: post
title:  "java 异常处理 以及 finally的使用"
date:   2014-03-12 22:00:13
categories: java
permalink: /blogs/exception-finally
---

(1)   There is always something more to learn. That was the lesson for me last week when I learned something new about the Java programming language, despite having used professionally it for almost 10 years.
I was upgrading a Java web application toWebSphere server version 6.1 and as the first step I switched the development environment toRational Application Developer version 7. With the new IDE came an improved compiler that reported additional warnings, so it didn't surprise me to see hundreds of new warnings. It is a standard practice of mine to eliminate warnings, even harmless ones, since a significant warning can easily be missed if harmless warnings are allowed to remain. As I worked my way through the warnings, I came across a new one I did not recognize: "Finally block does not complete normally". A simplified version of the code producing this warning is shown below:

```
  boolean performBusinessOperation() {
    boolean operationResult = false;
    try {
      // Perform some business logic...
      operationResult = true;
    } catch (IllegalStateException e) {
      // Handle this exception..
      operationResult = false;
    } catch (IllegalArgumentException e) {
      // Handle this exception...
      operationResult = false;
    } finally {
      // Common cleanup...
      // Following line produces warning
      // "Finally block does not complete normally"
      return operationResult;
    }
  }
  ```
This was not code I had written, so I spent some time trying to figure out the reason for the warning. In Java, the finally block is guaranteed to be executed after the contained try block finishes execution, even if an exception is thrown within the try block. If an exception is thrown and not caught within a function, the exception is propagated up the call stack until it encounters an appropriate catch block. But in this particular case within the performBusinessOperation method, if an uncaught exception is thrown, the finally block will run and perform the return statement. So which will win - the exception or the return? I was not sure, which in my mind explained the warning - it is bad practice to write code with unclear behavior. So I fixed the code by moving the return statement outside the finally block and moved on to the next warning.
Once I was finished eliminating warnings, I ran the entire suite of automated unit tests. To my surprise, I had a few failures. When I tracked down the offending code, I was surprised to see that it was my fix for the "Finally block does not complete normally" warning that broke the tests. How could that be? After further tracing and debugging, I finally found the reason: the unit test incorrectly invoked the method in question, causing it to throw a NullPointerException from within the try block. Having the return statement within the finally block apparently was causing the exception to be silently discarded. I found this shocking. This is dangerous behavior for a language, and I had problems believing that was actually the case. So I wrote a quick unit test for verification, shown below.

```
public class ReturnInFinallyBlockExample
  extends junit.framework.TestCase
{
  @SuppressWarnings("finally")
  private boolean isReturnWithinFinally() {
    try {
      if (true) throw new RuntimeException();
    } finally {
      return true; // This hides the exception
    }
  }

  private boolean isReturnOutsideFinally() {
    try {
      if (true) throw new RuntimeException();
    } finally {
      // Return outside finally block.
    }
    return true;
  }

  public void testReturnFromFinallyBlockWithUnhandledException() {
    assertTrue(isReturnWithinFinally());
    try {
      isReturnOutsideFinally();
      fail("Expect exception");
    } catch (RuntimeException e) {
      // Expected case.
    }
  }
}
```
This test case passes, demonstrating that the uncaught exception within the try block is silently discarded if the return statement is within the finally block. Note my use of the Java 5 annotation@SuppressWarnings("finally") in order to stop the compiler from reporting the "Finally block does not complete normally" warning for this example.
Perhaps I should have been less surprised by this behavior in Java given that I was already aware of another suboptimal situation regarding Java exception handling in finally blocks: if an uncaught exception is thrown in a try block and then another exception is thrown in the finally block, it will be the second exception that is propagated out of the method. The first exception will be silently lost. The following test case demonstrates this behavior:

```
public class ExceptionInFinallyBlockExample
  extends junit.framework.TestCase
{
  private void haveExceptionInFinallyBlock() {
    try {
      if (true) throw new IllegalArgumentException();
    } finally {
      if (true) throw new NullPointerException();
    }
  }
  public void testHaveExceptionInFinallyBlock() {
    try {
      haveExceptionInFinallyBlock();
      fail("Expect exception");
    } catch (NullPointerException e) {
      // Expected case.
    }
  }
}
```
Ignoring errors is dangerous, as I have discussed in my article on Error Handling and Reliability. So I strongly feel that the Java language should have prohibited return statements in finally blocks. Fortunately, modern Java IDEs like Eclipse can make up for this shortcoming by allowing you to flag this code construct as an error rather than a warning.

# finally的使用
  * 1 finally块中的return语句会覆盖try块、catch块中的return语句
  * 2 如果finally块中包含了return语句，即使前面的catch块重新抛出了异常，则调用该方法的语句也不会获得catch块重新抛出的异常，而是会得到finally块的返回值，并且不会捕获异常合理的做法是，既不在tryblock内部中使用return语句，也不在finally内部使用return语句，而应该在 finally 语句之后使用return来表示函数的结束和返回。
    
注：catch中的reture语句执行之前会先调用finally中的语句

