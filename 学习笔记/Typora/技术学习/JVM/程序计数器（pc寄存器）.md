# 程序计数器（pc寄存器）

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[Stone/](https://me.csdn.net/qq_44892091) 2020-01-23 21:23:20 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 1267 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏



版权

## 程序计数器（pc寄存器）：

pc寄存器 每个线程有一份
作用：PC寄存器用来存储指向下一条指令的地址，也即将要执行的指令代码。由执行引擎读取下一条指令。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200123110525525.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ0ODkyMDkx,size_16,color_FFFFFF,t_70)
pc寄存器它是一块很小的内存空间，几乎可以忽略不记。也是运行速度最快的存储区域。

在JVM规范中，每个线程都有它自己的程序计数器，是线程私有的，生命周期与线程的生命周期保持一致。

任何时间一个线程都只有一个方法在执行，也就是所谓的当前方法。程序计数器会存储当前线程正在执行的Java方法的JVM指令地址;或者，
如果是在执行native方法，则是未指定值(undefined)。

它是程序控制流的指示器,，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令。

它是唯一一个在Java虚拟机规范中没有规定任何OutotMemoryError情况的区域。

**pc寄存器的二个问题：**
问题一：使用PC寄存器存储字节码指令地址有什么用呢? 为什么使用PC寄存器记录当前线程的执行地址呢?

> 因为CPU需要不停的切换各个线程，这时候切换回来以后，就得知道接着从哪开始继续执行。
>
> JVM的字节码解释器就需要通过改变PC寄存器的值来明确下一条应该执行什么样的字节码指令。

问题二：PC寄存器为什么会被设定为线程私有?

> 为了能够准确地记录各个线程正在执行的当前字节码指令地址，最好的办法自然是为每一个线程都分配一个PC寄存器，这样一来各个线程之间便可以进行独立计算，从而不会出现相互干扰的情况。