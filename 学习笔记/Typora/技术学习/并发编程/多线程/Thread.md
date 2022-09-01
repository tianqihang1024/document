















### interrupt、interrupted`和`isInterrupted`的区别

`interrupted`：设置当前线程的中断状态。

`interrupted`：测试当前线程是否已被中断。此方法清除线程的中断状态。

`isInterrupted`：测试此线程是否已被中断。线程的中断状态不受此方法影响。

需要注意的是，`interrupted`是一个静态方法，测试调用方法的线程体的中断标识，而不是目标线程实例（内部使用`currentThread`获取当前线程，在使用`isInterrupted`获取当前线程的中断标识）。







### `Thread.sleep()`和`LockSupport.park()`

两者的作用都是将当前线程置为阻塞状态，区别在于使用上的差异。

`sleep`睡眠阻塞时被中断，会抛出







使用



