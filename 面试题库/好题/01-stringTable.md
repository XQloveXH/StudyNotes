### 一、存储结构

jdk8及以前String底层使用char[]，一个char是两个字节，jdk9开始改用byte[]加上编码标记节约空间。

![img](https://img-blog.csdnimg.cn/20200712110427380.png)

jdk9官网提供的String修改说明：http://openjdk.java.net/jeps/254

修改动机：

![img](https://img-blog.csdnimg.cn/20200712110701368.png)

![img](https://img-blog.csdnimg.cn/20200712110741184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

### 二、不可变性

1、当对字符串重新赋值时，需要重写指定内存区域赋值

2、当对现有字符串进行连接操作时，也需要重新指定内存区域。

3、调用String的replace方法时修改指定字符串或者字符时，也需要重新指定内存区域赋值。

4、通过字面量方式（区别于new）给一个字符串赋值，此时的字符串值声明在字符串常量池中。

### 三、字符串常量池是不会存储相同内容的字符串的。

1、String的String Pool是一个固定大小的HashTable，默认值大小长度是1009，如果放进String Pool的String非常多，就会造成Hash冲突严重，从而导致链表会很长，而链表长了后直接会造成的影响就是调用String.intern时性能会大幅下降。

2、-XX:StringTableSize可设置StringTable的长度。

3、在jdk6中StringTable是固定的，就是1009的长度，如果常量池中的字符串过多就会导致效率下降很快。

4、在jdk7中，StringTable的长度默认是60013，jdk8开始StringTable的1009是可设置的最小值，默认值是60013。

### 四、String的内存分配

jdk6及以前，字符串常量池存放在永久代。jdk7开始字符串常量池的位置调整到java堆中。

jdk8去除了永久代，开始使用元空间，字符串常量池存放于堆中。

### 五、字符串优化

1、常量和常量的拼接结果在常量池中，直接使用常量池的字面量，原理是编译器优化。

2、字符串拼接中，出现常量和字符串变量的拼接，那个字符串变量相当于在堆空间中new StringBuilder()，具体的内容为拼接的结果。

3、str.inter()，判断字符串常量池中是否存在str,如果存在，则返回常量池中str的地址，如果不存在则在常量池中加载一份str，同时返回str的地址。

例：观察s4=s1+s2

![img](https://img-blog.csdnimg.cn/20200719135902256.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

s4的编译执行过程

（1）生成StringBuilder()对象，在jdk5.0之前使用StringBuffer()对象

（2）调用StringBuilder对象的append方法加入常量池中"a"

（3）调用StringBuilder对象的append方法加入常量池中"b"

（4）调用StringBuilder对象的toString方法

注意：如果变量s1和s2前加入了final修饰符，此时编译器会在编译阶段做优化，不会生成StringBuilder对象，直接形成从常量池中找"ab"返回给s4，当然s3==s4也是true。

### 六、intern

str.inter()，判断字符串常量池中是否存在str,如果存在，则返回常量池中str的地址，如果不存在则在常量池中加载一份str，同时返回str的地址。

**1、jdk1.6，将字符串str尝试放入常量池**

（1）如果常量池中有，则并不会放入，返回已有常量池中的对象的地址

（2）如果没有，会把此对象复制一份，放入常量池，并返回常量池的对象地址。

![img](https://img-blog.csdnimg.cn/20200719152918684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

**2、jdk1.7及以后版本，将字符串str尝试放入常量池**

（1）如果常量池中有，则不会放入，返回已有的常量池中的对象地址

（2）如果没有，则把对象的引用地址复制一份，放入常量池中，并返回常量池中引用的地址。

![img](https://img-blog.csdnimg.cn/2020071915305927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20200719153136325.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

### 七、扩展知识

**1、new String("ab");会产生几个对象**

一共有2个对象，在第一个堆中new String生成，第二个是堆中常量池中的ab对象

**2、new String("a")+new String("b");会产生多少个对象**

一共会产生6个对象

（1）有拼接操作，会生成StringBuilder对象

（2）new String("a")对象

（3）常量池中的"a"对象

（4）new String("b")对象

（5）常量池中"b"对象

（6）拼接完后会调用StringBuilder对象的toString()方法，toString方法内会new String()对象

```
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

**3、intern在jdk版本中的区别**

```
public static void main(String[] args) {
   String s=new String("1");
   s.intern();//调用intern()前，字符串常量池中已经存在"1"了。
   String s2="1";
   System.out.println(s==s2);//jdk6：false，jdk7/8：false
   String s3=new String("1")+new String("1");
   s3.intern();
   String s4="11";
   System.out.println(s3==s4);//jdk6：false，jdk7/8：true
}
```

jdk6：

![img](https://img-blog.csdnimg.cn/20200719145650716.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

jdk7/8：

![img](https://img-blog.csdnimg.cn/20200719150059605.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

注意：s4和s3.intern()互换

```
String s3=new String("1")+new String("1");
String s4="11";
s3.intern();
System.out.println(s3==s4);//jdk6：false，jdk7/8：false
```

### 八、G1对String去重（默认是不开启的）

java工程师测试了大部分java应用出现如下现象

1、堆中存活的String对象占25%

2、堆中存活的重复的String对象有13.5%

3、String对象的平均长度是45

堆上存在重复的String对象势必造成内存的浪费，G1垃圾回收器会在自动持续的去重String对象，避免堆内存浪费。

G1去重的实现步骤：

![img](https://img-blog.csdnimg.cn/20200719160717543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p1b2Rhb3lvbmc=,size_16,color_FFFFFF,t_70)

 