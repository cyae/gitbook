## 背景

Java线程模型最早只支持Thread和Runnable：

| 缺陷             | 改进              |
| ---------------- | ----------------- |
| - 只能执行一次   | 线程池            |
| - 单独创建和销毁 |                   |
| 不能拿到运行结果 | Future + Callable |

有了以上工具，可以组合使用线程池 + Future + 同步阻塞机制，实现任务编排。

jdk1.8引入CompletableFuture，封装以上3个工具，简化使用与调试。

## 基础玩法

### 创建与执行

```java
// 使用内置线程池ForkJoinPool，不推荐
CompletableFuture<U> cf = CompletableFuture.supplyAsync(Supplier<U> supplier);

// 自定义线程池
CompletableFuture<U> cf = CompletableFuture.supplyAsync(Supplier<U> supplier, Executor executor);

// 阻塞获取执行结果，抛出受检异常
U res = cf.get();

// 阻塞获取执行结果，抛出非受检异常
U res = cf.join();

// 创建无返回值cf
CompletableFuture<Void> cf = CompletableFuture.runAsync(Comsumer<U> consumer);
```

### 一元依赖

```java
cf_cur = cf_pre.thenApply(Function<U, V> Function);
cf_cur = cf_pre.thenCompose(Function<U, CompletableFuture<V>> function);

cf_cur = cf_pre.thenAccept(Consumer<U> consumer);

cf_cur = cf_pre.thenRun(Runnable runnable);
```

### 二元依赖

```java
cf_AB = cf_A.thenCombine(cf_B, BiFunction<U, V, T> biFunc);
cf_AB = cf_A.applyToEither(cf_B, Function<U, V> func);

cf_AB = cf_A.thenAcceptBoth(cf_B, BiConsumer<U, V> biConsum);
cf_AB = cf_A.acceptEither(cf_B, Consumer<U> consum);

cf_AB = cf_A.runAfterBoth(cf_B, Runnable runnable);
cf_AB = cf_A.runAfterEither(cf_B, Runnable runnable);
```

### 多元依赖

```java
cf_agg = CompletableFuture.allOf(List<cf>);
cf_agg = CompletableFuture.anyOf(List<cf>);

cf_agg.thenApply(_ -> {
    cf1.join();
    cf2.join();

    return function(res_cf1, res_cf2);
})
```

## 进阶玩法

### 异常处理

```java
exceptionally(Function<Throwable, ? extends T> fn);

whenComplete(BiConsumer<? super T, ? super Throwable> action);

handle(BiFunction<? super T, ? extends U> handler);
```

### 超时机制

### 异步化

上述所有方法都有Async版本（exceptionally在jdk12），和同步版本区别在于：

- 同步版返回的stage使用依赖stage的线程
- 异步版则回收依赖stage的线程，然后从线程池分配新线程
