







### `ReentrantLock`公平和非公平的区别在哪里

非公平锁的待新增节点，总是尝试抢占锁，并不管等待队列是否存在。公平锁的节点，在等待队列初始化后，会老实入队，等待前面节点执行完成轮到自己。（非公平一开始就尝试去抢占锁。如果未抢占到锁，准备入对休眠前，锁被释放，会尝试再抢一次）





### `lock、tryLock、lockInterruptibly`的区别

-   `lock`：阻塞
-   `tryLock`：非阻塞响应中断
-   `lockInterruptibly`：阻塞响应中断





### `FIFO`等待队列节点的状态什么时候修改

等待节点的状态由后续节点维护，释放锁的时候会判断节点状态是否为`-1`，用于判断是否需要唤醒后续节点。





### `lock.unlock`唤醒后续节点时，为什么从尾向前循环

节点入队并不是原子操作，也就是说，`node.prev = pred; compareAndSetTail(pred, node)` 这两个地方可以看作Tail入队的原子操作，但是此时`pred.next = node;`还没执行，如果这个时候执行了`unparkSuccessor`方法，就没办法从前往后找了，所以需要从后往前找。还有一点原因，在产生`CANCELLED`状态节点的时候，先断开的是`Next`指针，`Prev`指针并未断开，因此也是必须要从后往前遍历才能够遍历完全部的`Node`。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer

private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) {
		node.prev = pred;
		if (compareAndSetTail(pred, node)) {
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}


/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```





### `lock.tryLock`指定等待获取锁时间时，没有抢占到锁，进入阻塞状态后怎么判断阻塞时长

阻塞是可以指定时长的，`LockSupport.parkNanos（Nanos：纳秒=挠挠）`函数可以到达时间后自动唤醒。再次试图抢占锁，不成功返回`false`。

-   `LockSupport.parkNanos(this, nanosTimeout);`：阻塞指定对象`this`，指定时长`nanosTimeout`，到期后自动唤醒。
-   `LockSupport.park(this);`：阻塞指定对象this，需要其他线程调用
-   



```
log.info("线程被打断了");
```



`park`函数阻塞的是线程还是锁对象





### `ReentrantLock`的死锁能否通过`interrupt`终止

可以的，使用











### 为什么只出现了一次`正常退出`：[ReentrantLock可中断锁_Terisadeng的博客-CSDN博客_reentrantlock可中断](https://blog.csdn.net/dongyuxu342719/article/details/94395877)

博主调用了`t1.interrupt();`使得`t1`因为中断异常，在未能获取`t2`锁的情况下就进入`finally`，但是代码里又直接对两个锁进行了释放，释放一个不属于自己的锁会抛出异常（代码里应该使用`lock.isHeldByCurrentThread`判断当前线程持有锁后再去`lock.unLock`）正常打印的线程，因为竞争者的异常退出而获取到另一把锁，最后是放的都是自身持有的锁。应该是这个问题导致的正常结束出现了一次。







### interrupt、interrupted`和`isInterrupted`的区别

`interrupte`：设置当前线程的中断状态。

`interrupted`：测试当前线程是否已被中断。此方法清除线程的中断状态。

`isInterrupted`：测试此线程是否已被中断。线程的中断状态不受此方法影响。

需要注意的是，`interrupted`是一个静态方法，测试调用方法的线程体的中断标识，而不是目标线程实例（内部使用`currentThread`获取当前线程，在使用`isInterrupted`获取当前线程的中断标识）。









### `lock、tryLock、lockInterruptibly`都可以响应中断，那么区别在那里

却别在于`tryLock、lockInterruptibly`立即响应中断，而`lock`在获取锁后，返回时才会检查时候被中断（`sleep、join、wait`）。

