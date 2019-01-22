---
title: Java Class Loader
date: 2019-01-18 10:49:28
tags:
- jvm
- java
- class loader
---
什么是ClassLoader？做了什么？Tomcat、Spring中的使用方式
<!-- more --> 
---

# JVM Class Loader
## ClassLoader是什么？
负责将 Class 的字节码形式转换成内存形式的 Class 对象。字节码可以来自于磁盘文件 `*.class`，也可以是 jar 包里的 `*.class`，也可以来自远程服务器提供的字节流，字节码的本质就是一个字节数组 `byte[]`，符合java规范的字节数组。

- 字节码解析一般流程
> `ClassLoader.load( byte[] )`

- 大多数字节码加密技术，就是在这个过程中执行的：
> 对字节码加密：`byte[] -> encrypt(byte[])`
> 通过定制的ClassLoader读取经过加密的字节码：`CustomClassLoader.load( encrypt(byte[]) )`

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-18-jvm-class-loader/jvm-class-loader-base-self.png)

每个 Class 对象的内部都有一个 classLoader 字段来标识自己是由哪个 ClassLoader 加载的。ClassLoader 在生成 Class 时传入；
```java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
  ···
  private final ClassLoader classLoader;
  private Class(ClassLoader loader) {
    classLoader = loader;
  }
  ···
```

## ClassLoader 基础体系
JVM 运行实例中会存在多个 ClassLoader，不同的 ClassLoader 会从不同的地方加载字节码文件。它可以从不同的文件目录加载，也可以从不同的 jar 文件中加载，也可以从网络上不同的静态文件服务器来下载字节码再加载。
JVM 中内置了三个重要的 ClassLoader，分别是 `BootstrapClassLoader`、`ExtensionClassLoader` 和 `AppClassLoader`。
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-18-jvm-class-loader/hvm-class-loader-hierachy.png)

- **`BootstrapClassLoader`**
`C++` 实现
负责加载 JVM 运行时核心类，这些类位于 `$JAVA_HOME/lib/rt.jar` 文件中，我们常用内置库 `java.xxx.*` 都在里面，比如 `java.util.*`、`java.io.*`、`java.nio.*`、`java.lang.*` 等等。这个 ClassLoader 比较特殊，它是由 C 代码实现的，我们将它称之为「根加载器」。
- **`ExtClassLoader`**
`sun.misc.Launcher.class` 中的静态内部类，Java实现；
引导路径：`System.getProperty("java.ext.dirs")`
负责加载 JVM 扩展类，比如 swing 系列、内置的 js 引擎、xml 解析器 等等，这些库名通常以 javax 开头，它们的 jar 包位于 `$JAVA_HOME/lib/ext/*.jar` 中，有很多 jar 包。
- **`AppClassLoader`**
`sun.misc.Launcher.class` 中的静态内部类，Java实现；
引导路径：`System.getProperty("java.class.path")`
直接面向我们用户的加载器，它会加载 Classpath 环境变量里定义的路径中的 jar 包和目录。我们自己编写的代码以及使用的第三方 jar 包通常都是由它来加载的。
- **`Launcher`** 引导类
引导路径：`System.getProperty("sun.boot.class.path")`

那些位于网络上静态文件服务器提供的 jar 包和 class文件，jdk 内置了一个 `URLClassLoader`，用户只需要传递规范的网络路径给构造器，就可以使用  `URLClassLoader` 来加载远程类库了。`URLClassLoader` 不但可以加载远程类库，还可以加载本地路径的类库，取决于构造器中不同的地址形式。`ExtensionClassLoader` 和 `AppClassLoader` 都是 `URLClassLoader` 的子类，它们都是从本地文件系统里加载类库。

`AppClassLoader` 可以由 `ClassLoader` 类提供的静态方法 `getSystemClassLoader()` 得到，它就是我们所说的「系统类加载器」，我们用户平时编写的类代码通常都是由它加载的。当我们的 main 方法执行的时候，这第一个用户类的加载器就是 `AppClassLoader`。

![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-18-jvm-class-loader/jvm-class-loader-class-hierachy.png)

### 加载器 Launcher 执行顺序
```java
public Launcher() {
  Launcher.ExtClassLoader var1;
  try {
    var1 = Launcher.ExtClassLoader.getExtClassLoader();
  } catch (IOException var10) {
     throw new InternalError("Could not create extension class loader", var10);
  }
  try {
    // 将 ExtClassLoader 作为 parent 传递给 AppClassLoader
    this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
  } catch (IOException var9) {
     throw new InternalError("Could not create application class loader", var9);
  }
  Thread.currentThread().setContextClassLoader(this.loader);
  String var2 = System.getProperty("java.security.manager");
  if (var2 != null) {
    SecurityManager var3 = null;
    if (!"".equals(var2) && !"default".equals(var2)) {
      try {
        var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
      } catch (IllegalAccessException var5) {
      } catch (InstantiationException var6) {
      } catch (ClassNotFoundException var7) {
      } catch (ClassCastException var8) {
      }
    } else {
      var3 = new SecurityManager();
    }
    if (var3 == null) {
      throw new InternalError("Could not create SecurityManager: " + var2);
    }
    System.setSecurityManager(var3);
  }
}
```

## 双亲委派 Parents Delegation Model
所谓的Parents Delegation Model其实就是一种类加载器之间的层次关系，具体的说就是它规定最顶层的类加载器必须为启动类加载器，其余的类加载器必须有自己的父类加载器，且上下层关系的加载器一般是**用组合而不是继承**来实现。
> 某个class loader会优先委派给它的parent classloader加载类；

### 双亲委派实现方式
如果当前类加载器的父类加载器不为空，就先让父类加载器加载 `name` 对应的类，`parent` 成员变量就是第二个问题中类加载器初始化时传递的父类加载器，这就解释了 `ExtClassLoader` 的父类加载器传递的是 `null`，就会执行 `else` 的逻辑，调用 `findBootstrapClassOrNull()` ，而该方法最终为 `native` 方法 `private native Class findBootstrapClass(String name);` ，实际上就是调用openjdk中 BootStrap ClassLoader 的实现去加载该类

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
    synchronized (getClassLoadingLock(name)) {
    // First, check if the class has already been loaded
    Class<?> c = findLoadedClass(name);
    if (c == null) {
      try {
        if (parent != null) {
          c = parent.loadClass(name, false);
        } else {
          c = findBootstrapClassOrNull(name);
        }
      } catch (ClassNotFoundException e) {
        // ClassNotFoundException thrown if class not found
        // from the non-null parent class loader
      }

      if (c == null) {
        // If still not found, then invoke findClass in order
        // to find the class.
        c = findClass(name);
      }
    }
    if (resolve) {
      resolveClass(c);
    }
    return c;
  }
}
```

## 组合而非继承
> 值得注意的是图中的 ExtensionClassLoader 的 parent 指针画了虚线，这是因为它的 parent 的值是 null，当 parent 字段是 null 时就表示它的父加载器是「根加载器」。如果某个 Class 对象的 classLoader 属性值是 null，那么就表示这个类也是「根加载器」加载的。注意这里的 parent 不是 super 不是父类，只是 ClassLoader 内部的字段。

# 动态加载
## Class.forName
我们通常使用 `Class.forName` 的方式，加载JDBC的驱动类
```java
Class.forName("com.mysql.cj.jdbc.Driver");
```
Driver类源自 `mysql-connector-java` 包，6.x.x 以后的版本；
在加载 `Driver` 类后，静态方法会被执行，完成 MySql 的驱动注册；
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    //
    // Register ourselves with the DriverManager
    //
    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    /**
     * Construct a new driver and register it with DriverManager
     * 
     * @throws SQLException
     *             if a database error occurs.
     */
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```

forName 方法同样也是使用调用者 Class 对象的 ClassLoader 来加载目标类。不过 forName 还提供了多参数版本，可以指定使用哪个 ClassLoader 来加载
```java
public static Class<?> forName(String name, boolean initialize,
                                   ClassLoader loader);
```
通过这种形式的 forName 方法可以突破内置加载器的限制，通过使用自定类加载器允许我们自由加载其它任意来源的类库。根据 ClassLoader 的传递性，目标类库传递引用到的其它类库也将会使用自定义加载器加载。

## Class Loader 核心方法
ClassLoader中三个核心方法：
- `loadClass(...)`
加载目标类的入口，它首先会查找当前 ClassLoader 以及它的双亲里面是否已经加载了目标类，如果没有找到就会让双亲尝试加载，如果双亲都加载不了，就会调用 `findClass(...)` 让自定义加载器自己来加载目标类
- `findClass(...)` 
需要子类来覆盖的，不同的加载器将使用不同的逻辑来获取目标类的字节码。拿到这个字节码之后再调用 `defineClass(...)` 方法将字节码转换成 Class 对象
- `defineClass(...)`
调用 defineClass() 方法将字节码转换成 Class 对象；

其他辅助方法包括但不限于
- `findLoadedClass(...)`
充当一个缓存；
当请求 loadClass 装入类时，它调用该方法来查看 ClassLoader 是否已装入这个类，这样可以避免重新装入已存在类所造成的麻烦。应首先调用该方法；
- `findSystemClass(...)`
从本地文件系统装入文件。它在本地文件系统中寻找类文件，如果存在，就使用 `defineClass(...)` 将原始字节转换成 Class 对象，以将该文件转换成类。当运行 Java 应用程序时，这是 JVM 正常装入类的缺省机制。  
- `resolveClass(...)`
ClassLoader 可以不完全地（无 link ）装入类，也可以完全地（link）装入类。当编写我们自己的 loadClass 时，可以通过 `resolve` 参数，决定是否调用 `resolveClass(...)` 方法；
最终通过调用native方法resolve0实现；
```
private native void resolveClass0(Class<?> c);
```

## 自定义 Class Loader
```java
class ClassLoader {

  // 加载入口，定义了双亲委派规则
  Class loadClass(String name) {
    // 是否已经加载了
    Class t = this.findFromLoaded(name);
    if(t == null) {
      // 交给双亲
      t = this.parent.loadClass(name)
    }
    if(t == null) {
      // 双亲都不行，只能靠自己了
      t = this.findClass(name);
    }
    return t;
  }
  
  // 交给子类自己去实现
  Class findClass(String name) {
    throw ClassNotFoundException();
  }
  
  // 组装Class对象
  Class defineClass(byte[] code, String name) {
    return buildClassFromCode(code, name);
  }
}

class CustomClassLoader extends ClassLoader {

  Class findClass(String name) {
    // 寻找字节码
    byte[] code = findCodeFromSomewhere(name);
    // 组装Class对象
    return this.defineClass(code, name);
  }
}
```

## Class.forName vs ClassLoader.loadClass
这两个方法都可以用来加载目标类，它们之间有一个小小的区别，那就是 `Class.forName(...)` 方法可以获取原生类型的 Class，而 `ClassLoader.loadClass(...)` 则会报错。
```java
Class<?> x = Class.forName("[I");
System.out.println(x);

x = ClassLoader.getSystemClassLoader().loadClass("[I");
System.out.println(x);

---------------------
class [I

Exception in thread "main" java.lang.ClassNotFoundException: [I
...
```

# 钻石依赖
项目管理上有一个著名的概念叫着「钻石依赖」，是指软件依赖导致同一个软件包的两个版本需要共存而不能冲突。
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-18-jvm-class-loader/jvm-class-loader-diamond-dependency.png)
我们平时使用的 maven 是这样解决钻石依赖的，它会从多个冲突的版本中选择一个来使用，如果不同的版本之间兼容性很糟糕，那么程序将无法正常编译运行。Maven 这种形式叫「扁平化」依赖管理。
使用 ClassLoader 可以解决钻石依赖问题。不同版本的软件包使用不同的 ClassLoader 来加载，位于不同 ClassLoader 中名称一样的类实际上是不同的类。

# 应用
## 为甚使用反射加载jdbc驱动
```java
Class.forName("com.mysql.cj.jdbc.Driver");
```
JDBC 是 Java 的一种规范，通俗一点说就是 JDK 在 `java.sql.*` 下提供了一系列的接口（interface），但没有提供任何实现（也就是类）。 所以任何人都可以在接口规范之下写自己的 JDBC 实现（如MySQL）。而若调用者也只调用接口上的方法（如我们），那么当未来有任何变更需要时（例如要从 MySQL 迁移至 Oracle ），则理论上不需要对代码做任何修改就能直接切换（可惜SQL语法没能统一规范）

这意味着什么？意味着你的代码中不应该引用任何与实现相关的东西，你的代码只知道 `java.sql.*` ，而不应该知道 `com.mysql.*` 或是 `com.oracle.*` ，以避免或减少未来切换数据源时对代码的变更。

注意，我们使用的所有其他API包括`Connection`/`Statement`/`ResultSet` 等都是`java.sql.*` 的东西，甚至`com.mysql.cj.jdbc.Driver` 类也是：
```java
package com.mysql.jdbc;
public class Driver ... implements java.sql.Driver {
    ...
}
```
因此，直接 `import com.mysql.jdbc.Driver`，违反了开闭原则（OCP，对扩展开放，对修改关闭）。