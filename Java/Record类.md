我们先举一个简单例子，声明一个用户 Record。

```java
public record User(long id, String name, int age) {}
```

这样编写代码之后，Record 类默认包含的元素和方法实现包括：

1. record 头指定的组成元素（`int id, String name, int age`），并且，这些元素都是 final 的。
2. record 默认只有一个构造器，是包含所有元素的构造器。
3. record 的每个元素都有一个对应的 getter（但这种 getter 并不是 getxxx()，而是直接用变量名命名，所以使用序列化框架，DAO 框架都要注意这一点）
4. 实现好的 hashCode()，equals()，toString() 方法（通过自动在编译阶段生成关于 hashCode()，equals()，toString() 方法实现的字节码实现）。

我们来使用下这个 Record ：

```java
User zhx = new User(1, "zhx", 29);
User ttj = new User(2, "ttj", 25);
System.out.println(zhx.id());//1
System.out.println(zhx.name());//zhx
System.out.println(zhx.age());//29
System.out.println(zhx.equals(ttj));//false
System.out.println(zhx.toString());//User[id=1, name=zhx, age=29]
System.out.println(zhx.hashCode());//3739156
```

## Record 的结构是如何实现的

### 编译后插入相关域与方法的字节码

查看上面举得例子的字节码，有两种方式，一是通过  `javap -v User.class`  命令查看文字版的字节码，截取重要的字节码如下所示：

```undefined
//省略文件头，文件常量池部分
{
  //public 构造器，全部属性作为参数，并给每个 Field 赋值
  public com.github.hashzhang.basetest.User(long, java.lang.String, int);
    descriptor: (JLjava/lang/String;I)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=3, locals=5, args_size=4
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Record."<init>":()V
         4: aload_0
         5: lload_1
         6: putfield      #7                  // Field id:J
         9: aload_0
        10: aload_3
        11: putfield      #13                 // Field name:Ljava/lang/String;
        14: aload_0
        15: iload         4
        17: putfield      #17                 // Field age:I
        20: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      21     0  this   Lcom/github/hashzhang/basetest/User;
            0      21     1    id   J
            0      21     3  name   Ljava/lang/String;
            0      21     4   age   I
    MethodParameters:
      Name                           Flags
      id
      name
      age

  //public final 修饰的 toString 方法
  public final java.lang.String toString();
    descriptor: ()Ljava/lang/String;
    flags: (0x0011) ACC_PUBLIC, ACC_FINAL
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         //核心实现是这个 invokedynamic，我们后面会分析
         1: invokedynamic #21,  0             // InvokeDynamic #0:toString:(Lcom/github/hashzhang/basetest/User;)Ljava/lang/String;
         6: areturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/github/hashzhang/basetest/User;
  //public final 修饰的 hashCode 方法
  public final int hashCode();
    descriptor: ()I
    flags: (0x0011) ACC_PUBLIC, ACC_FINAL
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         //核心实现是这个 invokedynamic，我们后面会分析
         1: invokedynamic #25,  0             // InvokeDynamic #0:hashCode:(Lcom/github/hashzhang/basetest/User;)I
         6: ireturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/github/hashzhang/basetest/User;
  //public final 修饰的 equals 方法
  public final boolean equals(java.lang.Object);
    descriptor: (Ljava/lang/Object;)Z
    flags: (0x0011) ACC_PUBLIC, ACC_FINAL
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         //核心实现是这个 invokedynamic，我们后面会分析
         2: invokedynamic #29,  0             // InvokeDynamic #0:equals:(Lcom/github/hashzhang/basetest/User;Ljava/lang/Object;)Z
         7: ireturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0  this   Lcom/github/hashzhang/basetest/User;
            0       8     1     o   Ljava/lang/Object;
  //public 修饰的 id 的 getter
  public long id();
    descriptor: ()J
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #7                  // Field id:J
         4: lreturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/github/hashzhang/basetest/User;
  //public 修饰的 name 的 getter
  public java.lang.String name();
    descriptor: ()Ljava/lang/String;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #13                 // Field name:Ljava/lang/String;
         4: areturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/github/hashzhang/basetest/User;
  //public 修饰的 age 的 getter
  public int age();
    descriptor: ()I
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #17                 // Field age:I
         4: ireturn
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/github/hashzhang/basetest/User;
}
SourceFile: "User.java"
Record:
  long id;
    descriptor: J

  java.lang.String name;
    descriptor: Ljava/lang/String;

  int age;
    descriptor: I

//以下是 invokedynamic 会调用的方法以及参数信息，我们后面会分析
BootstrapMethods:
  0: #50 REF_invokeStatic java/lang/runtime/ObjectMethods.bootstrap:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/TypeDescriptor;Ljava/lang/Class;Ljava/lang/String;[Ljava/lang/invoke/MethodHandle;)Ljava
/lang/Object;
    Method arguments:
      #8 com/github/hashzhang/basetest/User
      #57 id;name;age
      #59 REF_getField com/github/hashzhang/basetest/User.id:J
      #60 REF_getField com/github/hashzhang/basetest/User.name:Ljava/lang/String;
      #61 REF_getField com/github/hashzhang/basetest/User.age:I
InnerClasses:
  public static final #67= #63 of #65;    // Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
```

另一种是通过 IDE 的 jclasslib 插件查看，我推荐使用这种方法，查看效果如下：

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_01.png)

## 自动生成的 private final field

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_02.png)

## 自动生成的全属性构造器

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_03.png)

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_04.png)

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_05.png)

## 自动生成的 public getter 方法

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_06.png)

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_07.png)

## 自动生成的 hashCode()，equals()，toString() 方法

这些方法的核心就是  **invokedynamic**：

![](https://zhxhash-blog.oss-cn-beijing.aliyuncs.com/%E5%AE%9E%E6%88%98%20Java%2016%20%E5%80%BC%E7%B1%BB%E5%9E%8B%20Record/1.%20Record%20%E7%9A%84%E9%BB%98%E8%AE%A4%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8%E4%BB%A5%E5%8F%8A%E5%9F%BA%E4%BA%8E%E9%A2%84%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9B%B8%E5%85%B3%E5%AD%97%E8%8A%82%E7%A0%81%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/User_jclasslib_08.png)

看上去**貌似是调用另外一个方法**，这种间接调用**难道没有性能损耗问题么**？这一点 JVM 开发者已经想到了。我们先来来了解下  **invokedynamic**。

### invokedynamic 产生的背景

Java 最早是一种静态类型语言，也就是说它的类型检查的主体过程主要是在编译期而不是运行期。为了兼容动态类型语法，也是为了 JVM 能够兼容动态语言（JVM 设计初衷并不是只能运行 Java），在 Java 7 引入了字节码指令  **invokedynamic**。这也是后来 Java 8 的拉姆达表达式以及 var 语法的实现基础。

### invokedynamic 与 MethodHandle

invokedynamic 离不开对 java.lang.invoke 包的使用。这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这种方式以外，提供一种新的动态确定目标方法的机制，称为`MethodHandle`。

通过  `MethodHandle`  可以动态获取想调用的方法进行调用，和 Java `Reflection`  反射类似，但是为了追求性能效率，需要用  `MethodHandle`，主要原因是： `Reflection`  仅仅是 Java 语言上补充针对反射的实现，并没有考虑效率的问题，**尤其是 JIT 基本无法针对这种反射调用进行有效的优化**。`MethodHandle`  更是像是对于字节码的方法指令调用的模拟，适当使用的话 JIT 也能对于它进行优化，例如将  `MethodHandle`  相关方法引用声明为 static final 的：

```java
private static final MutableCallSite callSite = new MutableCallSite(
        MethodType.methodType(int.class, int.class, int.class));

private static final MethodHandle invoker = callSite.dynamicInvoker();
```

### 自动生成的 toString(), hashcode(), equals() 的实现

通过字节码可以看出 incokedynamic 实际调用的是  `BoostrapMethods`  中的 #0 方法：

```basic
0 aload_0
1 invokedynamic #24 <hashCode, BootstrapMethods #0>
6 ireturn
```

Bootstap 方法表包括：

```php
BootstrapMethods:
  //调用的实际是 java.lang.runtime.ObjectMethods 的 boostrap 方法
  0: #50 REF_invokeStatic java/lang/runtime/ObjectMethods.bootstrap:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/TypeDescriptor;Ljava/lang/Class;Ljava/lang/String;[Ljava/lang/invoke/MethodHandle;)Ljava
/lang/Object;
    Method arguments:
      #8 com/github/hashzhang/basetest/User
      #57 id;name;age
      #59 REF_getField com/github/hashzhang/basetest/User.id:J
      #60 REF_getField com/github/hashzhang/basetest/User.name:Ljava/lang/String;
      #61 REF_getField com/github/hashzhang/basetest/User.age:I
InnerClasses:
  //声明 MethodHandles.Lookup 为 final，加快调用性能，这样调用 BootstrapMethods 里面的方法可以实现近似于直接调用的性能
  public static final #67= #63 of #65;    // Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles

```

从这里，我们就能看出，实际上 toString() 调用的是  `java.lang.runtime.ObjectMethods`  的  `bootstap()`  方法。其核心代码是：  
[`ObjectMethods.java`](https://github.com/openjdk/jdk/blob/jdk-16+36/src/java.base/share/classes/java/lang/runtime/ObjectMethods.java)

```java
public static Object bootstrap(MethodHandles.Lookup lookup, String methodName, TypeDescriptor type,
                                   Class<?> recordClass,
                                   String names,
                                   MethodHandle... getters) throws Throwable {
        MethodType methodType;
        if (type instanceof MethodType)
            methodType = (MethodType) type;
        else {
            methodType = null;
            if (!MethodHandle.class.equals(type))
                throw new IllegalArgumentException(type.toString());
        }
        List<MethodHandle> getterList = List.of(getters);
        MethodHandle handle;
        //根据 method 名称，处理对应的逻辑，分别对应了 equals()，hashCode()，toString() 的实现
        switch (methodName) {
            case "equals":
                if (methodType != null && !methodType.equals(MethodType.methodType(boolean.class, recordClass, Object.class)))
                    throw new IllegalArgumentException("Bad method type: " + methodType);
                handle = makeEquals(recordClass, getterList);
                return methodType != null ? new ConstantCallSite(handle) : handle;
            case "hashCode":
                if (methodType != null && !methodType.equals(MethodType.methodType(int.class, recordClass)))
                    throw new IllegalArgumentException("Bad method type: " + methodType);
                handle = makeHashCode(recordClass, getterList);
                return methodType != null ? new ConstantCallSite(handle) : handle;
            case "toString":
                if (methodType != null && !methodType.equals(MethodType.methodType(String.class, recordClass)))
                    throw new IllegalArgumentException("Bad method type: " + methodType);
                List<String> nameList = "".equals(names) ? List.of() : List.of(names.split(";"));
                if (nameList.size() != getterList.size())
                    throw new IllegalArgumentException("Name list and accessor list do not match");
                handle = makeToString(recordClass, getterList, nameList);
                return methodType != null ? new ConstantCallSite(handle) : handle;
            default:
                throw new IllegalArgumentException(methodName);
        }
    }
```

其中，toString() 方法 的核心实现逻辑，就要看`case "toString"`  这一分支了，核心逻辑是`makeToString(recordClass, getterList, nameList)`：

```java
private static MethodHandle makeToString(Class<?> receiverClass,
                                            //所有的 getter 方法
                                            List<MethodHandle> getters,
                                            //所有的 field 名称
                                            List<String> names) {
    assert getters.size() == names.size();
    int[] invArgs = new int[getters.size()];
    Arrays.fill(invArgs, 0);
    MethodHandle[] filters = new MethodHandle[getters.size()];
    StringBuilder sb = new StringBuilder();
    //先拼接类名称[
    sb.append(receiverClass.getSimpleName()).append("[");
    for (int i=0; i<getters.size(); i++) {
        MethodHandle getter = getters.get(i); // (R)T
        MethodHandle stringify = stringifier(getter.type().returnType()); // (T)String
        MethodHandle stringifyThisField = MethodHandles.filterArguments(stringify, 0, getter);    // (R)String
        filters[i] = stringifyThisField;
        //之后拼接 field 名称=值
        sb.append(names.get(i)).append("=%s");
        if (i != getters.size() - 1)
            sb.append(", ");
    }
    sb.append(']');
    String formatString = sb.toString();
    MethodHandle formatter = MethodHandles.insertArguments(STRING_FORMAT, 0, formatString)
                                          .asCollector(String[].class, getters.size()); // (R*)String
    if (getters.size() == 0) {
        // Add back extra R
        formatter = MethodHandles.dropArguments(formatter, 0, receiverClass);
    }
    else {
        MethodHandle filtered = MethodHandles.filterArguments(formatter, 0, filters);
        formatter = MethodHandles.permuteArguments(filtered, MethodType.methodType(String.class, receiverClass), invArgs);
    }
    return formatter;
}
```

同理，`hashcode()`  实现是：

```java
private static MethodHandle makeHashCode(Class<?> receiverClass,
                                            List<MethodHandle> getters) {
    MethodHandle accumulator = MethodHandles.dropArguments(ZERO, 0, receiverClass); // (R)I

    // 对于每一个 field，找到对应的 hashcode 方法，取 哈希值，最后组合在一起
    for (MethodHandle getter : getters) {
        MethodHandle hasher = hasher(getter.type().returnType()); // (T)I
        MethodHandle hashThisField = MethodHandles.filterArguments(hasher, 0, getter);    // (R)I
        MethodHandle combineHashes = MethodHandles.filterArguments(HASH_COMBINER, 0, accumulator, hashThisField); // (RR)I
        accumulator = MethodHandles.permuteArguments(combineHashes, accumulator.type(), 0, 0); // adapt (R)I to (RR)I
    }

    return accumulator;
}
```

同理，`equals()`  实现是：

```java
private static MethodHandle makeEquals(Class<?> receiverClass,
                                          List<MethodHandle> getters) {
        MethodType rr = MethodType.methodType(boolean.class, receiverClass, receiverClass);
        MethodType ro = MethodType.methodType(boolean.class, receiverClass, Object.class);
        MethodHandle instanceFalse = MethodHandles.dropArguments(FALSE, 0, receiverClass, Object.class); // (RO)Z
        MethodHandle instanceTrue = MethodHandles.dropArguments(TRUE, 0, receiverClass, Object.class); // (RO)Z
        MethodHandle isSameObject = OBJECT_EQ.asType(ro); // (RO)Z
        MethodHandle isInstance = MethodHandles.dropArguments(CLASS_IS_INSTANCE.bindTo(receiverClass), 0, receiverClass); // (RO)Z
        MethodHandle accumulator = MethodHandles.dropArguments(TRUE, 0, receiverClass, receiverClass); // (RR)Z
        //对比两个对象的每个 field 的 getter 获取的值是否一样，对于引用类型通过 Objects.equals 方法，对于原始类型直接通过 ==
        for (MethodHandle getter : getters) {
            MethodHandle equalator = equalator(getter.type().returnType()); // (TT)Z
            MethodHandle thisFieldEqual = MethodHandles.filterArguments(equalator, 0, getter, getter); // (RR)Z
            accumulator = MethodHandles.guardWithTest(thisFieldEqual, accumulator, instanceFalse.asType(rr));
        }

        return MethodHandles.guardWithTest(isSameObject,
                                           instanceTrue,
                                           MethodHandles.guardWithTest(isInstance, accumulator.asType(ro), instanceFalse));
    }
```

## 用法

### 声明一个 Record

Record 可以**单独作为一个文件的顶级类**，即:  
User.java 文件：

```Java
public record User(long id, String name, int age) {}
```

也可以**作为一个成员类**，即:

```Java
public class RecordTest {
    public record User(long id, String name, int age) {}
}
```

也可以**作为一个本地类**，即：

```java
public class RecordTest {
    public void test() {
        record Mail (long id, String content){}
        Mail mail = new Mail(10, "content");
    }
}
```

**不能用 abstract 修饰 Record 类**，会有编译错误。  
可以用 final 修饰 Record 类，但是这其实是没有必要的，因为  **Record 类本身就是 final 的**。

**成员 Record 类，还有本地 Record 类，本身就是 static 的**，也可以用 static 修饰，但是没有必要。

和普通类一样，Record 类可以被 public, protected, private 修饰，也可以不带这些修饰，这样就是 package-private 的。

和一般类不同的是，Record 类的直接父类不是  `java.lang.Object`  而是  **`java.lang.Record`**。但是，**Record 类不能使用 extends**，因为 Record 类不能继承任何类。

### Record 类的属性

一般，**在 Record 类声明头部指定这个 Record 类有哪些属性**：

```Java
public record User(long id, String name, int age) {}
```

同时，可以在头部的属性列表中运用注解：

```Java
@Target({ ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
public @interface A {}
@Target({ ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface B {}

public record User(@A @B long id, String name, int age) {}
```

但是，需要注意一点，这里通过反射获取 id 的注解的时候，需要通过对应的方式进行获取，否则获取不到，即  `ElementType.FIELD`  通过  `Field`  获取，`ElementType.RECORD_COMPONENT`  通过  `RecordComponent`  获取:

```Java
Field[] fields = User.class.getDeclaredFields();
Annotation[] annotations = fields[0].getAnnotations(); // 获取到注解 @B

RecordComponent[] recordComponents = User.class.getRecordComponents();
annotations = recordComponents[0].getAnnotations(); // 获取到注解 @A
```

### Record 类体

Record 类属性必须在头部声明，**在 Record 类体只能声明静态属性**：

```java
public record User(long id, String name, int age) {
    static long anotherId;
}
```

Record 类体可以声明成员方法和静态方法，和一般类一样。但是**不能声明 abstract 或者 native 方法**：

```java
public record User(long id, String name, int age) {
    public void test(){}
    public static void test2(){}
}
```

Record 类体也不能包含实例初始化块，例如：

```java
public record User(@A @B long id, String name, int age) {
    {
        System.out.println(); //编译异常
    }
}
```

### Record 成员

Record 的所有成员属性，都是  **public final 非 static 的**，对于每一个属性，**都有一个对应的无参数返回类型为属性类型方法名称为属性名称的方法，即这个属性的 accessor**。前面说这个方法是 getter 方法其实不太准确，因为方法名称中并没有 get 或者 is 而是只是纯属性名称作为方法名。

这个方法如果我们自己指定了，就不会自动生成：

```java
public record User(long id) {
    @Override
    public long id() {
        return id;
    }
}
```

如果没有自己指定，则会自动生成这样一个方法:

1. **方法名就是属性名称**
2. **返回类型就是对应的属性类型**
3. **是一个 public 方法，并且没有声明抛出任何异常**
4. **方法体就是返回对应属性**
5. **如果属性上面有任何注解，那么这个注解如果能加到方法上那么也会自动加到这个方法上**。例如：

```java
public record User(@A @B long id, String name, int age) {}
@Target({
        ElementType.RECORD_COMPONENT,
        ElementType.METHOD,
})
@Retention(RetentionPolicy.RUNTIME)
public @interface A {}
@Target({ ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface B {}
```

下面获取  `id()`  这个方法的注解则会获取到注解  `@A`

```java
Method id = User.class.getDeclaredMethod("id");
Annotation[] idAnnotations = id.getAnnotations(); //@A
```

由于会自动生成这些方法，所以  **Record 成员的名称不能和 Object 的某些不符合上述条件（即上面提到的 6 条）的方法的名称一样**，例如：

```java
public record User(
    int wait, //编译错误
    int hashcode, //这个不会有错误，因为 hashcode() 方法符合自动生成的 accessor 的限制条件。
    int toString, //编译错误
    int finalize //编译错误) {
}
```

Record 类如果没有指定，则默认会生成实现  `java.lang.Record`  的抽象方法，即`hashcode()`, `equals()`, `toStrng()`  这三个方法。这**三个方法的实现方式，在第一节已经详细分析过**，这里简单回顾下要点：

1. `hashcode()`  在编译的时候自动生成字节码实现，核心逻辑基于  `ObjectMethods`  的  `makeHashCode`  方法，里面的逻辑是对于每一个属性的哈希值移位组合起来。注意这里的所有调用（包括对于  `ObjectMethods`  的方法调用以及获取每个属性）都是利用  `MethodHandle`  实现的近似于直接调用的方式调用的。
2. `equals()`  在编译的时候自动生成字节码实现，核心逻辑基于  `ObjectMethods`  的  `makeEquals`  方法，里面的逻辑是对于两个 Record 对象每一个属性判断是否相等（对于引用类型用`Objects.equals()`，原始类型使用 `==`），注意这里的所有调用（包括对于  `ObjectMethods`  的方法调用以及获取每个属性）都是利用  `MethodHandle`  实现的近似于直接调用的方式调用的。
3. `toString()`  在编译的时候自动生成字节码实现，核心逻辑基于  `ObjectMethods`  的  `makeToString`  方法，里面的逻辑是对于每一个属性的值组合起来构成字符串。注意这里的所有调用（包括对于  `ObjectMethods`  的方法调用以及获取每个属性）都是利用  `MethodHandle`  实现的近似于直接调用的方式调用的。

### Record 构造器

如果没有指定构造器，Record 类会自动生成一个以所有属性为参数并且给每个属性赋值的构造器。我们可以通过两种方式自己声明构造器：

**第一种是明确声明以所有属性为参数并且给每个属性赋值的构造器**，这个构造器需要满足：

1. 构造器参数需要包括所有属性（同名同类型），并按照 Record 类头的声明顺序。
2. 不能声明抛出任何异常（不能使用 throws）
3. 不能调用其他构造器（即不能使用 this(xxx)）
4. 构造器需要对于每个属性进行赋值。

**对于其他构造器，需要明确调用这个包含所有属性的构造器**：

```java
public record User(long id, String name, int age) {
    public User(int age, long id) {
        this(id, "name", age);
    }
    public User(long id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }
}
```

**第二种是简略方式声明**，例如：

```java
public record User(long id, String name, int age) {
    public User {
        System.out.println("initialized");
        id = 1000 + id;
        name = "prefix_" + name;
        age = 1 + age;
        //在这之后，对每个属性赋值
    }
}
```

这种方式相当于省略了参数以及对于每个属性赋值，相当于对这种构造器的开头插入代码。
