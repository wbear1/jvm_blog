# 从HelloWorld程序分析java应用的内存

先看一个HelloWorld程序：

在4core, 8g机器centos操作系统（版本号：2.6.32-431.el6.x86_64）上执行如下命令：

java -Xmx20m -Xms20m -XX:PermSize=20m -XX:MaxPermSize=20m Test
![start](https://github.com/wbear1/jvm_blog/blob/master/img/memory/start.png)


执行结果不言而喻，但这时候该程序分别占用多少物理内存和多少虚拟内存呢？用top -p 命令查看这个数值如下：
![jps](https://github.com/wbear1/jvm_blog/blob/master/img/memory/jps.png)

![top](https://github.com/wbear1/jvm_blog/blob/master/img/memory/top.png)


物理内存占用35m, 虚拟内存占用1217m。如果这与你的结果有很大差异，也不必大惊小怪，看完下面的分析过程也能解释你的结果。

下面就分析这35m的物理内存和1217m的虚拟内存到底用在什么地方。

首先，第一个能想到的肯定是用jmap，执行jmap -heap 24697看看内存里对象占用情况。
![jmap](https://github.com/wbear1/jvm_blog/blob/master/img/memory/jmap.png)

新生代使用了0.86MB，永久代使用了2.39MB，加起来还不3.25MB，那35m是什么鬼? 1217m又去哪了？

通过查看jvm相关文档，知道jvm提供了Native Memory Tracking（简称：NMT）方法，可以跟踪显示java程序在操作系统内存的使用。不过需要在启动java程序时加上-XX:NativeMemoryTracking=detail参数。执行命令：jcmd 24697 VM.native_memory detail。（注意：24697在启动时已加上-XX:NativeMemoryTracking=detail参数 ），关于NMT和jcmd的详细使用可以参考：
http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
http://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr006.html#BABEHABG
![jcmd](https://github.com/wbear1/jvm_blog/blob/master/img/memory/jcmd.png)

执行结果汇总了24697程序的内存使用情况和详细的虚拟内存。reserved是jvm通过mmaped PROT_NONE申请的虚拟地址空间，在页表中已经存在记录，页是不可访问，commited是jvm向操作系统实际分配的内存，mmpaed PROT_READ|PORT_WRITE 页可读写。操作对于内存的分配是延迟分配，只有真正访问内存页才能真正属于该进程，因此reserved和commited的内存不能完全算作rss（即top输出的物理内存）。
这时候，我们使用pmap命令，从操作系统角度来查看该进程的内存映射情况。执行命令pmap -x 24697
![pmap](https://github.com/wbear1/jvm_blog/blob/master/img/memory/pmap.png)
![pmap1](https://github.com/wbear1/jvm_blog/blob/master/img/memory/pmap1.png)

这里显示了该进程内存的起始地址、长度、实际的物理内存长度、脏页、权限和映射（可能是文件、分配的物理内存anon和stack程序栈）。这里pmap输出结果跟top的输出结果是一致。再根据之前的jcmd的结果和pmap结果，我们可以作出如下分析：
![ans](https://github.com/wbear1/jvm_blog/blob/master/img/memory/ans.png)

左边是用jcmd展示的结果，右边是用pmap展示的结果。可以看出虚拟内存由堆内存和非堆内存组成，堆内存的大小由Xmx和PermSize设定了，对于非堆内存（包括GC、JIT、Threads、class和classloader、文件映射）。
仔细观察pmap结果可以发现，对于每个线程栈占用了1028KB虚拟内存，执行java -XX:+PrintFlagsFinal获取默认的ThreadStackSize，其值为1024
![printFlag](https://github.com/wbear1/jvm_blog/blob/master/img/memory/printFlag.png)

另外，pmap结果中还有12个连续的65404KB的地址空间，这应该是个什么特殊的东西吧？经过多方探查，发现glibc为了内存分配的性能问题，使用了很多叫做arena的memory pool，每个线程分分配一块arena，在64位的机器下每个arena默认是64M，一个进程最多可以有cpu cores * 8个arena。对于本次测试机器而言，最多可以有4 * 8 = 32个arena。当然这个可以通过环境变量进行设置。在redhat官网上有如下说明：
![note](https://github.com/wbear1/jvm_blog/blob/master/img/memory/note.png)

对于2.11版本之后的glibc，提供该功能。查看下本次测试系统的glibc版本， 2.12版本
![glibc](https://github.com/wbear1/jvm_blog/blob/master/img/memory/glibc.png)

综上：本文开头的两个问题也得到了解答。物理内存只有在真正访问物理内存空间时才算，对于jvm而言，包括堆内存和非堆内存，非堆内存是不包括在Xmx和PermSize参数里的；虚拟内存很高的原因主要有2个：glibc的内存管理机制，以及系统库的文件映射，当然还有一些零零碎碎的，此次未作深入分析。
在分析过程中也留有一些问题：

1）jvm运行时的进程数量，jstack的结果和top的结果会相符吗？

2）线程栈为什么占用1028KB，而不是1024KB，多出来的4KB去哪了？

3）pmap的结果中有一些零散的地址空间，在jcmd的结果中未找到映射，这部分空间用来干吗？

任重而道远，留待下次分析吧。

下面有几篇文章可供参考：
https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_glibc_2_10_rhel_6_malloc_may_show_excessive_virtual_memory_usage?lang=en

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/6.0_Release_Notes/compiler.html#idp942672

http://blog.jobbole.com/83878/