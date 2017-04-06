# jvm

## 前言
参考文档：
1. [http://blog.csdn.net/witsmakemen/article/details/28600127](http://blog.csdn.net/witsmakemen/article/details/28600127)
2. [http://blog.csdn.net/witsmakemen/article/details/28600127](http://blog.csdn.net/cutesource/article/details/5904542)
3. [http://www.cnblogs.com/ywl925/p/3925637.html](http://www.cnblogs.com/ywl925/p/3925637.html)
``

 这篇文章是基于自己学习jvm时做的一些总结，主要是为了给自己留笔记。内容主要来自博文，真实来源已不可考。其中加入了一些我自己的理解，文笔不好，但好在通俗。
``

  从几个方面来学习：

  1. jvm是如何工作的
  2. jvm结构体系
  3. gc及原理
  4. jvm优化和参数列表

## 一. jvm是如何工作的
### 1.jvm 虚拟机的生命周期

    main() 方法是jvm的起点，jvm会一直运行直到所有的非守护线程(Daemon Thread)执行完毕，当然如果有足够权限可以调用exit()强制结束。

    普通进程都是非守护进程，可以通过thread.setDaemon(true)将一个普通线程设置为守护线程。GC就是典型的守护线程。守护线程会在所有非守护线程结束后结束。

### 2.jvm编译执行原理

先看一张图：
![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/241533100174700.jpg)
<center style="font-size:14px;color:#999;">图1.1</center>



首先程序猿编写了.java文件，jvm编译器将.java文件进行编译，生成.class文件，编译过程如下:
* 分析和输入到符号表
* 注解处理
* 语义分析和生成class文件

最后生成的class文件由以下部分组成：

* 结构信息。包括class文件格式版本号及各部分的数量与大小的信息
* 元数据。对应于Java源码中声明与常量的信息。包含类/继承的超类/实现的接口的声明信息、域与方法声明信息和常量池
* 方法信息。对应Java源码中语句和表达式对应的信息。包含字节码、异常处理器表、求值栈与局部变量区大小、求值栈的类型记录、调试符号信息

Java字节码的执行是由JVM执行引擎来完成，流程图如下所示：
![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/241533362431950.jpg)
<center style="font-size:14px;color:#999;">图1.2</center>

在程序运行时，jvm会调用classloader加载class文件，并根据class文件记录的信息，存入内存空间中的方法区、堆、栈、本地方法栈中。执行引擎从java堆栈中获取数据和指令，并从方法区中获取地址，然后转换成汇编语言执行命令。如果是本地方法，会调用本地方法接口执行，这将脱离jvm控制。

### 类加载机制
JVM的类加载是通过ClassLoader及其子类来完成的，类的层次关系和加载顺序可以由下图来描述：
![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/241535328298264.jpg)
<center style="font-size:14px;color:#999;">图1.2</center>

1. Bootstrap ClassLoader 是c写的，会加载rt.jar或者Xbootclasspath选项指定的jar包。
2. Extension ClassLoader 是java写的。会加载JRE/lib/ext/下的jar包，都是一些扩展包。
3. App ClassLoader 是java写的。AppClassLoader继承自URLClassLoader，继续向上继承SecureClassLoader，最终继承java.lang.ClassLoader

ExtClassLoader和AppClassLoader 都是由classLoader子系统的入口 sun.misc.Launcher.Launcher进行调用的。核心方法有2个：findClass、loadClass。
loadClass是由它们共同的父类classLoader实现的，只是各自实现了自己的findClass方法。loadClass方法把上一个classLoader作为参数传入，以实现从下往上的检查和从上往下的加载。

程序猿可以编写自己的classLoader，继承java.lang.ClassLoader，并在调用时，指定父级加载器。

## 二. jvm结构体系
jvm基本结构如下图：

![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/241532275353627.jpg)
<center style="font-size:14px;color:#999;">图2.1</center>

从上图能清晰看到Java平台包含的各个逻辑模块，也能了解到JDK与JRE的区别。

JVM自身的物理结构：
![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/241532375103105.jpg)
<center style="font-size:14px;color:#999;">图2.2</center>

从图中能看到JVM内存结构主要包括两个子系统和两个组件。两个子系统分别是Classloader子系统和Executionengine(执行引擎)子系统；两个组件分别是Runtimedataarea(运行时数据区域即内存空间)组件和Nativeinterface(本地接口)组件。

Classloader子系统的作用：

根据给定的全限定名类名(如java.lang.Object)来装载class文件的内容到Runtimedataarea中的methodarea(方法区域)。Java程序员可以extendsjava.lang.ClassLoader类来写自己的Classloader。

Executionengine子系统的作用：

执行classes中的指令。任何JVMspecification实现(JDK)的核心都是Executionengine，不同的JDK例如Sun的JDK和IBM的JDK好坏主要就取决于他们各自实现的Executionengine的好坏。

Nativeinterface组件：

与nativelibraries交互，是其它编程语言交互的接口。当调用native方法的时候，就进入了一个全新的并且不再受虚拟机限制的世界，所以也很容易出现JVM无法控制的nativeheapOutOfMemory。

RuntimeDataArea组件：

这就是我们常说的JVM的内存了（第三章详细解释）。它主要分为五个部分——

1. Heap(堆)：一个Java虚拟实例中只存在一个堆空间。

2. MethodArea(方法区域)：被装载的class的信息存储在Methodarea的内存中。当虚拟机装载某个类型时，它使用类装载器定位相应的class文件，然后读入这个class文件内容并把它传输到虚拟机中。

3. JavaStack(java的栈)：虚拟机只会直接对Javastack执行两种操作：以帧为单位的压栈或出栈

4. ProgramCounter(程序计数器)：每一个线程都有它自己的PC寄存器，也是该线程启动时创建的。PC寄存器的内容总是指向下一条将被执行指令的饿地址，这里的地址可以是一个本地指针，也可以是在方法区中相对应于该方法起始指令的偏移量。

5. Nativemethodstack(本地方法栈)：保存native方法进入区域的地址

## 三. gc及原理

要谈gc，就先要谈JVM内存结构，所以我们再回头看下图2.2。

重新详细的看一下内存结构
1. 堆
所有通过new创建的对象的内存都在堆中分配，其大小可以通过-Xmx和-Xms来控制。堆被划分为新生代和旧生代，新生代又被进一步划分为Eden和Survivor区，最后Survivor由FromSpace和ToSpace组成，结构图如下所示：
![jvm结构](https://raw.githubusercontent.com/chopinsun/study_java/master/18100723-e6a134c47e9440cd99fe67f49079ff1f.gif)
<center style="font-size:14px;color:#999;">图3.1</center>


新生代。新建的对象都是用新生代分配内存，Eden空间不足的时候，会把存活的对象转移到Survivor中，新生代大小可以由-Xmn来控制，也可以用-XX:SurvivorRatio来控制Eden和Survivor的比例旧生代。  
用于存放新生代中经过多次垃圾回收仍然存活的对象
新生代通常存活时间较短，因此基于Copying算法来进行回收，所谓Copying算法就是扫描出存活的对象，并复制到一块新的完全未使用的空间中，对应于新生代，就是在Eden和FromSpace或ToSpace之间copy。  
新生代采用空闲指针的方式来控制GC触发，指针保持最后一个分配的对象在新生代区间的位置，当有新的对象要分配内存时，用于检查空间是否足够，不够就触发GC。当连续分配对象时，对象会逐渐从eden到survivor，最后到旧生代。

2. 栈
每个线程执行每个方法的时候都会在栈中申请一个栈帧，每个栈帧包括局部变量区和操作数栈，用于存放此次方法调用过程中的临时变量、参数和中间结果
3. 本地方法栈
用于支持native方法的执行，存储了每个native方法调用的状态
4. 方法区
存放了要加载的类信息、静态变量、final类型的常量、属性和方法信息。JVM用持久代(PermanetGeneration)来存放方法区，可通过-XX:PermSize和-XX:MaxPermSize来指定最小值和最大值。

了解了基本概念，那接下来看看gc怎么工作的

2. JVM垃圾回收机制

JVM分别对新生代和旧生代采用不同的垃圾回收机制

新生代的GC：

新生代通常存活时间较短，因此基于Copying算法来进行回收，所谓Copying算法就是扫描出存活的对象，并复制到一块新的完全未使用的空间中，对应于新生代，就是在Eden和FromSpace或ToSpace之间copy。新生代采用空闲指针的方式来控制GC触发，指针保持最后一个分配的对象在新生代区间的位置，当有新的对象要分配内存时，用于检查空间是否足够，不够就触发GC。当连续分配对象时，对象会逐渐从eden到survivor，最后到旧生代，用javavisualVM来查看，能明显观察到新生代满了后，会把对象转移到旧生代，然后清空继续装载，当旧生代也满了后，就会报outofmemory的异常
在执行机制上JVM提供了串行GC(SerialGC)、并行回收GC(ParallelScavenge)和并行GC(ParNew)

1)串行GC

在整个扫描和复制过程采用单线程的方式来进行，适用于单CPU、新生代空间较小及对暂停时间要求不是非常高的应用上，是client级别默认的GC方式，可以通过-XX:+UseSerialGC来强制指定

2)并行回收GC

在整个扫描和复制过程采用多线程的方式来进行，适用于多CPU、对暂停时间要求较短的应用上，是server级别默认采用的GC方式，可用-XX:+UseParallelGC来强制指定，用-XX:ParallelGCThreads=4来指定线程数

3)并行GC

与旧生代的并发GC配合使用

旧生代的GC：

旧生代与新生代不同，对象存活的时间比较长，比较稳定，因此采用标记(Mark)算法来进行回收，所谓标记就是扫描出存活的对象，然后再进行回收未被标记的对象，回收后对用空出的空间要么进行合并，要么标记出来便于下次进行分配，总之就是要减少内存碎片带来的效率损耗。在执行机制上JVM提供了串行GC(SerialMSC)、并行GC(parallelMSC)和并发GC(CMS)，具体算法细节还有待进一步深入研究。

以上各种GC机制是需要组合使用的，指定方式由下表所示：
![GC机制](https://raw.githubusercontent.com/chopinsun/study_java/master/18100725-71016a87e5b04416bcbf299519ff1114.png)

### 垃圾回收机制的算法

说到垃圾回收（Garbage Collection，GC），很多人就会自然而然地把它和Java联系起来。在Java中，程序员不需要去关心内存动态分配和垃圾回收的问题，这一切都交给了JVM来处理。顾名思义，垃圾回收就是释放垃圾占用的空间，那么在Java中，什么样的对象会被认定为“垃圾”？那么当一些对象被确定为垃圾之后，采用什么样的策略来进行回收（释放空间）？在目前的商业虚拟机中，有哪些典型的垃圾收集器？下面我们就来逐一探讨这些问题。以下是本文的目录大纲：

　　一.如何确定某个对象是“垃圾”？

　　二.典型的垃圾收集算法

　　三.典型的垃圾收集器

一.如何确定某个对象是“垃圾”？

　　在这一小节我们先了解一个最基本的问题：如果确定某个对象是“垃圾”？既然垃圾收集器的任务是回收垃圾对象所占的空间供新的对象使用，那么垃圾收集器如何确定某个对象是“垃圾”？—即通过什么方法判断一个对象可以被回收了。

　　在java中是通过引用来和对象进行关联的，也就是说如果要操作对象，必须通过引用来进行。那么很显然一个简单的办法就是通过引用计数来判断一个对象是否可以被回收。不失一般性，如果一个对象没有任何引用与之关联，则说明该对象基本不太可能在其他地方被使用到，那么这个对象就成为可被回收的对象了。这种方式成为引用计数法。

　　这种方式的特点是实现简单，而且效率较高，但是它无法解决循环引用的问题，因此在Java中并没有采用这种方式（Python采用的是引用计数法）。如果有A和B两个对象，分别访问对方，将导致它们的引用计数都不为0，那么垃圾收集器就永远不会回收它们。
　  为了解决这个问题，在Java中采取了 可达性分析法。该方法的基本思想是通过一系列的“GC Roots”对象作为起点进行搜索，如果在“GC Roots”和一个对象之间没有可达路径，则称该对象是不可达的，不过要注意的是被判定为不可达的对象不一定就会成为可回收对象。被判定为不可达的对象要成为可回收对象必须至少经历两次标记过程，如果在这两次标记过程中仍然没有逃脱成为可回收对象的可能性，则基本上就真的成为可回收对象了。

二.典型的垃圾收集算法

　　在确定了哪些垃圾可以被回收后，垃圾收集器要做的事情就是开始进行垃圾回收，但是这里面涉及到一个问题是：如何高效地进行垃圾回收。由于Java虚拟机规范并没有对如何实现垃圾收集器做出明确的规定，因此各个厂商的虚拟机可以采用不同的方式来实现垃圾收集器，所以在此只讨论几种常见的垃圾收集算法的核心思想。

　　1.Mark-Sweep（标记-清除）算法

　　这是最基础的垃圾回收算法，之所以说它是最基础的是因为它最容易实现，思想也是最简单的。标记-清除算法分为两个阶段：标记阶段和清除阶段。标记阶段的任务是标记出所有需要被回收的对象，清除阶段就是回收被标记的对象所占用的空间。具体过程如下图所示：
![GC机制](https://raw.githubusercontent.com/chopinsun/study_java/master/Mark-Sweep.jpg)
　 从图中可以很容易看出标记-清除算法实现起来比较容易，但是有一个比较严重的问题就是容易产生内存碎片，碎片太多可能会导致后续过程中需要为大对象分配空间时无法找到足够的空间而提前触发新的一次垃圾收集动作。
    2.Copying（复制）算法

　　为了解决Mark-Sweep算法的缺陷，Copying算法就被提了出来。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用的内存空间一次清理掉，这样一来就不容易出现内存碎片的问题。具体过程如下图所示：
![GC机制](https://raw.githubusercontent.com/chopinsun/study_java/master/Copying.jpg)

这种算法虽然实现简单，运行高效且不容易产生内存碎片，但是却对内存空间的使用做出了高昂的代价，因为能够使用的内存缩减到原来的一半。

　　很显然，Copying算法的效率跟存活对象的数目多少有很大的关系，如果存活对象很多，那么Copying算法的效率将会大大降低。

3.Mark-Compact（标记-整理）算法

　　为了解决Copying算法的缺陷，充分利用内存空间，提出了Mark-Compact算法。该算法标记阶段和Mark-Sweep一样，但是在完成标记之后，它不是直接清理可回收对象，而是将存活对象都向一端移动，然后清理掉端边界以外的内存。具体过程如下图所示：
![GC机制](https://raw.githubusercontent.com/chopinsun/study_java/master/Mark-Compact.jpg)

4.Generational Collection（分代收集）算法

　　分代收集算法是目前大部分JVM的垃圾收集器采用的算法。它的核心思想是根据对象存活的生命周期将内存划分为若干个不同的区域。一般情况下将堆区划分为老年代（Tenured Generation）和新生代（Young Generation），老年代的特点是每次垃圾收集时只有少量对象需要被回收，而新生代的特点是每次垃圾回收时都有大量的对象需要被回收，那么就可以根据不同代的特点采取最适合的收集算法。

　　目前大部分垃圾收集器对于新生代都采取Copying算法，因为新生代中每次垃圾回收都要回收大部分对象，也就是说需要复制的操作次数较少，但是实际中并不是按照1：1的比例来划分新生代的空间的，一般来说是将新生代划分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden空间和其中的一块Survivor空间，当进行回收时，将Eden和Survivor中还存活的对象复制到另一块Survivor空间中，然后清理掉Eden和刚才使用过的Survivor空间。

　　而由于老年代的特点是每次回收都只回收少量对象，一般使用的是Mark-Compact算法。

　　注意，在堆区之外还有一个代就是永久代（Permanet Generation），它用来存储class类、常量、方法描述等。对永久代的回收主要回收两部分内容：废弃常量和无用的类。

三.典型的垃圾收集器

　　垃圾收集算法是 内存回收的理论基础，而垃圾收集器就是内存回收的具体实现。下面介绍一下HotSpot（JDK 7)虚拟机提供的几种垃圾收集器，用户可以根据自己的需求组合出各个年代使用的收集器。

　　1.Serial/Serial Old

　　Serial/Serial Old收集器是最基本最古老的收集器，它是一个单线程收集器，并且在它进行垃圾收集时，必须暂停所有用户线程。Serial收集器是针对新生代的收集器，采用的是Copying算法，Serial Old收集器是针对老年代的收集器，采用的是Mark-Compact算法。它的优点是实现简单高效，但是缺点是会给用户带来停顿。

　　2.ParNew

　　ParNew收集器是Serial收集器的多线程版本，使用多个线程进行垃圾收集。

　　3.Parallel Scavenge

　　Parallel Scavenge收集器是一个新生代的多线程收集器（并行收集器），它在回收期间不需要暂停其他用户线程，其采用的是Copying算法，该收集器与前两个收集器有所不同，它主要是为了达到一个可控的吞吐量。

　　4.Parallel Old

　　Parallel Old是Parallel Scavenge收集器的老年代版本（并行收集器），使用多线程和Mark-Compact算法。

　　5.CMS

　　CMS（Current Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，它是一种并发收集器，采用的是Mark-Sweep算法。

　　6.G1

　　G1收集器是当今收集器技术发展最前沿的成果，它是一款面向服务端应用的收集器，它能充分利用多CPU、多核环境。因此它是一款并行与并发收集器，并且它能建立可预测的停顿时间模型。
