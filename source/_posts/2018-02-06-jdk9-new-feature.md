---
title: jdk9-new-feature
date: 2018-02-06 23:20:32
tag:
	- jdk9
	- feature
---


Java 9版本于2017年9月21日发布，本文将主要介绍Java 9中引入的若干新特性。在介绍之前，还是很必要回顾一下Java 8发布的内容，毕竟Java 9中的若干新特性（例，Stream 流处理）也是在前者基础上的展开的。

# Java 8特性回顾
Java 8是Java自Java 5（发布于2004年）之后的最重要的版本。这个版本包含语言、编译器、库、工具和JVM等方面的十多个新特性。主要有：

**Java语言**

- 方法引用
- 重复注解
- 类型推断
- 参数名称
- 拓宽注解的使用
- Lambda表达式
- 接口的默认方法和静态方法

**JDK新特性**

- Optional
- Streams
- Date/Time API
- Base64
- 并行数组
- 新增 StampedLock
- 原子性操作类添加新成员
- 优化HashMap

**JVM**

- 使用 元空间（Metaspace）代替永久代（PermGen space）
- 使用 -XX:MetaSpaceSize和 -XX:MaxMetaspaceSize 代替原来的 -XX:PermSize和-XX:MaxPermSize。

**工具/编译器**

- Nashorn引擎：jjs
- 类依赖分析器：jdeps

# Java 9的安装
## 下载
[Java SE Development Kit 9 Downloads](http://www.oracle.com/technetwork/java/javase/downloads/jdk9-downloads-3848520.html)

## 使用
将 JAVA_HOME 配置为 Java 9的路径后，查看 JDK 版本：
```
java -version
java version "9.0.1"
Java(TM) SE Runtime Environment (build 9.0.1+11)
Java HotSpot(TM) 64-Bit Server VM (build 9.0.1+11, mixed mode)
```
将 JAVA_HOME 配置为 Java 8的路径后，再次查看 JDK 版本：
```
java -version
java version "1.8.0_92"
Java(TM) SE Runtime Environment (build 1.8.0_92-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.92-b14, mixed mode)
```

注意这里的一个细节：**从 Java 9开始，Java 版本方案将根据业内软件版本编码的最佳实践进行修改**，即Java版本字符串将依次包含如下三个部分：主版本号、小（维护）版本号和安全版本号。这一变化可能会导致目前解析版本字符串而有假定版本号开头为1或点的应用程序出现问题。例如： System.getProperty("java.version").indexof('.');。

# Java 9新特性
## 概要
**Java语言**

- Jigsaw：模块化
- 接口中的私有方法

**JDK**

- 进程 API 增强
- 改进了 Stream API
- 集合类的工厂方法
- Try-With-Resources 改进
- HTTP/2
- 响应式流 Reactive Streams

**JVM**

- 修改默认垃圾回收器
- 优化字符串占用空间
- 竞争锁的性能优化
- 代码分段缓存

**工具/编译器**

- JShell：Java交互式REPL
- 改进的 Javadoc
- 多版本 JAR

## 详解

### 新特性1：Jigsaw：模块化
模块系统主要解决两个基本问题：

1. Jar包膨胀：以类和接口进行面向对象的设计在系统规模越来越大时因为隔离粒度问题而难以真正地封装代码，因为在更高粒度层面（JAR 文件，如tool.jar, rt.jar），系统的不同部分之间没有明确的依赖关系的概念。
2. 类使用的的安全性：类路径（classpath）中的任何其他类可以访问每个公共类，这会导致开发人员无意中使用不是公开API的类，严重影响系统后续维护升级。

模块化的引入使得JDK可以在更小的设备中使用，因为采用模块化系统的应用程序只需要这些应用程序所需的那部分 JDK 模块，而非是整个JDK框架了。

其次，模块化使得对 package 的控制更精细了，这与 maven 的通过依赖不同之处，在于 maven 使用了dependency 进行管理依赖，而 jigsaw 使用 requiure 进行管理依赖，通过 export，可以只暴露某个模块下指定的包给调用者（写到这里我联想到了 Javascript 里面的 requirejs 模块，Java 这两个关键字的灵感来自于此？）。

例子：

编写模块化的代码和没有模块化的代码区别并不大，Java 9 通过根目录下的 module-info.java 文件进行区分。

整个工程的结构为：
```
src
└── main
└── java
├── com
│ └── sk
│ └── scalpel
│ └── priviate
│ └── PrivateClass.java
│ └── common
│ └── CommonClass.java
└── module-info.java
```
src/main/java/module-info.java 内容为：
```
module commonClass.jigsaw {
    exports com.sk.ocelot.common;
    requires sso;
}
```
在这个模块描述符中，<font color=#0099ff>通过 requires 语句来表示对其他模块的依赖，此外，exports 语句可以控制哪些package 可被其他模块使用</font>。

<font color=#0099ff>Java 平台本身也使用自己的模块系统进行了模块化</font>。通过封装 JDK 内部类，Java 平台变得更加安全并且让后续的持续演进变得容易得多。比如 rt.jar, tools.jar 两个核心 Jar包，被 JDK9/jmods 文件下若干个 \*.jmod 文件所替代。

下图是 Java9 中的模块依赖图，可以看到，所有的模块都依赖了一个叫做 java.base 的模块，而 java.base 中包含了Java 语言层面的一些基础包，如 java.net、 java.nio、java、util等，具体可参考 Module java.base

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/jdk9/jdk-modules.jpg)


## 新特性2：JShell：Java交互式REPL
REPL是一种快速运行语句的命令行工具，很多语言都具备这个能力，如 Python、Scala 等。我们可以从控制台启动 jshell，然后直接输入和执行 Java 代码，jshell的即时反馈使它成为探索API和尝试语言特性的好工具。

```
pwd
/Library/Java/JavaVirtualMachines/jdk-9.0.1.jdk/Contents/Home/bin
./jshell
| 欢迎使用 JShell -- 版本 9.0.1
| 要大致了解该版本, 请键入: /help intro
jshell> System.out.println("Current process id: " + ProcessHandle.current().pid());
Current process id: 9022
jshell>
```
有了 JShell 后，测试 API 再也不用去新建一个工程了~

### 新特性3：垃圾回收器
Garbage First（G1）的设计初衷是，以更高的计算成本为代价最小化STW中断时间。Java 9 使用G1作为默认的垃圾收集器，替代了之前默认使用的 Parallel GC。事实上它从 JDK 8u40 开始就已经十分完善，足以作为默认的垃圾收集器了。

当然，这里也存在若干争议，可以看下一下这篇文章：[Oracle Proposes G1 as the Default Garbage Collector for Java 9](https://www.infoq.com/news/2015/06/Oracle-Proposes-G1-Default-GC)。

### 新特性4：字符串优化
JDK 6 引入了可选的 Compressed String 功能，JDK 9 引入了 Compact String ，它们两者的设计目的都是优化字符串在 JVM 中的内存占用。因为 Java 内部使用 UTF-16，占用两个字节，但从统计角度来说只需 8 比特的情况占大多数，例如：LATIN-1。大多数情况下，字符串实例常占用比它实际需要的内存多一倍的空间。

Java 6中，在启动参数里使用 *-XX:+UseCompressedStrings* 可以启用 Compressed String 功能，字符串将以 byte[] 的形式存储，代替原来的 char[]，此功能最终在 JDK 7 中被移除，主要原因在于它将带来一些无法预料的性能问题（存储为 byte[] 后，字符串的操作方法都依赖于字符数组的表现形式，而非字节数组。介于此，很多的方法都需将压缩后的字节数组解压缩为字符数组，无形中影响了性能）。

Java 9重新采纳字符串压缩这一概念：通过引入了一个 final 修饰的成员变量 coder, 由它来保存当前字符串的编码信息。若该字符串为 LATIN-1 编码，则使用8个比特来存储，若干 UTF-16 编码，则使用16个比特来编码。代码如下：
```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
 
    /**
     * The value is used for character storage.
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     *
     * Additionally, it is marked with {@link Stable} to trust the contents
     * of the array. No other facility in JDK provides this functionality (yet).
     * {@link Stable} is safe here, because value is never null.
     */
    @Stable
    private final byte[] value;
 
    /**
     * The identifier of the encoding used to encode the bytes in
     * {@code value}. The supported values in this implementation are
     *
     * LATIN1
     * UTF16
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     */
    private final byte coder;
    //略
}
```
可以看到 新引入的 coder 字段的及其作用。

### 新特性5：HTTP/2
HTTP/2标准是HTTP协议的最新版本，上一个版本HTTP/1.1诞生于1999年。

Java对 HTTP 的支持是基于 HTTP/1.0 展开的，当时最好的设计思想已经无法满足现在的需求（如 HTTPS 的大规模普及）。同时，JDK 内核对 HTTP 的支持已经无法跟上现实网络的发展步伐。实际上，甚至JDK8也只不过是交付了一个支持 HTTP/1.0 的客户端，然而，大多数的开发者早已转而使用第三方客户端库了，比如Apache的HttpComponents。

Java 9提供了一种HTTP调用的新方式。对于旧的“HttpURLConnetion”API来说，这个迟来的替换也添加了对 WebSockets和 HTTP/2 的支持。

Java 9当前的代码库只支持 HTTP/1.1，但是已经包含了新的 API。这使得在对 HTTP/2 支持完成对过程中，开发者可以实验性地使用和验证新的 API。如下面的异步发送请求功能：
```
try {
    HttpClient httpClient = HttpClient.newHttpClient();
    HttpRequest httpRequest = HttpRequest.newBuilder().uri(new URI("https://www.bing.com/")).GET().build();
    Map< String, List < String >> headers = httpRequest.headers().map();
    CompletableFuture<HttpResponse< String >> asyncResponse = httpClient.sendAsync(httpRequest, HttpResponse.BodyHandler.asString());
    if (!asyncResponse.isDone()) {
        asyncResponse.cancel(true);
        System.out.println("Sedn async request failed...");
        return;
    }
    HttpResponse response = asyncResponse.get();
    System.out.println("Response body: " + response.body());
 
} catch (Exception e) {
    System.out.println("message " + e);
}
```
### 新特性6：接口中的私有方法
Java 8使用两个新概念扩展了接口的含义：默认方法和静态方法。设想这样一个场景：在定义的接口中有几个默认方法，代码几乎相同，我们需要重构这些方法来调用包含共享功能的私有方法。现在的问题是：默认方法不能是私有的。使用共享代码创建另一个默认方法不是一个可行的解决方案，因为该帮助方法会成为公共API的一部分。在Java 9中，可以向接口添加私有的帮助方法来解决此问题：
```
public interface IInterfaceDemo {
 
     default void method1() {
         init();
         // do something
     }
 
     default void method2() {
         init();
         // do something
     }
 
     // 公共方法声明为私有
     private void init() {
         System.out.println("Current process id: " + ProcessHandle.current().pid());
     }
 }
 ```

### 新特性7：进程增强API
新增加的 API 提供了一种java程序和操作系统交互的能力，我们可以在代码中来获取关于进程的一些信息，java 9中新增了一个类 ProcessHandle，使用这个类可以很方便的查询进程的一些信息。

下面略举几个例子，具体可参考 ProcessHandleImpl 这个实现类~

```
// 获取当前进程的PIDlong pid = ProcessHandle.current().pid();
 
// 获取当前进程的所有子进程
Stream<ProcessHandle> children = ProcessHandle.current().children();
 
// 获取进程的存活状态
boolean isProcessAlive = ProcessHandle.current().isAlive();
 
// 通过静态方法获取所有进程
ProcessHandle.allProcesses().forEach(new Consumer<ProcessHandle>() {
  @Override
  public void accept(ProcessHandle processHandle) {
    if (processHandle.pid() == pid) {
      System.out.println("I'm the Current Process:" + pid + "\n" + processHandle.info());
    }
  }
});
 ```

### 新特性8：改进了Stream API
Stream 接口新增方法：
```
default Stream<T> dropWhile(Predicate<? super T> predicate)
default Stream<T> takeWhile(Predicate<? super T> predicate)
static <T> Stream<T> ofNullable(T t)
static <T> Stream<T> iterate(T seed, Predicate<? super T> hasNext, UnaryOperator<T> next)
```
用法举例：
```
// 以下代码片段会生成包含1到100之间的所有整数的流：
Stream.iterate(1, n -> n <= 100, n -> n + 1)
 
 
// 以下代码输出 5, 6, 7
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
list.stream().dropWhile(x -> x < 5).forEach(System.out::println);
```
Collectors 类 新增方法 filtering 和 flatMapping：

```
<T,A,R> Collector<T,?,R> filtering(Predicate<? super T> predicate, Collector<? super T,A,R> downstream)
<T,U,A,R> Collector<T,?,R> flatMapping(Function<? super T,? extends Stream<? extends U>> mapper, Collector<? super U,A,R> downstream)
```
用法举例：
```
Map<String, List<String>> keyValues = list.stream()
        .collect(Collectors.toMap(String::valueOf, x -> List.of(String.valueOf(x), String.valueOf(x))));
 
keyValues.forEach((key, value) -> System.out.println("key: " + key + ", value: " + value));/**
 * 输出如下
 * key: 1, value: [1, 1]
 * key: 2, value: [2, 2]
 * key: 3, value: [3, 3]
 * key: 4, value: [4, 4]
 * key: 5, value: [5, 5]
 * key: 6, value: [6, 6]
 * key: 7, value: [7, 7]
 */List<String> values = keyValues.values()
 .stream()
 .collect(Collectors.flatMapping(e -> e.stream().distinct(), Collectors.toList()));values.stream().forEach(System.out::print);// 输出 12345678
 ```

### 新特性9：集合类的工厂方法
在java 9中，分别为List、Set、Map增加了of静态工厂方法，来获取一个不可变的List、Set、Map。不变的对象在单线程和多线程下执行是没有区别的，即不会有线程安全问题，会降低并发编程带来的风险。

Java 8 里：
```
// 声明空集合List<String> emptyList = Collections.emptyList();
// 声明一个单个元素的集合
List<String> singleItemList = Collections.singletonList("singleItem");
// 声明一个不可变的集合
List<String> unmodifiableList = Collections.unmodifiableList(new ArrayList<>(Arrays.asList("item1", "item2")));
```
Java 9里简化了声明：
```
// 声明空集合List<String> emptyList = List.of();
// 声明一个单个元素的集合
List<String> singleItemList = List.of("singleItem");
// 声明一个不可变的集合
List<String> unmodifiableList = List.of("item1", "item2");
```
<font color=#0099ff>建议通过工厂类进行Collection类的声明</font>。除了更简洁和更好阅读之外，这些方法还可以让开发人员不必为选择特定的集合类实现而“头痛”。事实上，从工厂方法返回的集合实现是根据初始化时输入的元素数量高度优化过的。

### 新特性10：Try-With-Resources改进
在 java 7 以前，程序中使用的资源需要被明确地关闭，这个体验有点繁琐。try-with-resources 语句会确保在 try 语句结束时关闭所有资源。实现了java.lang.AutoCloseable 或 java.io.Closeable 的对象都可以做为资源。

Java 7 之前的文件处理代码
```
private static void printFileBeforeJava7() throws IOException {
    InputStream input = null;
    try {
        input = new FileInputStream("file.txt");
        int data = input.read();
        while (data != -1) {
            System.out.print((char) data);
            data = input.read();
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (input != null) {
            input.close();
        }
    }
}
```
Java 7 或 Java 8里面里的文件处理代码
```
private static void printFileJava7OrJava8() throws IOException {
    FileInputStream input = new FileInputStream("file.txt")
     
    // 注意这里需要重新定义资源
    try (FileInputStream newInput = input) {
        int data = newInput.read();
        while (data != -1) {
            System.out.print((char) data);
            data = newInput.read();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
Java 9 里的文件处理代码
```
private static void printFileInJava9() throws IOException {
    FileInputStream input = new FileInputStream("file.txt")
     
    // 注意这里不需要重新定义资源
    try (input) {
        int data = input.read();
        while (data != -1) {
            System.out.print((char) data);
            data = input.read();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### 新特性11：多版本JAR
当新版本的Java出来时，某个类库的所有用户通常需要几年时间才能切换到这个新版本。这意味着类库必须向后兼容需要支持的最旧版本的Java（例如，在许多情况下为Java 6或7）。这实际上意味着很长一段时间内类库开发者无法在代码中使用Java 9的新功能。多版本JAR功能允许开发人员创建仅在特定Java版本上运行库时使用的类的备用版本：

新建一个工程，打包为JAR 内部结构看起来像这样：

```
multiVesionJarTest.jar
├── META-INF
│ └── versions
│ └── 9
│ └── package
│ └── Test.class
├── package
│ └──Test.class
└ └──Main.class
```
Java 8 或更早的版本会在直接使用 package 下面的 Test 类，但新版本会使在 META-INFO/versions 子目录中去找合适的内容来代替默认的，代码更为清爽。

### 新特性12：响应式流 Reactive Streams
JDK9中的 Flow API 对应响应式流规范，响应式流规范是一种事实标准，目标是使用非阻塞背压方式提供一个标准的异步流处理。

更确切地说，响应式流目的是“找到最小的一组接口，方法和协议，用来描述必要的操作和实体以实现这样的目标：以非阻塞背压方式实现数据的异步流”，更详细的介绍，可参考文章：响应式流Reactive Streams

java.util.concurrent.Flow包含的接口如下：

- Flow.Processor（处理器）
- Flow.Publisher（发布者）
- Flow.Subscriber（订阅者）
- Flow.Subscription（订阅管理器）

下面给一个具体的例子:
```
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Flow;
import java.util.concurrent.SubmissionPublisher;
 
/**
 * Created by yves on 2017/11/4.
 */
public class FlowTest {
    private static final ExecutorService PUBLISHE_EXECUTOR = Executors.newFixedThreadPool(1);
    private static final int MAX_CAPACITY = 1024;
 
    public static void main(String[] args) {
 
        // 创建生产者
        SubmissionPublisher<String> oracle = new SubmissionPublisher<>(PUBLISHE_EXECUTOR, MAX_CAPACITY);
 
        // 创建消费组
        StrugglingDeveloper<String> developers = new StrugglingDeveloper<>();
        oracle.subscribe(developers);
 
        // 发布消息
        List.of("2011年7月7日：Java 7发布了", "2014年3月19日：Java 8发布了", "2017年9月21日：Java 9发布了")
                .forEach(oracle::submit);
 
        oracle.close();
 
        System.out.println("<Ready to publish item..>");
        PUBLISHE_EXECUTOR.submit(() -> {
            //发布任务结束
        });
 
        PUBLISHE_EXECUTOR.shutdownNow();
 
    }
 
    static class StrugglingDeveloper<T> implements Flow.Subscriber<T> {
 
        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            subscription.request(5);
            System.out.println("<Start to Receive>");
        }
 
        @Override
        public void onNext(T item) {
            System.out.println("<Received>:" + item);
        }
 
        @Override
        public void onError(Throwable t) {
            t.printStackTrace();
        }
 
        @Override
        public void onComplete() {
            System.out.println("<onComplete>");
        }
    }
}
```
输出如下：
```
<Start to Receive>
<Ready to publish item..>
<Received>:2011年7月7日：Java 7发布了
<Received>:2014年3月19日：Java 8发布了
<Received>:2017年9月21日：Java 9发布了
<onComplete>
```

### 新特性13：代码分段缓存
Java 9的另一个性能提升来自于 JIT(Just-in-time) 编译器. 当某段代码被大量重复执行的时候, 虚拟机会把这段代码编译成机器码(native code)并储存在代码缓存里面, 进而通过访问缓存中不同分段的代码来提升编译器的效率。

和原来的单一缓存区域不同的是, 新的代码缓存根据代码自身的生命周期而分为三种:

- 永驻代码(JVM 内置 / 非方法代码)
- 短期代码(仅在某些条件下适用的配置性(profiled)代码)
- 长期代码(非配置性代码)
- 缓存分段会在各个方面提升程序的性能, 比如做垃圾回收扫描的时候可以直接跳过非方法代码(永驻代码), 从而提升效率

更多内容，请参考：[JEP 197: Segmented Code Cache](http://openjdk.java.net/jeps/197)

### 新特性14：其他新特性
- 轻量级 JSON API
- Optional的流式处理
- 改进的Javadoc（支持搜索 + 支持HTML5）
- 改进了竞争锁 ([JEP 143: Improve Contended Locking](http://openjdk.java.net/jeps/143))