## 生命周期
![image](https://img2022.cnblogs.com/blog/2827284/202208/2827284-20220820192516972-1185624100.png)
![image](https://img2022.cnblogs.com/blog/2827284/202208/2827284-20220820192525960-1575101719.png)

1. Bean 容器找到配置文件中 Spring Bean 的定义。
2. Bean 容器利用 Java Reflection API 创建一个 Bean 的实例。
3. 如果涉及到一些属性值 利用 set()方法设置一些属性值。
4. 如果 Bean 实现了 BeanNameAware 接口，调用 setBeanName()方法，传入 Bean 的名字。
5. 如果 Bean 实现了 BeanClassLoaderAware 接口，调用 setBeanClassLoader()方法，传入 ClassLoader 对象的实例。
6. 与上面的类似，如果实现了其他 \*.Aware 接口，就调用相应的方法。
7. 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 postProcessBeforeInitialization() 方法
8. 类中添加了注解 @PostConstruct 的方法
9. 如果 Bean 实现了 InitializingBean 接口，执行 afterPropertiesSet()方法。
10. 如果 Bean 在配置文件中的定义包含 init-method 属性，执行指定的方法。
11. 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行 postProcessAfterInitialization() 方法
12. 当要销毁 Bean 的时候，如果 Bean 实现了 DisposableBean 接口，执行 destroy() 方法。
13. 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性，执行指定的方法。

## 同 beanId 哪个生效

- bean 不存在覆盖一说，发现有重复 beanId 的 bean 时则不加载
- 如果明确要做 spring bean 覆盖，则应该使用 spring 父子容器, 这时应考虑加载顺序
  - spring bean 的加载顺序首先是根据依赖关系，生成 DAG（有向无环图），然后依次加载。
  - 通过指定`@Order`/`@Primary`来主动干涉 spring bean 的创建优先级。
  - 无任何扰动的情况下，spring 加载路径中，越靠后的越先加载

## bean循环依赖

在Spring中，所有的bean默认是单例的，即singleton。而多例模式下的循环依赖，Spring是无法解决的。首先看下源码，这里有一个prototypesCurrentlyInCreation变量，很是重要，Spring也给出了说明，这个变量用来记录当前正在创建的beanName。最后再着重介绍下到底为什么要设这么一个标志。

```java
/** Names of beans that are currently in creation */
private final ThreadLocal<Object> prototypesCurrentlyInCreation = new NamedThreadLocal<>("Prototype beans currently in creation");
```

在刚开始调用doGetBean(beanName)来创建bean的时候，会调用isPrototypeCurrentlyInCreation方法进行判断，判断当前的beanName是否在当前线程变量中，如果在则直接抛出BeanCurrentlyInCreationException异常。

```java
/*在刚进来创建时，会判断当前bean是否正在创建*/
if (isPrototypeCurrentlyInCreation(beanName)) {
    //如果正在创建，就会抛出异常
	throw new BeanCurrentlyInCreationException(beanName);
}
 
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	return (curVal != null && (curVal.equals(beanName) || 
(curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
	}
```

第一次进来创建时，变量中肯定没有存储，返回false。接着往下走，调用beforePrototypeCreation方法，将当前的beanName加入当前线程的变量中保存起来，打上一个标记，当前的bean已经开始创建了，为以后的循环依赖做准备。

```java
//IOC容器创建原型模式Bean实例对象
else if (mbd.isPrototype()) {
	// It's a prototype -> create a new instance.
	//原型模式(Prototype)是每次都会创建一个新的对象
	Object prototypeInstance = null;
	try {
		//重点一：回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
		beforePrototypeCreation(beanName);
		//重点二：创建指定Bean对象实例
		prototypeInstance = createBean(beanName, mbd, args);
	}
	finally {
		//回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建
		afterPrototypeCreation(beanName);
	}
	//获取给定Bean的实例对象
	bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
 
protected void beforePrototypeCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	if (curVal == null) {
		this.prototypesCurrentlyInCreation.set(beanName);
	}
	//...省略无关代码
}
```

第二步会调用createBean方法，而crateBean的真正是实现是doCreateBean，这里会完成bean的创建，属性依赖注入，初始化等一系列操作。

```java
//创建Bean实例对象
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {
 
	//...省略无关代码
	try {
		//真正的核心：创建Bean的入口
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		return beanInstance;
	}
	catch (BeanCreationException ex) {
		// A previously detected exception with proper bean creation context already...
		throw ex;
	}
	//...省略无关代码
}
 
//真正创建Bean的方法
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
		throws BeanCreationException {
	//...省略无关代码	
	
	//调用默认构造函数进行反射生成实例
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	//Bean对象的初始化，依赖注入在此触发
	populateBean(beanName, mbd, instanceWrapper);
	//初始化Bean对象
	exposedObject = initializeBean(beanName, exposedObject, mbd);
	
	//...省略无关代码
	return exposedObject;
}
```

在调用populateBean方法时，会去为每个属性赋值，以上面的TestA,TestB为例，当testA发现依赖testB，接着就会去getBean(testB)，testB的创建流程跟testA一样。首先会先去set集合中获取当前beanName是否正在创建当中，如果没有，会进行set操作，保存到当前线程变量中。然后继续进行bean的创建。当调用到populateBean方法时，testB会发现依赖testA，那么转过身又会回去创建testA，当调用到isPrototypeCurrentlyInCreation方法时，此时的testA由于还在进行属性赋值，并没有完全的创建成功，那么理所当然他会在set集合当中，此时程序会直接抛出异常，告知当前的bean正在创建当中。

那么问题来了，如果不抛异常，会怎么样呢？没有这么一个set集合又会怎么样呢？大家可以试想一下。如果没有这么一个标记，testA在进行属性赋值时，发现依赖testB，马上又去创建testB，到testB进行属性赋值时，又发现依赖了testA，转过头又去创建testA，而testA依赖了testB，则又会去创建testB。循环往复，形成了一个死循环，永远都出不来了。所以Spring为了避免这种现象，索性直接抛出异常，因为在Spring看来，它也不知道如何来解决这种情况。

## Bean加载时间优化
### 检测耗时
1. 实现spring bean的前后置接口BeanPostProcessor，打印时间
2. arthas trace命令查看调用链路耗时
### 解决思路
- 