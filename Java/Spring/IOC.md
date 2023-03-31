---
date created: 2023-03-08 19:35
---

## Spring BeanFactory 容器

- 最简单的容器，给 DI 提供了基本的支持
- 用 org.springframework.beans.factory.BeanFactory 接口来定义。
- BeanFactory 或者相关的接口，如 BeanFactoryAware，InitializingBean，DisposableBean，主要用于 Spring 整合的第三方框架反向兼容。

### BeanFactory API使用场景

1. BeanFactory接口设计了**getBean**方法，这个方法是**使用IoC容器API的主要方法**，通过这个方法，可以**取得IoC容器中管理的Bean**，Bean的取得是**通过指定名字**来**索引**的。

2. 如果需要在获取Bean时对Bean进行**类型检查**，BeanFactory接口定义了带有参数的getBean方法，这个方法的使用与不带参数的getBean方法类似，不同的是增加了对Bean的类型的要求。

3. 用户可以通过BeanFactory接口方法中的getBean来使用Bean名字，从而在获取Bean时，如果需要获取的Bean是prototype类型的，用户还可以为这个prototype类型的Bean生成指定构造函数的对应参数。这使得在一定程度上可以控制生成prototype类型的Bean。有了BeanFactory的定义，用户可以执行以下操作。

- 通过接口方法containsBean让用户能够判断容器是否含有指定名字的Bean。
- 通过接口方法isSingleton来查询指定名字的Bean是否是Singleton类型的Bean。对于Singleton属性，用户可以在BeanDefinition中指定。
- 通过接口方法isPrototype来查询指定名字的Bean是否是prototype类型的。与Singleton属性一样，这个属性也可以由用户在BeanDefinition中指定。
- 通过接口方法isTypeMatch来查询指定了名字的Bean的Class类型是否是特定的Class类型。这个Class类型可以由用户来指定。
- 通过接口方法getType来查询指定名字的Bean的Class类型。
- 通过接口方法getAliases来查询指定了名字的Bean的所有别名，这些别名都是用户在BeanDefinition中定义的。

这些接口方法勾画出了IoC容器的基本特性，定义了IoC容器。

可以看到，这里定义的只是一系列的接口方法，通过这一系列的BeanFactory接口，可以使用不同的Bean检索方法，很方便从IoC容器中得到需要的Bean，从而忽略具体的IoC容器的实现。从这个角度上看，这些检索方法代表的是最为基本的容器入口。

## Spring ApplicationContext 容器

- 该容器添加了更多的企业特定的功能，例如从一个属性文件中解析文本信息的能力，发布应用程序事件给感兴趣的事件监听器的能力。
- 该容器是由 org.springframework.context.ApplicationContext 接口定义。

最常使用的 ApplicationContext 接口实现：

- FileSystemXmlApplicationContext：该容器从 XML 文件中加载已被定义的 bean。在这里，你需要提供给构造器 XML 文件的完整路径。
- ClassPathXmlApplicationContext：该容器从 XML 文件中加载已被定义的 bean。在这里，你不需要提供 XML 文件的完整路径，只需正确配置 CLASSPATH 环境变量即可，因为，容器会从 CLASSPATH 中搜索 bean 配置文件。
- WebXmlApplicationContext：该容器会在一个 web 应用程序的范围内加载在 XML 文件中已被定义的 bean。

## Bean 作用域

注意，如果你使用 web-aware ApplicationContext 时，其中三个是可用的。

| 作用域         | 描述                                                                                                 |
| -------------- | ---------------------------------------------------------------------------------------------------- |
| singleton      | 在 spring IoC 容器仅存在一个 Bean 实例，Bean 以单例方式存在，默认值                                  |
| prototype      | 每次从容器中调用 Bean 时，都返回一个新的实例，即每次调用 getBean()时，相当于执行 newXxxBean()        |
| request        | 每次 HTTP 请求都会创建一个新的 Bean，该作用域仅适用于 WebApplicationContext 环境                     |
| session        | 同一个 HTTP Session 共享一个 Bean，不同 Session 使用不同的 Bean，仅适用于 WebApplicationContext 环境 |
| global-session | 一般用于 Portlet 应用环境，该作用域仅适用于 WebApplicationContext 环境                               |
