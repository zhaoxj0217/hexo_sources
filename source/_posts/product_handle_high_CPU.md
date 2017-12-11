---

title: 线上问题处理-CPU过高
data: 2017-11-29
categories: accident handling,linux
tags: [accident handling,linux]

---

某服务器上部署了若干tomcat实例，即若干垂直切分的Java站点服务，以及若干Java微服务，突然收到运维的CPU异常告警。  
问：如何定位是哪个服务进程导致CPU过载，哪个线程导致CPU过载，哪段代码导致CPU过载？


**步骤一、找到最耗CPU的进程**

工具：top  
方法：  
•	执行top -c ，显示进程运行信息列表  
•	键入P (大写p)，进程按照CPU使用率排序  
![top_linux.png](https://i.loli.net/2017/12/06/5a279cce5c796.png)
如上图，最耗CPU的进程PID为10765

**步骤二：找到最耗CPU的线程**  
工具：top  
方法：  
•	top -Hp 10765 ，显示一个进程的线程运行信息列表  
•	键入P (大写p)，线程按照CPU使用率排序  
![top_hp_linux.png](https://i.loli.net/2017/12/06/5a279d2c284d8.png)
如上图，进程10765内，最耗CPU的线程PID为10804

**步骤三：将线程PID转化为16进制**  
工具：printf   
方法：printf “%x\n” 10804   
![printf_16_linux.png](https://i.loli.net/2017/12/06/5a279db425e22.png)
如上图，10804对应的16进制是0x2a34，当然，这一步可以用计算器。
 
之所以要转化为16进制，是因为堆栈里，线程id是用16进制表示的。


**步骤四：查看堆栈，找到线程在干嘛**  
工具：pstack/jstack/grep  
方法：jstack 10765 | grep ‘0x2a34’ -C5 --color  
•	打印进程堆栈  
•	通过线程id，过滤得到线程堆栈  
![jstack_grep_linux.png](https://i.loli.net/2017/12/06/5a279e337516e.png)
如上图，找到了耗CPU高的线程对应的线程名称“AsyncLogger-1”，以及看到了该线程正在执行代码的堆栈。

