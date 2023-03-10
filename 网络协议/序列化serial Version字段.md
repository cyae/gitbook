阿里巴巴开发手册中，第四章 OOP 规约的第 13 条解释如下：

> 【强制】序列化类新增属性时，请不要修改 serialVersionUID 字段，避免反序列失败；如果 完全不兼容升级，避免反序列化混乱，那么请修改 serialVersionUID 值。说明：注意 serialVersionUID 不一致会抛出序列化运行时异常。

简单来说，序列化就是把不适于存储或传输的数据，转化为另一种形式的数据，使得数据能够得以保存或传输。相对的，反序列化就是将数据形式转化这个过程逆向进行。

## 例子

比如说 Java 对象，就可以序列化为 JSON，或者是 Byte，也可以是我们的自定义形式，比如 key-value 形式，如下图。

![Java序列化与反序列化](https://img2020.cnblogs.com/blog/1902696/202003/1902696-20200325123432193-194512546.png)

在 Java 中，默认提供了一种序列化方式。就是对应类实现 java.io.Serializable 接口，就可以做到序列化和反序列化。下面分别是实现序列化的类 User 和测试类 SerializerTest。

```java
public class User implements java.io.Serializable {
    private String name;
    public User(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return name;
    }
}
```

```java
public class SerializerTest {
    public static void main(String[] args) throws Exception {
        User user = new User("LeoSyn");
        FileOutputStream fo = new FileOutputStream("user.bytes");
        ObjectOutputStream so = new ObjectOutputStream(fo);
        so.writeObject(user);
        so.close();
        FileInputStream fi = new FileInputStream("user.bytes");
        ObjectInputStream si = new ObjectInputStream(fi);
        user = (User) si.readObject();
        System.out.println(user);
        si.close();
    }
}
```

实现了 Serializable 接口的 User 类通过 ObjectOutputStream 转化为字节码存入 user.bytes，再使用 ObjectInputStream 把字节码从 user.bytes 读入内存。

![Java序列化与反序列化例子](https://img2020.cnblogs.com/blog/1902696/202003/1902696-20200325123501744-194934628.png)

---

在 Java 中，类的 serialVersionUID 用于验证序列化和反序列化的类的版本是否一致。为何不能轻易修改 serialVersionUID？调整一下例子，再测试一次。

## 例子

首先使用 UserSerializeTest 类对上一节例子中的 User 类进行序列化，存储到 user.bytes 中。

```java
public class UserSerializeTest {
    public static void main(String[] args) throws Exception {
        User user = new User("LeoSyn");
        FileOutputStream fo = new FileOutputStream("user.bytes");
        ObjectOutputStream so = new ObjectOutputStream(fo);
        so.writeObject(user);
        so.close();
    }
}
```

随后对 User 类进行修改，增加一个新的变量 desc。使用 UserDeserializeTest 类对 user.bytes 进行反序列化，生成 User 类对象。

```java
public class UserDeserializeTest {
    public static void main(String[] args) throws Exception {
        FileInputStream fi = new FileInputStream("user.bytes");
        ObjectInputStream si = new ObjectInputStream(fi);
        User user = (User) si.readObject();
        System.out.println(user);
        si.close();
    }
}
```

```java
public class User implements java.io.Serializable {
    private String name;
    private String desc;
    public User(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return name;
    }
}
```

执行结果如下。报错显示 serialVersionUID 不同，反序列化失败。但是代码中并没有定义 serialVersionUID，原理是什么呢？

```mipsasm
Exception in thread "main" java.io.InvalidClassException: com.serializationTest.User;
local class incompatible: stream classdesc serialVersionUID = 6360520658036414457,
local class serialVersionUID = 5259168347869896042
        at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:699)
        at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1885)
        at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1751)
        at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2042)
        at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1573)
        at java.io.ObjectInputStream.readObject(ObjectInputStream.java:431)
        at com.serializationTest.UserDeserializeTest.main(UserDeserializeTest.java:10)
```

## 源码解析

在 java.io.ObjectStreamClass#writeNonProxy 中，如果当前类没有定义 serialVersionUID，就会调用 java.io.ObjectStreamClass#computeDefaultSUID 生成默认的序列化唯一标识。代码中生成唯一标识的规则是根据类名，接口名，方法和属性等参数生成的 hash 值，所以给 User 添加了 desc 属性，对应的 serialVersionUID 肯定会变化。

![Java序列化与反序列化例子_报错](https://img2020.cnblogs.com/blog/1902696/202003/1902696-20200325125345035-1803624156.jpg)

JVM 规范里也有具体的解释：

> The stream-unique identifier is a 64-bit hash of the class name, interface class names, methods, and fields.

## 修改方案

在上一节例子场景中，只要给 User 类定义一个 serialVersionUID，即使在序列化后对 User 类进行修改，再进行反序列化，也可以成功执行代码。代码如下：

```java
public class User implements java.io.Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    public User(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return name;
    }
}
```

![Java序列化与反序列化例子_修改](https://img2020.cnblogs.com/blog/1902696/202003/1902696-20200325125408212-1384032993.jpg)
