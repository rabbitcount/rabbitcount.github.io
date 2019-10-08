title: Arthas 源码
date: 2019-09-11 19:12:50
tags: [arthas]
categories: []
---
arthas 是Alibaba开源的Java诊断工具，基于`jvm Agent`方式，使用`Instrumentation`方式修改字节码方式以及使用`java.lang.management`包提供的管理接口的方式进行java应用诊断。详细的介绍可以参考官方文档。
官方文档地址： [https://alibaba.github.io/arthas/](https://alibaba.github.io/arthas/)
GitHub地址： [https://github.com/alibaba/arthas/](https://github.com/alibaba/arthas/)
<!-- more -->

# 组成模块

arthas有多个模块组成，如下图所示：

![arthas模块图.png](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-09-11-narration-of-arthas/arthas.png)

##  **arthas\-boot.jar** 和 **as.sh** 
模块功能类似，分别使用java和shell脚本，下载对应的jar包（如 arthas\-core.jar 或 arthas\-agent.jar），并生成服务端和客户端的启动命令，然后启动客户端和服务端。

##  **arthas\-core.jar** 
是服务端程序的启动入口类（入口main函数位于 `com.taobao.arthas.core.Arthas`），会调用`virtualMachine#attach`到目标进程，并加载arthas\-agent.jar作为agent jar包。

##  **arthas\-agent.jar**
> 既可以使用 **premain** 方式（**在目标进程启动之前，通过\-agent参数静态指定**）；
> 也可以通过 **agentmain** 方式（**在进程启动之后attach上去**）。
   
arthas\-agent 会使用自定义的 classloader(`ArthasClassLoader`) 加载 arthas\-core.jar 里面的 `com.taobao.arthas.core.config.Configure` 类，以及 `com.taobao.arthas.core.server.ArthasBootstrap`。 
    同时程序运行的时候会使用arthas\-spy.jar。

##  **arthas\-spy.jar** 
只包含 Spy 类，目的是为了将 Spy 类使用 `BootstrapClassLoader` 来加载，从而使目标进程的 java 应用可以访问 Spy 类。通过 ASM 修改字节码，可以将 Spy 类的方法`ON_BEFORE_METHOD`， `ON_RETURN_METHOD`等编织到目标类里面。Spy类你可以简单理解为类似spring aop的Advice，有前置方法、后置方法等。
##  **arthas\-client.jar** 
客户端程序（入口类 `com.taobao.arthas.client.TelnetConsole`），用来连接arthas\-core.jar启动的服务端代码，使用 telnet 方式。一般由 arthas\-boot.jar 和 as.sh 来负责启动。

# 服务端代码分析

## 读取配置、加载AgentBootstrap、ArthasBootstrap
> **关键字**：`arthas-core.jar`  `Arthas.class` `com.taobao.arthas.agent.AgentBootstrap` `com.taobao.arthas.core.server.ArthasBootstrap`
> 主要做两件事情：
> - 加载 `Configure` 配置 并运行 `com.taobao.arthas.agent.AgentBootstrap`
> 使用`ArthasClassloader`加载`com.taobao.arthas.core.config.Configure`类(位于arthas\-core.jar)，并将传递过来的序列化之后的config，反序列化成对应的`Configure`对象。
> - 通过 `com.taobao.arthas.agent.AgentBootstrap` 加载 `com.taobao.arthas.core.server.ArthasBootstrap`
> 使用`ArthasClassloader`加载`com.taobao.arthas.core.server.ArthasBootstrap`类（位于arthas\-core.jar），并调用其中`bind`方法完成进程绑定。

从 as.sh 的启动命令可知，是从 `arthas-core.jar` 开始启动，arthas\-core 的 pom.xml 文件里面指定了 mainClass 为 `com.taobao.arthas.core.Arthas` ，使得程序启动的时候从该类的 main 方法开始运行。Arthas源码如下：

```java
public class Arthas {

    private Arthas(String[] args) throws Exception {
        attachAgent(parse(args));
    }

    private Configure parse(String[] args) {
        // 省略非关键代码，解析启动参数作为配置，并填充到configure对象里面
        return configure;
    }

    private void attachAgent(Configure configure) throws Exception {
           // 省略非关键代码，attach到目标进程
          virtualMachine = VirtualMachine.attach("" + configure.getJavaPid());
          virtualMachine.loadAgent(configure.getArthasAgent(),
                            configure.getArthasCore() + ";" + configure.toString());
    }

    public static void main(String[] args) {
            new Arthas(args);
    }
}

```

### 解析入参为 `Configure`
Arthas首先解析入参，生成`com.taobao.arthas.core.config.Configure`类；`Configure` 包含了相关配置信息；
### 加载 `arthas-agent.jar`
使用 `jdk-tools` 里面的`VirtualMachine.loadAgent`；加载 `arthas-agent.jar` 包，并运行；
- 第一个参数为agent路径；
- 第二个参数向jar包中的agentmain()方法传递参数（此处为agent\-core.jar包路径和config序列化之后的字符串）；

### 运行 `arthas-agent.jar` 中的 `AgentBootstrap`
arthas\-agent.jar包，指定了Agent\-Class为`com.taobao.arthas.agent.AgentBootstrap`，同时可以使用Premain的方式和目标进程同时启动
```xml
<manifestEntries>
    <Premain-Class>com.taobao.arthas.agent.AgentBootstrap</Premain-Class>
    <Agent-Class>com.taobao.arthas.agent.AgentBootstrap</Agent-Class>
</manifestEntries>
```

其中 `Premain-Class` 的 `premain` 和 `Agent-Class` 的 `agentmain` 都调用main方法。

#### `AgentBootstrap` 功能
`AgentBootstrap` 的 main方法主要做4件事情：
##### 加载 `arthas-spy.jar`
找到 arthas\-spy.jar 路径，并调用 `Instrumentation#appendToBootstrapClassLoaderSearch` 方法，使用 `bootstrapClassLoader` 来加载 arthas\-spy.jar 里的 Spy 类。
#####  自定义classLoader加载`arthas-agent`
arthas\-agent 路径传递给自定义的 classloader(`ArthasClassloader`) ，用来隔离 arthas 本身的类和目标进程的类。
##### 加载 `AdviceWeaver`，并传递给Spy
`loadOrDefineClassLoader` 方法使用 `ArthasClassloader#loadClass` 方法，加载 `com.taobao.arthas.core.advisor.AdviceWeaver` 类，并将里面的 `methodOnBegin`、`methodOnReturnEnd`、`methodOnThrowingEnd` 等方法取出赋值给 Spy 类对应的方法。同时 Spy 类里面的方法又会通过 ASM 字节码增强的方式，编织到目标代码的方法里面。使得 Spy  间谍类可以关联由 `AppClassLoader` 加载的目标进程的业务类和 `ArthasClassloader` 加载的 arthas 类，因此 Spy 类可以看做两者之间的桥梁。根据 classloader 双亲委派特性，子 classloader 可以访问父 classloader 加载的类。源码如下：
```java
private static ClassLoader getClassLoader(Instrumentation inst, File spyJarFile, File agentJarFile) throws Throwable {
    // 将Spy添加到BootstrapClassLoader
    inst.appendToBootstrapClassLoaderSearch(new JarFile(spyJarFile));
    
    // 构造自定义的类加载器ArthasClassloader，尽量减少Arthas对现有工程的侵蚀
    return loadOrDefineClassLoader(agentJarFile);
}
    
private static void initSpy(ClassLoader classLoader) throws ClassNotFoundException, NoSuchMethodException {
    // 该classLoader为ArthasClassloader
    Class<?> adviceWeaverClass = classLoader.loadClass(ADVICEWEAVER);
    Method onBefore = adviceWeaverClass.getMethod(ON_BEFORE, int.class, ClassLoader.class, String.class,
            String.class, String.class, Object.class, Object[].class);
    Method onReturn = adviceWeaverClass.getMethod(ON_RETURN, Object.class);
    Method onThrows = adviceWeaverClass.getMethod(ON_THROWS, Throwable.class);
    Method beforeInvoke = adviceWeaverClass.getMethod(BEFORE_INVOint int.class, String.class, String.class, String.class);
    adviceWeaverClass.getMethod(AFTER_INVOKE,   int.class, Str int  int.class, String.class, String.class, String.class);
    Method throwInvoke = adviceWeaverClass.getMethod(THROW_INVOKE, int.class, String.class, String.class, String.class);
    Method reset = AgentBootstrap.class.getMethod(RESET);
    Spy.initFoitFoitFotFooooorAgentLauncher(classLoader, onBefore, onReturn, IInvoke, afterInvoke, throwInvoke, reset);
 }
```
classloader关系如下：
```
+-BootstrapClassLoader
+-sun.misc.Launcher$ExtClassLoader@7bf2dede
	+-coooClassloader@51a10fc8
	+-sun.misc.Lau.Lauuncher$AppClassLo.Lauuncher$AppClassLoader@18b4aac2
```

##### `AgentBootstrap` 异步调用 `ArthasBootstrap#bind` 方法
异步调用 bind 方法，将通过反射调用 `com.taobao.arthas.core.server.ArthasBootstrap` 的 bind 方法，用于启动监听；
```java
Thread bindingThread = new Thread() {
  @Override
  public void run() {
    try {
      bind(inst, agentLoader, agentArgs);
    } catch (Throwable throwable) {
      throwable.printStackTrace(ps);
    }
  }
};
    
private static void bind(Instrumentation inst, ClassLoader agentLoader, String args) throws Throwable {
  /**
  * <pre>
  * Configure configure = Configure.toConfigure(args);
  * int javaPid = configure.getJavaPid();
  * ArthasBootstrap bootstrap = ArthasBootstrap.getInstance(javaPid, inst);
  * </pre>
  */
  Class<?> classOfConfigure = agentLoader.loadClass(ARTHAS_CONFIGURE);
  Object configure = classOfConfigure.getMethod(TO_CONFIGURE, String.class).invoke(null, args);
  int javaPid = (Integer) classOfConfigure.getMethod(GPID).invoke(configure);
  Class<?> bootstraaapClasapClass = agentLoader.lladClass(ARTHAS_BOOTSTRAP);
  Object bootstrap = botratrapClass.getMethod(GET_INSTANCE, inCE, inCE, inE, in ininnstrumentation.class).invoke(null, javaPi          boolean isBind = (Boolean) bootstrapClass.getMethod(IS_BIND).invoke(bootstrap);
  if (!isBind) {
    try {
      ps.println("Arthas start to bind...");
      bootstrapClass.getMethod(BIND, classOfConfigure).invoke(bootstrap, configure);
      ps.println("Arthas server bind success.");
      return;
    } catch (Exception e) {
      ps.println("Arthas server port binding failed! Please check $HOME/logs/arthas/arthas.log for more details.");
      throw e;
    }
  }
  ps.println("Arthas server already bind.");
}
```


## 启动服务器并监听

下面重点看下`com.taobao.arthas.core.server.ArthasBootstrap#bind`方法

```java
/**
 * Bootstrap arthas server
 *
 * @param configure 配置信息
 * @throws IOException 服务器启动失败
 */
public void bind(Configure configure) throws Throwable {

  long start = System.currentTimeMillis();

  if (!isBindRef.compareAndSet(false, true)) {
    throw new IllegalStateException("already bind");
  }

  try {
    ShellServerOptions options = new ShellServerOptions()
          .setInstrumentation(instrumentation)
          .setPid(pid)
          .setSessionTimeout(configure.getSessionTimeout() * 1000);
    shellServer = new ShellServerImpl(options, this);
    BuiltinCommandPack builtinCommands = new BuiltinCommandPack();
    List<CommandResolver> resolvers = new ArrayList<CommandResolver>();
    resolvers.add(builtinCommands);
    // TODO: discover user provided command resolver
    if (configure.getTelnetPort() > 0) {
      // telnet方式的server
      shellServer.registerTermServer(new TelnetTermServer(configure.getIp(), configure.getTelnetPort(),
           options.getConnectionTimeout()));
    } else {
      logger.info("telnet port is {}, skip bind telnet server.", configure.getTelnetPort());
    }
    if (configure.getHttpPort() > 0) {
      // websocket方式的server
      shellServer.registerTermServer(new HttpTermServer(onfigure.getIp(), configure.getHttpPort(),
            options.getConnectionTimeout()));
    } else {
      logger.info("http port is {}, skip bind http server.", configure.getHttpPort());
    }

    for (CommandResolver resolver : resolvers) {
      shellServer.registerCommandResolver(resolver);
    }

    shellServer.listen(new BindHandler(isBindRef));

    logger.info("as-server listening on network={};telnet={};http={};timeout={};", configure.getIp(),
          configure.getTelnetPort(), configure.getHttpPort(), options.getConnectionTimeout());
    // 异步回报启动次数
    UserStatUtil.arthasStart();

    logger.info("as-server started in {} ms", System.currentTimeMillis() - start );
  } catch (Throwable e) {
    logger.error(null, "Error during bind to port " + configure.getTelnetPort(), e);
    if (shellServer != null) {
      shellServer.close();
    }
    throw e;
  }
}
```

可以看到有两种类型的server，`TelnetTermServer`和`HttpTermServer`。同时会在BuiltinCommandPack里添加所有的命令Command，添加命令的源码如下：

```java
public class BuiltinCommandPack implements CommandResolver {

    private static List<Command> commands = new ArrayList<Command>();

    static {
        initCommands();
    }

    @Override
    public List<Command> commands() {
        return commands;
    }

    private static void initCommands() {
        commands.add(Command.create(HelpCommand.class));
        commands.add(Command.create(KeymapCommand.class));
        commands.add(Command.create(SearchClassCommand.class));
        commands.add(Command.create(SearchMethodCommand.class));
        commands.add(Command.create(ClassLoaderCommand.class));
        commands.add(Command.create(JadCommand.class));
        commands.add(Command.create(GetStaticCommand.class));
        commands.add(Command.create(MonitorCommand.class));
        commands.add(Command.create(StackCommand.class));
        commands.add(Command.create(ThreadCommand.class));
        commands.add(Command.create(TraceCommand.class));
        commands.add(Command.create(WatchCommand.class));
        commands.add(Command.create(TimeTunnelCommand.class));
        commands.add(Command.create(JvmCommand.class));
        // commands.add(Command.create(GroovyScriptCommand.class));
        commands.add(Command.create(OgnlCommand.class));
        commands.add(Command.create(DashboadCommand.class));
        commands.add(Command.create(DumpClassCCommand.class));
        commands.add(Command.create(JulyCommand.class));
        commands.add(Command.create(ThanksCommand.class));
        commands.add(Command.create(OptionsCommand.class));
        commands.add(Command.create(ClsCommand.class));
        commands.add(Command.create(ResetCommand.class));
        commands.add(Command.create(VersionCommand.class));
        commands.add(Command.create(ShutdownCommand.class));
        commands.add(Command.create(SessionCommand.class));
        commands.add(Command.create(SystemPropertyCommand.class));
        commands.add(Command.create(SystemEnvCommand.class));
        commands.add(Command.create(RedefineCommand.class));
        commands.add(Command.create(HistoryCommand.class));
    }
}

```

调用`shellServer.registerTermServer`，`shellServer.registerTermServer`，`shellServer.registerCommandResolve` 注册到`ShellServer`里；
`ShellServer`是整个服务端的门面类，调用`listen`方法启动`ShellServer`。


`ShellServer#listen`会调用所有注册的`TermServer的listen`方法，比如`TelnetTermServer`。然后`TelnetTermServer`的`listen`方法会注册一个回调类，该回调类在有新的客户端连接时会调用`TermServerTermHandler`的`handle`方法处理。

```java
bootstrap = new NettyTelnetTtyBootstrap().setHost(hostIp).setPort(port);
  try {
    bootstrap.start(new Consumer<TtyConnection>() {
      @Override
      public void accept(final TtyConnection conn) {
        termHandler.handle(new TermImpl(Helper.loadKeymap(), conn));
      }
    }).get(connectionTimeout, TimeUnit.MILLISECONDS);
    listenHandler.handle(Future.<TermServer>succeededFuture());

```

该方法会接着调用`ShellServerImpl`的`handleTerm`方法进行处理，`ShellServerImpl`的`handleTerm`方法会调用`ShellImpl`的`readline`方法。该方法会注册`ShellLineHandler`作为回调类，服务端接收到客户端发送的请求行之后，会回调`ShellLineHandler`的`handle`方法处理请求。`readline`方法源码如下：

```java
public void readline(String prompt, Handler<String> lineHandler, Handler<Completion> completionHandler) {
  if (conn.getStdinHandler() != echoHandler) {
    throw new IllegalStateException();
  }
  if (inReadline) {
    throw new IllegalStateException();
  }
  inReadline = true;
  // 注册回调类RequestHandler，该类包装了ShellLineHandler，处理逻辑还是在ShellLineHandler类里面
  readline.readline(conn, prompt, new RequestHandler(this, lineHandler), new CompletionHandler(completionHandler, session));
}
```

## 处理客户端请求

`ShellLineHandler` 的 `handle` 方法会根据不同的请求命令执行不同的逻辑：

1. 如果是 exit, logout, quit, jobs, fg, bg, kill 等直接执行。
2. 如果是其他的命令，则创建Job，并运行。创建Job的类图如下：
![Arthas createJob](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-09-11-narration-of-arthas/ShellLinkeHandlerCreateJob.png)
3. 创建 `Job` 时，会根据具体客户端传递的命令，找到对应的 `Command` ，并包装成 `Process` , `Process` 再被包装成Job。
4. 运行 `Job` 时，反向先调用`Process`，再找到对应的 `Command` ，最终调用 `Command` 的 `process` 处理请求。

## Command处理流程

`Command`主要分为两类：

- 不需要使用字节码增强的命令
其中JVM相关的使用 `java.lang.management` 提供的管理接口，来查看具体的运行时数据。
- 需要使用字节码增强的命令
![arthas command asm](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-09-11-narration-of-arthas/arthas-asm.png)

### 字节码增强的 Command
字节码增强的命令统一继承`EnhancerCommand`类；
> `com.taobao.arthas.core.command.monitor200.EnhancerCommand#process` 
> -> `com.taobao.arthas.core.command.monitor200.EnhancerCommand#enhance`
> -> (**静态方法**) `com.taobao.arthas.core.advisor.Enhancer.enhance`

在 静态方法 `Enhancer.enhance` 中：
- 调用 `inst.addTransformer` 添加自定义的 `ClassFileTransformer` （`Enhancer implements ClassFileTransformer`）；
- 在 `ClassFileTransformer#transform` 实现方法中，使用 `com.taobao.arthas.core.advisor.AdviceWeaver` (继承 `ClassVisitor` ) 修改类的字节码；
其中覆写了 `visitMethod` 方法，`visitMethod`方法里面使用了`AdviceAdapter`（继承了`MethodVisitor`类），在`onMethodEnter`方法, `onMethodExit`方法中，把`Spy`类对应的方法（`ON_BEFORE_METHOD`， `ON_RETURN_METHOD`， `ON_THROWS_METHOD`等）编织到目标类的方法对应的位置。

在前面`Spy`初始化的时候可以看到，这几个方法其实指向的是`AdviceWeaver`类的`methodOnBegin`， `methodOnReturnEnd`等。在这些方法里面都会根据`adviceId`查找对应的`AdviceListener`，并调用`AdviceListener`的对应的方法，比如`before`,`afterReturning`, `afterThrowing`。

通过这种方式，可以实现不同的`Command`使用不同的`AdviceListener`，从而实现不同的处理逻辑。下面找几个常用的`AdviceListener`介绍下:
- `StackAdviceListener`
在方法执行前，记录堆栈和方法的耗时。
- `WatchAdviceListener`
满足条件时打印打印参数或者结果，条件表达式使用Ognl语法。
- `TraceAdviceListener`
在每个方法前后都记录，并维护一个调用树结构。

# arthas客户端代码分析

客户端代码在 `arthas-client` 模块里面，入口类是 `com.taobao.arthas.client.TelnetConsole`。主要使用apache commons-net.jar 进行 telnet 连接，关键的代码有下面几步：
1.  构造 `TelnetClient` 对象，并初始化；
2.  构造 `ConsoleReader` 对象，并初始化；
3.  调用 `IOUtil.readWrite(telnet.getInputStream(), telnet.getOutputStream(), System.in, consoleReader.getOutput())` 处理各个流，一共有四个流：
	- `telnet.getInputStream()`
	- `telnet.getOutputStream()`
	- `System.in`
	- `consoleReader.getOutput()`

请求时：从本地 `System.in` 读取，发送到 `telnet.getOutputStream()`，即发送给远程服务端。
响应时：从 `telnet.getInputStream()` 读取远程服务端发送过来的响应，并传递给 `consoleReader.getOutput()`，即在本地控制台输出。