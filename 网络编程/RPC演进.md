## 写在前面

单机应用中的方法调用很简单，直接调用就行，像这样

```java
String msg = helloService.say("Hello!");
```

因为调用方与被调用方在一个进程内

随着业务的发展，单机应用会越来越力不从心，势必会引入分布式来解决单机的问题，那么调用方如何调用另一台机器上的方法呢 ？

这就涉及到分布式通信方式，从单机走向分布式，产生了很多通信方式

![](https://img2020.cnblogs.com/blog/747662/202101/747662-20210116174526112-536414049.png)

## RPC 的演进过程

先说明一下，下文中的示例虽然是 Java 代码实现的，但原理是通用的，重点是理解其中的原理

### 第一版

两台机器之间进行交互，那么肯定离不开网络通信协议，TCP / IP 也就成了绕不开的点，所以先辈们最初想到的方法就是通过 TCP / IP 来实现远程方法的调用

而操作系统是没有直接暴露 TCP / IP 接口的，而是通过 Socket 抽象了 TCP / IP 接口，所以我们可以通过 Socket 来实现最初版的远程方法调用

完整示例代码：[rpc-01](https://gitee.com/youzhibing/rpc/tree/master/rpc-01)，核心代码如下

Server：

```java
public class Server {
    private static boolean is_running = true;

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8888);
        while (is_running) {
            System.out.println("等待 client 连接");
            Socket client = serverSocket.accept();
            System.out.println("获取到 client...");
            handle(client);
            client.close();
        }
        serverSocket.close();
    }

    private static void handle(Socket client) throws Exception {
        InputStream in = client.getInputStream();
        OutputStream out = client.getOutputStream();
        DataInputStream dis = new DataInputStream(in);
        DataOutputStream dos = new DataOutputStream(out);

        // 从 socket 读取参数
        int id = dis.readInt();
        System.out.println("id = " + id);

        // 查询本地数据
        IUserService userService = new UserServiceImpl();
        User user = userService.getUserById(id);

        // 往 socket 写响应值
        dos.writeInt(user.getId());
        dos.writeUTF(user.getName());
        dos.flush();

        dis.close();
        dos.close();
    }
}
```

Client：

```java
public class Client {

    public static void main(String[] args) throws Exception {
        Socket s = new Socket("127.0.0.1", 8888);

        // 网络传输数据
        // 往 socket 写请求参数
        DataOutputStream dos = new DataOutputStream(s.getOutputStream());
        dos.writeInt(18);
        // 从 socket 读响应值
        DataInputStream  dis = new DataInputStream(s.getInputStream());
        int id = dis.readInt();
        String name = dis.readUTF();
        // 将响应值封装成 User 对象
        User user = new User(id, name);
        dos.close();
        dis.close();
        s.close();

        // 进行业务处理
        System.out.println(user);
    }
}
```

代码很简单，就是一个简单的 Socket 通信；如果看不懂，那就需要去补充下 Socket 和 IO 的知识

测试结果如下

![](https://img2020.cnblogs.com/blog/747662/202101/747662-20210117094821285-494811440.gif)

可以看到 Client 与 Server 之间是可以进行通信的；但是，这种方式非常麻烦，有太多缺点，最明显的一个就是

- Client 端业务代码 与 网络传输代码 混合在一起，没有明确的模块划分
- 如果有多个开发者同时进行 Client 开发，那么他们都需要知道 Socket、IO

### 第二版

针对第一版的缺点，演进出了这一版，引进  Stub （早期的叫法，不用深究，理解成 sidecar 代理就行）实现 Client 端网络传输代码的封装

完整示例代码：[rpc-02](https://gitee.com/youzhibing/rpc/tree/master/rpc-02)，改动部分如下

Stub：

```java
public class Stub {

    public User getUserById(Integer id) throws Exception {
        Socket s = new Socket("127.0.0.1", 8888);

        // 网络传输数据
        // 往 socket 写请求参数
        DataOutputStream dos = new DataOutputStream(s.getOutputStream());
        dos.writeInt(id);
        // 从 socket 读响应值
        DataInputStream dis = new DataInputStream(s.getInputStream());
        int userId = dis.readInt();
        String name = dis.readUTF();
        // 将响应值封装成 User 对象
        User user = new User(userId, name);
        dos.close();
        dis.close();
        s.close();

        return user;
    }
}
```

Client：

```java
public class Client {

    public static void main(String[] args) throws Exception {

        // 不再关注网络传输
        Stub stub = new Stub();
        User user = stub.getUserById(18);

        // 进行业务处理
        System.out.println(user);
    }
}
```

Client 不再关注网络数据传输，一心关注业务代码就好

有小伙伴可能就杠上了：这不就是把网络传输代码移了个位置嘛，这也算改进？

迭代开发是一个逐步完善的过程，而这也算是一个改进哦

但这一版还是有很多缺点，最明显的一个就是

- Stub 只能代理  IUserService  的一个方法  getUserById ，局限性太大，不够通用
- 如果想在  IUserService  新增一个方法： getUserByName ，那么需要在 Stub 中新增对应的方法，Server 端也需要做对应的修改来支持

### 第三版

第二版中的  Stub  代理功能太弱了，那有没有什么方式可以增强 Stub 的代理功能了？

前面的 Stub 相当于是一个**静态代理**，所以功能有限，那静态代理的增强版是什么了，没错，就是：**动态代理**

不熟悉动态代理的小伙伴，一定要先弄懂动态代理：[设计模式之代理，手动实现动态代理，揭秘原理实现](https://www.cnblogs.com/youzhibing/p/10464274.html)

JDK 有动态代理的 API，我们就用它来实现

完整示例代码：[rpc-03](https://gitee.com/youzhibing/rpc/tree/master/rpc-03)，相较于第二版，改动比较大，大家需要仔细看

Server：

```java
public class Server {
    private static boolean is_running = true;

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8888);
        while (is_running) {
            System.out.println("等待 client 连接");
            Socket client = serverSocket.accept();
            System.out.println("获取到 client...");
            handle(client);
            client.close();
        }
        serverSocket.close();
    }

    private static void handle(Socket client) throws Exception {
        InputStream in = client.getInputStream();
        OutputStream out = client.getOutputStream();
        ObjectInputStream ois = new ObjectInputStream(in);
        ObjectOutputStream oos = new ObjectOutputStream(out);

        // 获取方法名、方法的参数类型、方法的参数值
        String methodName = ois.readUTF();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        IUserService userService = new UserServiceImpl();
        Method method = userService.getClass().getMethod(methodName, parameterTypes);
        User user = (User) method.invoke(userService, args);

        // 往 socket 写响应值；直接写可序列化对象（实现 Serializable 接口）
        oos.writeObject(user);
        oos.flush();

        ois.close();
        oos.close();
    }
}
```

Stub：

```java
public class Stub {

    public static IUserService getStub() {
        Object obj = Proxy.newProxyInstance(IUserService.class.getClassLoader(),
                new Class[]{IUserService.class}, new NetInvocationHandler());
        return (IUserService)obj;
    }

    static class NetInvocationHandler implements InvocationHandler {

        /**
         *
         * @param proxy
         * @param method
         * @param args
         * @return
         * @throws Throwable
         */
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Socket s = new Socket("127.0.0.1", 8888);

            // 网络传输数据
            ObjectOutputStream oos = new ObjectOutputStream(s.getOutputStream());
            // 传输方法名、方法参数类型、方法参数值；可能会有方法重载，所以要传参数列表
            oos.writeUTF(method.getName());
            Class[] parameterTypes = method.getParameterTypes();
            oos.writeObject(parameterTypes);
            oos.writeObject(args);

            // 从 socket 读响应值
            ObjectInputStream ois = new ObjectInputStream(s.getInputStream());
            User user = (User) ois.readObject();

            oos.close();
            ois.close();
            s.close();

            return user;
        }
    }
}
```

Client：

```java
public class Client {

    public static void main(String[] args) throws Exception {

        IUserService userService = Stub.getStub();
        //User user = userService.getUserById(23);

        User user = userService.getUserByName("李小龙");
        // 进行业务处理
        System.out.println(user);
    }
}
```

我们来看下效果

![](https://img2020.cnblogs.com/blog/747662/202101/747662-20210117142138970-1651969384.gif)

此时， IUserService  接口的方法都能被代理了，即使它新增接口， Stub  不用做任何修改也能代理上

另外， Server  端的响应值改成了对象，而不是单个属性逐一返回，那么无论  User  是新增属性，还是删减属性，Client 和 Server 都不受影响了

这一版的改进是非常大的进步；但还是存在比较明显的缺点

- 只支持  IUserService ，通用性还是不够完美
- 如果新引进了一个  IPersonService ，那怎么办 ？

### 第四版

第三版相当于 Client 与 Server 端约定好了，只进行 User 服务的交互，所以 User 之外的服务，两边是通信不上的

如果还需要进行其他服务的交互，那么 Client 就需要**将请求的服务名作为参数**传递给 Server，告诉 Server 我需要和哪个服务进行交互

所以，Client 和 Server 都需要进行改造

完整示例代码：[rpc-04](https://gitee.com/youzhibing/rpc/tree/master/rpc-04)，相较于第三版，改动比较小，相信大家都能看懂

Server：

```java
public class Server {
    private static boolean is_running = true;

    private static final HashMap<String, Object> REGISTRY_MAP = new HashMap();

    public static void main(String[] args) throws Exception {

        // 向注册中心注册服务
        REGISTRY_MAP.put("com.qsl.rpc.service.IUserService", new UserServiceImpl());
        REGISTRY_MAP.put("com.qsl.rpc.service.IPersonService", new PersonServiceImpl());

        ServerSocket serverSocket = new ServerSocket(8888);
        while (is_running) {
            System.out.println("等待 client 连接");
            Socket client = serverSocket.accept();
            System.out.println("获取到 client...");
            handle(client);
            client.close();
        }
        serverSocket.close();
    }

    private static void handle(Socket client) throws Exception {
        InputStream in = client.getInputStream();
        OutputStream out = client.getOutputStream();
        ObjectInputStream ois = new ObjectInputStream(in);
        ObjectOutputStream oos = new ObjectOutputStream(out);

        // 获取服务名
        String serviceName = ois.readUTF();
        System.out.println("serviceName = " + serviceName);

        // 获取方法名、方法的参数类型、方法的参数值
        String methodName = ois.readUTF();
        Class[] parameterTypes = (Class[]) ois.readObject();
        Object[] args = (Object[]) ois.readObject();

        // 获取服务；从服务注册中心获取服务
        Object serverObject = REGISTRY_MAP.get(serviceName);

        // 通过反射调用服务的方法
        Method method = serverObject.getClass().getMethod(methodName, parameterTypes);
        Object resp = method.invoke(serverObject, args);

        // 往 socket 写响应值；直接写可序列化对象（实现 Serializable 接口）
        oos.writeObject(resp);
        oos.flush();

        ois.close();
        oos.close();
    }
}
```

Stub：

```java
public class Stub {

    public static Object getStub(Class clazz) {
        Object obj = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, new NetInvocationHandler(clazz));
        return obj;
    }

    static class NetInvocationHandler implements InvocationHandler {

        private Class clazz;

        NetInvocationHandler(Class clazz){
            this.clazz = clazz;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Socket s = new Socket("127.0.0.1", 8888);

            // 网络传输数据
            ObjectOutputStream oos = new ObjectOutputStream(s.getOutputStream());

            // 传输接口名，告诉服务端，我要哪个服务
            oos.writeUTF(clazz.getName());

            // 传输方法名、方法参数类型、方法参数值；可能会有方法重载，所以要传参数列表
            oos.writeUTF(method.getName());
            Class[] parameterTypes = method.getParameterTypes();
            oos.writeObject(parameterTypes);
            oos.writeObject(args);

            // 从 socket 读响应值
            ObjectInputStream ois = new ObjectInputStream(s.getInputStream());
            Object resp = ois.readObject();

            oos.close();
            ois.close();
            s.close();

            return resp;
        }
    }
}
```

Client：

```java
public class Client {

    public static void main(String[] args) throws Exception {

        /*IUserService userService = (IUserService)Stub.getStub(IUserService.class);
        User user = userService.getUserByName("青石路");
        System.out.println(user);*/

        IPersonService personService = (IPersonService)Stub.getStub(IPersonService.class);
        Person person = personService.getPersonByPhoneNumber("123");
        System.out.println(person);
    }
}
```

此版本抽象的比较好了，屏蔽了底层细节，支持任何服务的任意方法，算是一个比较完美的版本了

至此，一个最基础的 RPC 就已经实现了

但是，还是有大量的细节可以改善

- 序列化与反序列化
- 传输控制协议
- 其他服务治理功能

网络中数据的传输都是二进制，所以请求参数需要序列化成二进制，响应参数需要反序列化成对象

而 JDK 自带的序列化与反序列化，具有语言局限性、效率慢、序列化后的长度太长等缺点

序列化与反序列化协议非常多，常见的有

![](https://img2020.cnblogs.com/blog/747662/202101/747662-20210117185449939-165339340.png)

## 总结

### 1、RPC 的演进过程

![](https://img2020.cnblogs.com/blog/747662/202101/747662-20210118100300116-396926556.png)

### 2、RPC 的组成要素

三要素：动态代理、序列化与反序列化协议、网络通信协议

网络通信协议可以是 TCP、UDP，也可以是 HTTP 1.x、HTTP 2，甚至有能力可以是自定义协议

### 3、RPC 框架

RPC 不等同于 RPC 框架，RPC 是一个概念，是一个分布式通信方式

基于 RPC 产生了很多 RPC 框架：Dubbo、Netty、gRPC、BRPC、Thrift、JSON-RPC 等等

RPC 框架对 RPC 进行了功能丰富，包括：服务注册、服务发现、服务治理、服务监控、服务负载均衡等功能

## 参考

[36 行代码透彻解析 RPC](https://www.bilibili.com/video/av93511815)

[序列化和反序列化](https://tech.meituan.com/2015/02/26/serialization-vs-deserialization.html)

[你应该知道的 RPC 原理](https://www.cnblogs.com/LBSer/p/4853234.html)
