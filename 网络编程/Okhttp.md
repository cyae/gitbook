OkHttp 是一套处理 HTTP 网络请求的依赖库，由 Square 公司设计研发并开源，目前可以在 Java 和 Kotlin 中使用。对于 Android App 来说，OkHttp 现在几乎已经占据了所有的网络请求操作，RetroFit + OkHttp 实现网络请求似乎成了一种标配。因此它也是每一个 Android 开发工程师的必备技能，了解其内部实现原理可以更好地进行功能扩展、封装以及优化。

## 网络请求流程分析

先看下 OkHttp 的基本使用：

![123](Pasted%20image%2020231021154514.png)

除了直接 new OkHttpClient 之外，还可以使用内部工厂类 Builder 来设置 OkHttpClient。如下所示：
![](Pasted%20image%2020231021154522.png)
请求操作的起点从 OkHttpClient.newCall().enqueue() 方法开始：

newCall
![](Pasted%20image%2020231021154526.png)
RealCall.enqueue
![](Pasted%20image%2020231021154530.png)
调用 Dispatcher 的入队方法，执行一个异步网络请求的操作。

可以看出，最终请求操作是委托给 Dispatcher的enqueue 方法内实现的。

Dispatcher 是 OkHttpClient 的调度器，是一种门户模式。主要用来实现执行、取消异步请求操作。本质上是内部维护了一个线程池去执行异步操作，

并且在 Dispatcher 内部根据一定的策略，保证最大并发个数、同一 host 主机允许执行请求的线程个数等。

Dispatcher的enqueue 方法的具体实现如下：
![](Pasted%20image%2020231021154537.png)
可以看出，实际上就是使用线程池执行了一个 AsyncCall，而 AsyncCall 实现了 Runnable 接口，因此整个操作会在一个子线程（非 UI 线程）中执行。

继续查看 AsyncCall 中的 run 方法如下：
![](Pasted%20image%2020231021154543.png)
在 run 方法中执行了另一个 execute 方法，而真正获取请求结果的方法是在 getResponseWithInterceptorChain 方法中，从名字也能看出其内部是一个拦截器的调用链，具体代码如下：
![](Pasted%20image%2020231021154556.png)
每一个拦截器的作用如下。

- BridgeInterceptor：主要对 Request 中的 Head 设置默认值，比如 Content-Type、Keep-Alive、Cookie 等。
- CacheInterceptor：负责 HTTP 请求的缓存处理。
- ConnectInterceptor：负责建立与服务器地址之间的连接，也就是 TCP 链接。
- CallServerInterceptor：负责向服务器发送请求，并从服务器拿到远端数据结果。
在添加上述几个拦截器之前，会调用 client.interceptors 将开发人员设置的拦截器添加到列表当中。
对于 Request 的 Head 以及 TCP 链接，我们能控制修改的成分不是很多。

## 总结

1. Okhttp是对Socket的封装。有三个主要的类，Request，Response，Call

2. 默认使用new OkHttpClient() 创建初client对象。如果需要初始化网络请求的参数，如timeout,interceptor等，可以创建Builder，通过builder.build() 创建初client对象。

3. 使用new Request.Builder().url().builder()创建初requst对象。

4. 通过client.newCall()创建出call对象，同步使用call.excute(), 异步使用call,enqueue(). 这里Call是个接口，具体的实现在RealCall这个实现类里面。

Okhttp的高效体现在，okhttp内有个Dispatcher类，是okhttp内部维护的一个线程池，对最大连接数，host最大访问量做了初始定义。维护3个队列及1个线程池

readyAsyncCalls

待访问请求队列，里面存储准备执行的请求。

runningAsyncCalls

异步请求队列，里面存储正在执行，包含已经取消但是还没有结束的请求。

runningSyncCalls

同步请求队列，正在执行的请求，包含已经取消但是还没有结束的请求。

ExecutorService

线程池，最小0，最大Max的线程池

在执行call.excute()的时候，调用到realcall类里的excute方法，这个是同步方法，在方法的第一行就加了锁，判断executed标记，如果是true就抛出异常，保证一个请求只被执行一次。false的话继续向下执行。调用client.dispatcher.excute()进入到dispatcher类中，向runningSyncCalls队列中添加当前这个请求。执行结束会调用finished方法

如果是异步操作，会创建一个RealCall.AsyncCall对象，AsyncCall继承的NamedRunnable接口，NamedRunnable是个runnable。进入到Dispatcher的enqueue()方法中，首先判断线程池中线程的数据，host的访问量，如果都没有达到那么加入到runningAsyncCalls中，并执行。否则加入到readyAsyncCalls队列中。
finished方法，如果是异步操作，promoteCall方法，promoteCalls()中用迭代器遍历readyAsyncCalls 然后加入到runningAsyncCalls

RealConnection

真正的连接操作类，对soket封装，http1/http2的选择，ssl协议等等信息。一个recalconnection就是一次链接

ConnectionPool

链接池，管理http1/http2的连接，同一个address共享一个connection，实现链接的复用。

StreamAlloction

保存了链接信息，address，HttpCodec，realconnection，connectionpool等信息

## 常见问题

### OKHttp有哪些拦截器，分别起什么作用?

OKHTTP的拦截器是把所有的拦截器放到一个list里，然后每次依次执行拦截器，并且在每个拦截器分成三部分：

预处理拦截器内容
通过proceed方法把请求交给下一个拦截器
下一个拦截器处理完成并返回，后续处理工作。
这样依次下去就形成了一个链式调用，看看源码，具体有哪些拦截器：

```java
Response getResponseWithInterceptorChain() throws IOException {

// Build a full stack of interceptors.

List<Interceptor> interceptors = new ArrayList<>();

interceptors.addAll(client.interceptors());

interceptors.add(retryAndFollowUpInterceptor);

interceptors.add(new BridgeInterceptor(client.cookieJar()));

interceptors.add(new CacheInterceptor(client.internalCache()));

interceptors.add(new ConnectInterceptor(client));

if (!forWebSocket) {

interceptors.addAll(client.networkInterceptors());

}

interceptors.add(new CallServerInterceptor(forWebSocket));

Interceptor.Chain chain = new RealInterceptorChain(

interceptors, null, null, null, 0, originalRequest);

return chain.proceed(originalRequest);

}
```

根据源码可知，一共七个拦截器：

1. addInterceptor(Interceptor)，这是由开发者设置的，会按照开发者的要求，在所有的拦截器处理之前进行最早的拦截处理，比如一些公共参数，Header都可以在这里添加。
2. RetryAndFollowUpInterceptor，这里会对连接做一些初始化工作，以及请求失败的充实工作，重定向的后续请求工作。跟他的名字一样，就是做重试工作还有一些连接跟踪工作。
3. BridgeInterceptor，这里会为用户构建一个能够进行网络访问的请求，同时后续工作将网络请求回来的响应Response转化为用户可用的Response，比如添加文件类型，content-length计算添加，gzip解包。
4. CacheInterceptor，这里主要是处理cache相关处理，会根据OkHttpClient对象的配置以及缓存策略对请求值进行缓存，而且如果本地有了可⽤的Cache，就可以在没有网络交互的情况下就返回缓存结果。
5. ConnectInterceptor，这里主要就是负责建立连接了，会建立TCP连接或者TLS连接，以及负责编码解码的HttpCodec
6. networkInterceptors，这里也是开发者自己设置的，所以本质上和第一个拦截器差不多，但是由于位置不同，所以用处也不同。这个位置添加的拦截器可以看到请求和响应的数据了，所以可以做一些网络调试。
7. CallServerInterceptor，这里就是进行网络数据的请求和响应了，也就是实际的网络I/O操作，通过socket读写数据。

### OkHttp怎么实现连接池

为什么需要连接池？
频繁的进行建立Sokcet连接和断开Socket是非常消耗网络资源和浪费时间的，所以HTTP中的keepalive连接对于降低延迟和提升速度有非常重要的作用。keepalive机制是什么呢？也就是可以在一次TCP连接中可以持续发送多份数据而不会断开连接。所以连接的多次使用，也就是复用就变得格外重要了，而复用连接就需要对连接进行管理，于是就有了连接池的概念。

OkHttp中使用ConectionPool实现连接池，默认支持5个并发KeepAlive，默认链路生命为5分钟。

怎么实现的？
1）首先，ConectionPool中维护了一个双端队列Deque，也就是两端都可以进出的队列，用来存储连接。
2）然后在ConnectInterceptor，也就是负责建立连接的拦截器中，首先会找可用连接，也就是从连接池中去获取连接，具体的就是会调用到ConectionPool的get方法。

```JAVA
RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {

assert (Thread.holdsLock(this));

for (RealConnection connection : connections) {

if (connection.isEligible(address, route)) {

streamAllocation.acquire(connection, true);

return connection;

}

}

return null;

}
```

也就是遍历了双端队列，如果连接有效，就会调用acquire方法计数并返回这个连接。

3）如果没找到可用连接，就会创建新连接，并会把这个建立的连接加入到双端队列中，同时开始运行线程池中的线程，其实就是调用了ConectionPool的put方法。

```java
public final class ConnectionPool {

void put(RealConnection connection) {

if (!cleanupRunning) {

//没有连接的时候调用

cleanupRunning = true;

executor.execute(cleanupRunnable);

}

connections.add(connection);

}

}
```

其实这个线程池中只有一个线程，是用来清理连接的，也就是上述的cleanupRunnable

```java
private final Runnable cleanupRunnable = new Runnable() {

@Override

public void run() {

while (true) {

//执行清理，并返回下次需要清理的时间。

long waitNanos = cleanup(System.nanoTime());

if (waitNanos == -1) return;

if (waitNanos > 0) {

long waitMillis = waitNanos / 1000000L;

waitNanos -= (waitMillis * 1000000L);

synchronized (ConnectionPool.this) {

//在timeout时间内释放锁

try {

ConnectionPool.this.wait(waitMillis, (int) waitNanos);

} catch (InterruptedException ignored) {

}

}

}

}

}

};
```

这个runnable会不停的调用cleanup方法清理线程池，并返回下一次清理的时间间隔，然后进入wait等待。

怎么清理的呢？看看源码：

```JAVA
long cleanup(long now) {

synchronized (this) {

//遍历连接

for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {

RealConnection connection = i.next();

//检查连接是否是空闲状态，

//不是，则inUseConnectionCount + 1

//是 ，则idleConnectionCount + 1

if (pruneAndGetAllocationCount(connection, now) > 0) {

inUseConnectionCount++;

continue;

}

idleConnectionCount++;

// If the connection is ready to be evicted, we're done.

long idleDurationNs = now - connection.idleAtNanos;

if (idleDurationNs > longestIdleDurationNs) {

longestIdleDurationNs = idleDurationNs;

longestIdleConnection = connection;

}

}

//如果超过keepAliveDurationNs或maxIdleConnections，

//从双端队列connections中移除

if (longestIdleDurationNs >= this.keepAliveDurationNs

|| idleConnectionCount > this.maxIdleConnections) {

connections.remove(longestIdleConnection);

} else if (idleConnectionCount > 0) { //如果空闲连接次数>0,返回将要到期的时间

// A connection will be ready to evict soon.

return keepAliveDurationNs - longestIdleDurationNs;

} else if (inUseConnectionCount > 0) {

// 连接依然在使用中，返回保持连接的周期5分钟

return keepAliveDurationNs;

} else {

// No connections, idle or in use.

cleanupRunning = false;

return -1;

}

}

closeQuietly(longestIdleConnection.socket());

// Cleanup again immediately.

return 0;

}
```

也就是当如果空闲连接maxIdleConnections超过5个或者keepalive时间大于5分钟，则将该连接清理掉。

4）这里有个问题，怎样属于空闲连接？

其实就是有关刚才说到的一个方法acquire计数方法：

```JAVA
public void acquire(RealConnection connection, boolean reportedAcquired) {

assert (Thread.holdsLock(connectionPool));

if (this.connection != null) throw new IllegalStateException();

this.connection = connection;

this.reportedAcquired = reportedAcquired;

connection.allocations.add(new StreamAllocationReference(this, callStackTrace));

}
```

在RealConnection中，有一个StreamAllocation虚引用列表allocations。每创建一个连接，就会把连接对应的StreamAllocationReference添加进该列表中，如果连接关闭以后就将该对象移除。

5）连接池的工作就这么多，并不复杂，主要就是管理双端队列Deque\<RealConnection\>，可以用的连接就直接用，然后定期清理连接，同时通过对StreamAllocation的引用计数实现自动回收。

### OkHttp里面用到了什么设计模式

责任链模式
这个不要太明显，可以说是okhttp的精髓所在了，主要体现就是拦截器的使用，具体代码可以看看上述的拦截器介绍。

建造者模式
在Okhttp中，建造者模式也是用的挺多的，主要用处是将对象的创建与表示相分离，用Builder组装各项配置。
比如Request：

```java

public class Request {

public static class Builder {

@Nullable HttpUrl url;

String method;

Headers.Builder headers;

@Nullable RequestBody body;

public Request build() {

return new Request(this);

}

}

}
```

工厂模式
工厂模式和建造者模式类似，区别就在于工厂模式侧重点在于对象的生成过程，而建造者模式主要是侧重对象的各个参数配置。
例子有CacheInterceptor拦截器中又个CacheStrategy对象：

```java
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();

public Factory(long nowMillis, Request request, Response cacheResponse) {

this.nowMillis = nowMillis;

this.request = request;

this.cacheResponse = cacheResponse;

if (cacheResponse != null) {

this.sentRequestMillis = cacheResponse.sentRequestAtMillis();

this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();

Headers headers = cacheResponse.headers();

for (int i = 0, size = headers.size(); i < size; i++) {

String fieldName = headers.name(i);

String value = headers.value(i);

if ("Date".equalsIgnoreCase(fieldName)) {

servedDate = HttpDate.parse(value);

servedDateString = value;

} else if ("Expires".equalsIgnoreCase(fieldName)) {

expires = HttpDate.parse(value);

} else if ("Last-Modified".equalsIgnoreCase(fieldName)) {

lastModified = HttpDate.parse(value);

lastModifiedString = value;

} else if ("ETag".equalsIgnoreCase(fieldName)) {

etag = value;

} else if ("Age".equalsIgnoreCase(fieldName)) {

ageSeconds = HttpHeaders.parseSeconds(value, -1);

}

}

}

}
```

观察者模式
之前我写过一篇文章，是关于Okhttp中websocket的使用，由于webSocket属于长连接，所以需要进行监听，这里是用到了观察者模式：

```java
final WebSocketListener listener;

@Override public void onReadMessage(String text) throws IOException {

listener.onMessage(this, text);

}

```

单例模式
这个就不举例了，每个项目都会有

### 怎么设计一个自己的网络访问框架，为什么这么设计？

我目前还没有正式设计过网络访问框架，

是我自己设计的话，我会从以下两个方面考虑

先参考现有的框架，找一个比较合适的框架作为启动点，比如说，基于上面讲到的okhttp的优点，选择okhttp的源码进行阅读，并且将主线的流程抽取出，为什么这么做，因为okhttp里面虽然涉及到了很多的内容，但是我们用到的内容并不是特别多；保证先能运行起来一个基本的框架；
考虑拓展，有了基本框架之后，我会按照我目前在项目中遇到的一些需求或者网路方面的问题，看看能不能基于我这个框架进行优化，比如服务器它设置的缓存策略，
我应该如何去编写客户端的缓存策略去对应服务器的，还比如说，可能刚刚去建立基本的框架时，不会考虑HTTPS的问题，那么也会基于后来都要求https，进行拓展；
为什么要基于Okhttp，就是因为它是基于Socket，从我个人角度讲，如果能更底层的深入了解相关知识，这对我未来的技术有很大的帮助；

### 如何考虑app的安全性？

1. 使用https协议进行交互
2. 数据交互时，根据业务分出哪些是敏感信息，凡是敏感信息使用对称加密方式，如果是类似密码的，则使用不可逆的加密方式；md5
3. 考虑跟钱相关，或者同等重要的数据接口，需要做多重验证，比如：前端加密请求参数，合并请求参数生成MD5码，服务器端做多重认证，最好能对比本地数据库或者缓存之类的信息；
4. 混淆，
5. app加固，dex文件进行加密，这种方式，可以通过”内存下载“，不安全，也只是为了增加破解难度；
6. 将加密算法，一些核心数据添加到so文件中；
