# 奇葩的排查经历

简述下背景，在通过op发布系统发布java后台项目iot-hub-parser（由于此次问题跟iot-hub-parser业务实现无关，不再介绍iot-hub-parser业务用途 ）后，程序无法启动，下面是开始问题排查了：

1. 查看日志
![log](https://github.com/wbear1/jvm_blog/blob/master/img/strange/log.png)

日志比较清楚，在初始化过程出现了NullPointerException，那翻下代码看看：
![src1](https://github.com/wbear1/jvm_blog/blob/master/img/strange/src1.png)

![src2](https://github.com/wbear1/jvm_blog/blob/master/img/strange/src2.png)

![src3](https://github.com/wbear1/jvm_blog/blob/master/img/strange/src3.png)

很明显就是没有读取Lion配置项iot-mqtt-broker.downstream.topics，首先是去lion检查配置项，确定已配置该项。这里也尝试打开debug日志，并未获取更有价值的信息。

2. 查看系统环境app.properties配置

在配置项已存在的情况，无法读取的配置，开始怀疑是环境的关系，于是立马检查该机器的环境配置，由于是线上机器，这里隐去数据，确认也是没问题的。
![env](https://github.com/wbear1/jvm_blog/blob/master/img/strange/env.png)

3. 从测试环境拉包验证

在发布到生产环境之前，代码svn的tag已经发布到测试环境，并且经过验证是正常运行的。于是从测试环境把包拉到生产环境，发现能够正常运行。于是，开始怀疑从op系统拉包到生产环境的过程中，可能发生了文件损坏。于是比对两者的文件区别。

4. 比对测试环境和生产环境jar包md5值

可以看到，两者从启动脚本（bin目录中的文件）、依赖jar包（lib目录中的文件）到配置文件（resources目录中的文件），所有文件的md5值完全一样。这就有点抓狂了！

![md5](https://github.com/wbear1/jvm_blog/blob/master/img/strange/md5.png)

5. 文件替换初步定位问题

尽管文件md5值都一样，猜测两者之间肯定有什么地方不一样，于是采取文件替换的方式一步步排查。先替换bin目录，然后是resources目录，都不能正常运行，最后替换lib目录后发现能够正常运行。于是怀疑是不是文件访问属性的问题，检查发现文件访问属性也是一样的。而且还发现，只要将lib目录中的所有jar包mv到其它目录，然后把那些jar包mv回lib目录下，程序也能够正常运行起来。 这就很让人头疼了！！

》》》》》》》开启艰难的过程。。

1）既然都替换lib目录能够正常运行，那是不是目录本身有问题，于是mv/rm来了几下，也没发现目录有什么问题；

2）文件inode是不是变了，发现mv并不会改变文件inode；

3）由于是在azure虚拟机上运行，怀疑会不会是底层硬件的问题导致文件损坏。。。这个比较难排查，通过部署到其它线上机器，发现也不能运行，初步判定不大可能是硬件问题；

4）想通过jvm的角度来排查，看看类加载有什么区别，打开TraceClassLoading开关跟踪加载过程，发现jvm是采用并行加载的方式，每次加载顺序都不一样，也无法继续排查；

5）lib目录下面有169个依赖的jar包，于是采用一个个替换的方式，想找出最终是哪个jar包影响的。最后定位到是下面两个文件：将这两个文件从lib目录mv出去，再mv回lib目录，程序竟然就能够运行了，而且还必须是按这个顺序才行。于是，开始有点头绪了，怀疑是包冲突导致的jar包加载顺序发生了改变。 
![order](https://github.com/wbear1/jvm_blog/blob/master/img/strange/order.png)

6）开启TraceClassLoading跟踪类加载过程

再次跟踪类加载过程，这次主要关注netty依赖包的加载，果然，mv前后，同一个类会从不同的jar包加载。再去查看pom文件，确实是有包冲突的。解决包冲突后，再次发布，ok了。问题来了，为什么mv操作会导致jar包的加载顺序发生改变呢？
![compare](https://github.com/wbear1/jvm_blog/blob/master/img/strange/compare.png)

7）了解classpath内部机制

看下我们的程序是如何启动j的，在启动脚本中有如下启动命令：
![startup](https://github.com/wbear1/jvm_blog/blob/master/img/strange/startup.png)

执行输出日志如下：
![startlog](https://github.com/wbear1/jvm_blog/blob/master/img/strange/startlog.png)

可以看到classpath是一个包含*通配符的路径字符串，那jvm是如何是处理classpath的呢？且看下jvm官方文档https://docs.oracle.com/javase/6/docs/technotes/tools/windows/classpath.html中的解释：里面提到*并不包含目录里面的class文件，不包括子目录，只包含当前目录的jar文件和JAR文件，并且应用程序不应该依赖于classpath中jar包出现的顺序。
![wildcards](https://github.com/wbear1/jvm_blog/blob/master/img/strange/wildcards.png)

那jvm到底是如何对classpath进行设置的呢，从openjdk的源码看看实现：
![openjdk1](https://github.com/wbear1/jvm_blog/blob/master/img/strange/openjdk1.png)
![openjdk2](https://github.com/wbear1/jvm_blog/blob/master/img/strange/openjdk2.png)

通过调用dirent.h提供的readdir来遍历目录里面的所有jar文件，也就是直接读取direntry。这里其实和ls -U或者ls -f的执行结果是一致的，可以看下ls.c的源码：

先通过readdir系统调用读取目录内容，如果是-U或者-f的话，不排序直接返回
![ls1](https://github.com/wbear1/jvm_blog/blob/master/img/strange/ls1.png)
![ls2](https://github.com/wbear1/jvm_blog/blob/master/img/strange/ls2.png)

通过下面的一个小程序，也能证明结果的确如此：该程序的输出和ls -U的结果是一致的。
![ls3](https://github.com/wbear1/jvm_blog/blob/master/img/strange/ls3.png)

这个顺序和jvm中加载的顺序也是一致的，可以通过下面这个小程序来验证。
![check](https://github.com/wbear1/jvm_blog/blob/master/img/strange/check.png)

那关于readdir返回的direntry顺序，是和directory的存储设计相关。下面这篇文章https://utcc.utoronto.ca/~cks/space/blog/unix/ReaddirOrder解释的比较清楚：早期采用线线数组的方式存储，现在比较多采用平衡树的结构存储
![readdir](https://github.com/wbear1/jvm_blog/blob/master/img/strange/readdir.png)

因此，readdir返回顺序并不是绝对的，它依赖操作系统底层的实现，这也再次验证了jvm官方文档所提醒，classpath中不应该依赖jar的顺序。

而之所以测试环境能够通过，生产环境不能通过，因为两个环境的机器linux内核版本有差别，测试环境为2.6.*，生产环境为3.10.*。

将文件mv到目录后，如果目录的存储采用线性数组的数据结构，那么文件在目录内容的顺序不会改变；如果采用平衡树的话，很有可能就会发生改变。

8）总结
  
到这里，线上发布的问题清楚了，根本原因是由于maven依赖冲突导致的lion不能正常工作，无法获取到配置。而之所以排查耗费如此之久，一是某些jar把exception捕获后没有合理输出且没有向上抛出，二是缺少一定的敏感性，有中间的很多环节其实可以联想到maven依赖的，最后是对jvm的理解不够深入，在md5一样的情况下，应该想到加载顺序。