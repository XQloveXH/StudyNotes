## 根节点枚举 

### Stop The World 

迄今为止， **所有收集器在根节点枚举这一步骤时都是必须暂停用户线程的**， 现在可达性分析算法耗时
最长的查找引用链的过程已经可以做到与用户线程一起并发 。

根节点枚举始终还是**必须在一个能保障一致性的快照中才得以进行**——这里“一致性”的意思是整个枚举期间执行子系统看起来就像被冻结在某个时间点上， 不会出现分析过程中， 根节点集合的对象引用关系还在不断变化
的情况， 

### OopMap 

虚拟机应当是有办法直接得到哪些地方存放着对象引用的。 

在HotSpot的解决方案里， 是使用一组称为OopMap的数据结构来达到这个目的。 一旦类加载动作完成的时候，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来， 在即时编译（见第11章） 过程中， 也会在特定的位置记录下栈里和寄存器里哪些位置是引用。 这样收集器在扫描时就可以直接得知这些信息了， 并不需要真正一个不漏地从方法区等GC Roots开始查找。 

## 安全点

实际上HotSpot也的确没有为每条指令都生成OopMap， 前面已经提到， 只是在“特定的位置”记录
了这些信息， 这些位置被称为安全点（Safepoint） 。  

### 安全点的位置的选择

安全点位置的选取基本上是以“**是否具有让程序长时间执行的特征”**为标准进行选定的， 因为每条指令执行的时间都非常短暂， 程序不太可能因为指令流长度太长这样的原因而长时间执行， “长时间执行”的最明显特征就是指令序列的复用， 例如方法调用、 循环跳转、 异常跳转等都属于指令序列复用， 所以只有具有这些功能的指令才会产生安全点。 

### 到达安全点的两种方案

#### 抢先式中断

抢先式中断不需要线程的执行代码主动去配合， 在垃圾收集发生时， 系统首先把所有用户线程全部中断， 如果发现有用户线程中断的地方不在安全点上， 就恢复这条线程执行， 让它一会再重新中断， 直到跑到安全点上。 现在几乎没有虚拟机实现采用抢先式中断来暂停线程响应GC事件。 

#### 主动式中断

不直接对线程操作， 仅仅简单地设置一个标志位， 各个线程执行过程时会不停地主动去轮询这个标志， 一旦发现中断标志为真时就自己在最近的安全点上主动中断挂起。 轮询标志的地方和安全点是重合的， 另外还要加上所有创建对象和其他需要在Java堆上分配内存的地方， 这是为了检查是否即将要发生垃圾收集， 避免没有足够内存分配新对象。 

##### 轮询操作的实现

由于轮询操作在代码中会频繁出现， 这要求它必须足够高效。 HotSpot使用内存保护陷阱的方式，
把轮询操作精简至只有一条汇编指令的程度。 下面代码清单3-4中的test指令就是HotSpot生成的轮询指
令， 当需要暂停用户线程时， 虚拟机把0x160100的内存页设置为不可读， 那线程执行到test指令时就会
产生一个自陷异常信号， 然后在预先注册的异常处理器中挂起线程实现等待， 这样仅通过一条汇编指
令便完成安全点轮询和触发线程中断了。 

## 安全区域 

安全区域是指能够确保在某一段代码片段之中， 引用关系不会发生变化， 因此， 在这个区域中任
意地方开始垃圾收集都是安全的。 我们也可以把安全区域看作被扩展拉伸了的安全点。 

当用户线程执行到安全区域里面的代码时， 首先会标识自己已经进入了安全区域， 那样当这段时
间里虚拟机要发起垃圾收集时就不必去管这些已声明自己在安全区域内的线程了。 当线程要离开安全
区域时， 它要检查虚拟机是否已经完成了根节点枚举（或者垃圾收集过程中其他需要暂停用户线程的
阶段） ， 如果完成了， 那线程就当作没事发生过， 继续执行； 否则它就必须一直等待， 直到收到可以
离开安全区域的信号为止。 

## 记忆集与卡表 

**记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。** 

### 实现方法

- 字长精度： 每个记录精确到一个机器字长（就是处理器的寻址位数， 如常见的32位或64位， 这个
  精度决定了机器访问物理内存地址的指针长度） ， 该字包含跨代指针。
- 对象精度： 每个记录精确到一个对象， 该对象里有字段含有跨代指针。
- 卡精度： 每个记录精确到一块内存区域， 该区域内有对象含有跨代指针 

### 卡集

第三种“卡精度”所指的是用一种称为“卡表”（Card Table） 的方式去实现记忆集[1]， 这也是目前最常用的一种记忆集实现形式，  

字节数组CARD_TABLE的每一个元素都对应着其标识的内存区域中一块特定大小的内存块， 这个
内存块被称作“卡页”（Card Page） 。 一般来说， 卡页大小都是以2的N次幂的字节数， 通过上面代码可
以看出HotSpot中使用的卡页是2的9次幂， 即512字节（地址右移9位， 相当于用地址除以512） 。 那如
果卡表标识内存区域的起始地址是0x0000的话， 数组CARD_TABLE的第0、 1、 2号元素， 分别对应了
地址范围为0x0000～0x01FF、 0x0200～0x03FF、 0x0400～0x05FF的卡页内存块[4]， 如图3-5所示 

![1603268297714](E:\soft\Typora\img\1603268297714.png)

一个卡页的内存中通常包含不止一个对象， 只要卡页内有一个（或更多） 对象的字段存在着跨代
指针， 那就将对应卡表的数组元素的值标识为1， 称为这个元素变脏（Dirty） ， 没有则标识为0。 在垃
圾收集发生时， 只要筛选出卡表中变脏的元素， 就能轻易得出哪些卡页内存块中包含跨代指针， 把它
们加入GC Roots中一并扫描 

**卡表的维护和更新是通过写屏障来实现的**

## 并发的可达性分析 

想解决或者降低用户线程的停顿， 就要先搞清楚为什么必须在一个能保障一致性的快照上才能进
行对象图的遍历？ 为了能解释清楚这个问题， 我们引入三色标记（Tri-color Marking） [1]作为工具来辅
助推导， 把遍历对象图过程中遇到的对象， 按照“是否访问过”这个条件标记成以下三种颜色：

- **白色**： 表示对象尚未被垃圾收集器访问过。 显然在可达性分析刚刚开始的阶段， 所有的对象都是
  白色的， 若在分析结束的阶段， 仍然是白色的对象， 即代表不可达。
- **黑色**： 表示对象已经被垃圾收集器访问过， 且这个对象的所有引用都已经扫描过。 黑色的对象代
  表已经扫描过， 它是安全存活的， 如果有其他对象引用指向了黑色对象， 无须重新扫描一遍。 黑色对
  象不可能直接（不经过灰色对象） 指向某个白色对象。
- **灰色**： 表示对象已经被垃圾收集器访问过， 但这个对象上至少存在一个引用还没有被扫描过。 

![1603263907118](E:\soft\Typora\img\1603263907118.png)



### 对象消失

Wilson于1994年在理论上证明了， 当且仅当以下两个条件同时满足时， 会产生“对象消失”的问
题， 即原本应该是黑色的对象被误标为白色：

- 赋值器插入了一条或多条从黑色对象到白色对象的新引用；
- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。 

### 避免对象消失

#### 增量更新 

增量更新要破坏的是第一个条件， 当黑色对象插入新的指向白色对象的引用关系时， 就将这个新
插入的引用记录下来， 等并发扫描结束之后， 再将这些记录过的引用关系中的黑色对象为根， 重新扫
描一次。 **这可以简化理解为， 黑色对象一旦新插入了指向白色对象的引用之后， 它就变回灰色对象了。** 

#### 原始快照 

原始快照要破坏的是第二个条件， 当灰色对象要删除指向白色对象的引用关系时， 就将这个要删
除的引用记录下来， 在并发扫描结束之后， 再将这些记录过的引用关系中的灰色对象为根， 重新扫描
一次。 **这也可以简化理解为， 无论引用关系删除与否， 都会按照刚刚开始扫描那一刻的对象图快照来进行搜索** 04-