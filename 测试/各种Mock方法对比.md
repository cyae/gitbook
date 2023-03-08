---
date created: 2023-03-08 19:24
---

## 1. EasyMock

- 先录制, 后播放
- 无法mock静态方法, 构造器等
- 过程随着程序逻辑变复杂

```java
public class HelloTest {
    @Test
    public void world() {
        // 1.生成mock对象
        Hello hello = createMock(Hello.class);
        // 2.设置预期行为和输出(录制)
        expect(hello.hello("a")).andReturn("hello a mock");
        expect(hello.hello("b")).andReturn("hello b mock");
        // 3.把mock对象切换到播放模式
        replay(hello);
        // 4.调用mock对象
        assertEquals("hello a mock", hello.hello("a"));
        assertEquals("hello b mock", hello.hello("b"));
        // 5.验证mock对象的行为,包含执行的方法和次数
        verify(hello);
    }
}
```

## 2. Mockito

- spring默认Mock框架
- 无法mock静态方法, 构造器等

```java
@RunWith(MockitoJUnitRunner.class)
public class HelloTest {

    @Mock
    private Hello hello;

    @Before
    public void setUp() {
        // 初始化逻辑
    }

    @Test
    public void hello() {
        Mockito.when(hello.hello("a")).thenReturn("hello a mock");
        // 局部执行真实方法
        Mockito.when(hello.hello("b")).thenCallRealMethod();
        assertEquals("hello a mock", hello.hello("a"));
        assertEquals("hello b", hello.hello("b"));

        // 验证调用次数
        // 提供anyString()等参数匹配器
        Mockito.verify(hello, times(2)).hello(anyString());

        // 验证调用顺序
        InOrder inOrder = Mockito.inOrder(hello);
        inOrder.verify(hello).hello("a");
        inOrder.verify(hello).hello("b");
    }

    @Test
    public void hello_spy1() {
        Hello spy = Mockito.spy(new Hello());
        // 实际会执行hello("a")方法，只是mockito在最后替换了返回值
        Mockito.when(spy.hello("a")).thenReturn("hello a mock");
        assertEquals("hello a mock", spy.hello("a"));
        assertEquals("hello b", spy.hello("b"));
    }

    @Test
    public void hello_spy2() {
        Hello spy = Mockito.spy(new Hello());
        // 实际不会执行hello("a")方法
        Mockito.doReturn("hello a mock").when(spy).hello("a");
        assertEquals("hello a mock", spy.hello("a"));
        assertEquals("hello b", spy.hello("b"));
    }

    @Test(expected = RuntimeException.class)
    public void hello_exception() {
        Mockito.when(hello.hello("a")).thenThrow(new RuntimeException());
        hello.hello("a");
    }
}
```

## 3. PowerMock

- 用于在EasyMock和Mockito的基础上添加静态, 构造器等mock
- 理论上不应该mock静态方法, 构造器等, 先调整结构

### 3.1 mock普通方法

```java
// 1.必须使用PowerMockRunner
@RunWith(PowerMockRunner.class)
// 2.这里填调用被测试方法的类
@PrepareForTest({CallNewHello.class})
public class CallNewHelloTest {
    @Test
    public void hello() throws Exception {
        Hello hello = Mockito.mock(Hello.class);
        // 提供withNoArguments/withAnyArguments/withArguments/withParameterTypes等构造参数设置方式
        PowerMockito.whenNew(Hello.class).withNoArguments().thenReturn(hello);
        Mockito.when(hello.hello("a")).thenReturn("hello a mock");
        CallNewHello callNewHello = new CallNewHello();
        assertEquals("hello a mock", callNewHello.hello("a"));
    }
}

```

### 3.2 mock静态方法

```java
@RunWith(PowerMockRunner.class)
// 这里填需要mock的静态类
@PrepareForTest({HelloStatic.class})
public class HelloStaticTest {
    @Test
    public void hello() {
        PowerMockito.mockStatic(HelloStatic.class);
        Mockito.when(HelloStatic.hello("a")).thenReturn("hello a mock");
        Mockito.when(HelloStatic.hello("b")).thenReturn("hello b mock");
        assertEquals("hello a mock", HelloStatic.hello("a"));
        assertEquals("hello b mock", HelloStatic.hello("b"));
        // 校验是否调用HelloStatic.hello("a")及次数是否为1
        PowerMockito.verifyStatic(HelloStatic.class, Mockito.times(1));
        HelloStatic.hello("a");
        // 校验是否调用HelloStatic.hello("b")及次数是否为1
        PowerMockito.verifyStatic(HelloStatic.class, Mockito.times(1));
        HelloStatic.hello("b");
    }
}
```

### 3.3 mock单例

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest({HelloSingleton.class})
public class HelloSingletonTest {
    @Test
    public void hello() throws Exception {
        // mock构造函数要放在mockStatic前，否则构造函数内的代码会执行
        PowerMockito.whenNew(HelloSingleton.class).withNoArguments().thenReturn(null);
        HelloSingleton hello = Mockito.mock(HelloSingleton.class);
        // mock静态方法getInstance()的返回
        PowerMockito.mockStatic(HelloSingleton.class);
        Mockito.when(HelloSingleton.getInstance()).thenReturn(hello);
        Mockito.when(hello.hello("a")).thenReturn("hello a mock");
        assertEquals("hello a mock", HelloSingleton.getInstance().hello("a"));
        // 验证HelloSingleton.getInstance()是否被调用
        PowerMockito.verifyStatic(HelloSingleton.class, Mockito.times(1));
        HelloSingleton.getInstance();
        // 验证hello的mock对象是否调用hello("a")
        // 两者结合相当于验证HelloSingleton.getInstance().hello("a")
        Mockito.verify(hello).hello("a");
    }
}
```

### 3.4 mock私有方法

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest({HelloPrivate.class})
public class HelloPrivateTest {
    @Test
    public void hello() throws Exception {
        HelloPrivate hello = PowerMockito.mock(HelloPrivate.class);
        PowerMockito.when(hello, "privateHello", "a").thenReturn("hello a mock");
        Mockito.doCallRealMethod().when(hello).hello("a");
        assertEquals("hello a mock", hello.hello("a"));
        PowerMockito.verifyPrivate(hello, Mockito.times(1)).invoke("privateHello", "a");
    }
}
```

## 4. Mock环境变量

### 4.1 surefire 配置静态环境变量

```java
public class EnvironmentTest {
    @Test
    public void test() {
        assertEquals("env_a_value", System.getenv("env_a"));
        assertEquals("property_a_value", System.getProperty("property_a"));
    }
}
```

### 4.2 system-rules 配置动态环境变量

```java
public class EnvironmentTest {
    @Rule
    public final EnvironmentVariables environmentVariables = new EnvironmentVariables();

    @Test
    public void test() {
        environmentVariables.set("name", "value");
        assertEquals("value", System.getenv("name"));
    }

    @Rule
    public final ProvideSystemProperty myPropertyHasMyValue = new ProvideSystemProperty("MyProperty", "MyValue");

    @Rule
    public final ProvideSystemProperty otherPropertyIsMissing = new ProvideSystemProperty("OtherProperty", null);

    @Test
    public void test() {
        assertEquals("MyValue", System.getProperty("MyProperty"));
        assertNull(System.getProperty("OtherProperty"));
    }
}
```

## 5. Mockito集成spring

被测代码

```java
@RestController
public class HelloController {
    @Autowired
    private HelloService helloService;

    @GetMapping(path = "/hello", produces = MediaType.TEXT_PLAIN_VALUE)
    public ResponseEntity<String> hello() {
        return new ResponseEntity<>(helloService.getName(), HttpStatus.OK);
    }

    @GetMapping(path = "/hello/static", produces = MediaType.TEXT_PLAIN_VALUE)
    public ResponseEntity<String> helloStatic() {
        return new ResponseEntity<>(HelloStatic.getName(), HttpStatus.OK);
    }
}

@Service
public class HelloService {
    public String getName() {
        return "hello";
    }
}

public class HelloStatic {
    public static String getName() {
        return "hello static";
    }
}
```

### 5.1 拉起springTest

```java
// 这里需要使用SpringRunner
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@AutoConfigureMockMvc
public class HelloControllerTest {
    @Autowired
    private MockMvc mockMvc;

    // 通过@MockBean注解替换Spring中的Bean为Mock对象
    @MockBean
    private HelloService helloServiceMock;

    @Before
    public void setUp() throws Exception {
        Mockito.when(helloServiceMock.getName()).thenReturn("world");
    }

    @Test
    public void hello() throws Exception {
        mockMvc.perform(get("/hello"))
            .andExpect(status().isOk())
            .andExpect(content().string("world"));
    }
}
```

```java
@RunWith(PowerMockRunner.class)
// @RunWith 已经指定为 SpringRunner ，这时就需要 @PowerMockRunnerDelegate 注解代理SpringRunner
@PowerMockRunnerDelegate(SpringRunner.class)
@SpringBootTest
@PrepareForTest({HelloStatic.class})
@PowerMockIgnore({"javax.*", "com.sun.*", "org.xml.*", "org.w3c.*"})
@AutoConfigureMockMvc
public class HelloControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Before
    public void setUp() throws Exception {
        PowerMockito.mockStatic(HelloStatic.class);
        Mockito.when(HelloStatic.getName()).thenReturn("world static");
    }

    @Test
    public void hello() throws Exception {
        mockMvc.perform(get("/hello/static"))
            .andExpect(status().isOk())
            .andExpect(content().string("world static"));
    }
}
```

### 5.2 不拉起spring

```java
public class HelloControllerTest {
    @Mock
    private HelloService helloService;

    @Before
    public void setUp() throws Exception {
        MockitoAnnotations.initMocks(this);
        Mockito.when(helloService.getName()).thenReturn("world");
    }

    @Test
    public void hello() throws Exception {
        HelloController helloController = new HelloController(helloService);
        ResponseEntity<String> hello = helloController.hello();
        assertEquals(HttpStatus.OK, hello.getStatusCode());
        assertEquals("world", hello.getBody());
    }
}
```

## 6. mockHttp客户端请求

```xml
<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-netty</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-junit-rule</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
```

被测代码

```java
public class HelloHttp {
    private String host;
    private int port;
    public HelloHttp(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public String hello() {
        CloseableHttpClient client = HttpClients.createDefault();
        HttpGet httpGet = new HttpGet("http://" + host + ":" + port + "/hello");
        try (CloseableHttpResponse response = client.execute(httpGet)) {
            return EntityUtils.toString(response.getEntity());
        } catch (IOException e) {
            System.out.println("http get error:" + e.getMessage());
        }
        return null;
    }
}
```

```java
public class HelloHttpTest {
    private static final String HOST = "localhost";
    private static final int PORT = PORT;

    @Rule
    public MockServerRule server = new MockServerRule(this, PORT);
    
    @Test
    public void hello() {
        MockServerClient mockClient = new MockServerClient(HOST, PORT);
        mockClient.when(request().withPath("/hello").withMethod("GET"))
            .respond(response().withStatusCode(200).withBody("hello"));
        HelloHttp helloHttp = new HelloHttp(HOST, PORT);
        assertEquals("hello", helloHttp.hello());
    }
}
```

## 7. 组件提供的原生Mock工具(zk为例)

```yaml
spring:
  application:
    name: hello
    cloud:
      zookeeper:
        connect-string: ip:port
        discovery:
          enabled: true
          register: true
```

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-test</artifactId>
    <version>2.6.0</version>
    <scope>test</scope>
</dependency>
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@AutoConfigureMockMvc
public class ApplicationTest {
    private static TestingServer server;

    @Autowired
    private MockMvc mockMvc;

    @BeforeClass
    public static void beforeClass() throws Exception {
        server = new TestingServer(2181, true);
    }

    @AfterClass
    public static void afterClass() throws Exception {
        server.stop();
    }

    @Test
    public void test() throws Exception {
        mockMvc.perform(get("/hello"))
            .andExpect(status().is2xxSuccessful())
            .andExpect(content().string("hello"));
    }
}
```
