---
title: 初识 java.lang.instrument 包
date: 2019-01-07 17:33:01
tags:
- jvm
- javaagent
---
java.lang.instrument包，`VirtualMachine, InstrumentationImpl, ClassFileTransformer`，及实现原理；
启动Agent的三种方式
1. 命令行启动：-javaagent
2. VM进程启动后运行加载Agent
3. 在可执行的jar包中包含agent
<!-- more -->

---

# 初识 instrument
[Package java.lang.instrument](https://docs.oracle.com/javase/10/docs/api/java/lang/instrument/package-summary.html)

Provides services that allow Java programming language agents to instrument programs running on the JVM. The mechanism for instrumentation is modification of the byte-codes of methods.
An agent is deployed as a JAR file. An attribute in the JAR file manifest specifies the agent class which will be loaded to start the agent. 

## Java Instrument能做什么？最大的作用？

- 使开发者可以构建一个独立于应用程序的代理程序Agent，`用来监控和协助运行在JVM上的程序，更重要的是能够替换和修改某些类的定义`；
- 最大的作用：可以实现一种虚拟机级别支持的AOP实现方式；

## 在JDK 1.5 、1.6中，Java Instrument做了哪些变动支持？
- JDK 1.5：支持 **静态** Instrument，就是在JVM启动前静态设置Instrument；
- JDK 1.6：支持 **动态** Instrument，就是在JVM启动后动态设置Instrument；支持本地代码Instrument；支持动态改变classpath；

## Java Instrument的实现是基于JVM哪种机制？JVM TI是什么，可以做什么？
> 基于JVMTI代理程序；
JVM TI：一套代理程序机制，为JVM相关工具提供的本地编程接口集合；
JVM TI可以支持第三方工具程序以代理的方式连接和访问JVM，并利用JVMTI提供的丰富的编程接口，完成很多跟JVM相关的功能；

## premain、agentmain方法执行时机？
- premain执行时机
在JVM启动时，初始化函数`eventHandlerVMinit`会调用`sun.instrument.InstrumentationImpl`类的`loadClassAndCallPremain`方法，执行Premain-Class指定类的premain方法；
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-07-java-lang-instrument/java-instrument-call-premain.png)
- agentmain执行时机
在JVM启动后，通过 `com.sun.tools.attach.VirtualMachine.loadAgent` 附着一个Instrument，如：`vm.loadAgent(jar)`，会调用`sun.instrument.InstrumentationImpl`类的`loadClassAndCallAgentmain`方法去执行Agentmain-Class指定类的agentmain方法；
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-07-java-lang-instrument/java-lang-instrument-call-agentmain.png)

## premain、agentmain方法中两个参数agentArgs、inst代表什么？分别会有什么作用？
- agentArgs
代理程序命令行中输入参数，随同`-javaagent`一起传入，与main函数不同的是，这个参数是一个 **字符串** 而不是一个字符串数组；
- inst
`java.lang.instrument.Instrumentation`实例，由JVM自动传入，集中了几乎所有功能方法，如：类操作、classpath操作等；

## java.lang.instrument.ClassFileTransformer是什么，有什么作用？
- `ClassFileTransformer` 当中的 `transform` 方法可以对类定义进行操作修改；
- 在类字节码载入JVM前，JVM会调用`ClassFileTransformer.transform` 方法，从而实现对类定义进行操作修改，实现AOP功能；相对于JDK 动态代理、CGLIB等AOP实现技术，不会生成新类，也不需要原类有接口；

## 对于agentmain方法执行，如何进行动态attach agent？
> 通过`VirtualMachine` 加载一个 `Agent`，如：`vm.loadAgent(jar)`；

## META-INF/MAINFEST.MF参数清单？
- `Premain-Class`：指定包含premain方法的类名；
- `Agent-Class`：指定包含agentmain方法的类名；
- `Boot-Class-Path`：指定引导类加载器搜索的路径列表。查找类的特点于平台的机制失败后，引导类加载器会搜索这些路径；
- `Can-Redefine-Class`：是否能重新定义此代理所需的类，默认为false；
- `Can-Retransform-Class`：是否能重新转换此代理所需的类，默认为false；
- `Can-Set-Native-Method-Prefix`：是否能设置此代理所需的本机方法前缀，默认值为false；

## 两个核心API ClassFileTransformer、Instrumention？
- ClassFileTransformer：定义了类加载前的预处理类；
- Instrumentation：增强器
    1. `add/removeTransformer`：添加/删除 `ClasFileTransformer`；
    2. `retransformerClasses`：指定哪些类，在已加载的情况下，重新进行转换处理，即触发重新加载类定义；对于重新加载的类不能修改旧有的类声明，比如：不能增加属性、不能修改方法声明等；
    3. redefineClasses：指定哪些类，触发重新加载类定义，与上面不同的是不会重新进行转换处理，而是把处理结果bytecode直接给JVM；
    4. getAllLoadedClasses：获取当前已加载的Class集合；
    5. getInitiatedClasses：获取由某个特定ClassLoader加载的类定义；
    6. getObjectSize：获得一个对象占用的空间大小
    7. appendToBootstrapClassLoaderSearch/appentToSystemClassLoaderSearch：增加BootstrapClassLoader/SystemClassLoader搜索路径；
    8. isNativeMethodPrefixSupported/SetNativeMethodPrefix：判断JVM是否支持拦截Native Method；
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-07-java-lang-instrument/java-instrument-Instrumention.png)

## Java Instrument工作原理
### 实现原理
1. 在JVM启动时，通过JVM参数`-javaagent`，传入agent jar，Instrument Agent被加载；
2. 在Instrument Agent 初始化时，注册了 JVM TI 初始化函数 `eventHandlerVMInit`；
3. 在JVM启动时，会调用初始化函数`eventHandlerVMInit`，启动了`Instrument Agent，用`sun.instrument.InstrumentationImpl`类里的方法 `loadClassAndCallPremain` 方法去初始化 `Premain-Class` 指定类的 `premain` 方法；
4. 初始化函数`eventHandlerVMinit`，注册了class解析的`ClassFileLoadHook`函数；
5. 在解析Class之前，JVM调用 JVM TI 的 `ClassFileLoadHook` 函数，钩子函数调用`sun.instrument.InstrumentationImpl`类里的`transform`方法，通过`TransformerManager的transformer`方法最终调用我们自定义（Custom）的Transformer类的transform方法；
6. 因为字节码在解析Class之前改的，直接使用修改后的字节码的数据流替代，最后进入Class解析，对整个Class解析无影响；
7. 重新加载Class依然重新走5-6步骤；

> 支持回调的两个阶段

### 依赖函数
依赖函数 `eventHandlerVMInit`， `eventHandlerClassFileLoadHook`说明；
```C
/*
 *  src/java.instrument/share/native/libinstrument/InvocationAdapter.c
 *  JVMTI callback support
 *
 *  We have two "stages" of callback support.
 *  At OnLoad time, we install a VMInit handler.
 *  When the VMInit handler runs, we remove the VMInit handler and install a
 *  ClassFileLoadHook handler.
 */
void JNICALL
eventHandlerVMInit( jvmtiEnv *      jvmtienv,
                    JNIEnv *        jnienv,
                    jthread         thread) {
                    
void JNICALL
eventHandlerClassFileLoadHook(  jvmtiEnv *              jvmtienv,
                                JNIEnv *                jnienv,
                                jclass                  class_being_redefined,
                                jobject                 loader,
                                const char*             name,
                                jobject                 protectionDomain,
                                jint                    class_data_len,
                                const unsigned char*    class_data,
                                jint*                   new_class_data_len,
                                unsigned char**         new_class_data) {
```

# 启动 Agent 的三种方式
## 命令行启动Agent
在命令行中增加option：
```java
-javaagent:<jarpath>[=<options>]
```
## VM 启动后运行 Agent

## Agent在可执行的jar包中
MANIFEST 中需要包含 `Launcher-Agent-Class` 

## 编写 Agent
在三种启动方式中，都需要预先生成 Agent 的 jar 包。
1. Agent 的 jar 包的MANIFEST.MF文件中，必须包含 `Agent-class` 属性，指明 agent class。
2. `agent class`必须实现 `public static agentmain` 方法；

### Agent 类方法
agentmain方法拥有以下两种重构形态，JVM会首先尝试访问双参数的方法，访问失败后，会尝试访问单参数的函数版本；
```java
public static void agentmain(String agentArgs, Instrumentation inst)
public static void agentmain(String agentArgs)
```
如果通过命令行方式启动agent，agent class可能包含 `premain` 方法；当agent晚于VM启动时，不会访问 `premain` 方法；
agent 通过 agentArgs 参数，传递 option 参数

### 生成 MANIFEST 
#### MANIFEST 文档配置
`META-INF -> MANIFEST.MF` 文件中，
```
Agent-Class: ${full-path-of-Agent-class}
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Manifest-Version: 1.0
Permissions: all-permissions
```
#### 通过 Maven 插件生成
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-assembly-plugin</artifactId>
      <executions>
        <execution>
          <id>make-assembly</id>
          <phase>package</phase>
          <goals>
            <goal>single</goal>
          </goals>
          <configuration>
            <archive>
              <manifestEntries>
                <Agent-Class>${full-path-of-Agent-class}</Agent-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
                <Manifest-Version>1.0</Manifest-Version>
                <Permissions>all-permissions</Permissions>
              </manifestEntries>
            </archive>
            <descriptorRefs>
              <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>
          </configuration>
        </execution>
      </executions>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.8.0</version>
      <configuration>
        <source>8</source>
        <target>8</target>
      </configuration>
    </plugin>
  </plugins>
</build>
```
其中 maven-compiler-plugin 的配置不是必须的，但是由于Agent需要区分1.6前或1.6后，所以建议指定jdk的版本；
执行 `mvn clean package` 后，生成的jar包为 `\*-jar-with-dependencies.jar`

### 补充
> The agentmain method should do any necessary initialization required to start the agent. When startup is complete the method should return. If the agent cannot be started (for example, because the agent class cannot be loaded, or because the agent class does not have a conformant agentmain method), the JVM will not abort. If the agentmain method throws an uncaught exception it will be ignored (but may be logged by the JVM for troubleshooting purposes).

