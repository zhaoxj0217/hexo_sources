---

title: 线上问题处理-服务假死
data: 2017-11-29
categories: accident handling,linux
tags: [accident handling,linux]

---

1.服务假死  
现象一般如下，比如：服务器没有后续日志输出，或者接口只有进的日志，没有出的日志。

操作步骤：  
 1）联系运维，在出问题的服务器上输入以下命令，保存当前stack日志：  

         jstack -l  服务pid > stack.log
 2）打开保存的日志，分析主线当前状态，<font color=#DC143C>搜索"main"</font>，比如下面日志标识主线程当前在sleep：  
     
    "main" #1 prio=5 os_prio=0 tid=0x00007faf9c00a000 nid=0x7661 waiting on condition [0x00007fafa36a8000]
    java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)

 3）打开保存的日志，分析可能存在的死锁，<font color=#DC143C>搜索"deadlock"或者"BLOCKED"</font>， 比如下面日志，表示可能存在死锁：  

     "IdleRemover" daemon prio=10 tid=0x00007f6b2c148800 nid=0x11d7 waiting on condition [0x00007f6b222e1000]
       java.lang.Thread.State: TIMED_WAITING (parking)
     at sun.misc.Unsafe.park(Native Method)
	......
	 Locked ownable synchronizers://正常情况下，线程不会有：Locked ownable synchronizers信息
 	 - <0x0000000765df4068> (a java.util.concurrent.ThreadPoolExecutor$Worker)
 
4）如果线程信息没有异常，需要确认FGC是否存在问题？比如： 频繁full gc 或者单词full gc时间较长。
  
    jstat  -gcutil pid 2500 70
    [root@iZbp1aehttqhrozii8qa5iZ Python-3.6.3]# jstat -gcutil 10567 2500 70
    S0  S1   E   O  MCCS  YGC YGCT FGC FGCT GCT 
    0.00 54.17 19.32 17.82 97.59 95.77 1777 11.596 4 0.898 12.494
    0.00 54.17 19.58 17.82 97.59 95.77 1777 11.596 4 0.898 12.494
    0.00 54.17 19.82 17.82 97.59 95.77 1777 11.596 4 0.898 12.494
    0.00 54.17 20.16 17.82 97.59 95.77 1777 11.596 4 0.898 12.494

S0 — Heap上的 Survivor space 0 区已使用空间的百分比  
S1 — Heap上的 Survivor space 1 区已使用空间的百分比  
E — Heap上的 Eden space 区已使用空间的百分比  
O — Heap上的 Old space 区已使用空间的百分比  
P — Perm space 区已使用空间的百分比  
YGC — 从应用程序启动到采样时发生 Young GC 的次数  
YGCT– 从应用程序启动到采样时 Young GC 所用的时间（单位秒）  
FGC — 从应用程序启动到采样时发生 Full GC 的次数  
FGCT– 从应用程序启动到采样时 Full GC 所用的时间（单位秒）  
GCT — 从应用程序启动到采样时用于垃圾回收的总时间（单位秒）  


**jstack Dump 日志文件中的线程状态**  
dump 文件里，值得关注的线程状态有：  
- 1.	死锁，<font color=#DC143C> Deadlock（重点关注）</font>   
- 2.	执行中，Runnable     
- 3.	等待资源，<font color=#DC143C> Waiting on condition（重点关注</font>）   
- 4.	等待获取监视器，<font color=#DC143C> Waiting on monitor entry（重点关注）</font>  
- 5.	暂停，Suspended  
- 6.	对象等待中，Object.wait() 或 TIMED_WAITING  
- 7.	阻塞，<font color=#DC143C> Blocked（重点关注）</font>    
- 8.	停止，Parked  


**Dump文件中的线程状态含义及注意事项**

•	**Deadlock**：死锁线程，一般指多个线程调用间，进入相互资源占用，导致一直等待无法释放的情况。

•	**Runnable**：一般指该线程正在执行状态中，该线程占用了资源，正在处理某个请求，有可能正在传递SQL到数据库执行，有可能在对某个文件操作，有可能进行数据类型等转换。

•	**Waiting on condition**：等待资源，或等待某个条件的发生。具体原因需结合 stacktrace来分析。 
 
- 如果堆栈信息明确是应用代码，则证明该线程正在等待资源。一般是大量读取某资源，且该资源采用了资源锁的情况下，线程进入等待状态，等待资源的读取。  
- 又或者，正在等待其他线程的执行等。  
- 如果发现有大量的线程都在处在 Wait on condition，从线程 stack看，正等待网络读写，这可能是一个网络瓶颈的征兆。因为网络阻塞导致线程无法执行。    
--	一种情况是网络非常忙，几乎消耗了所有的带宽，仍然有大量数据等待网络读写；  
-- 	另一种情况也可能是网络空闲，但由于路由等问题，导致包无法正常的到达。  
- 另外一种出现 Wait on condition的常见情况是该线程在 sleep，等待 sleep的时间到了时候，将被唤醒。  


•	**Blocked**：线程阻塞，是指当前线程执行过程中，所需要的资源长时间等待却一直未能获取到，被容器的线程管理器标识为阻塞状态，可以理解为等待资源超时的线程。   

•	**Waiting for monitor entry 和 in Object.wait()**：Monitor是 Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 monitor。从下图1中可以看出，每个 Monitor在某个时刻，只能被一个线程拥有，该线程就是 “Active Thread”，而其它线程都是 “Waiting Thread”，分别在两个队列 “ Entry Set”和 “Wait Set”里面等候。在 “Entry Set”中等待的线程状态是 “Waiting for monitor entry”，而在 “Wait Set”中等待的线程状态是 “in Object.wait()”。

![java_monitor.png](https://i.loli.net/2017/12/06/5a278e71877cf.png)


**示例一：Waiting to lock 和 Blocked**  
实例如下：  

    "RMI TCP Connection(267865)-172.16.5.25" daemon prio=10 tid=0x00007fd508371000 nid=0x55ae waiting for monitor entry [0x00007fd4f8684000]
       java.lang.Thread.State: BLOCKED (on object monitor)
    at org.apache.log4j.Category.callAppenders(Category.java:201)
    - waiting to lock <0x00000000acf4d0c0> (a org.apache.log4j.Logger)
    at org.apache.log4j.Category.forcedLog(Category.java:388)
    at org.apache.log4j.Category.log(Category.java:853)
    at org.apache.commons.logging.impl.Log4JLogger.warn(Log4JLogger.java:234)
    at com.tuan.core.common.lang.cache.remote.SpyMemcachedClient.get(SpyMemcachedClient.java:110)


1）线程状态是 **Blocked**，阻塞状态。说明线程等待资源超时！  
2）“ <font color=#DC143C> waiting to lock <0x00000000acf4d0c0></font>”指，线程在等待给这个 0x00000000acf4d0c0 地址上锁（英文可描述为：trying to obtain  0x00000000acf4d0c0 lock）。  
3）在 dump 日志里查找字符串 0x00000000acf4d0c0，发现有大量线程都在等待给这个地址上锁。如果能在日志里找到谁获得了这个锁（如locked < 0x00000000acf4d0c0 >），就可以顺藤摸瓜了。  
4）“<font color=#DC143C>waiting for monitor entry</font>”说明此线程通过 synchronized(obj) {……} 申请进入了临界区，从而进入了上图中的“Entry Set”队列，但该 obj 对应的 monitor 被其他线程拥有，所以本线程在 Entry Set 队列中等待。  
5）第一行里，"<font color=#DC143C>RMI TCP Connection(267865)-172.16.5.25</font>"是 Thread Name 。tid指Java Thread id。nid指native线程的id。prio是线程优先级。[0x00007fd4f8684000]是线程栈起始地址。  


**示例二：Waiting on condition 和 TIMED_WAITING**  
实例如下：  

    "RMI TCP Connection(idle)" daemon prio=10 tid=0x00007fd50834e800 nid=0x56b2 waiting on condition [0x00007fd4f1a59000]
       java.lang.Thread.State: TIMED_WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for  <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:198)
    at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:424)
    at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:323)
    at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:874)
    at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:945)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:907)
    at java.lang.Thread.run(Thread.java:662)

1）“<font color=#DC143C>TIMED_WAITING (parking)</font>”中的 timed_waiting 指等待状态，但这里指定了时间，到达指定的时间后自动退出等待状态；parking指线程处于挂起中。

2）“<font color=#DC143C>waiting on condition</font>”需要与堆栈中的“<font color=#DC143C>parking to wait for  <0x00000000acd84de8> (a java.util.concurrent.SynchronousQueue$TransferStack)</font>”结合来看。首先，本线程肯定是在等待某个条件的发生，来把自己唤醒。其次，SynchronousQueue 并不是一个队列，只是线程之间移交信息的机制，当我们把一个元素放入到 SynchronousQueue 中时必须有另一个线程正在等待接受移交的任务，因此这就是本线程在等待的条件。


**示例三：in Obejct.wait() 和 TIMED_WAITING**  
实例如下：  

    "RMI RenewClean-[172.16.5.19:28475]" daemon prio=10 tid=0x0000000041428800 nid=0xb09 in Object.wait() [0x00007f34f4bd0000]
       java.lang.Thread.State: TIMED_WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    - waiting on <0x00000000aa672478> (a java.lang.ref.ReferenceQueue$Lock)
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:118)
    - locked <0x00000000aa672478> (a java.lang.ref.ReferenceQueue$Lock)
    at sun.rmi.transport.DGCClient$EndpointEntry$RenewCleanThread.run(DGCClient.java:516)
    at java.lang.Thread.run(Thread.java:662)


1）“<font color=#DC143C>TIMED_WAITING (on object monitor)</font>”，对于本例而言，是因为本线程调用了 java.lang.Object.wait(long timeout) 而进入等待状态。  
2）“Wait Set”中等待的线程状态就是“ <font color=#DC143C>in Object.wait() </font>”。当线程获得了 Monitor，进入了临界区之后，如果发现线程继续运行的条件没有满足，它则调用对象（一般就是被 synchronized 的对象）的 wait() 方法，放弃了 Monitor，进入 “Wait Set”队列。只有当别的线程在该对象上调用了 notify() 或者 notifyAll() ，“ Wait Set”队列中线程才得到机会去竞争，但是只有一个线程获得对象的 Monitor，恢复到运行态。   
3）RMI RenewClean 是 DGCClient 的一部分。DGC 指的是 Distributed GC，即分布式垃圾回收。  
4）请注意，是先 <font color=#DC143C>locked <0x00000000aa672478></font>，后 <font color=#DC143C>waiting on <0x00000000aa672478></font>，之所以先锁再等同一个对象，请看下面它的代码实现：  

    static private class  Lock { };
    private Lock lock = new Lock();
    public Reference<? extends T> remove(long timeout)
    {
    	synchronized (lock) {
    		Reference<? extends T> r = reallyPoll();
    		if (r != null) return r;
    		for (;;) {
    			lock.wait(timeout);
    			r = reallyPoll();
   				……
       }
    }
即，线程的执行中，先用 synchronized 获得了这个对象的 Monitor（对应于 <font color=#DC143C> locked <0x00000000aa672478></font> ）；当执行到 lock.wait(timeout);，线程就放弃了 Monitor 的所有权，进入“Wait Set”队列（对应于  <font color=#DC143C>waiting on <0x00000000aa672478> </font>）。
5）从堆栈信息看，是正在清理 remote references to remote objects ，引用的租约到了，分布式垃圾回收在逐一清理呢。
