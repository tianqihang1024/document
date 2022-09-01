# @Autowried注解和@Resource注解的区别

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[根正苗红鹏哥哥](https://me.csdn.net/sinat_31155413) 2019-06-21 17:25:50 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 1324 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 5



分类专栏： [注解](https://blog.csdn.net/sinat_31155413/category_9049934.html) [spring boot](https://blog.csdn.net/sinat_31155413/category_9050724.html)

版权

### 1、@Autowried

用来装配bean, 可作用于字段上, 也可以作用于setter方法上.

是Spring的注解.

默认情况下要求对象必须存在, 它要求依赖对象必须存在. 若允许null值, 可以设置它的required为false.

默认按照类型byType进行装配注入. 如果想按照名称进行装配的话, 需要与Qualifer注解搭配使用

**如下：**

```java
@Autowired
@Qualifier("RedisTemplate")
private RedisTemplate redisTemplate;

//或者
@Autowired
private RedisTemplate redisTemplate;
```



### 2、@Resource

用来装配bean, 可作用于字段上, 也可以作用于setter方法上.

是J2EE的注解.

默认按照名称来装配注入, 只有找不到与名称匹配的bean才会按照类型来注入.

它有两个属性是比较重要的:

- name: Spring将name的属性值解析为bean的名称, 使用byName的自动注入策略
- type: Spring将type的属性值解析为bean的类型, 使用byType的自动注入策略，如果既不指定name属性又不指定type属性, Spring这时通过反射机制默认使用byName自动注入策略

 **以下这三种方式都可以成功注入：**

```java

//通过name属性注入
@Resource(name = "RedisTemplate")
private RedisTemplate redisTemplate;


//通过type属性注入
@Resource(type = RedisTemplate.class)
private RedisTemplate redisTemplate;


//通过name，type属性注入
@Resource(name = "RedisTemplate",type = RedisTemplate.class)
private RedisTemplate redisTemplate;
```

注：Resource注解是J2EE提供的, 而Autowried注解是Spring提供的,





