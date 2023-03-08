> 一种动态的JVM监控工具

## [原理](https://blog.csdn.net/hosaos/article/details/102931887#AgentClassAttachAgent_437)

通过**代理模式**, 对字节码进行修改或增强.

随着进程启动的Premain方式的Agent更偏向是一种初始化加载时的修改方式，而Attach API的loadAgent()方法，能够将打包好的Agent jar包动态Attach到目标JVM上，是一种**运行时注入Agent**、**修改字节码**的方式

## 用途

* 想知道某个方法有没被调用到，但是忘了打日志
* 想要确定环境上的代码和本地代码是否一致，需要多次拷贝才能到本地(k8s)，再执行反编译查看
* 想知道环境**运行时动态生成的**的**环境变量**长啥样
* 日志级别调的比较高，出问题时没有日志可定位
* 想知道某个对象的状态，但是不能引发业务中断去dump
* 代码有BUG，只有打补丁重启环境，无法热修改
* lombok等三方注解字节码动态增强出来的代码到底长啥样
* 查看CPU资源使用情况, 绘制火焰图

## 样例

假设只知道某jvm上运行服务的如下信息

* 是一个Spring boot 服务，Main Class:  `com.learn.arthas.ArthasLearnMain`  
* 在8080端口提供http rest服务(通过`netstat -anp | grep <java pid>`也能查看监听端口)

0. 在主机上准备arthas的jar包, 通过kubectl cp 命令拷入docker容器

1. xhsell连接目标pod, 打开三个窗口, 分别用于:
   * 日志
   * 发送请求
   * Arthas命令行界面

2. 启动Arthas

```bash
JAVA_HOME -jar arthas/arthas-boot.jar
```

3. 使用jad命令查看MainClass反编译码

```bash
jad com.learn.arthas.ArthasLearnMain
```

4. 使用sc命令查看所有类, 模型, 接口api定义, 类加载器

```bash
sc [-d 详细信息] com.learn.*
```

5. 找到接口后, 观察接口调用情况, 注意如果线上环境的接口被频繁调用, 以下所有命令后缀-n 限制执行次数, 防止xshell被打爆

```bash
# 统计成功/失败/错误率/时延
monitor [全类名] [方法名] [-c 监听周期]

# 方法入参、出参、异常
watch [全类名] [方法名] '{[入参params, 出参returnObj, ...]}' [-x 展开层数, 如结果是标量/一维数组, 则应展开为1/2层]

# 方法调用栈, 上卷
# 注意: 有时候调用者过多且根据入参无区分度, 无法确定落脚点, 改用trace + 参数条件限制查看下钻情况, 检查异常点
stack [全类名] [方法名] [参数条件表达式]

# 方法调用链时延统计, 下钻 (会将调用瓶颈标红)
# 注意: 此命令无法查看动态生成的代理类(如AOP切面), 因此当遇到动态类时, 需要分析接口内子调用对应的cutpoint, 重新确定落脚点
# 这也要求方法不能有过多子调用, 不能有过多平凡的入参, 否则排查落脚点困难
trace [全类名] [方法名] [参数条件表达式]
```

6. 调整日志级别

```bash
# 查看当前日志级别
logger

# 修改日志级别1, 对于某些不支持的日志框架无效
logger [-c 类加载器] [--name 日志名] [--level 日志级别]

# 修改日志级别2, 通过注入实现
# 使用ognl获取logger对象
ognl [-c 类加载器] ['@全类名@日志对象名.getClass().getName()']
# 调用logger对象的设置级别方法, 如log4j
ognl [-c 类加载器] ['@全类名@日志对象名.setLevel(@org.apache.logging.log4j.Level@INFO)']
```

7. 模拟rest调用

```bash
# 思路1: 在springBoot启动类Main Class里预留ApplicationContext ctx落脚点, 拿到springIOC容器, 然后获取里面的Controller/Service/Mapper Bean, 接着使用ognl调用对应方法
# 注意: 如果预见到参数可能在未来被arthas改变, 就不要使用static final修饰, 因为这种常量参数在编译器会被优化到形参里, 即使用arthas改了实参也不起作用
ognl [-c 类加载器] '@Main Class全类名@ctx.getBean("xxxService").方法名(参数...)/属性'
```

```bash
# 思路2: 没有预留ctx, 或不是spring框架没有IOC容器, 使用时光机tt获取快照信息, 但是需要等待客户端触发接口
tt -t [全类名] [方法名]
# 针对某一快照, 查看类属性
tt [-i 快照id] [-w 查看属性/方法]
```

8. 线程繁忙度
```bash
# thread繁忙程度前三个线程
thread -n 3
```

9. 绘制火焰图, 一般在压测时绘制
```bash
# profiler火焰图
profiler start [--event 监测事件] # 默认是按cpu占用率画
profiler status # 查看状态
profiler stop [--file 路径+文件名] # 停止，会自动生成svg图，也可指定为html格式
```

10. 详细命令[列表](https://alibaba.github.io/arthas/)
![image](https://img2022.cnblogs.com/blog/2827284/202208/2827284-20220819001430482-1904590557.png)