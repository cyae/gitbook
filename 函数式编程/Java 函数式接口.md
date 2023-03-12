### 基本数据类型

以上介绍的几个常用的函数式接口入参和返回，都是泛型对象的，也就是必须为引用类型。当我们传入或获取的是基本数据类型时，将会发生自动装箱和自动拆箱，带来不必要的性能损耗，比如：

```java
Supplier<Long> supplier = () -> System.currentTimeMillis();
long timeMillis = supplier.get();
```

在上面例子里，发生了一次自动装箱（long 被装箱为 Long）和一次自动拆箱（Long 被拆箱为 long），如何避免这种不必要的性能损耗呢？JDK 为我们提供相应的函数式接口，如 LongSupplier 接口，定义了一个名叫 getAsLong 的抽象方法，签名是() -> long。上面的例子可以优化为：

```java
LongSupplier supplier = () -> System.currentTimeMillis();
long timeMillis = supplier.getAsLong();
```

## 常用接口

### Supplier 相关的接口

| 接口名称        | 方法名称     | 方法签名      |
| --------------- | ------------ | ------------- |
| Supplier        | get          | () -> T       |
| BooleanSupplier | getAsBoolean | () -> boolean |
| DoubleSupplier  | getAsDouble  | () -> double  |
| IntSupplier     | getAsInt     | () -> int     |
| LongSupplier    | getAsLong    | () -> long    |

### Consumer 相关的接口

| 接口名称          | 方法名称 | 方法签名            |
| ----------------- | -------- | ------------------- |
| Consumer          | accept   | (T) -> void         |
| DoubleConsumer    | accept   | (double) -> void    |
| IntConsumer       | accept   | (int) -> void       |
| LongConsumer      | accept   | (long) -> void      |
| ObjDoubleConsumer | accept   | (T, double) -> void |
| ObjIntConsumer    | accept   | (T, int) -> void    |
| ObjLongConsumer   | accept   | (T, long) -> void   |

### Consumer 相关的接口

| 接口名称          | 方法名称 | 方法签名            |
| ----------------- | -------- | ------------------- |
| Consumer          | accept   | (T) -> void         |
| DoubleConsumer    | accept   | (double) -> void    |
| IntConsumer       | accept   | (int) -> void       |
| LongConsumer      | accept   | (long) -> void      |
| ObjDoubleConsumer | accept   | (T, double) -> void |
| ObjIntConsumer    | accept   | (T, int) -> void    |
| ObjLongConsumer   | accept   | (T, long) -> void   |

## Comparator

### Comparator 的使用

在之前文章的例子中，我们使用`Comparator.comparing`静态方法构建了一个`Comparator`接口的实例，我们再来简单介绍一下。先来看看 Mask 类是怎么写的：

```java
package one.more.study;

/**
 * 口罩
 * @author 万猫学社
 */
public class Mask {
    public Mask() {
    }

    public Mask(String brand, String type, double price) {
        this.brand = brand;
        this.type = type;
        this.price = price;
    }

    /**
     * 品牌
     */
    private String brand;
    /**
     * 类型
     */
    private String type;
    /**
     * 价格
     */
    private double price;

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }

    @Override
    public String toString() {
        return "Mask{" +
                "brand='" + brand + '\'' +
                ", type='" + type + '\'' +
                ", price=" + price +
                '}';
    }
}
```

然后，根据口罩品牌对口罩列表进行正序排序：

```java
List<Mask> maskList = new ArrayList<>();
maskList.add(new Mask("3M", "KN95",17.8));
maskList.add(new Mask("Honeywell", "KN95",18.8));
maskList.add(new Mask("3M", "FFP2",19.8));
maskList.add(new Mask("Honeywell", "N95",19.5));
maskList.sort(Comparator.comparing(Mask::getBrand));
for (Mask mask : maskList) {
    System.out.println(mask);
}
```

### 逆序

需求改了，要求按照口罩品牌进行逆序排列，这是还需不需要再构建一个`Comparator`接口的实例呢？答案是不需要，`Comparator`接口有一个默认方法`reversed`可以使其逆序，把上面的例子稍微修改一下：

```java
maskList.sort(Comparator.comparing(Mask::getBrand).reversed());
```

### 比较器链

需求又改了，先按照口罩品牌逆序排序，如果口罩品牌一样，再按照口罩类型正序排序。Comparator 接口还有一个默认方法 thenComparing 就是做这个的，它的入参也是一个 Function 接口的实例，如果前一个比较器的比较结果相同，就当前的比较器再进行比较，我们再来修改一下上面的例子：

```java
maskList.sort(Comparator.comparing(Mask::getBrand)
        .reversed()
        .thenComparing(Mask::getType));
```

需求又又改了，先按照口罩品牌逆序排序，如果口罩品牌一样，再按照口罩价格正序排序。口罩价格是 double 类型，如果使用 thenComparing 会导致自动装箱，造成资源的白白浪费。所以，推荐使用 thenComparingDouble 方法，它的入参是 ToDoubleFunction，代码修改如下：

```java
maskList.sort(Comparator.comparing(Mask::getBrand)
        .reversed()
        .thenComparingDouble(Mask::getPrice));
```

类似这样支持基础数据类型的方法还有两个，分别是：`thenComparingInt`方法，它的入参是`ToIntFunction`；`thenComparingLong`方法，它的入参是`ToLongFunction`。

### 总结

| 默认方法名称        | 作用     | 入参             | 入参签名      |
| ------------------- | -------- | ---------------- | ------------- |
| reversed            | 逆序     | 无               | 无            |
| thenComparing       | 比较器链 | Function         | (T) -> R      |
| thenComparingInt    | 比较器链 | ToIntFunction    | (T) -> int    |
| thenComparingLong   | 比较器链 | ToLongFunction   | (T) -> long   |
| thenComparingDouble | 比较器链 | ToDoubleFunction | (T) -> double |

## 接口复合

### Consumer 复合

`Consumer`接口中，有一个默认方法`andThen`，它的入参还是`Consumer`接口的实例。做完上一个`Consumer`的操作以后，再做当前`Consumer`的操作，就像工厂的流水线一样，比如：

```java
Consumer<Mask> brand = m -> m.setBrand("3M");
Consumer<Mask> type = m -> m.setType("N95");
Consumer<Mask> price = m -> m.setPrice(19.9);
Consumer<Mask> print = System.out::println;

brand.andThen(type)
        .andThen(price)
        .andThen(print)
        .accept(new Mask());
```

### Predicate 复合

Predicate 接口一共有 3 个默认方法：`negate`、`and`和`or`，用它们可以创建更加复杂的 Predicate 接口实例。

#### negate 方法

negate 方法就是做非运算。比如，判断口罩类型是 N95：

```java
Mask mask = new Mask("Honeywell", "N95",19.5);
Predicate<Mask> isN95 = m -> "N95".equals(m.getType());
System.out.println(isN95.test(mask));
```

那么，使用 negate 方法以后，就变成了判断口罩类型不是 N95：

```java
Mask mask = new Mask("Honeywell", "N95",19.5);
Predicate<Mask> isN95 = m -> "N95".equals(m.getType());
System.out.println(isN95.negate().test(mask));
```

#### and 方法

and 方法就是做与运算。比如：

```java
Mask mask = new Mask("Honeywell", "N95",19.5);
Predicate<Mask> isN95 = m -> "N95".equals(m.getType());
Predicate<Mask> lessThan20 = m -> m.getPrice() < 20.0;
System.out.println(isN95.and(lessThan20).test(mask));
```

上面的代码分别声明了 2 个`Predicate`接口的实例，分别是**判断口罩类型是 N95**和**判断口罩价格小于 20**，使用`and`方法以后，表示口罩类型是 N95 **并且**  口罩价格小于 20

#### or 方法

or 方法就是做或运算。比如：

```java
Mask mask = new Mask("Honeywell", "N95", 21.5);
Predicate<Mask> isN95 = m -> "N95".equals(m.getType());
Predicate<Mask> lessThan20 = m -> m.getPrice() < 20.0;
System.out.println(isN95.or(lessThan20).test(mask));
```

上面的代码分别声明了 2 个`Predicate`接口的实例，分别是**判断口罩类型是 N95**和**判断口罩价格小于 20**，使用`or`方法以后，表示口罩类型是 N95 **或者**  口罩价格小于 20

#### and 方法和 or 方法组合使用

当 and 方法和 or 方法组合使用时，优先级是由在 Lambda 表达式链中的位置决定的，从左到右优先级从高到低，比如：

```java
Mask mask = new Mask("3M", "N95", 21.5);
Predicate<Mask> is3M = m -> "3M".equals(m.getType());
Predicate<Mask> isN95 = m -> "N95".equals(m.getType());
Predicate<Mask> lessThan20 = m -> m.getPrice() < 20.0;
System.out.println(is3M.or(isN95).and(lessThan20).test(mask));
```

上面的代码分别声明了 3 个 Predicate 接口的实例，分别是判断口罩品牌是 3M、判断口罩类型是 N95 和判断口罩价格小于 20，3 个 Predicate 组合以后是 is3M.or(isN95).and(lessThan20)，根据从左到右优先级从高到低，组合以后的逻辑是(is3M || isN95 ) && lessThan20

### Function 复合

Function 接口一共有 2 个默认方法，分别是：andThen 和 compose，用它们可以创建更加复杂的 Function 接口实例。

#### andThen 方法

Function 接口的 andThen 方法，和 Consumer 接口的类似，它的入参还是 Function 接口的实例。做完上一个 Function 的操作以后，再做当前 Function 的操作，比如：

```java
Function<Integer, Integer> plusTwo = x -> x + 2;
Function<Integer, Integer> timesThree = x -> x * 3;
System.out.println(plusTwo.andThen(timesThree).apply(1));
System.out.println(plusTwo.andThen(timesThree).apply(2));
System.out.println(plusTwo.andThen(timesThree).apply(3));
```

上面的代码分别声明了 2 个`Function`接口的实例，先加 2，然后再乘以 3，也就是`(x + 2) * 3`

#### compose 方法

`Function`接口的`compose`方法，和`andThen`方法相反的，先做当前`Function`的操作，然后再做上一个`Function`的操作，比如：

```java
Function<Integer, Integer> plusTwo = x -> x + 2;
Function<Integer, Integer> timesThree = x -> x * 3;
System.out.println(plusTwo.compose(timesThree).apply(1));
System.out.println(plusTwo.compose(timesThree).apply(2));
System.out.println(plusTwo.compose(timesThree).apply(3));
```

上面的代码分别声明了 2 个`Function`接口的实例，先乘以 3，然后再加 2，也就是`(x * 3) + 2`
