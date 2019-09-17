title: Arthas 源码
date: 2019-09-11 19:12:50
tags: [arthas]
categories: []
---
# arthas简介

arthas 是Alibaba开源的Java诊断工具，基于`jvm Agent`方式，使用`Instrumentation`方式修改字节码方式以及使用`java.lang.management`包提供的管理接口的方式进行java应用诊断。详细的介绍可以参考官方文档。
官方文档地址： [https://alibaba.github.io/arthas/](https://alibaba.github.io/arthas/)
GitHub地址： [https://github.com/alibaba/arthas/](https://github.com/alibaba/arthas/)
本文主要分析arthas源码，主要分成下面几个部分：

1.  arthas组成模块
2.  arthas服务端代码分析
3.  arthas客户端代码分析

# arthas组成模块

arthas有多个模块组成，如下图所示：

![](https://upload-images.jianshu.io/upload_images/1324111-18a6654e4b1dcd08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/752/format/webp)

arthas模块图.png

1.  arthas\-boot.jar和as.sh模块功能类似，分别使用java和shell脚本，下载对应的jar包，并生成服务端和客户端的启动命令，然后启动客户端和服务端。服务端最终生成的启动命令如下：

```
${JAVA_HOME}"/bin/java \
     ${opts}  \
     -jar "${arthas_lib_dir}/arthas-core.jar" \
         -pid ${TARGET_PID} \             要注入的进程id
         -target-ip ${TARGET_IP} \       服务器ip地址
         -telnet-port ${TELNET_PORT} \  服务器telnet服务端口号
         -http-port ${HTTP_PORT} \      websocket服务端口号
         -core "${arthas_lib_dir}/arthas-core.jar" \      arthas-core目录
         -agent "${arthas_lib_dir}/arthas-agent.jar"    arthas-agent目录

```

2.  arthas\-core.jar是服务端程序的启动入口类，会调用`virtualMachine#attach`到目标进程，并加载arthas\-agent.jar作为agent jar包。
3.  arthas\-agent.jar既可以使用premain方式（在目标进程启动之前，通过\-agent参数静态指定），也可以通过agentmain方式（在进程启动之后attach上去）。arthas\-agent会使用自定义的classloader(`ArthasClassLoader`)加载arthas\-core.jar里面的`com.taobao.arthas.core.config.Configure`类以及`com.taobao.arthas.core.server.ArthasBootstrap`。 同时程序运行的时候会使用arthas\-spy.jar。
4.  arthas\-spy.jar里面只包含Spy类，目的是为了将Spy类使用`BootstrapClassLoader`来加载，从而使目标进程的java应用可以访问Spy类。通过ASM修改字节码，可以将Spy类的方法`ON_BEFORE_METHOD`， `ON_RETURN_METHOD`等编织到目标类里面。Spy类你可以简单理解为类似spring aop的Advice，有前置方法，后置方法等。
5.  arthas\-client.jar是客户端程序，用来连接arthas\-core.jar启动的服务端代码，使用telnet方式。一般由arthas\-boot.jar和as.sh来负责启动。

# arthas服务端代码分析

## 前置准备

看服务端启动命令可以知道 从 arthas\-core.jar开始启动，arthas\-core的pom.xml文件里面指定了mainClass为`com.taobao.arthas.core.Arthas`，使得程序启动的时候从该类的main方法开始运行。Arthas源码如下：

```
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

1.  Arthas首先解析入参，生成`com.taobao.arthas.core.config.Configure`类，包含了相关配置信息
2.  使用jdk\-tools里面的`VirtualMachine.loadAgent`，其中第一个参数为agent路径， 第二个参数向jar包中的agentmain()方法传递参数（此处为agent\-core.jar包路径和config序列化之后的字符串），加载arthas\-agent.jar包，并运行
3.  arthas\-agent.jar包，指定了Agent\-Class为`com.taobao.arthas.agent.AgentBootstrap`，同时可以使用Premain的方式和目标进程同时启动

```
<manifestEntries>
    <Premain-Class>com.taobao.arthas.agent.AgentBootstrap</Premain-Class>
    <Agent-Class>com.taobao.arthas.agent.AgentBootstrap</Agent-Class>
</manifestEntries>

```

其中`Premain-Class`的`premain`和`Agent-Class`的`agentmain`都调用main方法。
main方法主要做4件事情：

1.  找到arthas\-spy.jar路径，并调用`Instrumentation#appendToBootstrapClassLoaderSearch`方法，使用`bootstrapClassLoader`来加载arthas\-spy.jar里的Spy类。
2.  arthas\-agent路径传递给自定义的classloader(`ArthasClassloader`)，用来隔离arthas本身的类和目标进程的类。
3.  使用 `ArthasClassloader#loadClass`方法，加载`com.taobao.arthas.core.advisor.AdviceWeaver`类，并将里面的`methodOnBegin`、`methodOnReturnEnd`、`methodOnThrowingEnd`等方法取出赋值给Spy类对应的方法。同时Spy类里面的方法又会通过ASM字节码增强的方式，编织到目标代码的方法里面。使得Spy 间谍类可以关联由`AppClassLoader`加载的目标进程的业务类和`ArthasClassloader`加载的arthas类，因此Spy类可以看做两者之间的桥梁。根据classloader双亲委派特性，子classloader可以访问父classloader加载的类。源码如下：

```
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
        Method beforeInvoke = adviceWeaverClass.getMethod(BEFORE_INVOKE, int.class, String.class, String.class, String.class);
        Method afterInvoke = adviceWeaverClass.getMethod(AFTER_INVOKE, int.class, String.class, String.class, String.class);
        Method throwInvoke = adviceWeaverClass.getMethod(THROW_INVOKE, int.class, String.class, String.class, String.class);
        Method reset = AgentBootstrap.class.getMethod(RESET);
        Spy.initForAgentLauncher(classLoader, onBefore, onReturn, onThrows, beforeInvoke, afterInvoke, throwInvoke, reset);
    }

```

classloader关系如下：

```
+-BootstrapClassLoader
+-sun.misc.Launcher$ExtClassLoader@7bf2dede
  +-com.taobao.arthas.agent.ArthasClassloader@51a10fc8
  +-sun.misc.Launcher$AppClassLoader@18b4aac2

```

4.  异步调用bind方法，该方法最终启动server监听线程，监听客户端的连接，包括telnet和websocket两种通信方式。源码如下：

```
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
            int javaPid = (Integer) classOfConfigure.getMethod(GET_JAVA_PID).invoke(configure);
            Class<?> bootstrapClass = agentLoader.loadClass(ARTHAS_BOOTSTRAP);
            Object bootstrap = bootstrapClass.getMethod(GET_INSTANCE, int.class, Instrumentation.class).invoke(null, javaPid, inst);
            boolean isBind = (Boolean) bootstrapClass.getMethod(IS_BIND).invoke(bootstrap);
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

主要做两件事情：

*   使用`ArthasClassloader`加载`com.taobao.arthas.core.config.Configure`类(位于arthas\-core.jar)，并将传递过来的序列化之后的config，反序列化成对应的`Configure`对象。
*   使用`ArthasClassloader`加载`com.taobao.arthas.core.server.ArthasBootstrap`类（位于arthas\-core.jar），并调用`bind`方法。

## 启动服务器，并监听客户端请求

下面重点看下`com.taobao.arthas.core.server.ArthasBootstrap#bind`方法

```
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
                shellServer.registerTermServer(new HttpTermServer(configure.getIp(), configure.getHttpPort(),
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

```
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
        commands.add(Command.create(DashboardCommand.class));
        commands.add(Command.create(DumpClassCommand.class));
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

调用`shellServer.registerTermServer`，`shellServer.registerTermServer`，`shellServer.registerCommandResolve` 注册到`ShellServer`里，`ShellServer`是整个服务端的门面类，调用`listen`方法启动`ShellServer`。
`ShellServer`会使用一系列的类，细节比较复杂，可以见下面的类图。

![](https://upload-images.jianshu.io/upload_images/1324111-126a2a2d6faddb65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/906/format/webp)

Arthas\-服务端类图.png

`ShellServer#listen`会调用所有注册的`TermServer的listen`方法，比如`TelnetTermServer`。然后`TelnetTermServer`的`listen`方法会注册一个回调类，该回调类在有新的客户端连接时会调用`TermServerTermHandler`的`handle`方法处理。

```
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

```
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

`ShellLineHandler`的`handle`方法会根据不同的请求命令执行不同的逻辑：

1.  如果是exit,logout,quit, jobs,fg,bg,kill等直接执行。
2.  如果是其他的命令，则创建Job，并运行。创建Job的类图如下：

    ![](https://upload-images.jianshu.io/upload_images/1324111-8d234fd5c43829c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/903/format/webp)

    服务端\-创建job类图.png

    步骤比较多，就不一一细讲，总之：

3.  创建`Job`时，会根据具体客户端传递的命令，找到对应的`Command`，并包装成`Process`, `Process`再被包装成Job。
4.  运行`Job`时，反向先调用`Process`，再找到对应的`Command`，最终调用`Command`的`process`处理请求。

## Command处理流程

`Command`主要分为两类：

1.  不需要使用字节码增强的命令
    其中JVM相关的使用 `java.lang.management` 提供的管理接口，来查看具体的运行时数据。比较简单，就不介绍了。
2.  需要使用字节码增强的命令
    字节码增强的命令，可以参考下图：

    ![](https://upload-images.jianshu.io/upload_images/1324111-d1f5781d49aecef0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

    arthas\-command相关类图.png

字节码增加的命令统一继承`EnhancerCommand`类，`process`方法里面调用`enhance`方法进行增强。调用`Enhancer`类`enhance`方法，该方法内部调用`inst.addTransformer`方法添加自定义的`ClassFileTransformer`，这边是`Enhancer`类。

`Enhancer`类使用`AdviceWeaver`(继承`ClassVisitor`)，用来修改类的字节码。重写了`visitMethod`方法，在该方法里面修改类指定的方法。`visitMethod`方法里面使用了`AdviceAdapter`（继承了`MethodVisitor`类），在`onMethodEnter`方法, `onMethodExit`方法中，把`Spy`类对应的方法（`ON_BEFORE_METHOD`， `ON_RETURN_METHOD`， `ON_THROWS_METHOD`等）编织到目标类的方法对应的位置。

在前面`Spy`初始化的时候可以看到，这几个方法其实指向的是`AdviceWeaver`类的`methodOnBegin`， `methodOnReturnEnd`等。在这些方法里面都会根据`adviceId`查找对应的`AdviceListener`，并调用`AdviceListener`的对应的方法，比如`before`,`afterReturning`, `afterThrowing`。

通过这种方式，可以实现不同的`Command`使用不同的`AdviceListener`，从而实现不同的处理逻辑。下面找几个常用的`AdviceListener`介绍下:

1.  `StackAdviceListener`
    在方法执行前，记录堆栈和方法的耗时。
2.  `WatchAdviceListener`
    满足条件时打印打印参数或者结果，条件表达式使用Ognl语法。
3.  `TraceAdviceListener`
    在每个方法前后都记录，并维护一个调用树结构。

# arthas客户端代码分析

客户端代码在arthas\-client模块里面，入口类是`com.taobao.arthas.client.TelnetConsole`。主要使用apache commons\-net jar进行telnet连接，关键的代码有下面几步：

1.  构造`TelnetClient`对象，并初始化
2.  构造`ConsoleReader`对象，并初始化
3.  调用`IOUtil.readWrite(telnet.getInputStream(), telnet.getOutputStream(), System.in, consoleReader.getOutput())`处理各个流，一共有四个流：

*   `telnet.getInputStream()`
*   `telnet.getOutputStream()`
*   `System.in`
*   `consoleReader.getOutput()`

请求时：从本地`System.in`读取，发送到 `telnet.getOutputStream()`，即发送给远程服务端。
响应时：从 `telnet.getInputStream()`读取远程服务端发送过来的响应，并传递给 `consoleReader.getOutput()`，即在本地控制台输出。