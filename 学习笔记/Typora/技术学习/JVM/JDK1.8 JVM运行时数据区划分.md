# JDK1.8 JVM运行时数据区域划分

https://blog.csdn.net/bruce128/article/details/79357870



## 一、JDK1.8 JVM运行时数据区域概览

![这里写图片描述](https://img-blog.csdn.net/20180812235058303?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JydWNlMTI4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这里介绍的是JDK1.8 JVM运行时内存数据区域划分。1.8同1.7比，最大的差别就是：**元数据区取代了永久代**。元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元数据空间并不在虚拟机中，而是使用本地内存**。

## 二、各区域介绍

### 1. 程序计数器

每个线程一块，指向当前线程正在执行的字节码代码的行号。如果当前线程执行的是native方法，则其值为null。

### 2. Java虚拟机栈

![stack](https://img-blog.csdn.net/20180313100115189?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJ1Y2UxMjg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
线程私有，每个线程对应一个Java虚拟机栈，其生命周期与线程同进同退。每个Java方法在被调用的时候都会创建一个栈帧，并入栈。一旦完成调用，则出栈。所有的的栈帧都出栈后，线程也就完成了使命。

### 3. 本地方法栈

功能与Java虚拟机栈十分相同。区别在于，本地方法栈为虚拟机使用到的native方法服务。

### 4. 堆

![heap](https://img-blog.csdn.net/2018031000051841?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJ1Y2UxMjg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
堆是JVM内存占用最大，管理最复杂的一个区域。其唯一的用途就是存放对象实例：几乎所有的对象实例及数组都在对上进行分配。1.7后，字符串常量池从永久代中剥离出来，存放在堆中。堆有自己进一步的内存分块划分，按照GC分代收集角度的划分请参见上图。

#### 4.1 堆空间内存分配（默认情况下）

- 老年代 ： 三分之二的堆空间
- 年轻代 ： 三分之一的堆空间
	- eden区： 8/10 的年轻代空间
	- survivor0 : 1/10 的年轻代空间
	- survivor1 : 1/10 的年轻代空间

命令行上执行如下命令，查看所有默认的jvm参数

```shell
java -XX:+PrintFlagsFinal -version
1
```

##### 输出

输出有大几百行，这里只取其中的两个有关联的参数

```java
[Global flags]
    uintx InitialSurvivorRatio                      = 8                                   {product}
    uintx NewRatio                                  = 2                                   {product}
    ... ...
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
1234567
```

##### 参数解释

| 参数                     | 作用                              |
| ------------------------ | --------------------------------- |
| -XX:InitialSurvivorRatio | 新生代Eden/Survivor空间的初始比例 |
| -XX:Newratio             | Old区 和 Yong区 的内存比例        |

##### 一道推算题

> 默认参数下，如果仅给出eden区40M，求堆空间总大小

根据比例可以推算出，两个survivor区各5M，年轻代50M。老年代是年轻代的两倍，即100M。那么堆总大小就是150M。

#### 4.2 字符串常量池

JDK1.7 就开始“去永久代”的工作了。 1.7把字符串常量池从永久代中剥离出来，存放在堆空间中。

##### a. jvm参数配置

```java
-XX:MaxPermSize=10m
-XX:PermSize=10m
-Xms100m
-Xmx100m
-XX:-UseGCOverheadLimit
12345
```

##### b. 测试代码

```java
public class StringOomMock {
	
	public static void main(String[] args) {
		try {
			List<String> list = new ArrayList<String>();
			for (int i = 0; ; i++) {
				System.out.println(i);
				list.add(String.valueOf("String" + i++).intern());
			}
		} catch (java.lang.Exception e) {
			e.printStackTrace();
		}
	}
}
1234567891011121314
```

##### c. jdk1.6 下的运行结果

jdk1.6 环境下是永久代OOM

```java
153658
153660
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
	at java.lang.String.intern(Native Method)
	at com.jd.im.StringOomMock.main(StringOomMock.java:17)
12345
```

##### d. jdk1.7 下的运行结果

jdk1.7 下是堆OOM，并且伴随着频繁的FullGC， CPU一直高位运行

```java
2252792
2252794
2252796
2252798
*** java.lang.instrument ASSERTION FAILED ***: "!errorOutstanding" with message can't create name string at ../../../src/share/instrument/JPLISAgent.c line: 807
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.nio.CharBuffer.wrap(CharBuffer.java:369)
	at sun.nio.cs.StreamEncoder.implWrite(StreamEncoder.java:265)
	at sun.nio.cs.StreamEncoder.write(StreamEncoder.java:125)
	at java.io.OutputStreamWriter.write(OutputStreamWriter.java:207)
	at java.io.BufferedWriter.flushBuffer(BufferedWriter.java:129)
	at java.io.PrintStream.write(PrintStream.java:526)
	at java.io.PrintStream.print(PrintStream.java:597)
	at java.io.PrintStream.println(PrintStream.java:736)
	at com.jd.im.StringOomMock.main(StringOomMock.java:16)
123456789101112131415
```

##### e. jdk1.8 下的运行结果

jdk1.8的运行结果同1.7的一样，都是堆空间OOM。

```java
2236898
2236900
2236902
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.lang.Integer.toString(Integer.java:403)
	at java.lang.String.valueOf(String.java:3099)
	at java.io.PrintStream.print(PrintStream.java:597)
	at java.io.PrintStream.println(PrintStream.java:736)
	at com.jd.im.StringOomMock.main(StringOomMock.java:16)

12345678910
```

### 5. 元数据区

元数据区取代了1.7版本及以前的永久代。元数据区和永久代本质上都是方法区的实现。方法区存放虚拟机加载的类信息，静态变量，常量等数据。
元数据区OOM测试：

#### a. jvm参数配置

```java
-XX:MetaspaceSize=8m 
-XX:MaxMetaspaceSize=50m
12
```

#### b. 测试代码

借助cglib框架生成新类。

```java
public class MetaSpaceOomMock {
	
	public static void main(String[] args) {
		ClassLoadingMXBean loadingBean = ManagementFactory.getClassLoadingMXBean();
		while (true) {
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(MetaSpaceOomMock.class);
			enhancer.setCallbackTypes(new Class[]{Dispatcher.class, MethodInterceptor.class});
			enhancer.setCallbackFilter(new CallbackFilter() {
				@Override
				public int accept(Method method) {
					return 1;
				}
				
				@Override
				public boolean equals(Object obj) {
					return super.equals(obj);
				}
			});
			
			Class clazz = enhancer.createClass();
			System.out.println(clazz.getName());
			//显示数量信息（共加载过的类型数目，当前还有效的类型数目，已经被卸载的类型数目）
			System.out.println("total: " + loadingBean.getTotalLoadedClassCount());
			System.out.println("active: " + loadingBean.getLoadedClassCount());
			System.out.println("unloaded: " + loadingBean.getUnloadedClassCount());
		}
	}
}
1234567891011121314151617181920212223242526272829
```

#### c. 运行输出：

```java
jvm.MetaSpaceOomMock$$EnhancerByCGLIB$$567f7ec0
total: 6265
active: 6265
unloaded: 0
jvm.MetaSpaceOomMock$$EnhancerByCGLIB$$3501581b
total: 6266
active: 6266
unloaded: 0
Exception in thread "main" net.sf.cglib.core.CodeGenerationException: java.lang.reflect.InvocationTargetException-->null
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:345)
	at net.sf.cglib.proxy.Enhancer.generate(Enhancer.java:492)
	at net.sf.cglib.core.AbstractClassGenerator$ClassLoaderData$3.apply(AbstractClassGenerator.java:93)
	at net.sf.cglib.core.AbstractClassGenerator$ClassLoaderData$3.apply(AbstractClassGenerator.java:91)
	at net.sf.cglib.core.internal.LoadingCache$2.call(LoadingCache.java:54)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at net.sf.cglib.core.internal.LoadingCache.createEntry(LoadingCache.java:61)
	at net.sf.cglib.core.internal.LoadingCache.get(LoadingCache.java:34)
	at net.sf.cglib.core.AbstractClassGenerator$ClassLoaderData.get(AbstractClassGenerator.java:116)
	at net.sf.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:291)
	at net.sf.cglib.proxy.Enhancer.createHelper(Enhancer.java:480)
	at net.sf.cglib.proxy.Enhancer.createClass(Enhancer.java:337)
	at jvm.MetaSpaceOomMock.main(MetaSpaceOomMock.java:38)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at net.sf.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:413)
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:336)
	... 12 more
Caused by: java.lang.OutOfMemoryError: Metaspace
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	... 17 more
123456789101112131415161718192021222324252627282930313233
```

如果是1.7的jdk，那么报OOM的将是PermGen区域。

### 6. 直接内存

jdk1.4引入了NIO，它可以使用Native函数库直接分配堆外内存。