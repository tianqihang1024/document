## 一文彻底理解ReentrantLock可重入锁的使用

https://baijiahao.baidu.com/s?id=1648624077736116382&wfr=spider&for=pc

愚公要移山1

发布时间：19-10-2816:14科技达人，优质创作者

java除了使用关键字synchronized外，还可以使用ReentrantLock实现独占锁的功能。而且ReentrantLock相比synchronized而言功能更加丰富，使用起来更为灵活，也更适合复杂的并发场景。这篇文章主要是从使用的角度来分析一下ReentrantLock。

**一、简介**

ReentrantLock常常对比着synchronized来分析，我们先对比着来看然后再一点一点分析。

（1）synchronized是独占锁，加锁和解锁的过程自动进行，易于操作，但不够灵活。ReentrantLock也是独占锁，加锁和解锁的过程需要手动进行，不易操作，但非常灵活。

（2）synchronized可重入，因为加锁和解锁自动进行，不必担心最后是否释放锁；ReentrantLock也可重入，但加锁和解锁需要手动进行，且次数需一样，否则其他线程无法获得锁。

（3）synchronized不可响应中断，一个线程获取不到锁就一直等着；ReentrantLock可以相应中断。

ReentrantLock好像比synchronized关键字没好太多，我们再去看看synchronized所没有的，一个最主要的就是ReentrantLock还可以实现公平锁机制。什么叫公平锁呢？也就是在锁上等待时间最长的线程将获得锁的使用权。通俗的理解就是谁排队时间最长谁先执行获取锁。

字数写得多可能大家都会烦，干脆直接上代码演示。

**二、使用**

1、简单使用

我们先给出一个最基础的使用案例，也就是实现锁的功能。

![img](https://pics4.baidu.com/feed/1ad5ad6eddc451da3340cf0f31befb63d116326d.jpeg?token=453a71e9bc059e080fe73f0588151612&s=3281B14C12B4B26F16C9950B0000F0C1)

在这里我们定义了一个ReentrantLock，然后再test方法中分别lock和unlock，运行一边就可以实现我们的功能。这就是最简单的功能实现，代码很简单。我们再看看ReentrantLock和synchronized不一样的地方，那就是公平锁的实现。

2、公平锁实现

对于公平锁的实现，就要结合着我们的可重入性质了。公平锁的含义我们上面已经说了，就是谁等的时间最长，谁就先获取锁。

![img](https://pics5.baidu.com/feed/d53f8794a4c27d1e2bb5a7729c96046bdcc438ba.jpeg?token=b6d18757a2214aea7c53c09c6d95cd5c&s=B281B14C9AB4B24F46C9B40A000070C1)

首先new一个ReentrantLock的时候参数为true，表明实现公平锁机制。在这里我们多定义几个线程ABCDE，然后再test方法中循环执行了两次加锁和解锁的过程。

![img](https://pics1.baidu.com/feed/11385343fbf2b211aee9477b4dc3cc3d0dd78e8d.jpeg?token=d4df31a7b4fb1589ad81385fec7e8ac8&s=571CEE2A0F1A484146C101DB0000D0B6)

3、非公平锁实现

非公平锁那就随机的获取，谁运气好，cpu时间片轮到哪个线程，哪个线程就能获取锁，和上面公平锁的区别很简单，就在于先new一个ReentrantLock的时候参数为false，当然我们也可以不写，默认就是false。直接测试一下

![img](https://pics7.baidu.com/feed/267f9e2f07082838cf3ef17630da00044c08f16e.jpeg?token=58c526d42f44f49194aa99d59651fb6b&s=4BA438620B2B400950FC29DA0000C0B2)

4、响应中断

响应中断就是一个线程获取不到锁，不会傻傻的一直等下去，ReentrantLock会给予一个中断回应。在这里我们举一个死锁的案例。

首先我们定义一个测试类ReentrantLockTest3。

![img](https://pics3.baidu.com/feed/eaf81a4c510fd9f9ee7d853da26e7d2f2934a4c1.jpeg?token=428cef42e544014c91b9c79d5b72ee5c&s=3281B14483F0BC6850D84D8F0000E081)

在这里我们定义了两个锁lock1和lock2。然后使用两个线程thread和thread1构造死锁场景。正常情况下，这两个线程相互等待获取资源而处于死循环状态。但是我们此时thread中断，另外一个线程就可以获取资源，正常地执行了。

![img](https://pics5.baidu.com/feed/cc11728b4710b9121f73027a44be550693452249.jpeg?token=58a06ad7da017c17e044a00a1bf2bc56&s=BA81A14CDAB6864F56DD9D0B0000A0C1)

我们运行测试一下：

![img](https://pics3.baidu.com/feed/58ee3d6d55fbb2fb39f5e173c80989a14723dcba.jpeg?token=e50cd658872c42da3e3794101de3dafb&s=7AC6AC1A94E849035CE5D9CB020090B1)

5、限时等待

这个是什么意思呢？也就是通过我们的tryLock方法来实现，可以选择传入时间参数，表示等待指定的时间，无参则表示立即返回锁申请的结果：true表示获取锁成功，false表示获取锁失败。我们可以将这种方法用来解决死锁问题。

首先还是测试代码，不过在这里我们不需要再去中断其中的线程了，我们直接看线程类是如何实现的。

![img](https://pics6.baidu.com/feed/bd3eb13533fa828b293a59917a5ce831970a5a74.jpeg?token=9a6750f03573f41ff24ef8218d493b8e&s=BA81B14C12B4B46C56DDB50F0000A0C1)

在这个案例中，一个线程获取lock1时候第一次失败，那就等10毫秒之后第二次获取，就这样一直不停的调试，一直等到获取到相应的资源为止。

当然，我们可以设置tryLock的超时等待时间tryLock(long timeout,TimeUnit unit)，也就是说一个线程在指定的时间内没有获取锁，那就会返回false，就可以再去做其他事了。

OK，到这里我们就把ReentrantLock常见的方法说明了，所以其原理，还是主要通过源码来解释。而且分析起来还需要集合AQS和CAS机制来分析。我也会在下一篇文章来分析。感谢大家的持续关注和支持。