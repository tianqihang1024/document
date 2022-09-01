# `@DependsOn`



## 用途

注解`@DependsOn`位于如下包

```java
org.springframework.context.annotation
```


该注解用于声明当前`bean`依赖于另外一个`bean`。所依赖的`bean`会被容器确保在当前`bean`实例化之前被实例化。

举例来讲，如果容器通过`@DependsOn`注解方式定义了`bean plant`依赖于`bean water`,那么容器在会确保bean water的实例在实例化`bean plant`之前完成。

**一般用在一个`bean`没有通过属性或者构造函数参数显式依赖另外一个`bean`，但实际上会使用到那个`bean`或者那个`bean`产生的某些结果的情况。**



## 用法

直接或者间接标注在带有`@Component`注解的类上面;
使用`@DependsOn`注解到类层面仅仅在使用component scanning方式时才有效;

如果带有`@DependsOn`注解的类通过`XML`方式使用，该注解会被忽略，<bean depends-on="..."/>这种方式会生效;

直接或者间接标注在带有`@Bean` 注解的方法上面;



## 用法举例

`SpringCloud-Alibaba`整合`Seata`时，需要对`druidDataSource`进行代理，但是这个`@Bean`加载的顺序不固定，导致确实对象启动失败。接方法就是在代理那块添加这个注解，等待`druidDataSource`注入后在初始化代理。





