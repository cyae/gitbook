## 基础

### 通知（Advice）

通知描述了切面要完成的工作以及何时执行。比如我们的日志切面需要记录每个接口调用时长，就需要在接口调用前后分别记录当前时间，再取差值。

- 前置通知（Before）：在目标方法调用前调用通知功能；
- 后置通知（After）：在目标方法调用之后调用通知功能，不关心方法的返回结果；
- 返回通知（AfterReturning）：在目标方法成功执行之后调用通知功能；
- 异常通知（AfterThrowing）：在目标方法抛出异常后调用通知功能；
- 环绕通知（Around）：通知包裹了目标方法，在目标方法调用之前和之后执行自定义的行为。

### 切点（Pointcut）

切点定义了通知功能被应用的范围。比如日志切面的应用范围就是所有接口，即所有 controller 层的接口方法。

### 切面（Aspect）

切面是通知和切点的结合，定义了何时、何地应用通知功能。

### 引入（Introduction）

在无需修改现有类的情况下，向现有的类添加新方法或属性。

### 织入（Weaving）

把切面应用到目标对象并创建新的代理对象的过程。

### 连接点（JoinPoint）

通知功能被应用的时机。比如接口方法被调用的时候就是日志切面的连接点。

#### JointPoint 和 ProceedingJoinPoint

JointPoint 是程序运行过程中可识别的连接点，这个点可以用来作为 AOP 切入点。JointPoint 对象则包含了和切入相关的很多信息，比如切入点的方法，参数、注解、对象和属性等。我们可以通过反射的方式获取这些点的状态和信息，用于追踪 tracing 和记录 logging 应用信息。

##### JointPoint

通过 JpointPoint 对象可以获取到下面信息

```java
# 返回目标对象，即被代理的对象
Object getTarget();

# 返回切入点的参数
Object[] getArgs();

# 返回切入点的Signature
Signature getSignature();

# 返回切入的类型，比如method-call，field-get等等，感觉不重要
String getKind();
```

##### ProceedingJoinPoint

Proceedingjoinpoint 继承了 JoinPoint。是在 JoinPoint 的基础上暴露出 proceed 这个方法。proceed 很重要，这个是 aop 代理链执行的方法。环绕通知=前置+目标方法执行+后置通知，proceed 方法就是用于启动目标方法执行的。

暴露出这个方法，就能支持 aop:around 这种切面（而其他的几种切面只需要用到 JoinPoint，，这也是环绕通知和前置、后置通知方法的一个最大区别。这跟切面类型有关）， 能决定是否走代理链还是走自己拦截的其他逻辑。

ProceedingJoinPoint 可以获取切入点的信息:

- 切入点的方法名字及其参数
- 切入点方法标注的注解对象（通过该对象可以获取注解信息）
- 切入点目标对象（可以通过反射获取对象的类名，属性和方法名）

```java
//获取切入点方法的名字,getSignature());是获取到这样的信息 :修饰符+ 包名+组件名(类名) +方法名
String methodName = joinPoint.getSignature().getName()

//获取方法的参数,这里返回的是切入点方法的参数列表
Object[] args = joinPoint.getArgs();

//获取方法上的注解
Signature signature = joinPoint.getSignature();
MethodSignature methodSignature = (MethodSignature) signature;
Method method = methodSignature.getMethod();
if (method != null)
{
   xxxxxx annoObj= method.getAnnotation(xxxxxx.class);
}

//获取切入点所在目标对象
Object targetObj =joinPoint.getTarget();
//可以发挥反射的功能获取关于类的任何信息，例如获取类名如下
String className = joinPoint.getTarget().getClass().getName();
```

### AOP 相关注解

Spring 中使用注解创建切面

- @Aspect：用于定义切面
- @Before：通知方法会在目标方法调用之前执行
- @After：通知方法会在目标方法返回或抛出异常后执行
- @AfterReturning：通知方法会在目标方法返回后执行
- @AfterThrowing：通知方法会在目标方法抛出异常后执行
- @Around：通知方法会将目标方法封装起来
- @Pointcut：定义切点表达式
- 切点表达式：指定了通知被应用的范围，表达式格式：[AspectJ 切入点语法详解](http://jinnianshilongnian.iteye.com/blog/1415606)

```java
execution(方法修饰符 返回类型 方法所属的包.类名.方法名称(方法参数)
//com.hs.demo.controller包中所有类的public方法都应用切面里的通知
execution(public * com.hs.demo.controller.*.*(..))
//com.hs.demo.service包及其子包下所有类中的所有方法都应用切面里的通知
execution(* com.hs.demo.service..*.*(..))
//com.hs.demo.service.EmployeeService类中的所有方法都应用切面里的通知
execution(* com.hs.demo.service.EmployeeService.*(..))
```

#### @POINTCUT 定义切入点，有以下 2 种方式

> 方式一：设置为注解@LogFilter 标记的方法，有标记注解的方法触发该 AOP，没有标记就没有。

```java
@Aspect
@Component
public class LogFilter1Aspect {
    @Pointcut(value = "@annotation(com.hs.aop.annotation.LogFilter)")
    public void pointCut(){

    }
}
```

自定义注解 LogFilter：

```java
@Target(ElementType.METHOD)
@Retention(value = RetentionPolicy.RUNTIME)
public @interface LogFilter1 {
}
```

对应的 Controller 方法如下，手动添加@LogFilter 注解：

```java
@RestController
public class AopController {

    @RequestMapping("/aop")
    @LogFilter
    public String aop(){
        System.out.println("这是执行方法");
        return "success";
    }
}
```

> 方式二：采用表达式批量添加切入点，如下方法，表示 AopController 下的所有 public 方法都添加 LogFilter1 切面。

```java
@Pointcut(value = "execution(public * com.train.aop.controller.AopController.*(..))")
public void pointCut(){

}
```

#### @Around 环绕通知

@Around 集成了@Before、@AfterReturing、@AfterThrowing、@After 四大通知。需要注意的是，他和其他四大通知注解最大的不同是需要手动进行接口内方法的反射后才能执行接口中的方法，换言之，@Around 其实就是一个动态代理。

```java
   /**
     * 环绕通知是spring框架为我们提供的一种可以在代码中手动控制增强部分什么时候执行的方式。
     *
     */
    public void aroundPringLog(ProceedingJoinPoint pjp)
   {
      //拿到目标方法的方法签名
       Signature signature = pjp.getSignature();
      //获取方法名
      String name = signature.getName();

      try {
            //@Before
            System.out.println("【环绕前置通知】【"+name+"方法开始】");
            //这句相当于method.invoke(obj,args)，通过反射来执行接口中的方法
            proceed = pjp.proceed();
            //@AfterReturning
            System.out.println("【环绕返回通知】【"+name+"方法返回，返回值："+proceed+"】");
        } catch (Exception e) {
            //@AfterThrowing
            System.out.println("【环绕异常通知】【"+name+"方法异常，异常信息："+e+"】");
        }
        finally{
            //@After
            System.out.println("【环绕后置通知】【"+name+"方法结束】");
        }
    }
```

proceed = pjp.proceed(args)这条语句其实就是 method.invoke，以前手写版的动态代理，也是 method.invoke 执行了，jdk 才会利用反射进行动态代理的操作，在 Spring 的环绕通知里面，只有这条语句执行了，spring 才会去切入到目标方法中。

为什么说环绕通知就是一个动态代理呢？

> proceed = pjp.proceed(args)这条语句就是动态代理的开始，当我们把这条语句用 try-catch 包围起来的时候，在这条语句前面写的信息，就相当于前置通知，在它后面写的就相当于返回通知，在 catch 里面写的就相当于异常通知，在 finally 里写的就相当于后置通知。

## 实战

1. [利用 Aspectj 实现 Oval 的自动参数校验](https://www.cnblogs.com/mumuxinfei/p/9328057.html)
2. [类 Shiro 权限校验框架的设计和实现](https://www.cnblogs.com/mumuxinfei/p/9339086.html)

## 同一方法有多切面，默认优先级

- 在同一切面类内, 按照切入点的定义顺序来织入
- 在不同的切面类内, 都实现了`Ordered`接口, 按切入点的 Order 数值从小到达织入.
- 在不同的切面类内, 存在没实现`Ordered`接口的类, 则切入点的顺序不确定.

## 织入顺序

如果同一个函数调用, 涉及多个 AOP 的织入, 那么这些 AOP 的顺序该如何定义和指定? 为了解决这个问题, AspectJ 引入了 Order, 它约定了 order 数值越小, 优先级越高(越早被调用).

AspectJ 类指定顺序的方式有两种.

### 1. 引入注解@Order

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

### 2. 实现 Ordered 接口

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

*注: 只有满足切面 checkRole()规则, 同时调用函数签名的参数列表为 methodName(String, String), 才触发调用.*

### AspectJ 指示符

举例一下常见的一些指示符:

- execution：用于匹配方法执行的连接点.
- within：用于匹配指定类型内的方法执行.
- this：用于匹配当前 AOP 代理对象类型的执行方法, 注意是 AOP 代理对象的类型匹配，这样就可能包括引入接口也类型匹配.
- target：用于匹配当前目标对象类型的执行方法, 注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配.
- args：用于匹配当前执行的方法传入的参数为指定类型的执行方法.
- @annotation：用于匹配当前执行方法持有指定注解的方法.
- reference pointcut：表示引用其他命名切入点.
