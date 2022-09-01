

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



### `ThreadLocalMap`中的`key`存放的是`ThreadLocal`的句柄，既然是`Map`结构不应该，`Key`相同则覆盖Value`嘛？

`ThreadLocalMap`属于`Thread`，虽然你的`ThreadLocal`句柄是同一个，但是 它们是不同的容器，隶属于不同的`Thread`。



### 为什么说`ThreadLocalMap`的`Key`是软引用

`ThreadLocalMap`存放的是`Entry`对象。`Entry`继承`WeakReference`，有两个属性`referent、value`，`referent`被`WeakReference`的构造函数包裹裹成了软引用，而`value`是直接`=`号赋值强引用。