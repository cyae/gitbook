## 前言

其实 spring 的[aop](https://so.csdn.net/so/search?q=aop&spm=1001.2101.3001.7020)非常的强大, 因此研究一下 AspectJ 还是有必要, 而不是仅仅停留在初级的阶段.  

比如 spring 的事务是基于 aop 来实现的, 如果不能深入的研究, 可能很多知识点, 只知其然而不知其所以然.  
　　
本文将简单地讲述如何指定 AspectJ 的织入顺序, 以及如何指定通知参数.

## AspectJ 的博文

以下博文是之前实战中记录的.  
1. [利用 Aspectj 实现 Oval 的自动参数校验](https://www.cnblogs.com/mumuxinfei/p/9328057.html)
2. [类 Shiro 权限校验框架的设计和实现](https://www.cnblogs.com/mumuxinfei/p/9339086.html)
　　
以下博文是本文参考的文章(强烈推荐):  
1. [AspectJ 切入点语法详解](http://jinnianshilongnian.iteye.com/blog/1415606)

## 织入顺序

如果同一个函数调用, 涉及多个 AOP 的织入, 那么这些 AOP 的顺序该如何定义和指定? 为了解决这个问题, AspectJ 引入了 Order, 它约定了 order 数值越小, 优先级越高(越早被调用).  

AspectJ 类指定顺序的方式有两种.

### 1. 引入注解@Order\*\*

```java
import org.springframework.core.annotation.Order;

@Aspect
@Component
@Order(1)
public class MyAdvice1 {
    @Pointcut("execution(* com.springapp.mvc.controller.*.*(..))")
    public void pointCut() {
    }
}
```

### 2. 实现 Ordered 接口\*\*

```java
import org.springframework.core.Ordered;

@Aspect
@Componentpublic
class MyAdvice2 implements Ordered {
    @Pointcut("execution(* com.springapp.mvc.controller.*.*(..))")
    public void pointCut() {
    }

    @Override
    public int getOrder() {
        return 2;
    }
}
```

无论是那种, 其遵守的标准是一定的.  
总的来说, 其顺序规则如下:  
1. 在同一切面类内, 按照切入点的定义顺序来织入
2. 在不同的切面类内, 都实现了 Ordered 接口, 按切入点的 Order 数值从小到达织入.
3. 在不同的切面类内, 存在没实现 Ordered 接口的类, 则切入点的顺序不确定.

## 通知参数指定

通知参数的指定, 一定程度上是为方便编程, 提升了开发效率.  
我之前对 Aspectj 了解没那么深入的时候, 一直用吃力不讨好的方式在开发.  

比如之前写的权限校验小框架, 其核心代码如下:

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyRequiresRoles {
    String[] value();

    MyLogic logic() default MyLogic.AND;
}

@Aspect
@Componentpublic
class MyShiroAdvice {
    /**
     * 定义切点, 用于角色的校验
     */
    @Pointcut("@annotation(com.springapp.mvc.myshiro.MyRequiresRoles)")
    public void checkRoles() {
    }

    @Before("checkRoles()")
    public void doCheckRole(JoinPoint jp) throws Exception {
        // 从JointPoint变量中提取对应的注解
        MyRequiresRoles mrp = extractAnnotation((MethodInvocationProceedingJoinPoint) jp, MyRequiresRoles.class);
        try {
            // 获取注解设置的值(角色集合, 逻辑操作), 进行评估判断
            if (!MyShiroHelper.validateRoles(mrp.value(), mrp.logic())) {
                throw new Exception("access disallowed");
            }
        } catch (Exception e) {
            throw new Exception("invalid state");
        }
    }

    // 获取注解信息
    private static <T extends Annotation> T extractAnnotation(MethodInvocationProceedingJoinPoint mp, Class<T> clazz) throws Exception {
        Field proxy = mp.getClass().getDeclaredField("methodInvocation");
        proxy.setAccessible(true);
        ReflectiveMethodInvocation rmi = (ReflectiveMethodInvocation) proxy.get(mp);
        Method method = rmi.getMethod();
        return (T) method.getAnnotation(clazz);
    }
}
```

在具体的拦截方法中, **通过 JointPoint 对象, 获取对应的调用方法/注解/参数等信息**. 但这种方式不够简洁, 容易导致类型转换错误.  
　　
是否有一种办法, 能够做到所需参数的随叫随到, 而且避免了类型转换的坑.  
答案是肯定的, 这为大英雄就是**通知参数指定**.  

针对上面一个列子, 我们可以引入切面指示符@annotation 类实现:

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyRequiresRoles {
    String[] value();

    MyLogic logic() default MyLogic.AND;
}

@Aspect
@Componentpublic
class MyShiroAdvice {
    /**
     * 定义切点, 用于角色的校验
     */
    @Pointcut("@annotation(com.springapp.mvc.myshiro.MyRequiresRoles)")
    public void checkRoles() {
    }

    // 通知参数指定, 通过指示符@annotation()指定了注解@MyRequiresRoles参数
    @Before("checkRoles() && @annotation(mrp)")
    public void doCheckRole(MyRequiresRoles mrp) throws Exception {
        try {
            if (!MyShiroHelper.validateRoles(mrp.value(), mrp.logic())) {
                throw new Exception("access disallowed");
            }
        } catch (Exception e) {
            throw new Exception("invalid state");
        }
    }
}
```

注: **_对比上述两个代码, 功能不变, 却直接导入想要的注解信息(间接地规避了类型转换), 大大简化了代码编写_**.

我们再来一个切面指示符 args 的使用例子:

```java
@Before(value = "checkRoles() && args(k, v)", argNames = "k, v")    public void doCheckRole2(String k, String v) {
    // TODO
}
```

*注: 只有满足切面 checkRole()规则, 同时调用函数签名的参数列表为 methodName(String, String), 才触发调用.  
　　
这个例子确实轻而易举的获取了调用函数的参数.

### AspectJ 指示符

举例一下常见的一些指示符:

- execution：用于匹配方法执行的连接点.
- within：用于匹配指定类型内的方法执行.
- this：用于匹配当前 AOP 代理对象类型的执行方法, 注意是 AOP 代理对象的类型匹配，这样就可能包括引入接口也类型匹配.
- target：用于匹配当前目标对象类型的执行方法, 注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配.
- args：用于匹配当前执行的方法传入的参数为指定类型的执行方法.
- @annotation：用于匹配当前执行方法持有指定注解的方法.
- reference pointcut：表示引用其他命名切入点.
