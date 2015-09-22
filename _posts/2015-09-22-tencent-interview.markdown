---
title: 腾讯面经
layout: post
guid: urn:uuid:35D5B065-3EC6-4DFD-BA91-40CFAD9C6E88
tags:
  - blog
  - interview
summary:
   腾讯SNG二面面经
--- 

#总结
总得来说问题比较大，很多问到的问题没有打出来，还有一些应该能回答的问题没有回答到点子上。  
一个可能是因为之前一直在准备托福，Android有些生疏；另外就是有点自满了，面试之前还有一天多的时间也没有好好准备，之前一面回答的不好的题也没有搞清楚。  

#问题
##1. 说一下对Android的理解  
这个问题完全在我意料之外，实在是有些大。我就从我知道的开始扯：Android基于Linux，启动方式是从Zygote（面试的时候错误的说成dalvik） folk新的进程来承载app。  
然后讲到Android runtime从Dalvik转变到现在新的ART，区别在于Dalvik是JIT而ART是AOT。

##2. Handler是怎么工作的？postDelayed是如何实现的？消息移除是怎么实现的？
这个问题一面的时候问到了，当时回答的不是特别好，但是没有引起足够的重视。  
下面摘抄几个概念：  
  * Message 消息，理解为线程间通讯的数据单元。例如后台线程在处理数据完毕后需要更新UI，则可发送一条包含更新信息的Message给UI线程。
  * Message Queue 消息队列，用来存放通过Handler发布的消息，按照先进先出执行。
  * Handler Handler是Message的主要处理者，负责将Message添加到消息队列以及对消息队列中的Message进行处理。
  * Looper 循环器，扮演Message Queue和Handler之间桥梁的角色，循环取出Message Queue里面的Message，并交付给相应的Handler进行处理。
  * 线程 UI thread 通常就是main thread，而Android启动程序时会替它建立一个Message Queue。每一个线程里可含有一个Looper对象以及一个MessageQueue数据结构。在你的应用程序里，可以定义Handler的子类别来接收Looper所送出的消息。  
  Looper实际上就是消息队列+消息循环的封装。  
  Handler可以被看做Looper的接口，用来向指定的Looper发送消息以及定义处理方法。  
  Hander handler=new Handler();等价于Handler handler=new Handler(Looper.myLooper());  
  Looper.prepare(): 创建looper，新建messagequeue，并将looper对象装入ThreadLocal中。  
  Looper.loop(): 让Looper开始工作，infinite loop，取messagequeue的下一个message对象  
  通过查阅源码，我发现postDelayed会被传递给messageQueue的queue.enqueueMessage(msg, uptimeMillis)，这里的时间会是当前时间加上delay的时间，之后这个时间会被设置为message.when
  而在messagequeue的next()方法中：  
  {% highlight Java %}
  final Message next() {
     int pendingIdleHandlerCount = -1; // -1 only during first iteration
     int nextPollTimeoutMillis = 0;
 
     for (;;) {
         if (nextPollTimeoutMillis != 0) {
             Binder.flushPendingCommands();
         }
         nativePollOnce(mPtr, nextPollTimeoutMillis);
 
         synchronized (this) {
             if (mQuiting) {
                 return null;
             }
 
             // Try to retrieve the next message.  Return if found.
             final long now = SystemClock.uptimeMillis();
             Message prevMsg = null;
             Message msg = mMessages;
             if (msg != null && msg.target == null) {
                 // Stalled by a barrier.  Find the next asynchronous message in the queue.
                 do {
                     prevMsg = msg;
                     msg = msg.next;
                 } while (msg != null && !msg.isAsynchronous());
             }
             if (msg != null) {
                 if (now < msg.when) {
                     // Next message is not ready.  Set a timeout to wake up when it is ready.
                     nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                 } else {
                     // Got a message.
                     mBlocked = false;
                     if (prevMsg != null) {
                         prevMsg.next = msg.next;
                     } else {
                         mMessages = msg.next;
                     }
                     msg.next = null;
                     if (false) Log.v("MessageQueue", "Returning message: " + msg);
                     msg.markInUse();
                     return msg;
                 }
             } else {
                 // No more messages.
                 nextPollTimeoutMillis = -1;
             }
 
             // If first time idle, then get the number of idlers to run.
             // Idle handles only run if the queue is empty or if the first message
             // in the queue (possibly a barrier) is due to be handled in the future.
             if (pendingIdleHandlerCount < 0
                     && (mMessages == null || now < mMessages.when)) {
                 pendingIdleHandlerCount = mIdleHandlers.size();
             }
             if (pendingIdleHandlerCount <= 0) {
                 // No idle handlers to run.  Loop and wait some more.
                 mBlocked = true;
                 continue;
             }
 
             if (mPendingIdleHandlers == null) {
                 mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
             }
             mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
         }
 
         // Run the idle handlers.
         // We only ever reach this code block during the first iteration.
         for (int i = 0; i < pendingIdleHandlerCount; i++) {
             final IdleHandler idler = mPendingIdleHandlers[i];
             mPendingIdleHandlers[i] = null; // release the reference to the handler
 
             boolean keep = false;
             try {
                 keep = idler.queueIdle();
             } catch (Throwable t) {
                 Log.wtf("MessageQueue", "IdleHandler threw exception", t);
             }
 
             if (!keep) {
                 synchronized (this) {
                     mIdleHandlers.remove(idler);
                 }
             }
         }
 
         // Reset the idle handler count to 0 so we do not run them again.
         pendingIdleHandlerCount = 0;
 
         // While calling an idle handler, a new message could have been delivered
         // so go back and look again for a pending message without waiting.
         nextPollTimeoutMillis = 0;
     }
 }
  {% endhighlight %}
  可以看到取下一个message时会判断时间，如果队列中没有消息或者第一个待处理的消息时机未到，且也没有其他利用队列空闲要处理的事务，则将队列设置为设置 blocked 状态，进入等待状态；否则就利用队列空闲处理其它事务。  
  enqueueMessage会根据when把新入的message放在合适的位置；removeMessage会遍历整条queue移除符合要求的message  
  *[reference](http://www.cnblogs.com/kesalin/p/android_messagequeue.html)*
  
##3. Binder&&IPC的实现原理
binder其实不是android首先提出来的IPC机制，它是基于OpenBinder来实现的。OpenBinder有许可问题，andriod不能直接使用，故此重新开发了自己的一套binder实现，基于宽松的Apache协议发布，架构与OpenBinder类似，相关信息可以参考: [http://www.open-binder.org](http://www.open-binder.org)。  
这个问题还没有研究透彻，先留下一个[reference](http://blog.csdn.net/universus/article/details/6211589)

##4. Activity如何收到Touch事件
在ViewRootImpl的setView方法中，用户的触摸按键消息是体现在窗体上的，而windowManagerService则是管理这些窗口，它一旦接收到用户对窗体的一些触摸按键消息，会进行相应的动作，这种动作是需要体现在具体的view上面，在Android中，一个具体的界面是由一个Activity呈现的，而Activity中则包含了一个window，此window中又包含了一个phoneWindow，这个phoneWindow才是真正意义上的窗口，它把一个框架布局进行了一定的包装，并提供了具体的窗口操作接口，phoneWindow中包含了一个DecorView，这个view才是包含整个Activity的ui，它将被attach到Activity主窗口中。所以说用户触摸按键的消息是由windowManagerService捕捉到然后交给phoneWindow中的DecorView进行相应的处理，而连接两者的桥梁则是一个ViewRoot类，ViewRoot类由windowManagerService创建，其内部有一个W类，这个W类是一个binder，负责WindowManagerService的ipc调用，W接收到windowManagerService发送过来的消息后，把消息传递给ViewRoot，进而传递给ActivityThread解析做出处理。  
*[reference](http://blog.csdn.net/andywuchuanlong/article/details/46762877)*

##5. 什么是NIO
非阻塞的I/O交互。通道是对原I/O包的流的模拟，缓冲区是一个容器对象，发送给一个通道的所有对象必须先放到缓冲区中；从通道中读取的数据都要读到缓冲区中。  
* 容量：Capacity  
* 上界：Limit  
* 位置：Position  
* 标记：Mark  
0 <= mark <= position <= limit <= capacity  
*[reference](http://www.yangyong.me/java-nio%E5%85%A5%E9%97%A8%E4%B8%8E%E8%AF%A6%E8%A7%A3/)*
