---
layout: post
title:  "wait-- notify/notifyAll的使用"
date:   2015-10-21 14:17
categories: java
permalink: /blogs/wait-notify
---
# 注意：
1. wait 和notify都必须在获取锁的基础之上才能起作用。
2. wait和notify必须在同一对象锁上才能相互作用。
3. synchronized( this)是对象锁
4. synchronized 在静态方法前，是类级别的锁
5. 对象中的this，特别是这个对象的属性被其他对象多次引用时，执行该对象的属性时，其this仍指向初始创建该属性的那个类的那个对象。即t某对象被创建出来（属性初始化完成）时，属性的所有者就是this，以后不管该属性被任何对象引用，该属性的所有者仍然是最初的this。

# 以下三个类用来模拟wait、notify的使用
public class TestMyCallabe extends Context{
          
           public Callable<Void> callable = new Callable<Void>() {
                    @Override
                    public Void call() throws Exception {
                             say();
                              return null;
                   }
          };
          
           public void myexecute( ){
                   System. out.println( "--------------------wait");
                    synchronized( this){
                             System. out.println( this);
                              //super.setValue("key", this.callable);
                              super.setCallable( callable);
                              try {
                                      String name = Thread. currentThread().getName();
                                      System. out.println(name);
                                      wait();
                                      System. out.println(name+ ":::EEEENNNDDDD");
                             } catch (InterruptedException e) {
                                       // TODO Auto-generated catch block
                                      e.printStackTrace();
                             }
                   }
          }
          
          
          
           private synchronized void say(){
                   String name = Thread. currentThread().getName();
                   System. out.println(name+ "OKOKOKOOOOO::::::"+this );
                   notifyAll();
                   System. out.println(name+ ":::notifyAll");
          }
          

}


           public class TestForWait extends Context{

                    static Map<String, Object>  map = new HashMap<String, Object>();
                    private static ExecutorService service = Executors.newFixedThreadPool(10);
                   
                   
              public static void main(String[] args) throws InterruptedException {
                    
                     Runnable runnable1 =  new Runnable() {
                                       @Override
                                       public void run() {
                                                TestMyCallabe testMyCallabe = new TestMyCallabe();
                                                testMyCallabe.myexecute();
                                      }
                             };
                              service.submit(runnable1);
                             
                              Thread.currentThread().sleep(100);
                             
                             
                             Callable<Void> callable = (Callable<Void>) getValue("key") ;
                              service.submit(callable);
                              service.shutdown();
              }
             
          
}

public class Context {
          
           private static Map<String, Object> map = new HashMap<String, Object>();
          
           private Callable< Void> callable;
          
           public void setCallable(Callable<Void> callable) {
                    this. callable = callable;
              setValue("key", this.callable);
          }
          
           public Callable<Void> getCallable() {
                    return callable;
          }
           public static Object getValue(String key){
                    return  map.get(key);
          }

           public static void setValue(String key, Object object){
                    map.put(key, object);
          }
}

