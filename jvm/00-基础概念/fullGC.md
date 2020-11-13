

## 一、gc的定义

GC，即就是Java垃圾回收机制。目前主流的JVM（HotSpot）采用的是分代收集算法。与C++不同的是，Java采用的是类似于树形结构的可达性分析法来判断对象是否还存在引用。即：从gcroot开始，把所有可以搜索得到的对象标记为存活对象。

## 二、gc的基础知识准备

要了解GC的触发条件，就要先对 JVM的内存结构有一定的了解。我们通常所说的GC主要是针对运行的数据区的。作为程序员要关注的区域主要有5块，分别是方法区(Method Area)，Java栈(Java stack)，本地方法栈(Native Method Stack)，堆(Heap)，程序计数器(Program Counter Register)。实际jvm在管理内存的时候，比这个分的更细致，只不过做应用程序开发，我们只需要关注这5块就可以了。

![img](https://pics1.baidu.com/feed/9213b07eca806538423c454deb176b40af3482ed.jpeg?token=f3ede83915f27a5cc3e9456255e0842d&s=1211662604797D8A5859807A020010F3)

堆(Heap)，是Jvm管理的内存中最大的一块。程序的主要数据也都是存放在堆内存中的，这一块区域被所有的线程所共享，通常出现线程安全问题的一般都是这个区域的数据出现的问题。

方法区(Method Area),与Heap一样，也是各个线程共享的内存域，这块区域主要是用来存储类加载器加载的类信息，常量，静态变量，通俗的讲就是编译后的class文件信息。

Jvm栈，与程序计数器一样，它是每个线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

本地方法栈（Native Method Stacks）,本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的Native方法服务。

程序计数器，个人感觉的他就是为多线程准备的，程序计数器是每个线程独有的，所以是线程安全的。它主要用于记录每个线程的执行情况。

通常我们所说的gc主要是针对java heap这块区域的。下面来了解一下heap区。

![img](https://pics4.baidu.com/feed/060828381f30e92418589d6531c2a4021c95f731.jpeg?token=ed923d2ad501743fe7d47e65da2b176b&s=E987B71E16AA45094C70B66C03009078)

从图中我们可以看出jvm heap区域是分代的，分为年轻代，老年代和持久代。 JVM的堆区对象分配的一般规则：

\1. 对象优先在Eden区分配

\2. 大对象直接进入老年代（-XX:PretenureSizeThreshold=3145728 这个参数来定义多大的对象直接进入老年代）

\3. 长期存活的对象将进入老年代(在JDK8中测试，-XX:MaxTenuringThreshold=1的阀值设定根本没用)

\4. 动态对象年龄判定（虚拟机并不会永远地要求对象的年龄都必须达到MaxTenuringThreshold才能晋升老年代，如果Survivor空间中相同年龄的所有对象的大小总和大于Survivor的一半，年龄大于或等于该年龄的对象就可以直接进入老年代）

\5. 空间分配担保

\6. 只要老年代的连续空间大于（新生代所有对象的总大小或者历次晋升的平均大小）就会进行minor GC，否则会进行full GC

## 三、gc的触发条件

充分了解了jvm的内存结构之后，下面我们就来说说什么情况下会触发gc。触发full gc的情况主要有这几种：

（1）System.gc()方法的调用。此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定，但很多情况下它会触发 Full GC,从而增加Full GC的频率，也即增加了间歇性停顿的次数。强烈影响系建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过-XX:+ DisableExplicitGC来禁止RMI（Java远程方法调用）调用System.gc。

（2）旧生代空间不足。旧生代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出错误：java.lang.OutOfMemoryError: Java heap space 。为避免以上两种状况引起的FullGC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

（3）Permanet Generation空间满了。Permanet Generation中存放的为一些class的信息等，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出错误信息：java.lang.OutOfMemoryError: PermGen space 。为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存。如果发现统计数据说之前Minor GC的平均晋升大小比目前old gen剩余的空间大，则不会触发Minor GC而是转为触发full GC。

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

四、gc回收的内容

知道了gc触发的条件之后，我们就能知道gc主要回收什么了？gc的主要作用是回收堆中的对象。通过可达性分析一个对象的引用是否存在，如果不存在，就可以被回收了。

## 四、gc的具体过程

那么，gc的是如何实现的，这个主要看是用的哪一种回收算法以及用的什么垃圾回收集了。回收算法主要有：

标记-清除复制算法标记-整理(Mark-Compat)算法分代收集(Generational Collection)算法这里针对不同的代，可以使用一些相对合适的算法。

新生代中，每次垃圾收集时都有大批对象死去，只有少量存活，就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集；

老年代中，其存活率较高、没有额外空间对它进行分配担保，就应该使用“标记-整理”或“标记-清理”算法进行回收。

常用的垃圾回收器：

Serial收集器ParNew收集器Parallel Scavenge收集器CMS(Concurrent Mark Sweep)收集器G1(Garbage First)收集器(从JDK1.7 Update 14之后的HotSpot虚拟机正式提供了商用的G1收集器)关于垃圾回收器的使用，这里也有一个组合建议共大家参考：

![img](https://pics6.baidu.com/feed/b90e7bec54e736d19298c506e59a85c6d4626900.jpeg?token=68cf5995b0bcf2a7465a18ad7a2c471b&s=EEE1F85A11AEC54F4A504E52030040F5)