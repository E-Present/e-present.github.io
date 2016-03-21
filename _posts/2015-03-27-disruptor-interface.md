---
layout: post
title:  "Disruptor的消费者接口"
date:   2015-03-27 13:00
categories: java
permalink: /blogs/disruptor-interface
---

1.WorkHandler
每个WorkerHandler对应处理一条消息，也就是说消息会被分发到每一个线程中去执行,例如：

   disruptor.handleEventsWithWorkerPool(new WorkHandlerObject(), new WorkHandlerObject(), new WorkHandlerObject());
     //此处的处理方式，每个work会处理一条消息，当前顺序如下：
   >        /*      QQ0>>>pool-1-thread-1
   >               QQ1>>>pool-1-thread-3
   >              QQ2>>>pool-1-thread-2
   >               QQ3>>>pool-1-thread-1
   >                QQ5>>>pool-1-thread-2
   >                QQ4>>>pool-1-thread-3
   >                QQ6>>>pool-1-thread-1
   >                QQ7>>>pool-1-thread-2
   >                QQ8>>>pool-1-thread-3
   >                QQ9>>>pool-1-thread-1
   >                QQ10>>>pool-1-thread-2
   >                QQ11>>>pool-1-thread-3*/

2.EventHandler
每个EnventHandler会串行的处理同一条消息，也就是说，每条信息都会被添加的EventHandler处理。例如：
/*      ObjectEventHandler handler1 = new ObjectEventHandler();
          ObjectEventHandler handler2 = new ObjectEventHandler();
          ObjectEventHandler handler3 = new ObjectEventHandler();
    disruptor.handleEventsWith(handler1, handler2, handler3);*/
 //此方式，每个消息都会被当前添加的EvenHandler顺序处理，此处的处理顺序如下：
/*                QQ0>>>pool-1-thread-3
                   QQ0>>>pool-1-thread-2
                   QQ0>>>pool-1-thread-1
                   QQ1>>>pool-1-thread-2
                   QQ1>>>pool-1-thread-3
                   QQ1>>>pool-1-thread-1
                   QQ2>>>pool-1-thread-2
                   QQ2>>>pool-1-thread-1
                   QQ2>>>pool-1-thread-3*/

3. 接口EventProcessor其中的实现类：BatchEventProcessor的run方法的部分源码：
final long availableSequence = sequenceBarrier .waitFor(nextSequence);
                    while (nextSequence <= availableSequence)
                    {
                        event = dataProvider.get(nextSequence);
                        eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                        nextSequence++;
                    }
                    sequence.set( availableSequence);
从这可以看出当前的一个线程执行时，他会先通过获取可用的sequece，然后从ringbuffer中消费
