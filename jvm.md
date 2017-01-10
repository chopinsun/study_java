# jvm

## 前言
参考文档：
1. [http://blog.csdn.net/witsmakemen/article/details/28600127](http://blog.csdn.net/witsmakemen/article/details/28600127)
2. [http://blog.csdn.net/witsmakemen/article/details/28600127](http://blog.csdn.net/cutesource/article/details/5904542)
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

从图中能看到JVM内存结构主要包括两个子系统和两个组件。两个子系统分别是Classloader子系统和Executionengine(执行引擎)子系统；两个组件分别是Runtimedataarea(运行时数据区域即内存空间)组件和Nativeinterface(本地接口)组件。

Classloader子系统的作用：

根据给定的全限定名类名(如java.lang.Object)来装载class文件的内容到Runtimedataarea中的methodarea(方法区域)。Java程序员可以extendsjava.lang.ClassLoader类来写自己的Classloader。

Executionengine子系统的作用：

执行classes中的指令。任何JVMspecification实现(JDK)的核心都是Executionengine，不同的JDK例如Sun的JDK和IBM的JDK好坏主要就取决于他们各自实现的Executionengine的好坏。

Nativeinterface组件：

与nativelibraries交互，是其它编程语言交互的接口。当调用native方法的时候，就进入了一个全新的并且不再受虚拟机限制的世界，所以也很容易出现JVM无法控制的nativeheapOutOfMemory。

RuntimeDataArea组件：

这就是我们常说的JVM的内存了。它主要分为五个部分——

1. Heap(堆)：一个Java虚拟实例中只存在一个堆空间

2. MethodArea(方法区域)：被装载的class的信息存储在Methodarea的内存中。当虚拟机装载某个类型时，它使用类装载器定位相应的class文件，然后读入这个class文件内容并把它传输到虚拟机中。

3. JavaStack(java的栈)：虚拟机只会直接对Javastack执行两种操作：以帧为单位的压栈或出栈

4. ProgramCounter(程序计数器)：每一个线程都有它自己的PC寄存器，也是该线程启动时创建的。PC寄存器的内容总是指向下一条将被执行指令的饿地址，这里的地址可以是一个本地指针，也可以是在方法区中相对应于该方法起始指令的偏移量。

5. Nativemethodstack(本地方法栈)：保存native方法进入区域的地址
