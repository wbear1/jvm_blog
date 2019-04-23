# 填坑之旅

在某次xx项目发布后，8core的机器cpu很快被打爆，如下图所示：
![start](https://github.com/wbear1/jvm_blog/blob/master/img/sad/start.png)

根据jstack进行进程dump stack，经过分析，发现有多个线程都处于ResourceLeakDetector.newRecord方法上，如下所示：
![newRecord](https://github.com/wbear1/jvm_blog/blob/master/img/sad/newRecord.png)

从完整的堆栈来看，这些线程也是处于正常的工作流程。之前对Netty的ResourceLeakDetector也并不了解，翻了下源码：ResourceLeakDetector类是Netyy通过java对象引用来检测对象内存是否泄露的，并且其有一个-Dio.netty.leakDetectionLevel的配置，用于配置检测的不同级别。有disabled、simple、advanced和paranoid四种级别，检测粒度逐级递增。所谓的检测粒度是指检测的采样频率的高低，disabled级别是不检测，simple和advanced级别是采样检测，paranoid级别是每个对象都检测。被检测对象在被调用时，会触发记录调用栈帧，这个过程是比较耗时的（这个可以实验统计下？）。
![code1](https://github.com/wbear1/jvm_blog/blob/master/img/sad/code1.png)
![code2](https://github.com/wbear1/jvm_blog/blob/master/img/sad/code2.png)
![code3](https://github.com/wbear1/jvm_blog/blob/master/img/sad/code3.png)

发现启动脚本中指定了检测级别为paranoid，所以导致了上述的打爆cpu的现象。在测试环境下，通过模拟线上成比例的压力，也能复现这个问题，而将检测级别调成disabled，cpu使用率立马降下来了。
![script](https://github.com/wbear1/jvm_blog/blob/master/img/sad/script.png)

不过，此次发布并未修改启动脚本，理论上之前的版本也会有问题才对。

经过分析，认为此次的代码修改并不会导致cpu打爆，有可能是修改了pom.xml文件（将一些无关依赖去掉）导致的。两个版本比对后，发现相对于上一个版本，依赖包有很多不同（见下图，这些jar包是上一个版本包含，此版本不包含的）。

那合理的解释就是：这些jar包导致了之前的版本中io.netty.leakDetectionLevel设置未生效。接下来的问题就变成，哪些jar包会影响leak detection呢？
![jar](https://github.com/wbear1/jvm_blog/blob/master/img/sad/jar.png)

到这里有两种思路：

1）研究Netty的resource detection机制

2）逐一排查jar包，看到底是哪个jar包导致的，然后再反向去查找原因

不管采用何种思路，最终要彻底搞清楚这个问题，都需要理解resource detection机制（后面写文章详解）。

再回过头看看之前dump出来的stack，发现主要是针对ByteBuf对象的检测，而ByteBuf是Netty中用途十分广泛的数据结构，如果真的对每个ByteBuf对象进行检测的话，必然会有问题。那顺着这个思路，看看netty中是如何管理ByteBuf的。

netty中ByteBuf的抽象程度很高，设计复杂程序较高，包含如下种类的ByteBuf：
![bytebuf](https://github.com/wbear1/jvm_blog/blob/master/img/sad/bytebuf.png)

具体要使用何种ByteBuf，是由ByteBufAllocator来负责分配，allocator根据环境和配置来决定使用何种ByteBuf。

比如：是否使用堆外内存、是否使用Unsafe、是否是Andriod环境等，来选择使用何种ByteBuf(heap还是direct)，并且在最后，根据io.netty.leakDetectionLevel配置的级别，使用SimpleLeakAwareByteBuf或者AdvancedLeakAwareByteBuf包装ByteBuf，来进行内存泄露
检测。这里有一个例外，那就是如果是android环境，那么是不会使用内存泄露检测的。而检测是否android运行环境的方法，也很简单粗暴：能否通过反射来实例化android.app.Application类，如果可以，就说明当前运行环境是android环境。再看看之前两个eos_mqtt版本的jar包对比，其中有一个就是android-4.1.1.4.jar，到这里，似乎就快接近真相了。为了验证这个android依赖包就是真凶，做了一个实验，单独把这个android-4.1.1.4.jar加入这次发布版本的lib目录下，重新测试运行，果然，cpu很平稳，没有再出现打爆的现象。

分配流程可见下图：
![flow](https://github.com/wbear1/jvm_blog/blob/master/img/sad/flow.png)

检测是否andriod的方法：
![andriod](https://github.com/wbear1/jvm_blog/blob/master/img/sad/andriod.png)

那为什么毫不相关的android依赖包会进入到纯后台的服务中呢？查看maven的dependency:tree，看看是如何引入的？额，这就尴尬了，竟然是某个其它服务的sdk依赖。。。
![tree](https://github.com/wbear1/jvm_blog/blob/master/img/sad/tree.png)

到这里，算是真相大白了：不知从何开始，启动脚本加上了参数io.netty.leakDetectionLevel=paranoid，开启了内存泄露检测，但这时候流量不大，并不会导致什么问题；后来引入了android-4.1.1.4.jar包，这个时候，io.netty.leakDetectionLevel开始失效了；随着业务增长，流量变大，再开启paranoid级别的检测，机器就会不堪重负，这次发布就是踩到这个坑了。

最后作个总结：

**1）线上环境，不要把用于调试或者测试的方法、参数、配置放上去，比如这次的io.netty.leakDetectionLevel=paranoid，否则就是在埋雷；**

**2）发布sdk依赖时，尽可能精简，无关依赖一律不加，否则就是在坑队友；**

**3）每次改动，引入或者删除新的配置、依赖等，做到心中有数，尤其是依赖，当项目复杂时，会触发很多不可预知的行为；**

**4）有时间继续研究netty的内存管理、内存检测等，在排查过程中发现设计还是很精妙的。**
