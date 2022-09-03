

## `ThreadLocal`是什么

创建线程局部变量，存放只有当前线程能够访问的对象。



## 用法简介

### 创建，支持泛型

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();
```

### 赋值

```java
threadLocal.set(IdWorker.get32UUID());
```

### 取值

```java
threadLocal.get()
```

### 删除

```java
threadLocal.remove();
```



### `ThreadLocal`内存泄漏如何防止

确定不使用后调用`remove`函数清除，`get、set`函数虽说能够清除失效的条目，但是得命中失效的条目后才会进行替换，存在命中率的问题。命中失败后在`get`函数中会通过`getEntryAfterMiss`清除失效条目，`set`则是`replaceStaleEntry`。



### `ThreadLocalMap`中的`key`存放的是`ThreadLocal`的句柄，既然是`Map`结构不应该，`Key`相同则覆盖`Value`嘛？

初始情况下`ThreadLocal<String> a = new ThreadLocal();`直接定义在`class`中，使用`set`方法`Key`相同则覆盖`Value`。这么想是没错的，但是有种情况就是，`a = new ThreadLocal();`这样的话，`a`就更换了引用，从老的对象切换到了新的对象，在下次`GC`之前，调用`set`赋值的话，当前线程的`ThreadLocalMap`会有两个`Entry`对象，一个是老的，一个是新的。这就是（不过不用担心，`ThreadLocal`中的`get、set、remove`方法都有清除过时值的逻辑）



### 为什么说`ThreadLocalMap`的`Key`是弱引用

- `ThreadLocalMap`存放的是`Entry`对象。`Entry`继承`WeakReference`，有两个属性`referent、value`，`referent`被`WeakReference`的构造函数包裹裹成了弱引用，而`value`是直接`=`号赋值强引用。
- 初始情况下`ThreadLocal<String> a = new ThreadLocal();`声明了成员变量，后续确定不使用后使用`a = null;`断开引用，下次`GC`时这个`a`对象会被回收。如果`ThreadLocalMap`的`key`使用强引用的话，`key`依旧持有`a`对象这个引用，根据可达性算法`a`依旧被引用，不能被回收，造成内存泄漏。弱引用在没有就没有这方面的顾虑，当`ThreadLocal`对象没有强应用后`GC`能够直接将器回收。回收后`ThreadLocalMap`的`key`为`null`，在后续调用`get、set、remove`方法时，都有清除过时值的逻辑，清除所有key为`null`的`Entry`。











