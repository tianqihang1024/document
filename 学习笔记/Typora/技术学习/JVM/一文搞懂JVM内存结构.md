# 一文搞懂JVM内存结构

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[_Rt](https://me.csdn.net/rongtaoup) 2019-04-11 20:30:23 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 13353 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 101



分类专栏： [跟我学JVM系列](https://blog.csdn.net/rongtaoup/category_8840316.html)

版权

# 1. 前言

​		Java 虚拟机是中、高级开发人员必须修炼的知识，有着较高的学习门槛，很多人都不情愿去接触它。可能是觉得学习成本较高又或者是感觉没什么实用性，所以干脆懒得“搭理”它了。其实这种想法是错误的。举个最简单的例子，JVM 基本上是每家招聘公司都会问到的问题，它们会这么无聊问这些不切实际的问题吗？很显然不是。由 JVM 引发的故障问题，无论在我们开发过程中还是生产环境下都是非常常见的。比如 OutOfMemoryError(OOM) 内存溢出问题，你应该遇到过 Tomcat 容器中加载项目过多导致的 OOM 问题，导致 Web 项目无法启动。这就是JVM引发的故障问题。那到底JVM哪里发生内存溢出了呢？为什么会内存溢出呢？如何监控？最重要的就是如何解决问题呢？**能解决问题的技术才是最实用最好的技术**。然而你对JVM的内存结构都不清楚，就妄想解决JVM引发的故障问题，是不切实际的。只有基础打好了，对于JVM故障问题才能“披荆斩棘”。本文通过代码与图示详细讲解了JVM内存区域，相信阅读本文之后，你将对JVM内存的堆、栈、方法区等有一个清晰的认知。2. 运行时数据区

Java 虚拟机在执行 Java 程序的过程中会把它管理的内存划分为若干个不同的数据区域。每个区域都有各自的作用。

分析 JVM 内存结构，主要就是分析 JVM 运行时数据存储区域。JVM 的运行时数据区主要包括：**堆、栈、方法区、程序计数器**等。而 JVM 的优化问题主要在**线程共享的数据区**中：**堆、方法区**。

------

## 2.1 程序计数器

**程序计数器（Program Counter Register）**是一块较小的内存空间，可以看作是**当前线程**所执行字节码的**行号指示器**，指向下一个将要执行的指令代码，由执行引擎来读取下一条指令。更确切的说，**一个线程的执行，是通过字节码解释器改变当前线程的计数器的值，来获取下一条需要执行的字节码指令，从而确保线程的正确执行**。

为了确保线程切换后（**上下文切换**）能恢复到正确的执行位置，**每个线程都有一个独立的程序计数器**，各个线程的计数器互不影响，独立存储。也就是说程序计数器是**线程私有的内存**。

如果线程执行 Java 方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果执行的是 Native 方法，计数器值为Undefined。

**程序计数器不会发生内存溢出（OutOfMemoryError即OOM）问题。**

------

## **2.2 栈**

JVM 中的栈包括 **Java 虚拟机栈**和本地方法栈，两者的**区别**就是，Java 虚拟机栈为 JVM 执行 Java 方法服务，本地方法栈则为 JVM 使用到的 Native 方法服务。两者作用是极其相似的，本文主要介绍 Java 虚拟机栈，以下简称栈。

> **Native 方法是什么？**

JDK 中有很多方法是使用 Native 修饰的。Native 方法**不是以 Java 语言实现**的，而是以本地语言实现的（比如 C 或 C++）。个人理解Native 方法是与操作系统直接交互的。比如通知垃圾收集器进行垃圾回收的代码 System.gc()，就是使用 native 修饰的。

```java
public final class System {



    public static void gc() {



        Runtime.getRuntime().gc();



    }



}



 



public class Runtime {



    //使用native修饰



     public native void gc();
```

> **什么是栈？**

**定义：\**限定仅在表头进行插入和删除操作的线性表\****。即**压栈（入栈）和弹栈（出栈）**都是对**栈顶元素**进行操作的。所以栈是**后进先出**的。

栈是**线程私有**的，他的生命周期与线程相同。每个线程都会分配一个栈的空间，即**每个线程拥有独立的栈空间**。

![img](https://img-blog.csdnimg.cn/20190409163129199.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

> **栈中存储的是什么？**

**栈帧**是栈的元素。**每个方法在执行时都会创建一个栈帧**。栈帧中存储了**局部变量表、操作数栈、动态连接和方法出口**等信息。每个方法从调用到运行结束的过程，就对应着一个栈帧在栈中压栈到出栈的过程。

![img](https://img-blog.csdnimg.cn/20190409182042686.png)

### 2.2.1 局部变量表

栈帧中，由一个**局部变量表存储数据**。局部变量表中存储了**基本数据类型**（boolean、byte、char、short、int、float、long、double）的**局部变量（包括参数）、和对象的引用（String、数组、对象等），但是不存储对象的内容**。局部变量表所需的内存空间在**编译期间完成分配**，在方法运行期间不会改变局部变量表的大小。

局部变量的容量以**变量槽（Variable Slot）**为最小单位，每个变量槽最大存储32位的数据类型。对于64位的数据类型（long、double），JVM 会为其分配两个连续的变量槽来存储。以下简称 Slot 。

JVM 通过索引定位的方式使用局部变量表，索引的范围从0开始至局部变量表中最大的 Slot 数量。**普通方法与 static 方法**在第 0 个槽位的存储有所不同。非 static 方法的第 0 个槽位存储方法所属对象实例的引用。

![img](https://img-blog.csdnimg.cn/20190410183454222.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

> **Slot 复用？**

为了尽可能的节省栈帧空间，局部变量表中的 **Slot 是可以复用**的。方法中定义的局部变量，其作用域不一定会覆盖整个方法。当方法运行时，如果已经**超出了某个变量的作用域**，即变量失效了，那这个变量对应的 Slot 就可以交给其他变量使用，也就是所谓的 **Slot 复用**。通过一个例子来理解变量“失效”。

```java
public void test(boolean flag)



{



    if(flag)



    {



        int a = 66;



    }



    



    int b = 55;



}
```

当虚拟机运行 test 方法，就会创建一个栈帧，并压入到当前线程的栈中。当运行到 int a = 66时，在当前栈帧的局部变量中创建一个 Slot 存储变量 a，当运行到 int b = 55时，此时已经超出变量 a 的作用域了（变量 a 的作用域在{}所包含的代码块中），此时 a 就失效了，变量a 占用的 Slot 就可以交给b来使用，这就是 Slot 复用。

凡事有利弊。Slot 复用虽然节省了栈帧空间，但是会伴随一些额外的**副作用**。比如，Slot 的复用会直接影响到系统的垃圾收集行为。

```java
public class TestDemo {



 



    public static void main(String[] args){



        



        byte[] placeholder = new byte[64 * 1024 * 1024];



        



        System.gc();



    }



}
```

上段代码很简单，先向内存中填充了 64M 的数据，然后通知虚拟机进行垃圾回收。为了更清晰的查看垃圾回收的过程，我们再虚拟机的运行参数中加上“**-verbose:gc**”,这个参数的作用就是**打印 GC 信息**。

![img](https://img-blog.csdnimg.cn/20190410185810286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

打印的GC信息如下：

![img](https://img-blog.csdnimg.cn/20190410185953669.png)

可以看到虚拟机没有回收这 64M 内存。为什么没有被回收？其实很好理解，当执行 System.gc() 方法时，变量 placeholder 还在作用域范围之内，虚拟机是不会回收的，它还是“有效”的。

我们对上面的代码稍作修改，使其作用域“失效”。

 

```java
public class TestDemo {



 



    public static void main(String[] args){



        {



            byte[] placeholder = new byte[64 * 1024 * 1024];



        }



        System.gc();



    }



}
```

当运行到 System.gc() 方法时，变量 placeholder 的作用域已经失效了。它已经“无用”了，虚拟机会回收它所占用的内存了吧？

运行结果：

![img](https://img-blog.csdnimg.cn/20190410190827875.png)

发现虚拟机还是没有回收 placeholder 变量占用的 64M 内存。为什么所想非所见呢？在解释之前，我们再对代码稍作修改。在System.gc()方法执行之前，加入一个局部变量。

 

```java
public class TestDemo {



 



    public static void main(String[] args){



        {



            byte[] placeholder = new byte[64 * 1024 * 1024];



        }



        int a = 0;



        System.gc();



    }



}
```

在 System.gc() 方法之前，加入 int a = 0，再执行方法，查看垃圾回收情况。

![img](https://img-blog.csdnimg.cn/20190410191854837.png)

发现 placeholder 变量占用的64M内存空间被回收了，如果不理解局部变量表的Slot复用，很难理解这种现象的。

而 **placeholder 变量能否被回收的关键**就在于：局部变量表中的 Slot 是否还存有关于 placeholder 对象的引用。

第一次修改中，限定了 placeholder 的作用域，但**之后并没有任何对局部变量表的读写操作**，placeholder 变量在局部变量表中占用的Slot没有被其它变量所复用，所以作为 GC Roots 一部分的局部变量表仍然保持着对它的关联。所以 placeholder 变量没有被回收。

第二次修改后，运行到 int a = 0 时，已经超过了 placeholder 变量的作用域，此时 placeholder 在局部变量表中占用的Slot可以交给其他变量使用。**而变量a正好复用了 placeholder 占用的 Slot**，至此局部变量表中的 Slot 已经没有 placeholder 的引用了，虚拟机就回收了placeholder 占用的 64M 内存空间。

------

### 2.2.2 操作数栈

**操作数栈**是一个**后进先出**栈。操作数栈的**元素可以是任意的Java数据类型**。方法刚开始执行时，操作数栈是**空的**，在方法执行过程中，通过字节码指令对操作数栈进行**压栈和出栈**的操作。**通常进行****算数运算****的时候是通过操作数栈来进行的****，****又或者是在调用其他方法的时候通过操作数栈进行****参数传递****。操作数栈可以理解为栈帧中用于****计算****的临时数据存储区**。

通过一段代码来了解操作数栈。

```java
public class OperandStack{



 



    public static int add(int a, int b){



        int c = a + b;



        return c;



    }



 



    public static void main(String[] args){



        add(100, 98);



    }



}
```

使用 javap **反编译** OperandStack 后，根据虚拟机指令集，得出操作数栈的运行流程如下：

![img](https://img-blog.csdnimg.cn/20190409231224263.png)

add 方法刚开始执行时，操作数栈是空的。当执行 iload_0 时，把局部变量 0 压栈，即 100 入操作数栈。然后执行 iload_1，把局部变量1压栈，即 98 入操作数栈。接着执行 iadd，弹出两个变量（100 和 98 出操作数栈），对 100 和 98 进行求和，然后将结果 198 压栈。然后执行 istore_2，弹出结果（出栈）。

下面通过一张图，对比执行100+98操作，**局部变量表和操作数栈的变化情况**。

![img](https://img-blog.csdnimg.cn/20190409205401344.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

> **栈中可能出现哪些异常？**

**StackOverflowError：栈溢出错误**

如果一个线程在计算时所需要用到栈大小 > 配置允许最大的栈大小，那么Java虚拟机将抛出 StackOverflowError

**OutOfMemoryError：内存不足**

 栈进行动态扩展时如果无法申请到足够内存，会抛出 OutOfMemoryError 异常。

> **如何设置栈参数？**

使用 **-Xss** 设置栈大小，通常几百K就够用了。由于栈是**线程私有**的，**线程数越多，占用栈空间越大**。

**栈决定了函数调用的深度**。这也是慎用递归调用的原因。递归调用时，每次调用方法都会创建栈帧并压栈。当调用一定次数之后，所需栈的大小已经超过了虚拟机运行配置的最大栈参数，就会抛出 StackOverflowError 异常。

------

## 2.3 Java堆

堆是Java虚拟机所管理的内存中最大的一块存储区域。堆内存被所有**线程共享**。主要存放使用**new**关键字创建的对象。所有**对象实例以及数组**都要在堆上分配。垃圾收集器就是根据GC算法，收集堆上**对象所占用的内存空间（收集的是对象占用的空间而不是对象本身）**。

Java堆分为**年轻代**（Young Generation）和**老年代**（Old Generation）；**年轻代又分为伊甸园（Eden）和幸存区（Survivor区）；幸存区又分为From Survivor空间和 To Survivor空间。**

年轻代存储“新生对象”，我们新创建的对象存储在年轻代中。当年轻内存占满后，会触发**Minor GC**，清理年轻代内存空间。

老年代存储**长期存活的对象和大对象**。年轻代中存储的对象，经过多次GC后仍然存活的对象会移动到老年代中进行存储。老年代空间占满后，会触发**Full GC**。

***\*注：\**Full GC**是清理整个堆空间，包括年轻代和老年代。如果Full GC之后，堆中仍然无法存储对象，就会抛出**OutOfMemoryError**异常。

![img](https://img-blog.csdnimg.cn/2019041019553012.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

> Java堆设置常用参数

| 参数                            | 描述                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| -Xms                            | 堆内存初始大小                                               |
| -Xmx（MaxHeapSize）             | 堆内存最大允许大小，一般不要大于物理内存的80%                |
| -XX:NewSize（-Xns）             | 年轻代内存初始大小                                           |
| -XX:MaxNewSize（-Xmn）          | 年轻代内存最大允许大小，也可以缩写                           |
| -XX:NewRatio                    | 新生代和老年代的比值值为4 表示 新生代:老年代=1:4，即年轻代占堆的1/5 |
| -XX:SurvivorRatio=8             | 年轻代中Eden区与Survivor区的容量比例值，默认为8表示两个Survivor :eden=2:8，即一个Survivor占年轻代的1/10 |
| -XX:+HeapDumpOnOutOfMemoryError | 内存溢出时，导出堆信息到文件                                 |
| -XX:+HeapDumpPath               | 堆Dump路径-Xmx20m -Xms5m-XX:+HeapDumpOnOutOfMemoryError-XX:HeapDumpPath=d:/a.dump |
| -XX:OnOutOfMemoryError          | 当发生OOM内存溢出时，执行一个脚本-XX:OnOutOfMemoryError=D:/tools/jdk1.7_40/bin/printstack.bat %p%p表示线程的id pid |
| -XX:MaxTenuringThreshold=7      | 表示如果在幸存区移动多少次没有被垃圾回收，进入老年代         |

------

## **2.4 方法区（Method Area）**

方法区同 Java 堆一样是被所有**线程共享**的区间，用于存储**已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码**。更具体的说，静态变量+常量+类信息（版本、方法、字段等）+运行时常量池存在方法区中。**常量池是方法区的一部分**。

![img](https://img-blog.csdnimg.cn/20190410205726197.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

***\*注\****：JDK1.8 使用元空间 **MetaSpace** 替代方法区，元空间并不在 JVM中，而是使用本地内存。元空间两个参数：

1.  MetaSpaceSize：初始化元空间大小，控制发生GC阈值
2.  MaxMetaspaceSize ： 限制元空间大小上限，防止异常占用过多物理内存

 

常量池中存储编译器生成的各种**字面量和符号引用**。字面量就是Java中常量的意思。比如文本字符串，final修饰的常量等。方法引用则包括类和接口的全限定名，方法名和描述符，字段名和描述符等。

![img](https://img-blog.csdnimg.cn/20190410210752310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Jvbmd0YW91cA==,size_16,color_FFFFFF,t_70)

> **常量池有什么用 ？**

**优点：**常量池避免了频繁的创建和销毁对象而影响系统性能，其实现了对象的共享。

举个栗子： Integer 常量池（缓存池），和字符串常量池

> **Integer常量池：**

我们知道 **== 基本数据类型比较的是数值，而引用数据类型比较的是内存地址**。

```java
public void TestIntegerCache()



{



    public static void main(String[] args)



    {



        



        Integer i1 = new Integer(66);



        Integer i2 = new integer(66);



        Integer i3 = 66;



        Integer i4 = 66;



        Integer i5 = 150;



        Integer i6 = 150;



        System.out.println(i1 == i2);//false



        System.out.println(i3 == i4);//true



        System.out.println(i5 == i6);//false



    }



    



}
```

i1 和 i2 使用 new 关键字，每 new 一次都会在堆上创建一个对象，所以 i1 == i2 为 false。

i3 == i4 为什么是 true 呢？Integer i3 = 66 实际上有一步**装箱**的操作，即将 int 型的 66 装箱成 Integer，通过 Integer 的 valueOf 方法。

```java
public static Integer valueOf(int i) {



        if (i >= IntegerCache.low && i <= IntegerCache.high)



            return IntegerCache.cache[i + (-IntegerCache.low)];



        return new Integer(i);



    }
```

Integer 的 valueOf 方法很简单，它判断变量是否在 **IntegerCache** 的最小值（-128）和最大值（127）之间，如果在，则返回常量池中的内容，否则 new 一个 Integer 对象。

而 **IntegerCache** 是 **Integer的静态内部类**，作用就是将 [-128,127] 之间的数“缓存”在 IntegerCache 类的 cache 数组中，valueOf 方法就是调用常量池的 cache 数组，不过是将 i3、i4 **变量引用指向常量池中，没有真正的创建对象**。而new Integer(i)则是直接在堆中创建对象。

IntegerCache 类中，包含一个构造方法，三个静态变量：low最小值、high最大值、和Integer数组，还有一个静态代码块。静态代码块的作用就是在 IntegerCache 类加载的时候，对high最大值以及 Integer 数组初始化。也就是说当 IntegerCache 类加载的时候，最大最小值，和 Integer 数组就已经初始化好了。这个 Integer 数组其实就是包含了 -128到127之间的所有值。

> **IntegerCache 源码**

```java
private static class IntegerCache {



        static final int low = -128;//最小值



        static final int high;//最大值



        static final Integer cache[];//缓存数组



 



        //私有化构造方法，不让别人创建它。单例模式的思想



        private IntegerCache() {}



 



        //类加载的时候，执行静态代码块。作用是将-128到127之间的数缓冲在cache[]数组中



        static {



            // high value may be configured by property



            int h = 127;



            String integerCacheHighPropValue =



                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");



            if (integerCacheHighPropValue != null) {



                try {



                    int i = parseInt(integerCacheHighPropValue);



                    i = Math.max(i, 127);



                    // Maximum array size is Integer.MAX_VALUE



                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);



                } catch( NumberFormatException nfe) {



                    // If the property cannot be parsed into an int, ignore it.



                }



            }



            high = h;



 



            cache = new Integer[(high - low) + 1];//初始化cache数组，根据最大最小值确定



            int j = low;



            for(int k = 0; k < cache.length; k++)//遍历将数据放入cache数组中



                cache[k] = new Integer(j++);



 



            // range [-128, 127] must be interned (JLS7 5.1.7)



            assert IntegerCache.high >= 127;



        }



 



    }
```

而 i5 == i6 为 false，就是因为 **150 不在 Integer 常量池的最大最小值之间【-128,127】，从而 new 了一个对象，**所以为 false。

再看一段**拆箱**的代码。

```java
public static void main(String[] args){



       Integer i1 = new Integer(4);



       Integer i2 = new Integer(6);



       Integer i3 = new Integer(10);



       System.out.print(i3 == i1+i2);//true



    }
```

由于 i1 和 i2 是 Integer 对象，是不能使用+运算符的。首先 i1 和 i2 进行**自动拆箱**操作，拆箱成int后再进行数值加法运算。i3 也是拆箱后再与之比较数值是否相等的。所以 i3 == i1+i2 其实是比较的 int 型数值是否相等，所以为true。

------

> **String常量池：**

**String 是由 \**final\** 修饰的类，是不可以被继承的**。通常有两种方式来创建对象。

```java
//1、



String str = new String("abcd");



 



//2、



String str = "abcd";
```

第一种使用 new 创建的对象，存放在堆中。每次调用都会创建一个新的对象。

第二种先在栈上创建一个 String 类的对象引用变量 str，然后通过符号引用去字符串常量池中找有没有 “abcd”，如果没有，则将“abcd”存放到字符串常量池中，并将栈上的 str 变量引用指向常量池中的“abcd”。如果常量池中已经有“abcd”了，则不会再常量池中创建“abcd”，而是直接将 str 引用指向常量池中的“abcd”。

对于 String 类，equals 方法用于比较字符串内容是否相同； == 号用于比较内存地址是否相同，即是否指向同一个对象。通过代码验证上面理论。

```java
public static void main(String[] args){



       String str1 = "abcd";



       String str2 = "abcd";



       System.out.print(str1 == str2);//true



    }
```

首先在栈上存放变量引用 str1，然后通过符号引用去常量池中找是否有 abcd，没有，则将 abcd 存储在常量池中，然后将 str1 指向常量池的 abcd。当创建 str2 对象，去常量池中发现已经有 abcd 了，就将 str2 引用直接指向 abcd 。所以str1 == str2，指向**同一个内存地址**。

```java
public static void main(String[] args){



       String str1 = new String("abcd");



       String str2 = new String("abcd");



       System.out.print(str1 == str2);//false



    }
```

str1 和 str2 使用 new 创建对象，分别在堆上创建了不同的对象。两个引用指向堆中两个不同的对象，所以为 false。

> **关于字符串 + 号连接问题：**

对于字符串常量的 + 号连接，在程序编译期，JVM就会将其优化为 + 号连接后的值。所以在**编译期其字符串常量的值就确定了**。

```java
String a = "a1";   



String b = "a" + 1;   



System.out.println((a == b)); //result = true  



 



String a = "atrue";   



String b = "a" + "true";   



System.out.println((a == b)); //result = true 



 



String a = "a3.4";   



String b = "a" + 3.4;   



System.out.println((a == b)); //result = true 
```

> **关于字符串引用 + 号连接问题：**

对于字符串***\*引用\****的 + 号连接问题，由于字符串引用在**编译期是无法确定**下来的，在程序的**运行期动态分配并创建新的地址存储对象**。

```java
public static void main(String[] args){



       String str1 = "a";



	   String str2 = "ab";



	   String str3 = str1 + "b";



	   System.out.print(str2 == str3);//false



    }
```

对于上边代码，str3 等于 str1 引用 + 字符串常量“b”，在编译期无法确定，在运行期动态的分配并将连接后的新地址赋给 str3，所以 str2 和 str3 引用的内存地址不同，所以 str2 == str3 结果为 false

通过 jad 反编译工具，分析上述代码到底做了什么。编译指令如下：

![img](https://img-blog.csdnimg.cn/20190411121054923.png)

经过 jad 反编译工具反编译代码后，代码如下

```java
public class TestDemo



{

    public TestDemo()

    {

    }

    public static void main(String args[])

    {

        String s = "a";

        String s1 = "ab";

        String s2 = (new StringBuilder()).append(s).append("b").toString();

        System.out.print(s1 = s2);
    }
}
```

发现 **new 了一个 StringBuilder 对象，然后使用 append 方法优化了 + 操作符**。new 在堆上创建对象，而 String s1=“ab”则是在常量池中创建对象，两个应用所指向的内存地址是不同的，所以 s1 == s2 结果为 false。

**注**：我们已经知道了字符串引用的 + 号连接问题，其实是在运行期间创建一个 StringBuilder 对象，使用其 append 方法将字符串连接起来。这个也是我们开发中需要注意的一个问题，就是**尽量不要在 for 循环中使用 + 号来操作字符串**。看下面一段代码：

```java
public static void main(String[] args){

        String s = null;

        for(int i = 0; i < 100; i++){

            s = s + "a";
        }
    }
```

在 for 循环中使用 + 连接字符串，**每循环一次，就会新建 StringBuilder 对象，append 后就“抛弃”了它**。如果我们在循环外创建StringBuilder 对象，然后在循环中使用 append 方法追加字符串，就可以**节省 n-1 次创建和销毁对象的时间**。所以在循环中连接字符串，一般使用 StringBuilder 或者 StringBuffer，而不是使用 + 号操作。

```java
public static void main(String[] args){

        StringBuilder s = new StringBuilder();

        for(int i = 0; i < 100; i++){

            s.append("a");
        }
    }
```

> **使用final修饰的字符串**

```java
public static void main(String[] args){

        final String str1 = "a";
    
        String str2 = "ab";
    
        String str3 = str1 + "b";
    
        System.out.print(str2 == str3);//true
    }
```

final 修饰的变量是一个常量，编译期就能确定其值。所以 str1 + "b"就等同于 "a" + "b"，所以结果是 true。

> **String对象的intern方法。**

```java
public static void main(String[] args){
    
        String s = "ab";
        String s1 = "a";
        String s2 = "b";
        String s3 = s1 + s2;

        System.out.println(s3 == s);//false
        System.out.println(s3.intern() == s);//true
    }
```

通过前面学习我们知道，s1+s2 实际上在堆上 new 了一个 StringBuilder 对象，而 s 在常量池中创建对象 “ab”，所以 s3 == s 为 false。但是 s3 调用 **intern 方法，返回的是s3的内容（ab）在常量池中的地址值**。所以 s3.intern() == s 结果为 true。