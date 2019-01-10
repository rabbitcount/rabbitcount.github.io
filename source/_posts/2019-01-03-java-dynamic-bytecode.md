---
title: java动态字节码技术
date: 2019-01-03 11:53:00
tags:
- bytecode
- 字节码
- btrace
- visitor
- jvm
---
1. **读、写、修改字节码（byte code）**；
基于 ASM 的 `ClassReader， ClassWriter， ClassVisitor`
2. **自定义类转换器，供JVM加载重新生成后的字节码（byte code）**；
通过java类库 `instrument`，JVM通过加载 `ClassFileTransformer.transform()` 方法返回的 `byte[]`，实现重新加载；
3. **JVM调用自定义类转换器的实现方式（JVM TI，JPDA组成部分）**；
基于JVM TI，使用自定义的VM Agent
<!-- more --> 

---
{% note info %} 本文所有jdk文档，均使用jdk10 {% endnote %}

# 简单来说
动态字节码技术：如何改变原有代码的行为
> 修改字节码 + 重新加载字节码

## 读、写、修改字节码
```java
TestClassVisitor
```
读写字节码常见工具：BCEL、Javassist、ASM（使用Visitor模式接口）、CGLib（底层为ASM实现）
此处关注通过ASM对字节码进行读写和修改；实现代码见页尾；

### ASM 使用方式
ASM实现中采用访问者模式（Visitor Pattern），ASM 将对代码的读取和操作都包装成一个访问者，在解析 JVM 加载到的字节码时调用。核心入口为以下三个类：`ClassReader， ClassWriter， ClassVisitor`
> ClassReader 是 ASM 代码的入口，通过它解析二进制字节码，实例化时它时，我们需要传入一个 ClassVisitor，在这个 Visitor 里，我们可以实现 `visitMethod()/visitAnnotation()` 等方法，用以定义对类结构（如方法、字段、注解）的访问方法。
> 
> 而 ClassWriter 接口继承了 ClassVisitor 接口，我们在实例化类访问器时，将 ClassWriter "注入" 到里面，以实现对类写入的声明。

### 使用插件获得字节码
> Q：如何快速生成asm代码？
> A：`ASM Bytecode Outline` 插件，当前较新的版本是 `ASM Bytecode Outline 2017`

编译后，在指定的类上右键选择 `Show bytecode Outline`
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-03-java-dynamic-bytecode/java-dynamic-bytecode-asm-plugin.png)

## 自定义类转换器
```java
TestTransformer
```
为实现JVM动态加载修改后的字节码，可以依靠 java 的类库 `instrument`，通过`instrument` 类库可以修改已加载的类文件；

|       |         版本差别    |
|:----------:|:-------------|
| 1.6前 | 只能在 JVM 刚启动开始加载类时生效 | 
| 1.6后（含1.6）| 支持了在运行时对类定义的修改 | 

### 使用`instrument`
要使用 `instrument` 的类修改功能，我们需要实现它的 `ClassFileTransformer` 接口定义一个类文件转换器。它唯一的一个 `transform()` 方法会在类文件被加载时调用，在 `transform` 方法里，我们可以对传入的二进制字节码进行改写或替换，生成新的字节码数组后返回，JVM 会使用 `transform` 方法返回的字节码数据进行类的加载。
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-03-java-dynamic-bytecode/ClassFileTransformer%23transform.png)

## JVM调用类转换器的机制
```java
TestAgent， Attacher
```
我们可以通过JVM TI 加载修改后的字节码，此处使用 Java Agent 实现

[参考 java.lang.instrument, Agent](/2019/01/07/java-lang-instrument/#%E7%BC%96%E5%86%99-Agent)

# 源码
## 源码汇总
### TransformTarget
TransformTarget 是要被修改的目标类，正常执行时，它会三秒输出一次 “hello”。
```java
public class TransformTarget {
    public static void main(String[] args) {
        while (true) {
            try {
                Thread.sleep(3000L);
            } catch (Exception e) {
                break;
            }
            printSomething();
        }
    }

    public static void printSomething() {
        System.out.println("hello");
    }
}
```
### Agent 及打包方式
Agent 是执行修改类的主体，它使用 ASM 修改 TransformTarget 类的方法，并使用 instrument 包将修改提交给 JVM。
入口类，也是代理的 Agent-Class。
```java
public class TestAgent {
    public static void agentmain(String args, Instrumentation inst) {
        inst.addTransformer(new TestTransformer(), true);
        try {
            inst.retransformClasses(TransformTarget.class);
            System.out.println("Agent Load Done.");
        } catch (Exception e) {
            System.out.println("agent load failed!");
        }
    }
}
```

执行字节码修改和转换的类。
```java
public class TestTransformer implements ClassFileTransformer {

    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        System.out.println("Transforming " + className);
        ClassReader reader = new ClassReader(classfileBuffer);
        ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        ClassVisitor classVisitor = new TestClassVisitor(Opcodes.ASM5, classWriter);
        reader.accept(classVisitor, ClassReader.SKIP_DEBUG);
        return classWriter.toByteArray();
    }

    class TestClassVisitor extends ClassVisitor implements Opcodes {
        TestClassVisitor(int api, ClassVisitor classVisitor) {
            super(api, classVisitor);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
            MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);
            if (name.equals("printSomething")) {
                mv.visitCode();
                Label l0 = new Label();
                mv.visitLabel(l0);
                mv.visitLineNumber(19, l0);
                mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                mv.visitLdcInsn("bytecode replaced!");
                mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
                Label l1 = new Label();
                mv.visitLabel(l1);
                mv.visitLineNumber(20, l1);
                mv.visitInsn(Opcodes.RETURN);
                mv.visitMaxs(2, 0);
                mv.visitEnd();
                TransformTarget.printSomething();
            }
            return mv;
        }
    }
}
```
Agent打包Maven插件
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-assembly-plugin</artifactId>
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
Edit configuration
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-03-java-dynamic-bytecode/assembly-agent.png)

### Attacher

```java
public class Attacher {
    public static void main(String[] args) throws AttachNotSupportedException, IOException, AgentLoadException, AgentInitializationException {
        VirtualMachine vm = VirtualMachine.attach("${TransformTarget进程的PID}");
        vm.loadAgent("${/path/to/TestAgent.jar}");
    }
}
```

## 源码执行方式
### 前置准备
打包 Agent；
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-03-java-dynamic-bytecode/dynamic-bytecode-assembly-agent.png)
### 启动顺序
1. 启动 TransformTarget；
2. 查询 TransformTarget 的进程号，更新 Attacher 中的进程ID并启动；
### 执行结果
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-03-java-dynamic-bytecode/dynamic-byte-code-result.png)

