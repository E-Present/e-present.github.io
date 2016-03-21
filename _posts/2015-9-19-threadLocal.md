---
layout: post
title:  "ThreadLocal 和 InheritableThreadLocal的使用"
date:   2015-9-19 12:38
categories: Java
permalink: /blogs/ThreadLocal
---
1.  ThreadLocal
       Creates a thread local variable.
       保存只有当前线程可以使用的变量

2. InheritableThreadLocal
      This class extends ThreadLocal to provide inheritance of values from parent thread to child thread: when a child thread is created, the child receives initial values for all inheritable thread-local variables for which the parent has values. Normally the child's values will be identical to the parent's; however, the child's value can be made an arbitrary function of the parent's by overriding the childValue method in this class.

      Inheritable thread-local variables are used in preference to ordinary thread-local variables when the per-thread-attribute being maintained in the variable (e.g., User ID, Transaction ID) must be automatically transmitted to any child threads that are created.
    一般是把父线程需要传给子线程的变量保存在这里，保存的变量可以被子线程访问到。若子线程将其改变，不会影响父线程中的值。子线程只能通过修改可变性（Mutable）对象对主线程才是可见的，即才能将修改传递给主线程
    注意：
        1.如果使用线程池，可能实现不了此结果，特别是当当前线程池已被占满，当有新的任务时，就会出现子线程中获取不到父线程中的值。
        2. 线程池使用完后，最好关闭。exec.shutdown();
        3.若在线程池中使用InheritableThreadLocal来实现父线程传值给子线程，会发生线程池中固定线程中的InheritableThreadLocal保存的变量值是一定的，即当此线程产生时就固定下来了；由于线程是循环使用的，故很可能会出现传值出现错误（既当线程数小于任务数时）。
        4.注意ThreadLocal的内存泄露问题，可以采用在finally块中调用remove方法来规避此问题。
实例：
public class TestThreadLocal {
          
           static final String value01= "VALUE01";
           static final String value02 = "VALUE02";
          
    public static void main(String[] args) throws InterruptedException {
                   ThreadLocal<String> threadLocal = new ThreadLocal<String>();
                   threadLocal.set( value01);
          
          InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<String>();
          inheritableThreadLocal.set( value01);
          
          Thread thread_1 = new MyThread(threadLocal, inheritableThreadLocal);
          thread_1.setName( "Thread01");
          thread_1.start();
          
          thread_1.join();
          
            System. out.println(Thread. currentThread().getName() + "******************************************" ); 
          System. out.println(Thread. currentThread().getName() + "\tThreadLocal: " + threadLocal.get()); 
          System. out.println(Thread. currentThread().getName() + "\tInheritableThreadLocal: " + inheritableThreadLocal.get()); 
          }
   
 static   class MyThread extends Thread{
          ThreadLocal<String> threadLocal;
          InheritableThreadLocal<String> inheritableThreadLocal;
          
        public MyThread(ThreadLocal<String> threadLocal, InheritableThreadLocal<String> inheritableThreadLocal) { 
            super(); 
            this. threadLocal = threadLocal; 
            this. inheritableThreadLocal = inheritableThreadLocal; 
        }
       
        @Override
        public void run() {
            System. out.println(Thread. currentThread().getName() + "******************************************" ); 
              System. out.println(Thread. currentThread().getName() + "\tThreadLocal: " + threadLocal .get()); 
              System. out.println(Thread. currentThread().getName() + "\tInheritableThreadLocal: " + inheritableThreadLocal .get()); 
       
              threadLocal.set(TestThreadLocal. value02); 
              inheritableThreadLocal.set(TestThreadLocal. value02); 
       
              System. out.println(Thread. currentThread().getName() + "(Reset Value)*****************************"); 
              System. out.println(Thread. currentThread().getName() + "\tThreadLocal: " + threadLocal .get()); 
              System. out.println(Thread. currentThread().getName() + "\tInheritableThreadLocal: " + inheritableThreadLocal .get()); 
        }   
    }

}

