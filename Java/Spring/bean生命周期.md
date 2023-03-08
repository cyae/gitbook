
![image](https://img2022.cnblogs.com/blog/2827284/202208/2827284-20220820192516972-1185624100.png)
![image](https://img2022.cnblogs.com/blog/2827284/202208/2827284-20220820192525960-1575101719.png)

1. Bean 容器找到配置文件中 Spring Bean 的定义。
2. Bean 容器利用 Java Reflection API 创建一个Bean的实例。
3. 如果涉及到一些属性值 利用 set()方法设置一些属性值。
4. 如果 Bean 实现了 BeanNameAware 接口，调用 setBeanName()方法，传入Bean的名字。
5. 如果 Bean 实现了 BeanClassLoaderAware 接口，调用 setBeanClassLoader()方法，传入 ClassLoader对象的实例。
6. 与上面的类似，如果实现了其他 *.Aware接口，就调用相应的方法。
7. 如果有和加载这个 Bean 的 Spring 容器相关的 BeanPostProcessor 对象，执行postProcessBeforeInitialization() 方法
8. 如果Bean实现了InitializingBean接口，执行afterPropertiesSet()方法。
9. 如果 Bean 在配置文件中的定义包含 init-method 属性，执行指定的方法。
10. 如果有和加载这个 Bean的 Spring 容器相关的 BeanPostProcessor 对象，执行postProcessAfterInitialization() 方法
11. 当要销毁 Bean 的时候，如果 Bean 实现了 DisposableBean 接口，执行 destroy() 方法。
12. 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性，执行指定的方法。