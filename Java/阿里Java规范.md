---
date created: 2023-03-08 19:18
---

## **1. 变量命名**

### **1.1 POJO 类中的任何布尔类型的变量，都不要加 `is` 前缀，否则部分框架解析会引起序列化错误。**

**说明**：本文 MySQL 规约中的建表约定第 1 条，表达是与否的变量采用 `is_xxx` 的命名方式，所以需要在`<resultMap>`设置从 `is_xxx` 到 `xxx` 的映射关系。

**反例**：定义为布尔类型 `Boolean isDeleted` 的字段，它的 `getter` 方法也是 `isDeleted()`，部分框架在反向解析时，“误以为”对应的字段名称是 `deleted`，导致字段获取不到，得到意料之外的结果或抛出异常。

### **1.2 包名统一使用小写，点分隔符之间有且仅有一个自然语义的英语单词。包名统一使用单数形式，但是类名如果有复数含义，类名可以使用复数形式。**

**正例**：应用工具类包名为 `com.alibaba.ei.kunlun.aap.util`；类名为 `MessageUtils`（此规则参考 spring 的框架结构）。

### **1.3 避免在子父类的成员变量之间、或者不同代码块的局部变量之间采用完全相同的命名，使可理解性降低。**

**说明**：子类、父类成员变量名相同，即使是 `public` 也是能够通过编译，而局部变量在同一方法内的不同代码块中同名也是合法的，但是要避免使用。对于非 `setter` / `getter` 的参数名称也要避免与成员变量名称相同。

**反例**：

```java
class Son extends ConfusingName {
    // 不允许与父类的成员变量名称相同
    private int stock;
}
```

```java
public class ConfusingName {
    protected int stock;
    protected String alibaba;
    // 非 setter/getter 的参数名称，不允许与本类成员变量同名
    public void access(String alibaba) {
        if (condition) {
            final int money = 666;
            // ...
        }
        for (int i = 0; i < 10; i++) {
            // 在同一方法体中，不允许与其它代码块中的 money 命名相同
            final int money = 15978;
            // ...
        }
    }
}
```

### **1.4 为了达到代码自解释的目标，任何自定义编程元素在命名时，使用完整的单词组合来表达。**

**正例**：在 JDK 中，对某个对象引用的 `volatile` 字段进行原子更新的类名为 `AtomicReferenceFieldUpdater`。

**反例**：常见的方法内变量为 `int a`; 的定义方式。

### **1.5 接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的 Javadoc 注释。尽量不要在接口里定义常量，如果一定要定义，最好确定该常量与接口的方法相关，并且是整个应用的基础常量。**

**正例**：接口方法签名 `void commit()`; 接口基础常量 `String COMPANY = "alibaba"`;

**反例**：接口方法定义 `public abstract void commit()`;

**说明**：JDK8 中接口允许有默认实现，那么这个 `default` 方法，是对所有实现类都有价值的默认实现。

### **1.6 枚举类名带上 `Enum` 后缀，枚举成员名称需要全大写，单词间用下划线隔开。**

**说明**：枚举其实就是特殊的常量类，且构造方法被默认强制是私有。

**正例**：枚举名字为 `ProcessStatusEnum` 的成员名称：`SUCCESS` / `UNKNOWN_REASON`

### **1.7 领域模型命名规约：**

1）数据对象：xxxDO，xxx 即为数据表名。\
2）数据传输对象：xxxDTO，xxx 为业务领域相关的名称。\
3）展示对象：xxxVO，xxx 一般为网页名称。\
4）POJO 是 DO / DTO / BO / VO 的统称，禁止命名成 xxxPOJO。

---

## **2. 常量命名**

### **2.1 不允许任何魔法值（即未经预先定义的常量）直接出现在代码中。**

**反例**：

```java
// 开发者 A 定义了缓存的 key。
String key = "Id#taobao_" + tradeId;
cache.put(key, value);
```

```java
// 开发者 B 使用缓存时直接复制少了下划线，即 key 是"Id#taobao" + tradeId，导致出现故障。
String key = "Id#taobao" + tradeId;
cache.get(key);
```

### **2.2 浮点数类型的数值后缀统一为大写的 `D` 或 `F`, 长整型后缀统一为的写的`L`。**

**正例**：

```java
public static final double HEIGHT = 175.5D;
public static final float WEIGHT = 150.3F;
public static final long LENGTH = 254L;
```

**反例**:

```java
public static final float WRIGHT = 150.3; // 默认为double类型, 造成隐式类型转换和传参类型错误
public static final long LENGTH = 254; // 默认为int类型, 造成隐式类型转换和传参类型错误
```

### **2.3 有限状态变量值定义为枚举类**

```java
public enum SeasonEnum {
    SPRING(1), SUMMER(2), AUTUMN(3), WINTER(4);
    private int seq;
    SeasonEnum(int seq) {
        this.seq = seq;
    }
    public int getSeq() {
        return seq;
    }
}
```

---

## **3. 代码格式**

### **3.1 在进行类型强制转换时，右括号与强制转换值之间不需要任何空格隔开。**

```java
double first = 3.2D;
int second = (int)first + 2;
```

---

## **4. OOP 规约**

### **4.1 相同参数类型，相同业务含义，才可以使用的可变参数，参数类型避免定义为 Object。**

**说明**：可变参数必须放置在参数列表的最后。（建议开发者尽量不用可变参数编程）

**正例**：

```java
public List<User> listUsers(String type, Long... ids) {...}
```

**反例**:

```java
public List<User> listUsers(Object... objs, String type);
```

### **4.2 外部正在调用的接口或者二方库依赖的接口，不允许修改方法签名，避免对接口调用方产生影响。接口过时必须加 `@Deprecated` 注解，并清晰地说明采用的新接口或者新服务是什么。**

### **4.3 不使用过时方法. 方法过时意味着原方法有更好的实现, 应使用对应新方法.**

### **4.4 Object 的 equals 方法容易抛空指针异常，应使用常量或确定有值的对象来调用 equals**

**正例**：`"test".equals(param);`

**反例**：`param.equals("test"); // param为空则报NPE`

**说明**：推荐使用 JDK7 引入的工具类 `java.util.Objects.equals(Object a, Object b)`

### **4.5 所有整型包装类对象之间值的比较，全部使用 equals 方法比较**

**说明**：对于 Integer var = ? 在 -128 至 127 之间的赋值，Integer 对象是在缓冲池 `IntegerCache.cache` 产生，会复用已有对象，这个区间内的 Integer 值可以直接使用 == 进行判断\
但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象，这是一个大坑，推荐使用 equals 方法进行判断。

### **4.6 任何货币金额，均以**_最小货币单位_**且为**_整型类型_**进行存储。**

**反例**:

```java
double deposit = 123.45D;
```

**正例**:

```java
long depositAsFen = 12345L;
```

### **4.7 涉及到浮点数作为条件的判断, 不能使用 == 和 .equals**

**反例**:

```java
float a = 1.2F;
float b = 1.1F;
if (a - b == 0.1F)

Float c = Float.valueOf(a);
Float d = Float.valueOf(b);
if (c.equals(d))
```

**正例**:\
(1) 指定误差范围 esp\
(2) 使用`BigDecimal`类的`.add`/`.subtract`/`.compareTo`方法\
(3) 使用`strictfp`关键字

### **4.8 禁止使用构造方法 `BigDecimal(double)` 的方式把 double 值转化为 BigDecimal 对象。**

**反例**:\
`BigDecimal(double)` 存在精度损失风险，在精确计算或值比较的场景中可能会导致业务逻辑异常。如：

```java
BigDecimal g = new BigDecimal(0.1F)；// 实际的存储值为：0.100000001490116119384765625
```

**正例**:\
优先推荐入参为 String 的构造方法，或使用 BigDecimal 的 `valueOf` 方法，此方法内部其实执行了 Double 的 toString，而 Double 的 toString 按 double 的实际能表达的精度对尾数进行了截断。

```java
BigDecimal recommend1 = new BigDecimal("0.1");
BigDecimal recommend2 = BigDecimal.valueOf(0.1);
```

### **4.9 定义数据对象 DO 类时，属性类型要与数据库字段类型相匹配。**

| MySQL 数据类型 | Java 实体类属性类型 | 说明                                                                            |
| -------------- | ------------------- | ------------------------------------------------------------------------------- |
| int            | Integer             | 不管是 signed 还是 unsigned，Java 实体类型都是 Integer                          |
| bigint         | Long                | 不管是 bigint(xxx)括号多少位，不管 signed 还是 unsigned，Java 实体类型都是 Long |
| varchar        | String              | -                                                                               |
| bit            | byte[]              | -                                                                               |
| tinyint        | Byte                | 不管是 signed 还是 unsigned，Java 实体类型都是 Byte，在 java.lang 包下          |
| smallint       | Short               | 不管是 signed 还是 unsigned，Java 实体类型都是 Short                            |
| char           | String              | 不管 char 是 gbk、utf8、utf8mb4 等编码类型，Java 实体类型都是 String            |
| date           | Date                | java.util.Date                                                                  |
| datetime       | Date                | java.util.Date                                                                  |
| timestamp      | Date                | java.util.Date                                                                  |
| time           | Date                | java.util.Date                                                                  |
| float          | Float               | 不管是 signed 还是 unsigned，Java 实体类型都是 Float                            |
| decimal        | BigDecimal          | -                                                                               |
| numeric        | Long                | -                                                                               |
| double         | Double              | 不管是 signed 还是 unsigned，Java 实体类型都是 Double                           |
| tinytext       | String              | -                                                                               |
| text           | String              | -                                                                               |
| year           | Date                | java.util.Date                                                                  |
| enum           | String              | -                                                                               |
| tinyint        | Boolean             | 0, 1 表示 false, true                                                           |

### **4.10 关于基本数据类型与包装数据类型的使用标准如下：**

1）所有的 POJO 类属性必须使用包装数据类型。\
2）RPC 方法的返回值和参数必须使用包装数据类型。\
3）所有的局部变量使用基本数据类型。

**说明**：POJO 类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何 NPE 问题，或者入库检查，都由使用者来保证。

**正例**：数据库的查询结果可能是 null，因为自动拆箱，用基本数据类型接收有 NPE 风险。

**反例**：某业务的交易报表上显示成交总额涨跌情况，即正负 x%，x 为基本数据类型，调用的 RPC 服务，调用不成功时，返回的是默认值，页面显示为 0%，这是不合理的，应该显示成中划线-。所以包装数据类型的 null 值，能够表示额外的信息，如：远程调用失败，异常退出。

### **4.11 定义 DO / PO / DTO / VO 等 POJO 类时，不要设定任何属性默认值。**

**反例**：某业务的 DO 的 createTime 默认值为 `new Date()`；但是这个属性在数据提取时并没有置入具体值，在更新其它字段时又附带更新了此字段，导致创建时间被修改成当前时间。

### **4.12 序列化类新增属性时，请不要修改 serialVersionUID 字段，避免反序列失败；如果完全不兼容升级，避免反序列化混乱，那么请修改 serialVersionUID 值。**

**说明**: 当一个类实现`Serializable`接口, 有两种生成可传输对象的方式\
(1) 显式指定: `private static final long serialVersionUID = 1L`;\
(2) 隐式生成: 如果没有显式指定, 在传输该对象时, 底层会调用`java.io.ObjectStreamClass#computeDefaultSUID`, 根据类名, 属性, 方法等样本信息生成 hash 值, 作为`serialVersionUID`;

- 如果序列化和反序列化的`serialVersionUID`不同, 则报

`InvalidClassException`异常

**反例**:

```java
// 如果使用隐式生成, 后续又改变了类的样本信息, 则会导致反序列化失败
public class Cat implement Serializable {
    private name;
    // getter and setter
}

// 序列化...

// 在后续版本中, 添加新属性
public class Cat implement Serializable {
    private String name;
    private Integer age;
    // getter and setter
}

// 再反序列化会失败...
```

**正例**:

```java
public class Dog implement Serializable {
    private static final long serialVersionUID = 134530430050463L;
    private String name;
    // getter and setter
}

// 序列化...

// 在后续版本中, 添加新属性
public class Dog implement Serializable {
    private static final long serialVersionUID = 134530430050463L; // 不要修改
    private String name;
    private Integer age;
    // getter and setter
}

// 再反序列化, 新增属性age会设为默认值
```

### **4.13 构造方法里面禁止加入任何业务逻辑，如果有初始化逻辑，请放在 init 方法中。可以使用`@PostConstruct`注解修饰 init**

### **4.14 POJO 类的子类重写`toString()`方法时要加`super.toString()`**

### **4.15 在 POJO 类中，禁止同时存在对应属性 xxx 的 `isXxx()` 和 `getXxx()` 方法**

**说明**:POJO 类的相关规范中，boolean 类型的属性一般对应 isXxx()，一般 Xxx 是变量名首字母大写，而如果变量名本身就是 isXxx ，生成的对应方法还是 isXxx()。Boolean 类型的属性一般对应 getXxx()。

### **4.16 使用索引访问用 String 的 split 方法得到的数组时，需做最后一个分隔符后有无内容的检查，否则会有抛 IndexOutOfBoundsException 的风险。**

**说明**：

```java
String str = "a,b,c,,";
String[] ary = str.split(",");
// 预期大于 3，结果等于 3
System.out.println(ary.length);
```

### **4.17 类内方法定义的顺序依次是：公有方法或保护方法 > 私有方法 > getter / setter 方法。**

**说明**：\
(1) 公有方法是类的调用者和维护者最关心的方法，首屏展示最好；\
(2) 保护方法虽然只是子类关心，也可能是“模板设计模式”下的核心方法；\
(3) 私有方法外部一般不需要特别关心，是一个黑盒实现；\
(4) 因为承载的信息价值较低，所有 Service 和 DAO 的 getter / setter 方法放在类体最后。

- 但对于方法重载, 可以将所有访问域的同名方法放在一起, 便于查看

### **4.18 循环体内，字符串的连接方式，使用 StringBuilder 的 append 方法进行扩展。**

**反例**：

```java
String str = "start";
for (int i = 0; i < 100; i++) {
    str = str + "hello";
}
```

**说明**：反编译出的字节码文件显示每次循环都会 new 出一个 StringBuilder 对象，然后进行 append 操作，最后通过`toString()` 返回 String 对象，造成内存资源浪费。

### **4.18 如果要深拷贝对象, 则需重写 Object 的 clone 方法, 不重写默认为浅拷贝**

### **4.19 类成员与方法的访问域遵从最小权限原则：**

1）如果不允许外部直接通过 new 来创建对象，那么构造方法必须是 private。\
2）**工具类**不允许有 public 或 default 构造方法。\
3）类非 static 成员变量并且**与子类共享**，必须是 protected。\
4）类非 static 成员变量并且**仅在本类使用**，必须是 private。\
5）类 static 成员变量如果**仅在本类使用**，必须是 private。\
6）若是 static 成员变量，考虑是否为 final。\
7）类成员**方法只供类内部调用**，必须是 private。\
8）类成员**方法只对继承类公开**，那么限制为 protected。

**说明**：任何类、方法、参数、变量，严控访问范围。过于宽泛的访问范围，不利于模块解耦。

**思考**：如果是一个 private 的方法，想删除就删除，可是一个 public 的 service 成员方法或成员变量，删除一下，不得手心冒点汗吗？变量像自己的小孩，尽量在自己的视线内，变量作用域太大，无限制的到处跑，那么你会担心的。

---

## **5. 日期时间**

### **5.1 在日期格式中分清楚大写的 M 和小写的 m，大写的 H 和小写的 h，大写的 Y 和小写的 y 分别指代的意义。**

**说明**：日期格式中的这两对字母表意如下：\
1）表示月份是大写的 M\
2）表示分钟则是小写的 m\
3）24 小时制的是大写的 H\
4）12 小时制的则是小写的 h\
5）表示当天所在年是 y 6) 表示当周所在年是 Y, 如 2017 年 12 月 31 日 执行结果为 2018/12/31

### **5.2 获取时间**

| 精确度               | 方法                                                                     |
| -------------------- | ------------------------------------------------------------------------ |
| 毫秒                 | `System.currentTimeMillis()`<br>`new Date().getTime()`也可以, 但会创建类 |
| 纳秒                 | `System.nanoTime`                                                        |
| 不带时区的格式化纳秒 | `Instant`类                                                              |
| 带时区的格式化纳秒   | `LocalDateTime`类                                                        |

### **5.3 不允许在程序任何地方中使用：**

| 禁用方法             | 原因                                                                                                         |
| -------------------- | ------------------------------------------------------------------------------------------------------------ |
| `java.sql.Date`      | 不记录时间，时间相关的操作都会抛出异常                                                                       |
| `java.sql.Time`      | 不记录日期，有关日期的操作都会抛出异常                                                                       |
| `java.sql.Timestamp` | 继承自 Date，精确到纳秒, 但是在构造方法对毫秒四舍五入存到 nanos， 如果作为参数传给 Date 的方法, 造成精度损失 |

### **5.4 禁止在程序中写死一年为 365 天，避免在公历闰年时出现日期转换错误或程序逻辑错误。注意闰年 2 月**

### **5.5 使用枚举值来指代月份。如果使用数字，注意 Date，Calendar 等日期相关类的月份 month 取值范围是从 0 到 11 之间。**

**说明**：参考 JDK 原生注释，Month value is 0-based. e.g., 0 for January.

**正例**：Calendar.JANUARY，Calendar.FEBRUARY，Calendar.MARCH 等来指代相应月份来进行传参或比较。

---

## **6. 集合处理**

### **6.1 关于 hashCode 和 equals 的处理，遵循如下规则：**

1）只要覆写 equals，就必须覆写 hashCode。\
2）因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的对象必须覆写这两种方法。\
3）同理, 如果自定义对象作为 Map 的键，那么必须覆写 hashCode 和 equals。

- String 因为覆写了 hashCode 和 equals 方法，所以可以愉快地将 String 对象作为 key 来使用。

### **6.2 在使用 `java.util.stream.Collectors` 类的 `toMap()` 方法转为 Map 集合时，一定要使用参数类型为 `BinaryOperator`，参数名为 `mergeFunction` 的方法，否则当出现相同 key 时会抛出`IllegalStateException` 异常。**

**说明**：参数 `mergeFunction` 的作用是当出现 key 重复时，自定义对 value 取值的选取策略。

**正例**：

```java
List<Pair<String, Double>> pairArrayList = new ArrayList<>(3);
pairArrayList.add(new Pair<>("version", 12.10));
pairArrayList.add(new Pair<>("version", 12.19));
pairArrayList.add(new Pair<>("version", 6.28));
// 当key重复, 取最后一个key对应的value
// 生成的 map 集合中只有一个键值对：{version=6.28}
Map<String, Double> map = pairArrayList.stream()
.collect(Collectors.toMap(Pair::getKey, Pair::getValue, (v1, v2) -> v2));
```

**反例**：

```java
String[] departments = new String[]{"RDC", "RDC", "KKB"};
// key重复且未指定取值策略, 抛出 IllegalStateException 异常
Map<Integer, String> map = Arrays.stream(departments)
.collect(Collectors.toMap(String::hashCode, str -> str));
```

### **6.3 在使用 `java.util.stream.Collectors` 类的 `toMap()` 方法转为 Map 集合时，一定要注意有些 map 的 value 是要求非空的, 一旦存在 null 的 value, 即使按照取值策略取不到, 也会抛 NPE**

**说明**:

| 集合类                                  | Key           | Value         | Super       | 说明                    |
| --------------------------------------- | ------------- | ------------- | ----------- | ----------------------- |
| HashMap<br>LinkedHashMap<br>WeakHashMap | 允许为 null   | 允许为 null   | AbstractMap | 线程不安全              |
| TreeMap                                 | 不允许为 null | 允许为 null   | AbstractMap | 线程不安全              |
| ConcurrentHashMap                       | 不允许为 null | 不允许为 null | AbstractMap | 锁分段（JDK8:CAS 细化） |
| Hashtable                               | 不允许为 null | 不允许为 null | Dictionary  | 线程安全                |
| ConcurrentSkipListMap                   | 不允许为 null | 不允许为 null | AbstractMap | 线程安全                |
| **反例**:                               |               |               |             |                         |

```java
List<Pair<String, Double>> list = new ArrayList<>();
list.add(new Pair<>("2", 0.1));
list.add(new Pair<>("2", 0.2));
list.add(new Pair<>("2", null));
Map<String, Double> collect =
    list.stream().collect(Collectors.toMap(Pair::getFirst, Pair::getSecond, (k1, k2) -> k1));
```

### **6.4 `ArrayList` 的 `subList()` 结果不可强转成 `ArrayList`，否则会抛出 `ClassCastException` 异常. 类似还有`Arrays.asList()`, `List.of()`方法**

**说明**：`subList()`返回的是 ArrayList 的**内部类** SubList，并不是 ArrayList 本身，而是 ArrayList 的一个视图，对于 SubList 的所有操作最终会反映到原列表上。

**反例**:

```java
ArrayList<Pair<String, Double>> list = new ArrayList<>();
list.add(new Pair<>("2", 0.1));
list.add(new Pair<>("2", 0.2));
list.add(new Pair<>("2", null));
// 运行时错误, 编译时ide语法层面不会报错
ArrayList<Pair<String, Double>> list1 = (ArrayList<Pair<String, Double>>) list.subList(0, 1);
```

### **6.5 在 subList 场景中，高度注意对父集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生 `ConcurrentModificationException` 异常。**

### **6.6 使用 Map 的方法 keySet() / values() / entrySet() 返回集合视图时，不可以对其进行添加元素操作，否则会抛出 UnsupportedOperationException 异常。删除, 清空是支持的**

### **6.7 `Collections` 类返回的对象，如：emptyList() / singletonList() 等都是 `immutableList`，不可对其进行添加或者删除元素的操作。**

**反例**：如果查询无结果，返回 `Collections.emptyList()` 空集合对象，调用方一旦在返回的集合中进行了添加元素的操作，就会触发 `UnsupportedOperationException` 异常。

### **6.8 使用集合转数组的方法，必须使用集合的 toArray(T[] array)，传入的是类型完全一致、长度为 0 的空数组。**

**反例**：直接使用 toArray 无参方法存在问题，此方法返回值只能是 Object[]类，若强转其它类型数组将出现`ClassCastException` 错误。

**正例**：

```java
List<String> list = new ArrayList<>(2);
list.add("guan");
list.add("bao");
String[] array = list.toArray(new String[0]);
```

**说明**：使用 toArray 带参方法，数组空间大小的 length：\
1）等于 0，动态创建与 size 相同的数组，性能最好。\
2）大于 0 但小于 size，重新创建大小等于 size 的数组，增加 GC 负担。\
3）等于 size，在高并发情况下，数组创建完成之后，size 正在变大的情况下，负面影响与 2 相同。\
4）大于 size，空间浪费，且在 size 处插入 null 值，存在 NPE 隐患。

### **6.9 泛型通配符`<? is extends of T>`来接收返回的数据，此写法的泛型集合不能使用 add 方法，而`<? is super of T>`不能使用 get 方法，两者在接口调用赋值的场景中容易出错。**

**说明**：扩展说一下 **PECS**(Producer Extends Consumer Super) 原则，即频繁往外供读取内容的，适合用`<? extends T>`，经常往里插入的，适合用`<? super T>`.\
本质上, java 泛型是靠擦除实现的, 只有编译器能明确类的结构时才能分配内存, 才能初始化类.

### **6.10 在无泛型限制定义的集合赋值给泛型限制的集合时，在使用集合元素时，需要进行 `instanceof` 判断，避免抛出 `ClassCastException` 异常。**

**说明**：毕竟泛型是在 JDK5 后才出现，为了向前兼容，编译器允许非泛型集合与泛型集合互相赋值。

**反例**：

```java
List<String> generics = null;
List notGenerics = new ArrayList(10);
notGenerics.add(new Object());
notGenerics.add(new Integer(1));
generics = notGenerics;
// 此处抛出 ClassCastException 异常
String string = generics.get(0);
```

### **6.11 不要在 `foreach` 循环里进行元素的 remove / add 操作。remove 元素请使用 `iterator` 方式，如果并发操作，需要对 `iterator` 对象加锁。**

### **6.12 集合初始化时，指定集合初始值大小。**

**说明**：HashMap 使用构造方法 `HashMap(int initialCapacity)` 进行初始化时，如果暂时无法确定集合大小，那么指定默认值（16）即可。

**正例**：`initialCapacity` = (需要存储的元素个数 / 负载因子) + 1。注意负载因子（即 `loaderfactor`）默认为 0.75，如果暂时无法确定初始值大小，请设置为 16（即默认值）。

**反例**：`HashMap` 需要放置 1024 个元素，由于没有设置容量初始大小，随着元素增加而被迫不断扩容，`resize()` 方法总共会调用 8 次，反复重建哈希表和数据迁移。当放置的集合元素个数达千万级时会影响程序性能。

### **6.13 使用 `entrySet` 遍历 `Map` 类集合 KV，而不是 `keySet` 方式进行遍历。**

**说明**：

- keySet 其实是遍历了 2 次，一次是转为 `Iterator` 对象，另一次是从 hashMap 中取出 key 所对应的 value。
- entrySet 只是遍历了一次就把 key 和 value 都放到了 entry 中，效率更高。
- 如果是 JDK8，使用 `Map.forEach` 方法。

**正例**：

- values() 返回的是 V 值集合，是一个 list 集合对象；
- keySet() 返回的是 K 值集合，是一个 Set 集合对象；
- entrySet() 返回的是 K-V 值组合的 Set 集合。

---

## **7. 并发处理**

### **7.1 获取单例对象需要保证线程安全，其中的方法也要保证线程安全。**

**说明**：资源驱动类、工具类、单例工厂类都需要注意。

### **7.2 创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。**

**正例**：自定义线程工厂，并且根据外部特征进行分组，比如，来自同一机房的调用，把机房编号赋值给
`whatFeatureOfGroup`：

```java
public class UserThreadFactory implements ThreadFactory {
    private final String namePrefix;
    private final AtomicInteger nextId = new AtomicInteger(1);
    // 定义线程组名称，在利用 jstack 来排查问题时，非常有帮助
    UserThreadFactory(String whatFeatureOfGroup) {
        namePrefix = "FromUserThreadFactory's" + whatFeatureOfGroup + "-Worker-";
    }
    @Override
    public Thread newThread(Runnable task) {
        String name = namePrefix + nextId.getAndIncrement();
        Thread thread = new Thread(null, task, name, 0, false);
        System.out.println(thread.getName());
        return thread;
    }
}

class A {
    Thread getInstance(String context) {
        return new UserThreadFactory(context).newThread(
            () -> ...
        );
    }
}
```

### **7.3 线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。**

**说明**：线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。

### **7.4 线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。**

**说明**：Executors 返回的线程池对象的弊端如下：

| 阻塞队列                                                   | 弊端                                                                       |
| ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| FixedThreadPool<br>SingleThreadPool<br>ScheduledThreadPool | 允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM |
| CachedThreadPool                                           | 允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM |

### **7.5 `SimpleDateFormat` 是线程不安全的类，一般不要定义为 `static` 变量，如果定义为 `static`，必须加锁，或者使用 `DateUtils` 工具类。**

**正例**：注意线程安全，使用 DateUtils。亦推荐如下处理：

```java
// ThreadLocal使得泛型类实例成为当前线程独有的一份拷贝, 从而做到共享资源在线程间隔离
// 当线程销毁时, ThreadLocal随之销毁
private static final ThreadLocal<DateFormat> dateStyle = new ThreadLocal<DateFormat>() {
    @Override
    protected DateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
}
```

**说明**：如果是 JDK8 的应用，可以

| 原先(线程不安全) | 替代(线程安全)    |
| ---------------- | ----------------- |
| Date             | Instant           |
| Calendar         | LocalDateTime     |
| SimpleDateFormat | DateTimeFormatter |

### **7.6 必须回收自定义的 `ThreadLocal` 变量记录的当前线程的值，尤其在线程池场景下，线程经常会被复用，如果不清理自定义的 `ThreadLocal` 变量，可能会影响后续业务逻辑和造成内存泄露等问题。尽量在代码中使用 try-finally 块进行回收。**

**说明**:

- ThreadLocal 底层使用了 ThreadLocalMap, 每一个泛型类实例都是一份拷贝并存进 value 中, 如果不主动释放 Entry, 会导致内存泄漏
- 这是因为线程池恰好是为了减少 创建/销毁 线程的开销而复用线程, 那么如果线程池内某线程用完后不释放, ThreadLocalMap 的 Entry 会越堆越多

**正例**：

```java
objectThreadLocal.set(userInfo);
    try {
        // ...
    } finally {
        objectThreadLocal.remove();
}
```

### **7.7 高并发时，同步调用应该去考量锁的性能损耗。能用无锁数据结构，就不要用锁；能锁区块，就不要锁整个方法体；能用对象锁(|非`static`实例|直接`synchronized`)，就不要用类锁(`static`实例|`.class`)。锁的粒度尽量要小**

**说明**：尽可能使加锁的代码块工作量尽可能的小，避免在锁代码块中调用 RPC 方法。

| 类型                               | 实现                            |
| ---------------------------------- | ------------------------------- |
| 细粒度                             | synchronized(this)              |
| 对象锁                             | synchronized(非 static obj)     |
| 只影响同一实例内同一把锁修饰的对象 | synchronized 修饰非 static 方法 |
| 粗粒度                             | synchronized(static obj)        |
| 类锁                               | synchronized 修饰 static 方法   |
| 影响全部实例                       | synchronized(.class)            |

### **7.8 对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造成死锁。**

**说明**：线程一需要对表 A、B、C 依次全部加锁后才可以进行更新操作，那么线程二的加锁顺序也必须是 A、B、C，否则可能出现死锁。

### **7.9 在使用阻塞等待获取锁的方式中，必须在 try 代码块之外，并且在加锁方法与 try 代码块之间没有任何可能抛出异常的方法调用，避免加锁成功后，在 finally 中无法解锁。**

**说明一**：在 lock 方法与 try 代码块之间的方法调用抛出异常，无法解锁，造成其它线程无法成功获取锁, 锁死。\
**说明二**：如果 lock 方法在 try 代码块之内，可能由于其它方法抛出异常，导致在 finally 代码块中，unlock 对未加锁的对象解锁，它会调用 AQS 的 tryRelease 方法（取决于具体实现类），抛出 `IllegalMonitorStateException` 异常。\
**说明三**：如果 lock 方法在 try 代码块之内，在 Lock 对象的 lock 方法实现中可能抛出 `unchecked` 异常，导致加锁失败, 产生的后果与说明二相同。

**正例**：

```java
Lock lock = new XxxLock();
lock.lock();
// 不能抛异常, 否则锁死
try {
    doSomething();
    doOthers();
} finally {
    lock.unlock();
}
```

**反例**：

```java
Lock lock = new XxxLock();
try {
    // 如果此处抛出异常，则直接执行 finally 代码块
    // 而监视器并未加锁, 抛IllegalMonitorStateException异常
    doSomething();
    // 无论加锁是否成功，finally 代码块都会执行
    lock.lock();
    doOthers();
} finally {
    lock.unlock();
}
```

### **7.10 在使用尝试机制来获取锁的方式中，进入业务代码块之前，必须先判断当前线程是否持有锁。锁的释放规则与锁的阻塞等待方式相同。**

**说明**：`Lock` 对象的 `unlock` 方法在执行时，它会调用 AQS 的 `tryRelease` 方法（取决于具体实现类），如果当前线程不持有锁，则抛出 `IllegalMonitorStateException` 异常。\
**正例**：

```java
Lock lock = new XxxLock();
// ...
boolean isLocked = lock.tryLock();
if (isLocked) {
    try {
        doSomething();
        doOthers();
    } finally {
        lock.unlock();
    }
}
```

### **7.11 并发修改同一记录时，避免更新丢失，需要加锁。要么在应用层加锁，要么在缓存加锁，要么在数据库层使用乐观锁，使用 version 作为更新依据。**

**说明**：如果每次访问冲突概率小于 **20**%，推荐使用乐观锁，否则使用悲观锁。乐观锁的重试次数不得小于 **3** 次。

### **7.12 多线程并行处理定时任务时，Timer 运行多个 TimeTask 时，只要其中之一没有捕获抛出的异常，其它任务便会自动终止运行，使用 `ScheduledExecutorService` 和 Quartz(?) 则没有这个问题。**

### **7.13 资金相关的金融敏感信息，使用悲观锁策略。**

**说明**：乐观锁在获得锁的同时已经完成了更新操作，校验逻辑容易出现漏洞，另外，乐观锁对冲突的解决策略有较复杂的要求，处理不当容易造成系统压力或数据异常，所以资金相关的金融敏感信息不建议使用乐观锁更新。

**正例**：悲观锁遵循**一锁二判三更新四释放**的原则。

### **7.14 使用 `CountDownLatch` 进行异步转同步操作，每个线程退出前必须调用 `countDown` 方法，线程执行代码注意 catch 异常，确保 `countDown` 方法被执行到，避免主线程一直阻塞等待 count 归零至 `await` 方法，直到超时才返回结果。**

**说明**：注意，子线程抛出异常堆栈，不能在主线程 try-catch 到。

### **7.15 避免 Random 实例被多线程使用，虽然共享该实例是线程安全的，但会因竞争同一 seed 导致的性能下降。**

**说明**：Random 实例包括 `java.util.Random` 的实例或者 `Math.random()` 的方式。\
**正例**：在 JDK7 之后，可以直接使用 API `ThreadLocalRandom`，而在 JDK7 之前，需要编码保证每个线程持有一个单独的 Random 实例。

### **7.16 通过双重检查锁（double-checked locking），实现延迟初始化需要将目标属性声明为 `volatile` 型，（比如修改 helper 的属性声明为 private volatile Helper helper = null;）。**

**说明**: <https://blog.csdn.net/zhxlxh/article/details/51036869>

**正例**：

```java
public class LazyInitDemo {
    private volatile Helper helper = null;
    public Helper getHelper() {
        if (helper == null) {
            synchronized(this) {
                if (helper == null) {
                    helper = new Helper();
                }
            }
        }
        return helper;
    }
    // other methods and fields...
}
```

### **7.17 volatile 解决多线程内存不可见问题对于一写多读，是可以解决变量同步问题，但是如果多写，同样无法解决线程安全问题。**

**说明**：需要配合`synchronized`, AQS 或 `Atomic`类保证原子性.\
如果是 `count++` 写操作，使用如下类实现：\
`AtomicInteger count = new AtomicInteger();`
`count.addAndGet(1);`\
如果是 JDK8，推荐使用 `LongAdder` 对象，比 `AtomicLong` 性能更好（减少乐观锁的重试次数）。

### **7.18 HashMap 在容量不够进行 resize 时由于高并发可能出现循环链，导致 CPU 飙升，在开发过程中注意规避此风险。**

### **7.19 ThreadLocal 对象使用 static 修饰，等于没用。**

**说明**：这个变量是针对一个线程内所有操作共享的，所以设置为静态变量，所有此类实例共享此静态变量，也就是说在类第一次被使用时装载，只分配一块存储空间，所有此类的对象（只要是这个线程内定义的）都可以操控这个变量。

### **7.20 在高并发场景中，避免使用“等于”判断作为中断或退出的条件。**

**说明**：如果并发控制没有处理好，容易产生等值判断被“击穿”的情况，使用大于或小于的区间判断条件来代替。

**反例**：判断剩余奖品数量等于 0 时，终止发放奖品，但因为并发处理错误导致奖品数量瞬间变成了负数，这样的话，活动无法终止。

---

## **8. 控制语句**

### **8.1 当 switch 括号内的变量类型为 String 并且此变量为外部参数时，必须先进行 null 判断。**

### **8.2 三目运算符 condition ? 表达式 1：表达式 2 中，高度注意表达式 1 和 2 在类型对齐时，可能抛出因自动拆箱导致的 NPE 异常。**

**说明**：以下两种场景会触发类型对齐的拆箱操作：\
1）表达式 1 或 表达式 2 的值只要有一个是原始类型。\
2）表达式 1 或 表达式 2 的值的类型不一致，会强制拆箱升级成表示范围更大的那个类型。

**反例**：

```java
Integer a = 1;
Integer b = 2;
Integer c = null;
Boolean flag = false;
// a*b 的结果是 int 类型，那么 c 会强制拆箱成 int 类型，抛出 NPE 异常
Integer result = (flag ? a * b : c);
```

### **8.3 不要再`if`条件里写复杂表达式, 会提高函数圈复杂度, 使用`boolean`**

### **8.4 循环体中的语句要考量性能，以下操作尽量移至循环体外处理，如定义对象、变量、获取数据库连接，进行不必要的 try-catch 操作（这个 try-catch 是否可以移至循环体外）。**

### **8.5 参数校验场景**

| 需要                                                                                                                     | 不需要                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 调用频次低的方法                                                                                                         | 极有可能被循环调用的方法。但在方法说明里必须注明外部参数检查                                                                                                                              |
| 执行时间开销很大的方法。此情形中，参数校验时间几乎可以忽略不计，但如果因为参数错误导致中间执行回退，或者错误，那得不偿失 | 底层调用频度比较高的方法。毕竟是像纯净水过滤的最后一道，参数错误不太可能到底层才会暴露问题。一般 DAO 层与 Service 层都在同一个应用中，部署在同一台服务器中，所以 DAO 的参数校验，可以省略 |
| 需要极高稳定性和可用性的方法                                                                                             | 被声明成 private 只会被自己代码所调用的方法，如果能够确定调用方法的代码传入参数已经做过检查或者肯定不会有问题，此时可以不校验参数。                                                       |
| 对外提供的开放接口，不管是 RPC / API / HTTP 接口                                                                         | ---                                                                                                                                                                                       |
| 敏感权限入口                                                                                                             | ---                                                                                                                                                                                       |

---

## **9. 注释规约**

### **9.1 类、类属性、类方法, 抽象方法（包括接口中的方法）必须使用 JavaDoc, 枚举类型字段必须添加含义注释**

### **9.2 特殊 javadoc**

1）**待办事宜（TODO）**：（标记人，标记时间，[预计处理时间]）表示需要实现，但目前还未实现的功能。只能应用于类，接口和方法（因为它是一个 Javadoc 标签）。\
2）**错误，不能工作（FIXME）**：（标记人，标记时间，[预计处理时间]）在注释中用 FIXME 标记某代码是错误的，而且不能工作，需要及时纠正的情况。

---

## **10. 前后端规约**

### **10.1 前后端交互的 API，需要明确协议、域名、路径、请求方法、请求内容、状态码、响应体。通过 yaml 规定**

**说明**：\
1）协议：生产环境必须使用 HTTPS。\
2）路径：每一个 API 需对应一个路径，表示 API 具体的请求地址：

- 代表一种资源，只能为名词，推荐使用复数，不能为动词，请求方法已经表达动作意义。
- URL 路径不能使用大写，单词如果需要分隔，统一使用下划线。
- 路径禁止携带表示请求内容类型的后缀，比如".json"，".xml"，通过 accept 头表达即可。

3）请求方法：对具体操作的定义，常见的请求方法如下：

- GET：从服务器取出资源。
- POST：在服务器新建一个资源。
- PUT：在服务器更新资源。
- DELETE：从服务器删除资源。

4）请求内容：URL 带的参数必须无敏感信息或符合安全要求；body 里带参数时必须设置 Content-Type。\
5）响应体：响应体 body 可放置多种数据类型，由 Content-Type 头来确定。

### **10.2 前后端数据列表相关的接口返回，如果为空，则返回空数组[]或空集合{}。**

**说明**：此条约定有利于数据层面上的协作更加高效，减少前端很多琐碎的 null 判断。

### **10.3 服务端发生错误时，返回给前端的响应信息必须包含 HTTP 状态码，errorCode、errorMessage、用户提示信息四个部分。**

**说明**：四个部分的涉众对象分别是浏览器、前端开发、错误排查人员、用户。其中输出给用户的提示信息要求：简短清晰、提示友好，引导用户进行下一步操作或解释错误原因，提示信息可以包括错误原因、上下文环境、推荐操作等。\
**errorCode**：从右向左, 以高位表示宏观错误, 低位表示具体错误。\
**errorMessage**：简要描述后端出错原因，便于错误排查人员快速定位问题，注意不要包含敏感数据信息。需要记录到日志

**正例**：常见的 HTTP 状态码如下

- 200 OK：表明该请求被成功地完成，所请求的资源发送到客户端。
- 401 Unauthorized：请求要求身份验证，常见对于需要登录而用户未登录的情况。
- 403 Forbidden：服务器拒绝请求，常见于机密信息或复制其它登录用户链接访问服务器的情况。
- 404 NotFound：服务器无法取得所请求的网页，请求资源不存在。
- 500 InternalServerError：服务器内部错误。

### **10.4 在前后端交互的 JSON 格式数据中，所有的 key 必须为小写字母开始的 lowerCamelCase 风格，符合英文表达习惯，且表意完整。**

**正例**：errorCode / errorMessage / assetStatus / menuList / orderList / configFlag\
**反例**：ERRORCODE / ERROR_CODE / error_message / error-message / errormessage

### **10.5 对于需要使用超大整数的场景，服务端一律使用 String 字符串类型返回，禁止使用 Long 类型。**

**说明**：Java 服务端如果直接返回 Long 整型数据给前端，Javascript 会自动转换为 Number 类型（注：此类型为双精度浮点数，表示原理与取值范围等同于 Java 中的 Double）。Long 类型能表示的最大值是 263-1，在取值范围之内，超过 253（9007199254740992）的数值转化为 Javascript 的 Number 时，有些数值会产生精度损失。\
**扩展说明**：在 Long 取值范围内，任何 2 的指数次的整数都是绝对不会存在精度损失的，所以说精度损失是一个概率问题。若浮点数尾数位与指数位空间不限，则可以精确表示任何整数，但很不幸，双精度浮点数的尾数位只有 52 位。\
**反例**：通常在订单号或交易号大于等于 16 位，大概率会出现前后端订单数据不一致的情况。比如，后端传输的 "orderId"：362909601374617692，前端拿到的值却是：362909601374617660

### **10.6 HTTP 请求通过 URL 传递参数时，不能超过 2048 字节。**

**说明**：不同浏览器对于 URL 的最大长度限制略有不同，并且对超出最大长度的处理逻辑也有差异，2048 字节是取所有浏览器的最小值。\
**反例**：某业务将退货的商品 id 列表放在 URL 中作为参数传递，当一次退货商品数量过多时，URL 参数超长，传递到后端的参数被截断，导致部分商品未能正确退货。

### **10.7 HTTP 请求通过 body 传递内容时，必须控制长度，超出最大长度后，后端解析会出错。**

**说明**：nginx 默认限制是 1MB，tomcat 默认限制为 2MB，当确实有业务需要传较大内容时，可以调大服务器端的限制。

---

## **11. 其他**

### **11.1 在使用正则表达式时，利用好其预编译功能，可以有效加快正则匹配速度。**

**说明**：不要在方法体内定义：`Pattern pattern = Pattern.compile("规则")`;

### **11.2【强制】避免用 `ApacheBeanutils` 进行属性的 copy。**

**说明**：`ApacheBeanUtils` 性能较差，可以使用其他方案比如 `SpringBeanUtils`，`CglibBeanCopier`，注意均是浅拷贝。

### **11.3【强制】velocity 调用 POJO 类的属性时，直接使用属性名取值即可，模板引擎会自动按规范调用 POJO 的 `getXxx()`，如果是 `boolean` 基本数据类型变量（`boolean` 命名不需要加 is 前缀），会自动调 `isXxx()`方法。**

**说明**：注意如果是 `Boolean` 包装类对象，优先调用 `getXxx()` 的方法。

### **11.4【强制】后台输送给页面的变量必须加 `$!{var}` ——中间的感叹号。**

**说明**：如果 `var` 等于 `null` 或者不存在，那么 `${var}` 会直接显示在页面上。

### **11.5 二方库的新增或升级，保持除功能点之外的其它 jar 包仲裁结果不变。如果有改变，必须明确评估和验证**

**说明**：在升级时，进行 dependency:resolve 前后信息比对，如果仲裁结果完全不一致，那么通过 dependency:tree 命令，找出差异点，进行`<exclude>`排除 jar 包。

### **11.6 二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚举类型或者包含枚举类型的 POJO 对象。**

### **11.7 依赖于一个二方库群时，必须定义一个统一的版本变量，避免版本号不一致**

**说明**：依赖 springframework-core，-context，-beans，它们都是同一个版本，可以定义一个变量来保存版本：`${spring.version}`，定义依赖的时候，引用该版本。

### **11.8 禁止在子项目的 pom 依赖中出现相同的 GroupId，相同的 ArtifactId，但是不同的 Version**

**说明**：在本地调试时会使用各子项目指定的版本号，但是合并成一个 war，只能有一个版本号出现在最后的 lib 目录中。曾经出现过线下调试是正确的，发布到线上却出故障的先例。

### **11.9 所有 pom 文件中的依赖声明放在`<dependencies>`语句块中，所有版本仲裁放在`<dependencyManagement>`语句块中。**

```xml
% 父pom
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>1.2.3.RELEASE</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

```xml
% 子pom无需声明version, scpoe
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### **11.10 调用远程操作必须有超时设置。**

**说明**：类似于 HttpClient 的超时设置需要自己明确去设置 Timeout。根据经验表明，无数次的故障都是因为没有设置超时时间。

### **11.11 高并发服务器建议调小 TCP 协议的 time_wait 超时时间。**

**说明**：操作系统默认 240 秒后，才会关闭处于 time_wait 状态的连接，在高并发访问下，服务器端会因为处于 time_wait 的连接数太多，可能无法建立新的连接，所以需要在服务器上调小此等待值。

**正例**：在 linux 服务器上请通过变更/etc/sysctl.conf 文件去修改该缺省值（秒）：net.ipv4.tcp_fin_timeout=30

### **11.12 调大服务器所支持的最大文件句柄数（File Descriptor，简写为 fd）**

**说明**：主流操作系统的设计是将 TCP / UDP 连接采用与文件一样的方式去管理，即一个连接对应于一个 fd。主流的 linux 服务器默认所支持最大 fd 数量为 1024，当并发连接数很大时很容易因为 fd 不足而出现“open too many files”错误，导致新的连接无法建立。建议将 linux 服务器所支持的最大句柄数调高数倍（与服务器的内存数量相关）。

### **11.13 给 JVM 环境参数设置-XX：+HeapDumpOnOutOfMemoryError 参数，让 JVM 碰到 OOM 场景时输出 dump 信息。**

**说明**：OOM 的发生是有概率的，甚至相隔数月才出现一例，出错时的堆内信息对解决问题非常有帮助。

### **11.14 在线上生产环境，JVM 的 Xms 和 Xmx 设置一样大小的内存容量，避免在 GC 后调整堆大小带来的压力。**

### **11.15 了解每个服务大致的平均耗时，可以通过独立配置线程池，将较慢的服务与主线程池隔离开，免得不同服务的线程同归于尽。**

### **11.16 UML 使用规范**

| 图     | 条件                                                                       |
| ------ | -------------------------------------------------------------------------- |
| 用例图 | 在需求分析阶段，如果与系统交互的 User 超过一类并且相关的 UseCase 超过 5 个 |
| 状态图 | 某个业务对象的状态超过 3 个                                                |
| 时序图 | 系统中某个功能的调用链路上的涉及对象超过 3 个                              |
| 类图   | 系统中模型类超过 5 个                                                      |
| 活动图 | 系统中超过 2 个对象之间存在协作关系                                        |

### **11.17 系统设计时要准确识别出弱依赖，并针对性地设计降级和应急预案，保证核心系统正常可用。**

**说明**：系统依赖的第三方服务被降级或屏蔽后，依然不会影响主干流程继续进行，仅影响信息展示、或消息通知等非关键功能，那么这些服务称为弱依赖。

**正例**：当系统弱依赖于多个外部服务时，如果下游服务耗时过长，则会严重影响当前调用者，必须采取相应降级措施，比如，当调用链路中某个下游服务调用的平均响应时间或错误率超过阈值时，系统自动进行降级或熔断操作，屏蔽弱依赖负面影响，保护当前系统主干功能可用。

**反例**：某个疫情相关的二维码出错：“服务器开了点小差，请稍后重试”，不可用时长持续很久，引起社会高度关注，原因可能为调用的外部依赖服务 RT 过高而导致系统假死，而在显示端没有做降级预案，只能直接抛错给用户。

### **11.18 设计模式原则**

| 原则         | 场景                                                                                 |
| ------------ | ------------------------------------------------------------------------------------ |
| 单一原则     | 类的功能单一职责                                                                     |
| 里氏代换原则 | 推荐使用接口聚合, 若要使用继承, 父子类应满足替换原则                                 |
| 依赖倒置原则 | 依赖抽象类与接口                                                                     |
| 开闭原则     | 极端情况下，交付的代码是不可修改的，同一业务域内的需求变化，通过模块或类的扩展来实现 |

---

## **12. 异常处理**

### **12.1 事务场景中，抛出异常被 catch 后，如果需要回滚，一定要注意手动回滚事务。**

### **12.2 finally 块必须对资源对象、流对象进行关闭. 除非是手写的事务框架 关闭资源有异常也要做 try-catch。**

**说明**：如果 JDK7，可以使用 try-with-resources 方式。

### **12.3 不要在 finally 块中使用 return**

**说明**：try 块中的 return 语句执行成功后，并不马上返回，而是继续执行 finally 块中的语句，如果此处存在 return 语句，则会在此直接返回，无情丢弃掉 try 块中的返回点。\
**反例**：

```java
private int x = 0;
public int checkReturn() {
    try {
        // x 等于 1，此处不返回
        return ++x;
    } finally {
        // 返回的结果是 2
        return ++x;
    }
}
```

### **12.4 在调用 RPC、二方包、或动态生成类的相关方法时，捕捉异常使用顶级 `Throwable` 类进行拦截。**

**说明**：通过反射机制来调用方法，如果找不到方法，抛出 `NoSuchMethodException`。什么情况会抛出
`NoSuchMethodError` 呢？二方包在类冲突时，**仲裁机制**可能导致引入非预期的版本使类的方法签名不匹配，或者在**字节码修改框架**（比如：ASM）动态创建或修改类时，修改了相应的方法签名。这些情况，即使代码编译期是正确的，但在代码运行期时，会抛出 `NoSuchMethodError`。\
**反例**：足迹服务引入了高版本的 spring，导致运行到某段核心逻辑时，抛出 `NoSuchMethodError` 错误，catch 用的类却是 `Exception`，堆栈向上抛，影响到上层业务。这是一个非核心功能点影响到核心应用的典型反例。

### **12.5 防止 NPE，是程序员的基本修养，注意 NPE 产生的场景：**

**反例**：

- 返回类型为基本数据类型，return 包装数据类型的对象时，自动拆箱有可能产生 NPE, `public int method() { return Integer 对象; }`，如果为 null，自动解箱抛 NPE。
- 数据库的查询结果可能为 null。
- 远程调用返回对象时，一律要求进行空指针判断，防止 NPE。
- 对于 Session 中获取的数据，建议进行 NPE 检查，避免空指针。
- 级联调用 obj.getA().getB().getC()；一连串调用，易产生 NPE。

**正例**：使用 JDK8 的 `Optional` 类来防止 NPE 问题。

### **12.6 定义自定义异常时区分 unchecked / checked 异常，避免直接抛出 new RuntimeException()，更不允许抛出 Exception 或者 Throwable，应使用有业务含义的自定义异常。推荐业界已定义过的自定义异常，如：DAOException / ServiceException 等。**

**说明**:

- 异常分为`Error`和`Exception`
- `Exception`又分为`RuntimeException`和普通`Exception`
- 三者中只有普通`Exception`可以在编译时确定, 需要`try-catch`捕捉, 称为 checked. `Error`和`RuntimeException`则只有在运行时才会出现, 无法捕捉, 称为 unchecked

### **12.7 分层异常处理规约**

- 在 DAO 层，产生的异常类型有很多，无法用细粒度的异常进行 catch，可以使用 catch(Exception e) 方式，并 throw new DAOException(e)，不需要打印日志，因为日志在 Manager 或 Service 层一定需要捕获并打印到日志文件中去，如果同台服务器再打日志，浪费性能和存储。
- 在 Service 层出现异常时，必须记录出错日志到磁盘，尽可能带上参数和上下文信息，相当于保护案发现场。Manager 层与 Service 同机部署，日志方式与 DAO 层处理一致，如果是单独部署，则采用与 Service 一致的处理方式。
- Web 层绝不应该继续往上抛异常，因为已经处于顶层，如果意识到这个异常将导致页面无法正常渲染，那么就应该直接跳转到友好错误页面，尽量加上友好的错误提示信息。
- 开放接口层要将异常处理成错误码和错误信息方式返回。

---

## **13. 日志规约**

### **13.1 应用中不可直接使用日志系统（Log4j、Logback）中的 API，而应依赖使用日志框架（SLF4J、JCL）中的 API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一**

**说明**：日志框架使用 SLF4J：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
private static final Logger LOGGER = LoggerFactory.getLogger(Test.class);

@Slf4j
public void do() {
    LOGGER.info("Triggered {} and {}", e1, e2);
    LOGGER.error("inputParams: {} and errorMessage: {}", intput.toString(), e.getMessage(), e);
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug(...);
    }
}

```

### **13.2 避免重复打印日志，浪费磁盘空间，务必在日志配置文件中设置 additivity=false**

**正例**：`<logger name="com.taobao.dubbo.config" additivity="false">`

### **13.3 谨慎地记录日志。生产环境禁止输出 debug 日志；有选择地输出 info 日志；如果使用 warn 来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志。**

**说明**：大量地输出无效日志，不利于系统性能提升，也不利于快速定位错误点。记录日志时请思考：这些日志真的有人看吗？看到这条日志你能做什么？能不能给问题排查带来好处？

### **13.4 可以使用 warn 日志级别来记录用户输入参数错误的情况，避免用户投诉时，没有依据。如非必要，请不要在此场景打出 error 级别，避免频繁告警。**

**说明**：注意日志输出的级别，error 级别只记录系统逻辑出错、异常或者重要的错误信息。

### **13.5 敏感信息进行脱敏**

---

## **14. UT 规约**

### **14.1 AIR 原则**

| 原则        | 解释                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------- |
| Automatic   | 结果判断是自动的, 无需人工识别-->使用 Assert                                                 |
| Independent | 单元测试用例之间决不能互相调用，也不能依赖执行的先后次序. 依赖可使用@Before 和@After 等注解  |
| Repeatable  | UT 是植入到 CI/CD 里的, 每当 push 代码, UT 会自动执行, 为保证排除环境影响, 应使用 Mock 或 DI |

### **14.2 BCDE 原则**

| 原则    | 解释             |
| ------- | ---------------- |
| Border  | 边界值测试       |
| Correct | 正确的输入       |
| Design  | 与设计文档相结合 |
| Error   | 错误信息输入     |

### **14.3 和数据库相关的单元测试，可以设定自动回滚机制，不给数据库造成脏数据。或者对单元测试产生的数据有明确的前后缀标识。**

**正例**：在基础技术部的内部单元测试中，使用 FOUNDATION*UNIT_TEST*的前缀来标识单元测试相关代码。

---

## **15. 安全规约**

---

## **16. 数据库规约**

### **16.1 表达是与否概念的字段，必须使用 is_xxx 的方式命名，数据类型是 unsigned tinyint（1 表示是，0 表示否）。**

**注意**：POJO 类中的任何布尔类型的变量，都不要加 is 前缀，所以，需要在`<resultMap>`设置从 is_xxx 到 Xxx 的映射关系。数据库表示是与否的值，使用 tinyint 类型，坚持 is_xxx 的命名方式是为了明确其取值含义与取值范围。\
**说明**：任何字段如果为非负数，必须是 unsigned。\
**正例**：表达逻辑删除的字段名 is_deleted，1 表示删除，0 表示未删除。

### **16.2 表名、字段名必须使用小写字母或数字，禁止出现数字开头禁止两个下划线中间只出现数字。数据库字段名的修改代价很大，因为无法进行预发布，所以字段名称需要慎重考虑。**

**说明**：MySQL 在 Windows 下不区分大小写，但在 Linux 下默认是区分大小写。因此，数据库名、表名、字段名，都不允许出现任何大写字母，避免节外生枝。

### **16.3 小数类型为 decimal，禁止使用 float 和 double。**

**说明**：在存储的时候，float 和 double 都存在精度损失的问题，很可能在比较值的时候，得到不正确的结果。如果存储的数据范围超过 decimal 的范围，建议将数据拆成整数和小数并分开存储。

### **16.4 如果存储的字符串长度几乎相等，使用 char 定长字符串类型。**

### **16.5 varchar 是可变长字符串，不预先分配存储空间，长度不要超过 5000，如果存储长度大于此值，定义字段类型为 text，独立出来一张表，用主键来对应，避免影响其它字段索引率。**

### **16.6 表必备三字段：id，create_time，update_time。**

**说明**：其中 id 必为主键，类型为 bigint unsigned、单表时自增、步长为 1。create_time，update_time 的类型均为 datetime 类型，如果要记录时区信息，那么类型设置为 timestamp。

### **16.7 在数据库中不能使用物理删除操作，要使用逻辑删除。**

**说明**：逻辑删除在数据删除后可以追溯到行为操作。不过会使得一些情况下的唯一主键变得不唯一，需要根据情况来酌情解决。

### **16.8 单表行数超过 500 万行或者单表容量超过 2GB，才推荐进行分库分表。**

**说明**：如果预计三年后的数据量根本达不到这个级别，请不要在创建表时就分库分表。

### **16.9 尽可能把所有列定义为 not null（索引 NULL 列需要额外的空间来保存，所以要占用更多的空间；进行比较和计算时要对 NULL 值做特别的处理）**

### **16.10 表和表之间不要添加外键联系（外键会影响父表和子表的写操作从而降低性能）**

### **16.11 索引可以增加查询效率，但同样也会降低插入和更新的效率**

### **16.12 避免数据类型的隐式转换, 如 WHERE 从句中对列进行函数转换和计算，隐式转换会导致索引失效**

### **16.13 不要使用 select \*，要显示指明所有的查询列；**

- 消耗更多的 CPU 和 IO 以网络带宽资源, 尤其是 text 类型的字段
- 无法使用覆盖索引
- 增加表结构变更带来的影响, 增减字段容易与 resultMap 配置不一致。

### **16.14 禁止使用不含字段列表的 INSERT 语句**

### **16.15 避免使用子查询，可以把子查询优化为 join 操作**

- 子查询的结果集无法使用索引，通常子查询的结果集会被存储到临时表中，不论是内存临时表还是磁盘临时表都不会存在索引，所以查询性能会受到一定的影响；
- 特别是对于返回结果集比较大的子查询，其对查询性能的影响也就越大；
- 由于子查询会产生大量的临时表也没有索引，所以会消耗过多的 CPU 和 IO 资源，产生大量的慢查询。

### **16.16 减少同数据库的交互次数，尽量写成批量操作；**

### **16.17 对应同一列进行 or 判断时，使用 in 代替 or。in 的值不要超过 500 可以更有效的利用索引，or 大多数情况下很少能利用到索引。但 in 的值也不要超过 1000 个**

### **16.18 业务上具有唯一特性的字段，即使是组合字段，也必须建成唯一索引。**

**说明**：不要以为唯一索引影响了 insert 速度，这个速度损耗可以忽略，但提高查找速度是明显的；另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必然有脏数据产生。

### **16.19 超过三个表禁止 join。需要 join 的字段，数据类型保持绝对一致；多表关联查询时，保证被关联的字段需要有索引。**

**说明**：即使双表 join 也要注意表索引、SQL 性能。防止笛卡尔积

### **16.20 在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据实际文本区分度决定索引长度。**

**说明**：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为 20 的索引，区分度会高达 90%以上，可以使用 `count(distinct left(列名，索引长度)) / count(*)` 的区分度来确定。

### **16.21 页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。**

**说明**：最左匹配原则

### **16.22 联合索引创建的难点在于字段顺序选择：**

- 如果存在多个等值查询，则选择性好的放在前面，选择性差的放在后面
- 如果存在等值查询和排序，则在创建复合索引时，将等值查询字段放在前面，排序放在最后面
- 如果存在等值查询、范围查询、排序。等值查询放在最前面，范围查询和排序需根据实际情况决定索引顺序
- 如果有 order by 的场景，请注意利用索引的有序性。order by 最后的字段是组合索引的一部分，并且放在创建的索引组合顺序的最后，避免出现 filesort 的情况，影响查询性能。
- MySQL5.6 以后引入索引下推, 即使范围查询放在前面, 等值查询放后面, 也可以在范围查询时进行等值过滤, 而非先范围后回表过滤, 减少回表次数

### **16.23 利用覆盖索引来进行查询操作，避免回表。**

聚簇索引包含所有数据, 非聚簇索引包含非主键的索引字段和主键(用于回表), 而当 sql 语句的所求查询字段（select 列）和查询条件字段（where 子句）全都包含在一个索引中（联合索引），可以直接使用索引查询而不需要回表。即在非聚簇索引树中一次得到所有所需结果

### **16.24 利用延迟关联或者子查询优化超多分页场景。**

**说明**：MySQL 并不是跳过 offset 行，而是取 offset+N 行，然后返回放弃前 offset 行，返回 N 行，那当 offset 特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行 SQL 改写。

**正例**：先快速定位需要获取的 id 段，然后再关联：

```sql
-- 子查询优化
SELECT t1.* FROM 表 1 as t1
(select id from 表 1 where 条件 LIMIT 100000 , 20) as t2 where t1.id = t2.id

-- 延迟关联
SELECT C.* FROM 表 1 as C inner join
(select id from 表 1 where 条件 > 100000 LIMIT 20) as T on C.id = T.id
```

**反例**:

```sql
-- OFFSET过大
SELECT id from 表 1 where 条件 LIMIT 100000 , 20
```

### **16.25 不要使用 count(列名) 或 count(常量) 来替代 count(_)，count(_) 是 SQL92 定义的标准统计行数的语法，跟数据库无关，跟 NULL 和非 NULL 无关。**

**说明**：count(\*) 会统计值为 NULL 的行，而 count(列名) 不会统计此列为 NULL 值的行。

### **16.26 当某一列的值全是 NULL 时，count(col) 的返回结果为 0；但 sum(col) 的返回结果为 NULL，因此使用 sum() 时需注意 NPE 问题。**

**正例**：可以使用如下方式来避免 sum 的 NPE 问题：`SELECT IFNULL(SUM(column) , 0) FROM table;`

### **16.27 使用 ISNULL() 来判断是否为 NULL 值。**

**说明**：NULL 与**任何值**的直接比较都为 NULL。

- NULL<>NULL 的返回结果是 NULL，而不是 false。
- NULL=NULL 的返回结果是 NULL，而不是 true。
- NULL<>1 的返回结果是 NULL，而不是 true。

**反例**：在 SQL 语句中，如果在 null 前换行，影响可读性。
`select * from table where column1 is null and column3 is not null`；而 `ISNULL(column)` 是一个整体，简洁易懂。
从性能数据上分析，`ISNULL(column)` 执行效率更快一些。

### **16.28 不得使用物理外键与级联，一切外键概念必须在应用层逻辑解决。**

**说明**：学生表中的 student_id 是主键，那么成绩表中的 student_id 则为外键。如果更新学生表中的 student_id，同时触发成绩表中的 student_id 更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。

### **16.29 数据订正（特别是删除或修改记录操作）时，要先 select，避免出现误删除的情况，确认无误才能执行更新语句。**

**正例**:

```sql
select * from ... where ... as t1
drop from ... where ... in t1
```

**反例**:

```sql
concat ... as @table_name
prepare @table_name
exec @tablename
```

### **16.30 多表操作, 考虑日后可能的调整, 必须在字段前加表别名**

### **16.31 考虑国际化需求, 所有表使用 utf8mb4 字符集, 可以存储 emoji**

### **16.32 TRUNCATE TABLE 比 DELETE 速度快，且使用的系统和事务日志资源少，但 TRUNCATE 无事务且不触发 trigger，有可能造成事故，故不建议在开发代码中使用此语句。**

**说明**：TRUNCATE TABLE 在功能上与不带 WHERE 子句的 DELETE 语句相同。

### **16.33 涉及 sql 语句拼接场景, 也需要当作变量初始化, 并且`where`后留 `1=1`, 方便拼接`AND`和`OR`**

---

## **17. ORM 规约, 包括 mybatis, JPA, hibernate, jooq**

### **17.1 不要用 resultClass 当返回参数, 也不要用 HashMap 接收结果集，即使所有类属性名与数据库字段一一对应，也需要定义`<resultMap>`；反过来，每一个表也必然有一个`<resultMap>`与之对应。**

**说明**：配置映射关系，使字段与 DO 类解耦，方便维护。

**反例**：某同学为避免写一个`<resultMap>`xxx`</resultMap>`，直接使用 Hashtable 来接收数据库返回结果，结果出现日常是把 bigint 转成 Long 值，而线上由于数据库版本不一样，解析成 BigInteger，导致线上问题。

### **17.2 sql.xml 配置参数使用：#{}，#param# 不要使用 ${} 此种方式容易出现 SQL 注入。**

### **17.3 iBATIS 自带的 `queryForList(String stmt，int start，int size)` 不推荐使用。**

**说明**：其实现方式类似 mysql 分页, 是在数据库取到 statementName 对应的 SQL 语句的所有记录，再通过 subList 取 start，size 的子集合，线上因为这个原因曾经出现过 OOM。

### **17.4 不要写一个大而全的数据更新接口。**

传入为 POJO 类，不管是不是自己的目标更新字段，都进行`update table set c1 = value1 , c2 = value2 , c3 = value3`；这是不对的。执行 SQL 时，不要更新无改动的字段，一是易出错；二是效率低；三是增加 binlog 存储。

### **17.5 @Transactional 事务不要滥用。事务会影响数据库的 QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等。**
