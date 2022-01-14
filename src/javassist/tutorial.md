
[TOC]
# 读写字节码

Javassist是一个处理Java字节码的类库。ava字节码存储在一个称为class文件的二进制文件中。每个class文件包含一个Java类或接口。

`Javassist.CtClass` 是class的抽象表示。一个 `CtClass` (compiletime class) 对象可以用来处理一个class文件。下面是简单示例：

 ```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("test.Rectangle");
  cc.setSuperclass(pool.get("test.Point"));
  cc.writeFile();
 ```

以上程序首先获取一个`ClassPool`对象，该对象使用Javassist控制字节码修改。`CtClass`对象代表一个class对象，`ClassPool`就是`CtClass`的容器。`ClassPool`根据需要读取class文件来构造`CtClass`对象，并放入池中。可以通过`get()`方法获取`CtClass`对象，然后修改class定义。在上面的例子中，`CtClass`对象代表了`test.Rectangle`的class文件，`ClassPool`的`getDefault()`方法返回的池对象使用系统默认的搜索路径。

`ClassPool`内部有一个缓存（已全类名为key的hash表）存储多个`CtClass`对象，如果从缓存中没有获取到，根据当前的搜索路径，从搜索路径下读取class文件来创建新的`CtClass`对象并放入到缓存中。

`CtClass`对象可以被修改，在上面的例子中，`CtClass`所代表的`test.Rectangle`的class文件的父类被改成了`test.Point`，通过`writeFile()`输出class文件，并替换原来的class文件。

`writeFile()` 将 `CtClass` 对象转换为class文件并写入到本地硬盘。Javassist同时提供了 `toBytecode()`方法可以将其直接转化为字节码：

 ```java
  byte[] b = cc.toBytecode();
 ```

或者直接加载进虚拟机：

 ```java
  Class clazz = cc.toClass();
 ```

`toClass()` 请求当前线程上下文的类加载器来由 `CtClass`表示的类文件。返回一个`java.lang.Class` 对象。

## 定义一个新Class

要从头定义一个新类， 可以调用`ClassPool`的`makeClass()`方法。

 ```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.makeClass("Point");
 ```

此方法定义了一个新class `Point` including no members。`Point`的方法可以通过`CtNewMethod`的工厂方法创建，并通过`Point`的`addMethod()`方法添加到`CtClass`中。

`makeClass()`不能创建接口。可以通过`ClassPool`的`makeInterface()`方法创建接口。接口的方法可以通过`CtNewMethod`的`abstractMethod()`方法创建。注意接口的方法必须为抽象方法。

## 冻结classes

如果一个`CtClass`对象已经通过`writeFile()`, `toClass()`,或者`toBytecode()`转换成了class文件，Javassist将冻结`CtClass`对象。后续对`CtClass`对象的修改都是禁止的。这是为了警告开发者当一个class文件被JVM加载后，是不能再次重新加载的。

冻结的`CtClass`可以被解冻，这样就能继续修改类定义：

```java
  CtClasss cc = ...;
      :
  cc.writeFile();
  cc.defrost();
  cc.setSuperclass(...);    // OK since the class is not frozen.
```

如果`ClassPool.doPruning`被设置为`true`, 那么当Javassist冻结CtClass对象时，为了减少内存消耗，Javassist将修建`CtClass`中的不必要的数据结构(`attribute_info`结构)。例如，Code_attribute结构(方法体)被丢弃。因此，在`CtClass`对象被修剪之后，除了方法名、签名和注释之外，方法的字节码是不可访问的。修剪过的`CtClass`对象不能再次解冻。`ClassPool.doPruning`的默认值是`false`。

要阻止指定`CtClass`的修剪动作，必须提前调用`stopPruning()`方法：

```java
  CtClasss cc = ...;
  cc.stopPruning(true);
      :
  cc.writeFile();                             // convert to a class file.
  // cc is not pruned.
```

上面的`CtClass`对象没有被修剪，因此可以在`writeFile()`之后解冻。

> **Note:** 在DEBUG的时候，如果配置的`ClassPool.doPruning`为`true`，你又想暂时停止修剪和冻结，可以通过`debugWriteFile()`方法输出文件，此方法可以将配置改为`false`，并在输出完文件后解冻，然后再把配置改为`true`。

## Class搜索路径

`ClassPool.getDefault()`方法返回的池对象，使用和当前JVM一样的相同路径搜索class文件，如果程序在服务器的Tomcat或者JBoss上面运行，池对象有可能找不到用户的class文件，因为程序会使用多个自定义类加载器和系统类加载器，在这种情况下必须向池对象注册一个额外的类路径：

```java
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath(new ClassClassPath(this.getClass()));
```
上面的语句表示`this`对象所在的类路径将被注册到池对象，可以使用任何Class对象作为参数，而不是使用`this.getClass()`。

也可以直接注册路径:

```java
  ClassPool pool = ClassPool.getDefault();
  pool.insertClassPath("/usr/local/javalib");
```

或者是注册URL路径:

```java
  ClassPool pool = ClassPool.getDefault();
  ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
  pool.insertClassPath(cp);
```

这段程序将URL "http://www.javassist.org:80/java/" 注册为类路径，当池对象要获取指定包路径`org.javassist`下的class文件时，会在URL路径下寻找。例如要搜索`org.javassist.test.Main`时，完整的搜索路径如下：

```java
  http://www.javassist.org:80/java/org/javassist/test/Main.class
```

此外，你还可以通过`ByteArrayClassPath`入参直接给池对象一个字节数组，然后这个字节数组可以直接转换为`CtClass`对象：

```java
  ClassPool cp = ClassPool.getDefault();
  byte[] b = a byte array;
  String name = class name;
  cp.insertClassPath(new ByteArrayClassPath(name, b));
  CtClass cc = cp.get(name);
```

上面代码中获取的`CtClass`对象所代表的class文件由指定的`b`字节数组定义。如果调用池对象的`get()`方法时传入的全类名与构造`ByteArrayClassPath`时传入的全类名一样，则池对象从`ByteArrayClassPath`中获取class文件信息。

如果不清楚类的全类名，可以通过池对象的`makeClass()`方法获取：

```java
  ClassPool cp = ClassPool.getDefault();
  InputStream ins = an input stream for reading a class file;
  CtClass cc = cp.makeClass(ins);
```

`makeClass()`通过传入的输入流构造`CtClass`对象。你可以通过`makeClass()`预加载class文件。当搜索路径下包含一个大的jar文件时，这种操作可能会提升性能。因为池对象是按需读取class文件，有可能在读取class文件时，每次都要搜索整个jar。`makeClass()`可以优化这种搜索方式，通过`makeClass()`构造的`CtClass`会缓存在池对象中，所以预加载后可以提升`get()`的效率。

用户可以扩展类搜索路径，通过实现`ClassPath`接口，并在池对象的`insertClassPath()`方法中传入实现的`ClassPath`实例，这允许加载一些非标准资源。

# ClassPool

`ClassPool`对象是`CtClass`对象的容器。一旦创建了一个`CtClass`对象，它就会永远记录在`ClassPool`中（缓存）。这是因为编译器在稍后编译引用该`CtClass`表示的类的源代码时，可能需要访问CtClass对象。

## 避免内存溢出

`ClassPool`的这种工作方式可能会导致一个池对象包含大量的`CtClass`对象，最终内存溢出（很少发生，javassist已经做了优化）。为了解决这个问题，可以通过调用`CtClass`对象的`detach()`方法从池对象中移除：

 ```
  CtClass cc = ... ;
  cc.writeFile();
  cc.detach();
 ```

在调用`detach()`方法后不能在`CtClass`对象上进行任何操作，不过可以通过池对象再次`get()`相同的class，池对象会再次读取class文件。

另一种方式是创建一个新的池对象来代替旧的池对象，当旧池对象被GC时，其内部的`CtClass`对象也会被回收，可以通过以下方式创建新的池对象：

 ```
  ClassPool cp = new ClassPool(true);
  // if needed, append an extra search path by appendClassPath()
 ```

不要使用`ClassPool.getDefault()`方法创建，这个方法只是为了方便而设计的单例工厂方法。

`new ClassPool(true)`是一个方便的构造函数，它构造了一个ClassPool对象，并将系统搜索路径附加到它。调用构造函数等价于下面的代码:

 ```
  ClassPool cp = new ClassPool();
  cp.appendSystemPath();  // or append another path by appendClassPath()
 ```

## 级联ClassPools

如果程序运行在服务器上，可能需要创建多个池对象实例，每个类加载器（例如容器）对应一个池对象实例。此时不能通过`getDefault()`获取池对象，而是通过构造函数创建池对象。

多个池对象可以像`java.lang.ClassLoader`一样串联起来：

 ```
  ClassPool parent = ClassPool.getDefault();
  ClassPool child = new ClassPool(parent);
  child.insertClassPath("./classes");
 ```

如果`child.get()`被调用，子池对象首先委托父池对象搜索class文件，如果父池对象搜索不到，那么子池对象会在`./classes`目录下搜索。

如果`child.childFirstLookup`设置为true，那么子池对象在委托父池对象之前就会搜索class文件：

 ```
  ClassPool parent = ClassPool.getDefault();
  ClassPool child = new ClassPool(parent);
  child.appendSystemPath();         // the same class path as the default one.
  child.childFirstLookup = true;    // changes the behavior of the child.
 ```

## 改变已有class名称来定义新class

A new class can be defined as a copy of an existing class. The program below does that:

```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.setName("Pair");
```

这段程序首先获取代表`Point`的`CtClass`对象，然后通过`setName()`将其全类名更改为`Pair`。此方法调用后，类文件中所有使用`Point`全类名的地方都会替换为新的全类名，其它定义不会改变。

`setName()`方法会将池对象中原来的缓存删除，然后用新的全类名为key，保存新的键值对，所以再次通过`get("Point")`获取`CtClass`对象时，会重新读取class文件。

```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  CtClass cc1 = pool.get("Point");   // cc1 is identical to cc.
  cc.setName("Pair");
  CtClass cc2 = pool.get("Pair");    // cc2 is identical to cc.
  CtClass cc3 = pool.get("Point");   // cc3 is not identical to cc.
```
池对象维护了class文件和`CtClass`对象的一对一关系，javassist不允许不同的`CtClass`对象表示同一个class文件，除非使用不同的池对象来创建，如果有两个池对象，可以通过两个池对象获取同一个class文件的不同`CtClass`对象，以便生成不同版本的class。

## 改变冻结class名称来定义新class

一旦`CtClass`对象通过`writeFile()`或者`toBytecode()`方法转换成class文件，javassist将不允许再次对`CtClass`对象进行修改。以下为代码示例：

 ```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.writeFile();
  cc.setName("Pair");    // wrong since writeFile() has been called.
 ```
为了避免这个限制，你可以调用`getAndRename()`，因为`getAndRename()`不使用缓存获取`CtClass`对象，而是直接读取class文件：

 ```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.writeFile();
  CtClass cc2 = pool.getAndRename("Point", "Pair");
 ```
# Class加载器

如果一个class是否被修改是在加载时才能确定，就必须让javassist和类加载器一起协作。javassist可以和类加载器一起工作，这样就可以在加载时修改字节码，你可以使用自定义的类加载器，也可以使用javassist提供的类加载器。

## `CtClass`对象的`toClass`方法

`CtClass`提供了一个便捷的方法`toClass()`，可以通过当前线程的上下文获取类加载器，来加载`CtClass`对象代表的class文件。要调用这个方法，调用者必须有合适的权限，否则会抛出`SecurityException`异常。

以下为代码展示：

 ```java
  public class Hello {
      public void say() {
          System.out.println("Hello");
      }
  }
  
  public class Test {
      public static void main(String[] args) throws Exception {
          ClassPool cp = ClassPool.getDefault();
          CtClass cc = cp.get("Hello");
          CtMethod m = cc.getDeclaredMethod("say");
          m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
          Class c = cc.toClass();
          Hello h = (Hello)c.newInstance();
          h.say();
      }
  }
 ```
`Test.main()`在`Hello`对象的`say()`方法的方法体之前插入了一行输出语句，然后通过修改后的class实例创建类对象，并调用`say()`方法。

注意：必须保证在调用`toClass()`方法之前，JVM没有加载过`Hello`的class文件，如果JVM在调用`toClass()`之前已经加载了`Hello`的class文件，当通过`toClass()`加载修改后的class文件时，会抛出`LinkageError`异常，以下为代码示例：

 ```java
  public static void main(String[] args) throws Exception {
      // 通过这种方式让JVM加载Hello的class文件
      Hello orig = new Hello();
      ClassPool cp = ClassPool.getDefault();
      CtClass cc = cp.get("Hello");
          :
  }
 ```
上面的代码会抛出异常，因为类加载器无法同时加载`Hello`不同版本的class文件。

如果程序运行在服务器上，那么`toClass()`方法通过线程上下文获取的类加载器可能不是你想要的类加载器，有可能会因为使用的类加载器错误而抛出`ClassCastException`异常。这时必须为`toClass()`指定一个合适的类加载器，以下为代码示例：

 ```
  CtClass cc = ...;
  Class c = cc.toClass(bean.getClass().getClassLoader());
 ```
## Java的类加载

在Java中，多个类加载器可以共存，每个类加载器都创建自己的名称空间。不同的类加载器可以用相同的类名加载不同的class文件。被加载的classes被视为不同的class。这个特性是我们能够在一个JVM上运行多个应用程序，即使这些程序包含相同名称的不同类。

> 注意：
JVM不允许动态重载class。一旦类加载器加载class完毕，就不允许在运行时重新加载。因此在JVM加载class完毕后，就不能修改class。但是JPDA（Java Platform Debugger Architecture）为重新加载提供了有限的支持（hotswap）。

如果两个不同的类加载器加载同一个class文件，JVM会以相同的名称和定义生成两个class对象。这两个class对象被认为是不同的，因此两个类之间的强制转换会失败并抛出`ClassCastException`。

以下为代码示例：

 ```java
  MyClassLoader myLoader = new MyClassLoader();
  Class clazz = myLoader.loadClass("Box");
  Object obj = clazz.newInstance();
  Box b = (Box)obj;    // this always throws ClassCastException.
 ```
上面的代码中，`Box`被两个类加载器加载。假设当前运行这段程序时使用的类加载器为CL，因为这段代码引用了`MyClassLoader`, `Class`, `Object`, 和 `Box`，因此CL也会加载这些类（除非CL委托给了父级加载器）。因此代码中的变量`b`代表了CL类加载器加载的`Box`类。另一方面，`myLoader`加载器也加载了`Box`类，变量`obj`代表了`myLoader`类加载器加载的`Box`类。因此最后一行代码将会抛出异常。

以下代码对JAVA的“双亲委派”机制做了简单示例，这里不再翻译相关的文档：

 ```java
  public class Point {    // loaded by PL
      private int x, y;
      public int getX() { return x; }
          :
  }
  
  public class Box {      // the initiator is L but the real loader is PL
      private Point upperLeft, size;
      public int getBaseX() { return upperLeft.x; }
          :
  }
  
  public class Window {    // loaded by a class loader L
      private Box box;
      public int getBaseX() { return box.getBaseX(); }
  }
 ```
假设`Window`的启动加载器和真正加载器都是L，因为`Window`引用了`Box`，那么L在加载`Box`的时候会委托给父加载器PL，而PL加载`Box`的时候发现它引用了`Point`，PL没有委托给其它加载器自己加载`Point`成功，至此PL加载`Box`成功，所以`Box`的启动加载器是L，实际加载器是PL。

接下来，我们看一个稍微修改后的示例：

 ```java
  public class Point {
      private int x, y;
      public int getX() { return x; }
          :
  }
  
  public class Box {      // the initiator is L but the real loader is PL
      private Point upperLeft, size;
      public Point getSize() { return size; }
          :
  }
  
  public class Window {    // loaded by a class loader L
      private Box box;
      public boolean widthIs(int w) {
          Point p = box.getSize();
          return w == p.getX();
      }
  }
 ```

在上面的代码中，`Window`的类定义引用了`Point`。在这种情况下，当类加载器L想要加载`Point`时，必须委托给父级PL类加载器。*必须避免两个类加载器同时加载同一个class*，其中一个类加载器必须委托给另一个。

如果类加载器L在加载`Point`时不委托给PL，那么`widthIs()`方法将会抛出`ClassCastException`异常。因为`Box`的实际类加载器是PL，所以`Box`中引用的`Point`的实际类加载也是PL。因此`getSize()`方法返回的`Point`实例是由PL加载器加载的。而接收返回实例的变量`p`的`Point`类型是由L加载器加载的。JVM认为它们是不同的类型，因此抛出异常。

这种方式虽然不方便，但是很有必要，如果下面的这行代码没有抛出异常，那么开发者就可以打破`Point`对象的封装。

 ```java
  Point p = box.getSize();
 ```

例如PL加载器加载的`Point`中的属性`x`是私有的，但是L加载器加载的类定义如下，此时如果不抛出异常，则在`Window`中就可以直接访问`Point`中的`x`属性。

 ```java
  public class Point {
      public int x, y;    // not private
      public int getX() { return x; }
          :
  }
 ```

要了解Java中类加载器的更多细节，下面的文章将会有所帮助：

Sheng Liang and Gilad Bracha, "Dynamic Class Loading in the Java Virtual Machine", *ACM OOPSLA'98*, pp.3644, 1998.

## 使用`javassist.Loader`

Javassist提供了一个类加载器`javassist.Loader`，这个类加载器通过传入的`javassist.ClassPool`对象读取class文件。

例如，`javassist.Loader`可以用来加载javassist修改的指定class：

 ```java
  import javassist.*;
  import test.Rectangle;
  
  public class Main {
    public static void main(String[] args) throws Throwable {
 Default();
       Loader cl = new Loader(pool);
  
       CtClass ct = pool.get("test.Rectangle");
       ct.setSuperclass(pool.get("test.Point"));
  
       Class c = cl.loadClass("test.Rectangle");
       Object rect = c.newInstance();
           :
    }
  }
 ```

上面的代码将`test.Rectangle`的父类设置为`test.Point`，然后加载修改后的类，并创建`test.Rectangle`class实例对象。

如果想要在加载时按需修改class，可以添加`javassist.Loader`的事件监听器。当类加载器加载class时，回触发监听器的调用，监听器必须实现下面的接口：

 ```java
  public interface Translator {
      public void start(ClassPool pool)
          throws NotFoundException, CannotCompileException;
      public void onLoad(ClassPool pool, String classname)
          throws NotFoundException, CannotCompileException;
  }
 ```

当通过`javassist.Loader`对象的`addTranslator()`方法添加监听器时，监听器的`start()`方法被调用。当`javassist.Loader`在加载class时，在加载逻辑之前，`onLoad()`方法被调用，在`onLoad()`方法中可以修改类定义。

例如下面的监听器在所有class加载之前，将其修饰符修改为`public`：

 ```java
  public class MyTranslator implements Translator {
      void start(ClassPool pool)
          throws NotFoundException, CannotCompileException {}
      void onLoad(ClassPool pool, String classname)
          throws NotFoundException, CannotCompileException
      {
          CtClass cc = pool.get(classname);
          cc.setModifiers(Modifier.PUBLIC);
      }
  }
 ```

> 注意：
>
> 在`onLoad()`方法中不必调用`toBytecode()`或者`writeFile()`，因为`javassist.Loader`会调用这些方法。

假设要给`MyApp`启动时添加监听器`MyTranslator`，可以使用以下写法：

 ```java
  import javassist.*;
  
  public class Main2 {
    public static void main(String[] args) throws Throwable {
       Translator t = new MyTranslator();
       ClassPool pool = ClassPool.getDefault();
       Loader cl = new Loader();
       cl.addTranslator(pool, t);
       cl.run("MyApp", args);
    }
  }
 ```

可以通过以下方式启动程序:

 ```
  % java Main2 arg1 arg2...
 ```

注意`MyApp`不能访问`Main2`, `MyTranslator`, 和 `ClassPool`类，因为他们是不同的类加载器加载的，`MyApp`是通过`javassist.Loader`加载的，而像`Main2`这种是通过`java.lang.ClassLoader`加载的。

`javassist.Loader`与`java.lang.ClassLoader`搜索class的顺序不同。`java.lang.ClassLoader`首先将加载操作委托给父加载器，只有在父加载器找不到class时才尝试加载，而`javassist.Loader`在委托给父加载器之前就会尝试加载，`javassist.Loader`只在以下情况会委托给父加载器：

* 在`ClassPool`对象中通过`get()`无法找到class
* 通过`delegateLoadingOf()`指定了加载时要使用父加载器

这个加载顺序允许javassist加载修改过的class，但是如果由于某些原因javassist无法找到修改的类，就会委托给父加载器。一旦一个class被父加载器加载后，该class中引用的其它所有class都会由父加载器加载。回想以下class C中引用的所有class，都是由C的实际加载器加载的。如果你的javassist程序无法加载一个修改后的class，首先要检查的是是否使用这个class的所有class都被javassist加载了。

## 编写类加载器

一个简单的javassist类加载器如下：

 ```java
  import javassist.*;
  
  public class SampleLoader extends ClassLoader {
      /* Call MyApp.main().
       */
      public static void main(String[] args) throws Throwable {
          SampleLoader s = new SampleLoader();
          Class c = s.loadClass("MyApp");
          c.getDeclaredMethod("main", new Class[] { String[].class })
           .invoke(null, new Object[] { args });
      }
  
      private ClassPool pool;
  
      public SampleLoader() throws NotFoundException {
          pool = new ClassPool();
          pool.insertClassPath("./class"); // MyApp.class must be there.
      }
  
      /* Finds a specified class.
       * The bytecode for that class can be modified.
       */
      protected Class findClass(String name) throws ClassNotFoundException {
          try {
              CtClass cc = pool.get(name);
              // modify the CtClass object here
              byte[] b = cc.toBytecode();
              return defineClass(name, b, 0, b.length);
          } catch (NotFoundException e) {
              throw new ClassNotFoundException();
          } catch (IOException e) {
              throw new ClassNotFoundException();
          } catch (CannotCompileException e) {
              throw new ClassNotFoundException();
          }
      }
  }
 ```

`MyApp`是一个应用程序，要执行这段程序，首先要将`MyApp`的class文件放到`./class`目录下，这个目录不能是系统的类搜索路径。否则系统默认的加载器就能加载到它。然后通过以下命令启动程序：`% java SampleLoader`

类加载器就会加载`MyApp` (`./class/MyApp.class`)，并通过命令行参数调用`MyApp.main()`。

这是使用javassist类加载器最简单的方式，如果你想编写一个更复杂的类加载器，你需要对Java的类加载机制有更详细的了解。

## 修改系统class

像`java.lang.String`这样的class只能由系统加载器加载，因此不同通过之前所说的javassist自定义加载器在加载时动态修改系统class。

想要修改系统class，必须通过硬编码（静态）的方式修改，例如下面的程序给系统class添加了一个属性：

 ```java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("java.lang.String");
  CtField f = new CtField(CtClass.intType, "hiddenValue", cc);
  f.setModifiers(Modifier.PUBLIC);
  cc.addField(f);
  cc.writeFile(".");
 ```

这段程序产生了这个文件:`"./java/lang/String.class"`.

假设应用程序`MyApp`逻辑如下：

 ```java
  public class MyApp {
      public static void main(String[] args) throws Exception {
          System.out.println(String.class.getField("hiddenValue").getName());
      }
  }
 ```
通过以下方式启动程序：

 ```
  % java Xbootclasspath/p:. MyApp arg1 arg2...
 ```
如果对`String`的修改生效，则`MyApp`可以输出`hiddenValue`。

注意：为了覆盖rt.jar中的系统类而使用此技术的应用程序不应该被部署，因为这样做会违反Java 2 Runtime Environment二进制代码许可证。

## 运行时重新加载class

如果JVM通过JPDA（Java Platform Debugger Architecture）启动，则可以动态的重载class。在JVM加载一个class之后，这个class可以被卸载同时重新加载进一个新的class。但是新class的定义必须兼容旧class。

javassist为运行时重载提供了一个便捷的类，更多信息请查看`javassist.tools.HotSwapper`文档。

# 默认和自定义

`CtClass`提供了一组默认的方法，这些方法与Java的反射API兼容，例如`getName()`, `getSuperclass()`, `getMethods()`等等。`CtClass`也提供了修改class定义的方法，比如添加新的属性、构造器和方法。

通过`CtMethod`对象表示方法。`CtMethod`提供了修改方法定义的一组方法。

注意：如果方法是子类从父类中继承的，那么`CtMethod`对象既代表父类的方法，也代表子类的方法。

例如一个类`Point`声明了`move()`，并且`ColorPoint`继承了`Point`并且没有覆写`move()`方法。那么`CtMethod`对象既代表`Point`中声明的`move()`方法，也代表子类继承的`move()`方法。如果此时修改了`CtMethod`的定义，那么两个方法都会收到影响。如果你指向修改子类的`move()`方法定义，首先要给子类`ColorPoint`添加一个`move()`方法的副本，副本可以通过`CtNewMethod.copy()`获取。

javassist不允许删除方法或者属性，但是允许重命名，所以想要废弃一个方法时，应该把方法重命名并将其修饰符改为私有。

javassist不允许给已经存在的方法添加额外的参数，但是可以通过重载的方式添加一个额外的方法，以下为示例：

  ```java
    void move(int newX, int newY) { x = newX; y = newY; }

    void move(int newX, int newY, int newZ) {
        // do what you want with newZ.
        move(newX, newY);
    }
  ```

javassist提供了一些低级API操作原始class文件。例如`CtClass`对象中的`getClassFile()`方法返回一个代表原始class文件的`ClassFile`对象。`CtMethod`对象中的`getMethodInfo()`方法返回一个代表class文件中`method_info`结构的`MethodInfo`对象。低级API使用JVM规范的词汇表，你必须对这部分有所了解才能使用。

如果你修改的class文件中使用了类似`$`这样的标识符，那么你需要引入`javassist.runtime`包来进行支持，否则不需要引入javassist的任何运行时包。

## 在方法开始结束时插入代码

`CtMethod`和`CtConstructor`提供了三个方法`insertBefore()`, `insertAfter()`, 和 `addCatch()`。这三个方法允许在已有方法体中插入代码片段。你可以通过Java的方式编写代码片段字符串，javassist提供了内置的Java编译器处理这些字符串文本，将其编译成Java字节码并内联到方法体中。

也可以在指定的行号处插入代码片段（class文件包含行号表的前提下），可以通过`CtMethod`和`CtConstructor`的`insertAt()`方法接收代码片段字符串和一个指定的源文件行号，并在指定的行号后插入编译后的字节码。

`insertBefore()`, `insertAfter()`, `addCatch()`传入的代码片段可以是以`;`结束的语句，或者是以`{}`包裹的多行语句。

 ```java
  System.out.println("Hello");
  { System.out.println("Hello"); }
  if (i < 0) { i = i; }
 ```

如果使用了-g模式的编译模式（在class文件中包含局部变量表），那么插入到方法中的代码片段中，就可以直接引用方法中传入的参数变量。否则，旧必须通过以下表格内指定的变量名访问方法参数例如`$0`, `$1`, `$2`等等。如果想要访问方法体内声明的局部变量，是不允许的，但是在插入的代码片段中声明局部变量是可以的。不过`insertAt()`方法允许插入的代码片段访问方法体内的局部变量，不过此局部变量必须在插入的行号处是可用的，并且必须使用-g编译模式。

通过`insertBefore()`, `insertAfter()`, `addCatch()`, 和 `insertAt()`插入的代码片段是由javassist内部的编译器编译的，而编译器是支持扩展的，所以在javassist内置的编译器中做了功能增强，可以以下的标识符都有特殊含义：


| 标识符              | 含义            |
| ------------------- | -------------------------------------- |
| `$0, $1, $2, ...  ` | 代表this和实际的参数      |
| `$args`             | 参数的数组类型为 Object[].         |
| `$$`                | 所有实际的参数 ，例如m($$)等效于m($1,$2,...)        |
| `$cflow(...)`       | cflow variable           |
| `$r`                | 返回值类型，用于类型转换|
| `$w`                | 包装器类型，用于类型转换（例如Integer和int）|
| `$_`                | 结果值 |
| `$sig`              | 代表参数类型的java.lang.Class对象数组 |
| `$type`             | 返回值类型的java.lang.Class对象|
| `$class`            | 当前类的java.lang.Class对象 |

###  `$0, $1, $2`, ...

传递给目标方法的形参通过`$1`, `$2`等代替。`$1`代表第一个参数，`$2`代表第二个参数以此类推。变量的类型于入参的类型相同。`$0`代表`this`，如果是静态方法，`$0`不可以使用。

假设有一个类`Point`：

```java
class Point {
    int x, y;
    void move(int dx, int dy) { x += dx; y += dy; }
}
```
想要在`move()`被调用时打印出`dx`和`dy`，可以通过以下方式：

 ```
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  CtMethod m = cc.getDeclaredMethod("move");
  m.insertBefore("{ System.out.println($1); System.out.println($2); }");
  cc.writeFile();
 ```
`dx`和`dy`分别被`$1`和`$2`替换，修改后的class文件如下：

```java
class Point {
    int x, y;
    void move(int dx, int dy) {
        { System.out.println(dx); System.out.println(dy); }
        x += dx; y += dy;
    }
}
```

`$1`, `$2`, `$3`是可以被更新的，假如给其中一个变量赋予一个新值，那么在后续的逻辑中都会生效。

### `$args`

`$args`代表所有参数的数组，数组类型为`Object`。如果一个参数的类型是原始数据类型（比如int），那么这个参数将会转化为对应的包装类型放到数组中。`$args[0]`不代表`$0`，`$0`代表的是`this`，所以`$args[0]`代表的是第一个参数`$1`.

如果对`$args`赋予一个新数组，那么数组里代表的每个参数都会更新，如果参数的类型是原始数据类型，那么数组中对应位置的数据类型必须为对应的包装类型，在赋值之前会自动拆箱。

###  `$$` 

`$$`是所有参数的缩写，通过逗号分隔。例如`move()`方法接收三个参数，下面两种写法是等效的：

```java
  move($$)

  move($1, $2, $3)
```
如果`move()`方法没有接收任何参数，那么`move($$)`等效于`move()`。

`$$`可以用在其它方法上，假如`move()`方法接收三个参数，你在`move()`方法内插入的代码片段中写了如下表达式：

```java
  exMove($$, context)
```
它等效于：

 ```java
  exMove($1, $2, $3, context)
 ```

### `$cflow`

`$cflow`表示"控制流"。这是一个只读变量，返回指定方法的递归调用深度。

假设`CtMethod`对象`cm`代表了如下方法：

 ```java
  int fact(int n) {
      if (n <= 1)
          return n;
      else
          return n * fact(n  1);
  }
 ```
要使用`$cflow`，首先要声明`$cflow`监控的方法`fact()`

 ```
  CtMethod cm = ...;
  cm.useCflow("fact");
 ```
`useCflow()`的参数是`$cflow`变量的标识符，任何有效的Java名称都可以使用，例如`"my.Test.fact"`就是一个有效的标识符。

然后`$cflow(fact)`就表示对`cm`所代表方法的调用深度。第一次调用时`$cflow(fact)`的值为0，之后每一次调用都累加1，例如：

 ```java
  cm.insertBefore("if ($cflow(fact) == 0)"
                + "    System.out.println(\"fact \" + $1);");
 ```

`$cflow`的值是当前线程栈空间中栈顶之下所有与指定方法`cm`关联的栈帧个数，所以`$cflow`也可以在当前栈空间下的其它方法内访问。

### `$r`

`$r`代表方法的返回值类型，必须用在类型转换的表达式中，例如：

 ```java
  Object result = ... ;
  $_ = ($r)result;
 ```

如果返回值类型为原始数据类型，例如int，那么`($r)`会自动拆箱。

如果返回值类型为`void`，那么`($r)`不做任何操作，但是如果你转换的是一个对返回`void`类型的方法调用，那么这个操作符的转换结果为`null`，例如下面的`$_`为`null`：

 ```
  $_ = ($r)foo();
 ```

转换操作符`($r)`一般用在`return`语句中，即使返回值类型为`void`，下面的语句也是有效的：

 ```java
  return ($r)result;
 ```

上面的语句中`result`是作为结果值的某个局部变量，由于指定了转换操作符，结果值将被丢弃，整个`return`语句将被认为是没有返回值的return，即等效于下面的语句：

 ```java
  return;
 ```

### `$w`

`$w`代表方法的返回值类型，必须用在类型转换的表达式中，`($w)`将原始数据类型转换为包装类型，例如：

 ```java
  Integer i = ($w)5;
 ```
如果要操作的数据的类型不是原始数据类型，则`($w)`不做任何事。

### `$_`

`CtMethod`和`CtConstructor`的`insertAfter()`方法会将编译后的代码片段插入到方法体的最后面，所以在`insertAfter()`方法中除了可以使用方法参数的标识符`$0`, `$1`以外，还可以使用返回值的标识符`$_`。

`$_`代表了方法的返回值，它的类型就是方法的返回值类型，如果方法的返回值类型为`void`，则`$_`的类型为`Object`，值为`null`。


尽管`insertAfter()`方法插入的代码片段是在方法体正常结束后，返回之前执行，也可以通过设置`insertAfter()` 方法发的`asFinally`参数为`true`，来让其在方法抛出异常时也能执行。

### `$sig`

`$sig`代表了方法参数类型的`java.lang.Class`实例数组。

### `$type`

`$type`是返回值类型的`java.lang.Class`实例对象，如果是构造函数，则为`Void.class`。

### `$class`

`$class`代表当前编辑方法所在类的`java.lang.Class`实例对象，也是`$0`代表的`this`的类型的class实例对象。

### `addCatch()`

`addCatch()`是用来添加try-catch异常逻辑的，其中`$e`为异常对象的引用，例如：

 ```java
  CtMethod m = ...;
  CtClass etype = ClassPool.getDefault().get("java.io.IOException");
  m.addCatch("{ System.out.println($e); throw $e; }", etype);
 ```
修改后的class文件如下：

 ```java
  try {
      the original method body
  }
  catch (java.io.IOException e) {
      System.out.println(e);
      throw e;
  }
 ```
插入的代码片段必须以`return`或者`throw`结尾。

## 修改方法体

`CtMethod`和`CtConstructor`提供了`setBody()`来替换整个方法体。如果给定的代码片段为`null`，则替换后的方法体只包含一个`return`语句，返回值为0或者null，或者是`void`。

`setBody()`中可使用的标识符如下：

| 标识符                | 含义                                                         |
| --------------------- | ------------------------------------------------------------ |
| `$0, $1, $2, ...    ` | this and actual parameters                                   |
| `$args`               | An array of parameters. The type of $args is Object[].       |
| `$$`                  | All actual parameters.                                       |
| `$cflow(...)`         | cflow variable                                               |
| `$r`                  | The result type. It is used in a cast expression.            |
| `$w`                  | The wrapper type. It is used in a cast expression.           |
| `$sig`                | An array of java.lang.Class objects representing the formal parameter types. |
| `$type`               | A java.lang.Class object representing the formal result type. |
| `$class`              | A java.lang.Class object representing the class that declares the methodcurrently edited (the type of $0). |

注意`$_`是不可用的。

### 替换已有表达式

javassist只允许修改方法体内的表达式，用户可以创建`javassist.expr.ExprEditor`的子类来定义如何修改已有的表达式。

如果要使用`ExprEditor`对象，必须调用`CtMethod`或`CtClass`对象的`instrument()`方法，例如：

```java
  CtMethod cm = ... ;
  cm.instrument(
      new ExprEditor() {
          public void edit(MethodCall m)
                        throws CannotCompileException
          {
              if (m.getClassName().equals("Point")
                            && m.getMethodName().equals("move"))
                  m.replace("{ $1 = 0; $_ = $proceed($$); }");
          }
      });
```
上面的代码会搜索`cm`代表的方法中的方法体，并把方法体内所有调用使用`Point`对象调用`move()`方法的表达式，替换为指定的代码片段：
 ```java
  { $1 = 0; $_ = $proceed($$); }
 ```
这段代码片段的作用会将`move()`方法的第一个参数永远置为0.

`instrument()`会对方法体进行搜索，并在发现方法调用、属性访问、对象创建等表达式的时候，回调传入的`ExprEditor`对象的`edit()`方法，传入`edit()`方法的参数，代表了发现的表达式（`edit()`方法有多个重载方法，入参类型不同，代表了不同类型的表达式），我们可以通过传入的参数对已有表达式进行修改。

调用`replace()`方法可以替换`edit()`方法中参数代表的表达式，如果替换的内容为空，例如`replace("{}")`，则原表达式会被删除。如果想在表达式前后插入自定义内容，可以使用以下写法：

 ```java
  { beforestatements;
    $_ = $proceed($$);
    afterstatements; }
 ```
### 表达式：MethodCall参数

`MethodCall`代表了方法调用的表达式，在`replace()`方法中传入的代码片段，可以使用同`insertBefore()`方法一样以`$`开头的特殊标识符：

| 含义              | 标识符                                                       |
| ----------------- | ------------------------------------------------------------ |
| `$0 `             | 方法调用的目标对象，与this不同，this代表的是调用方，如果是静态方法，$0为null|
| `$1, $2, ...    ` | MethodCall对象所代表的方法的入参                             |
| `$_`              | MethodCall对象所代表的方法的返回值                           |
| `$r`              | MethodCall对象所代表的方法的返回值类型                       |
| `$class`          | 声明方法所在类的class对象实例                                |
| `$sig`            | MethodCall对象所代表的方法的入参类型的class实例数组          |
| `$type `          | MethodCall对象所代表的方法的返回值的class实例                |
| `$proceed `       | 原有表达式调用方法的名称                                     |

除非方法调用的结果类型是`void`，否则必须在源文本中为`$_`赋值，并且`$_`的类型就是结果类型。如果结果类型为`void`， 则`$_`的类型为`Object`，赋给`$_`的值将被忽略。

`$proceed`后面必须紧接一个`( )`，括号内是方法调用的参数。

### 表达式：ConstructorCall参数

`ConstructorCall`对象代表构造器方法体内的`this()`和`super`表达式。以下为在使用`replace()`方法时可以使用的特殊表达式：

| 含义              | 标识符                                                       |
| ----------------- | ------------------------------------------------------------ |
| `$0` | 等效于`this`. |
| `$1`, `$2`, ... | 构造器的入参 |
| `$class`        | 声明构造器方法所在类的class实例对象 |
| `$sig`          | 构造器入参类型的class实例数组 |
| `$proceed`      | 原表达式中调用构造器的方法名|

由于构造器方法内都会调用父级构造器或者当前类的其它重载构造器方法，所以替换后的语句要包含一个构造器调用，即`$proceed()`。

### 表达式：FieldAccess参数

`FieldAccess`代表属性访问表达式，以下为替换方法中可以使用的特殊标识符：

| 含义       | 标识符                                                       |
| ---------- | ------------------------------------------------------------ |
| `$0 `      | The object containing the field accessed by the expression. This is not equivalent to this.this represents the object that the method including the expression is invoked on.$0 is null if the field is static. |
| `$1 `      | 如果表达式是赋值语句，此标识符代表所赋的值，如果不是赋值语句，则此标识符不可用 |
| `$_`       | 如果表达式是读取语句，此标识符代表读取的值，如果不是读取语句，则此标识符不可用 |
| `$r`       | 如果表达式是读取语句，此标识符代表读取的值的类型，如果不是读取语句，则此标识符为void  |
| `$class`   | 声明属性的类的class实例对象 |
| `$type`    | 属性类型的class实例对象      |
| `$proceed` | 执行原始访问属性操作|

### 表达式：NewExpr参数

`NewExpr`对象代表`new`表达式，不包含数组的创建，以下为替换方法中可以使用的特殊标识符：

| 含义            | 标识符                                                       |
| --------------- | ------------------------------------------------------------ |
| `$0`            | `null`.                                                      |
| `$1`, `$2`, ... | new后面构造器的入参                           |
| `$_`            | 创建对象后的结果|
| `$r`            | 创建对象的类型                              |
| `$sig`          | new后面构造器入参的class实例数组 |
| `$type`         | 创建对象类型的class实例 |
| `$proceed`      | 原始执行语句 |

### 表达式：NewArray参数

`NewArray`对象代表通过`new`创建数组的操作，以下为替换方法中可以使用的特殊标识符：

| 含义            | 标识符                                                       |
| --------------- | ------------------------------------------------------------ |
| `$0`            | `null`.                                                      |
| `$1`, `$2`, ... | 每个维度的大小                                |
| `$_`            | 数组创建的结果 |
| `$r`            | 创建数组的类型                               |
| `$type`         | 创建后数组的class实例对象 |
| `$proceed`      | 原始执行语句 |

`$w`, `$args`和`$$`是不可用的。

例如通过以下方式创建数组，此时`$1`和`$2`的值就是3和4，`$3`不可用。：
 ```javva
  String[][] s = new String[3][4];
 ```
如果使用以下方式创建数组，此时`$1`的值就是3，`$2`不可用。：
 ```java
  String[][] s = new String[3][];
 ```
### 表达式：Instanceof参数

`Instanceof`对象代表了`instanceof`表达式，以下为替换方法中可以使用的特殊标识符：

| 含义       | 标识符                                                       |
| ---------- | ------------------------------------------------------------ |
| `$0`       | `null`.                                                      |
| `$1`       | `instanceof`左侧的值 |
| `$_`       | 表达式的结果，`$_`是`boolean`类型的 |
| `$r`       | `instanceof`右侧的类型 |
| `$type`    | `instanceof`右侧类型的class实例对象 |
| `$proceed` | 原始语句的执行，接受一个入参，并返回结果 |

### 表达式：Cast参数

`Cast`对象代表显示类型转换表达式，以下为替换方法中可以使用的特殊标识符：

| 含义       | 标识符                                                       |
| ---------- | ------------------------------------------------------------ |
| `$0`       | `null`.                                                      |
| `$1`       | 被显示转换的值              |
| `$_`       | 显示转换后的值 |
| `$r`       | 显示转换后的类型 |
| `$type`    | 显示转换后的类型的class实例对象 |
| `$proceed` | 原始语句的执行，接受一个入参，并返回转换结果 |

### 表达式：Handler参数

`Handler`对象代表`try-catch`中的`catch`表达式，以下为替换方法中可以使用的特殊标识符：

| 含义    | 标识符                                                       |
| ------- | ------------------------------------------------------------ |
| `$1`    | `catch`中的异常对象           |
| `$r`    | `catch`中的异常对象的类型 |
| `$w`    | 包装类型           |
| `$type` | `catch`中的异常对象类型的class实例对象 |

## 添加新方法或属性

### 添加方法

Javassist允许创建新方法或者构造器。`CtNewMethod`和`CtNewConstructor`提供了相关的静态工厂方法，其中`make()`方法可以通过代码片段创建：

假设`Point`类中有一个`int`类型的属性`x`，可以通过以下方式添加一个新方法：
 ```java
  CtClass point = ClassPool.getDefault().get("Point");
  CtMethod m = CtNewMethod.make(
                   "public int xmove(int dx) { x += dx; }",
                   point);
  point.addMethod(m);
 ```
传给`make()`的代码片段可以使用`setBody()`中以`$`开头的所有特殊标识符（`$_`除外），如果指定了目标对象和目标方法，也可以使用`$proceed`标识符，例如：

 ```java
  CtClass point = ClassPool.getDefault().get("Point");
  CtMethod m = CtNewMethod.make(
                   "public int ymove(int dy) { $proceed(0, dy); }",
                   point, "this", "move");
 ```

以上代码创建了以下方法：
 ```java
  public int ymove(int dy) { this.move(0, dy); }
 ```

你也可以先创建抽象方法，然后再创建方法体，不过使用这种方式时，创建抽象方法后会自动将所在类变为抽象类，所以在创建方法体之后要把类改为非抽象：
 ```java
  CtClass cc = ... ;
  CtMethod m = new CtMethod(CtClass.intType, "move",
                            new CtClass[] { CtClass.intType }, cc);
  cc.addMethod(m);
  m.setBody("{ x += $1; }");
  cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
 ```
### 相互递归方法

如果在方法中调用的其它方法没有添加到当前的class中，javassist是无法编译的，所以如果有相互递归调用的场景，可以使用以下方式实现，假设`cc`类中的`m()`和`n()`方法互相递归调用：

 ```java
  CtClass cc = ... ;
  CtMethod m = CtNewMethod.make("public abstract int m(int i);", cc);
  CtMethod n = CtNewMethod.make("public abstract int n(int i);", cc);
  cc.addMethod(m);
  cc.addMethod(n);
  m.setBody("{ return ($1 <= 0) ? 1 : (n($1  1) * $1); }");
  n.setBody("{ return m($1); }");
  cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
 ```
上面的代码先添加了两个抽象方法，然后添加了相互调用的递归方法体。
### 添加属性

javassist通过以下方式添加新的属性：
 ```
  CtClass point = ClassPool.getDefault().get("Point");
  CtField f = new CtField(CtClass.intType, "z", point);
  point.addField(f);
 ```

This program adds a field named `z` to class `Point`.

If the initial value of the added field must be specified, the program shown above must be modified into:

 ```
  CtClass point = ClassPool.getDefault().get("Point");
  CtField f = new CtField(CtClass.intType, "z", point);
  point.addField(f, "0");    // initial value is 0.
 ```

Now, the method `addField()` receives the second parameter, which is the source text representing an expression computing the initial value. This source text can be any Java expression if the result type of the expression matches the type of the field. Note that an expression does not end with a semi colon (`;`).

Furthermore, the above code can be rewritten into the following simple code:

 ```
  CtClass point = ClassPool.getDefault().get("Point");
  CtField f = CtField.make("public int z = 0;", point);
  point.addField(f);
 ```

### 删除成员

To remove a field or a method, call `removeField()` or `removeMethod()` in `CtClass`. A `CtConstructor` can be removed by `removeConstructor()` in `CtClass`.

## 注解

`CtClass`, `CtMethod`, `CtField` and `CtConstructor` provides a convenient method `getAnnotations()` for reading annotations. It returns an annotationtype object.

For example, suppose the following annotation:

 ```
  public @interface Author {
      String name();
      int year();
  }
 ```

This annotation is used as the following:

 ```
  @Author(name="Chiba", year=2005)
  public class Point {
      int x, y;
  }
 ```

Then, the value of the annotation can be obtained by `getAnnotations()`. It returns an array containing annotationtype objects.

 ```
  CtClass cc = ClassPool.getDefault().get("Point");
  Object[] all = cc.getAnnotations();
  Author a = (Author)all[0];
  String name = a.name();
  int year = a.year();
  System.out.println("name: " + name + ", year: " + year);
 ```

This code snippet should print:

 ```
  name: Chiba, year: 2005
 ```

Since the annoation of `Point` is only `@Author`, the length of the array `all` is one and `all[0]` is an `Author` object. The member values of the annotation can be obtained by calling `name()` and `year()` on the `Author` object.

To use `getAnnotations()`, annotation types such as `Author` must be included in the current class path. *They must be also accessible from a ClassPool object.* If the class file of an annotation type is not found, Javassist cannot obtain the default values of the members of that annotation type.

## 运行时支持

In most cases, a class modified by Javassist does not require Javassist to run. However, some kinds of bytecode generated by the Javassist compiler need runtime support classes, which are in the `javassist.runtime` package (for details, please read the API reference of that package). Note that the `javassist.runtime` package is the only package that classes modified by Javassist may need for running. The other Javassist classes are never used at runtime of the modified classes.

## 导入

All the class names in source code must be fully qualified (they must include package names). However, the `java.lang` package is an exception; for example, the Javassist compiler can resolve `Object` as well as `java.lang.Object`.

To tell the compiler to search other packages when resolving a class name, call `importPackage()` in `ClassPool`. For example,

 ```
  ClassPool pool = ClassPool.getDefault();
  pool.importPackage("java.awt");
  CtClass cc = pool.makeClass("Test");
  CtField f = CtField.make("public Point p;", cc);
  cc.addField(f);
 ```

The seconde line instructs the compiler to import the `java.awt` package. Thus, the third line will not throw an exception. The compiler can recognize `Point` as `java.awt.Point`.

Note that `importPackage()` *does not* affect the `get()` method in `ClassPool`. Only the compiler considers the imported packages. The parameter to `get()` must be always a fully qualified name.

## 限制

In the current implementation, the Java compiler included in Javassist has several limitations with respect to the language that the compiler can accept. Those limitations are:

* The new syntax introduced by J2SE 5.0 (including enums and generics) has not been supported. Annotations are supported by the low level API of Javassist. See the `javassist.bytecode.annotation` package (and also `getAnnotations()` in `CtClass` and `CtBehavior`). Generics are also only partly supported. See [the latter section](http://www.javassist.org/tutorial/tutorial3.html#generics) for more details.

* Array initializers, a comma-separated list of expressions enclosed by braces `{` and `}`, are not available unless the array dimension is one.

* Inner classes or anonymous classes are not supported. Note that this is a limitation of the compiler only. It cannot compile source code including an anonymous-class declaration. Javassist can read and modify a class file of inner/anonymous class.

* Labeled `continue` and `break` statements are not supported.

* The compiler does not correctly implement the Java method dispatch algorithm. The compiler may confuse if methods defined in a class have the same name but take different parameter lists.

For example,

 ```
  class A {} 
  class B extends A {} 
  class C extends B {} 
  
  class X { 
      void foo(A a) { .. } 
      void foo(B b) { .. } 
  }
 ```

If the compiled expression is `x.foo(new C())`, where `x` is an instance of X, the compiler may produce a call to `foo(A)` although the compiler can correctly compile `foo((B)new C())`.

* The users are recommended to use `#` as the separator between a class name and a static method or field name. For example, in regular Java,
* ```
  javassist.CtClass.intType.getName()
  ```

calls a method `getName()` on the object indicated by the static field `intType` in `javassist.CtClass`. In Javassist, the users can write the expression shown above but they are recommended to write:

```
 javassist.CtClass#intType.getName()
```

so that the compiler can quickly parse the expression.

# 低级字节码API

Javassist also provides lowerlevel API for directly editing a class file. To use this level of API, you need detailed knowledge of the Java bytecode and the class file format while this level of API allows you any kind of modification of class files.

If you want to just produce a simple class file, `javassist.bytecode.ClassFileWriter` might provide the best API for you. It is much faster than `javassist.bytecode.ClassFile` although its API is minimum.

# 获取`ClassFile`对象

A `javassist.bytecode.ClassFile` object represents a class file. To obtian this object, `getClassFile()` in `CtClass` should be called.

Otherwise, you can construct a `javassist.bytecode.ClassFile` directly from a class file. For example,

 ```
  BufferedInputStream fin
      = new BufferedInputStream(new FileInputStream("Point.class"));
  ClassFile cf = new ClassFile(new DataInputStream(fin));
 ```

This code snippet creats a `ClassFile` object from `Point.class`.

A `ClassFile` object can be written back to a class file. `write()` in `ClassFile` writes the contents of the class file to a given `DataOutputStream`.

You can create a new class file from scratch. For example,

> ```
> ClassFile cf = new ClassFile(false, "test.Foo", null);
> cf.setInterfaces(new String[] { "java.lang.Cloneable" });
>  
> FieldInfo f = new FieldInfo(cf.getConstPool(), "width", "I");
> f.setAccessFlags(AccessFlag.PUBLIC);
> cf.addField(f);
> 
> cf.write(new DataOutputStream(new FileOutputStream("Foo.class")));
> ```

this code generates a class file `Foo.class` that contains the implementation of the following class:

> ```
> package test;
> class Foo implements Cloneable {
>     public int width;
> }
> ```



## 添加删除成员

`ClassFile` provides `addField()` and `addMethod()` for adding a field or a method (note that a constructor is regarded as a method at the bytecode level). It also provides `addAttribute()` for adding an attribute to the class file.

Note that `FieldInfo`, `MethodInfo`, and `AttributeInfo` objects include a link to a `ConstPool` (constant pool table) object. The `ConstPool` object must be common to the `ClassFile` object and a `FieldInfo` (or `MethodInfo` etc.) object that is added to that `ClassFile` object. In other words, a `FieldInfo` (or `MethodInfo` etc.) object must not be shared among different `ClassFile` objects.

To remove a field or a method from a `ClassFile` object, you must first obtain a `java.util.List` object containing all the fields of the class. `getFields()` and `getMethods()` return the lists. A field or a method can be removed by calling `remove()` on the `List` object. An attribute can be removed in a similar way. Call `getAttributes()` in `FieldInfo` or `MethodInfo` to obtain the list of attributes, and remove one from the list.



## 遍历方法体

To examine every bytecode instruction in a method body, `CodeIterator` is useful. To otbain this object, do as follows:

 ```
  ClassFile cf = ... ;
  MethodInfo minfo = cf.getMethod("move");    // we assume move is not overloaded.
  CodeAttribute ca = minfo.getCodeAttribute();
  CodeIterator i = ca.iterator();
 ```

A `CodeIterator` object allows you to visit every bytecode instruction one by one from the beginning to the end. The following methods are part of the methods declared in `CodeIterator`:

 `void begin()`
  Move to the first instruction.
 `void move(int index)`
  Move to the instruction specified by the given index.
 `boolean hasNext()`
  Returns true if there is more instructions.
 `int next()`
  Returns the index of the next instruction.
  *Note that it does not return the opcode of the next instruction.*
 `int byteAt(int index)`
  Returns the unsigned 8bit value at the index.
 `int u16bitAt(int index)`
  Returns the unsigned 16bit value at the index.
 `int write(byte[] code, int index)`
  Writes a byte array at the index.
 `void insert(int index, byte[] code)`
  Inserts a byte array at the index. Branch offsets etc. are automatically adjusted.

The following code snippet displays all the instructions included in a method body:

 ```
  CodeIterator ci = ... ;
  while (ci.hasNext()) {
      int index = ci.next();
      int op = ci.byteAt(index);
      System.out.println(Mnemonic.OPCODE[op]);
  }
 ```



## 生成字节码序列

A `Bytecode` object represents a sequence of bytecode instructions. It is a growable array of bytecode. Here is a sample code snippet:

 ```
  ConstPool cp = ...;    // constant pool table
  Bytecode b = new Bytecode(cp, 1, 0);
  b.addIconst(3);
  b.addReturn(CtClass.intType);
  CodeAttribute ca = b.toCodeAttribute();
 ```

This produces the code attribute representing the following sequence:

 ```
  iconst_3
  ireturn
 ```

You can also obtain a byte array containing this sequence by calling `get()` in `Bytecode`. The obtained array can be inserted in another code attribute.

While `Bytecode` provides a number of methods for adding a specific instruction to the sequence, it provides `addOpcode()` for adding an 8bit opcode and `addIndex()` for adding an index. The 8bit value of each opcode is defined in the `Opcode` interface.

`addOpcode()` and other methods for adding a specific instruction are automatically maintain the maximum stack depth unless the control flow does not include a branch. This value can be obtained by calling `getMaxStack()` on the `Bytecode` object. It is also reflected on the `CodeAttribute` object constructed from the `Bytecode` object. To recompute the maximum stack depth of a method body, call `computeMaxStack()` in `CodeAttribute`.

`Bytecode` can be used to construct a method. For example,

> ```
> ClassFile cf = ...
> Bytecode code = new Bytecode(cf.getConstPool());
> code.addAload(0);
> code.addInvokespecial("java/lang/Object", MethodInfo.nameInit, "()V");
> code.addReturn(null);
> code.setMaxLocals(1);
> 
> MethodInfo minfo = new MethodInfo(cf.getConstPool(), MethodInfo.nameInit, "()V");
> minfo.setCodeAttribute(code.toCodeAttribute());
> cf.addMethod(minfo);
> ```

this code makes the default constructor and adds it to the class specified by `cf`. The `Bytecode` object is first converted into a `CodeAttribute` object and then added to the method specified by `minfo`. The method is finally added to a class file `cf`.



# 注解（元数据标签）

Annotations are stored in a class file as runtime invisible (or visible) annotations attribute. These attributes can be obtained from `ClassFile`, `MethodInfo`, or `FieldInfo` objects. Call `getAttribute(AnnotationsAttribute.invisibleTag)` on those objects. For more details, see the javadoc manual of `javassist.bytecode.AnnotationsAttribute` class and the `javassist.bytecode.annotation` package.

Javassist also let you access annotations by the higherlevel API. If you want to access annotations through `CtClass`, call `getAnnotations()` in `CtClass` or `CtBehavior`.

# 泛型

The lowerlevel API of Javassist fully supports generics introduced by Java 5. On the other hand, the higherlevel API such as `CtClass` does not directly support generics. However, this is not a serious problem for bytecode transformation.

The generics of Java is implemented by the erasure technique. After compilation, all type parameters are dropped off. For example, suppose that your source code declares a parameterized type `Vector<String>`:

 ```
  Vector<String> v = new Vector<String>();
    :
  String s = v.get(0);
 ```

The compiled bytecode is equivalent to the following code:

 ```
  Vector v = new Vector();
    :
  String s = (String)v.get(0);
 ```

So when you write a bytecode transformer, you can just drop off all type parameters. Because the compiler embedded in Javassist does not support generics, you must insert an explicit type cast at the caller site if the source code is compiled by Javassist, for example, through `CtMethod.make()`. No type cast is necessary if the source code is compiled by a normal Java compiler such as `javac`.

For example, if you have a class:

 ```
  public class Wrapper<T> {
    T value;
    public Wrapper(T t) { value = t; }
  }
 ```

and want to add an interface `Getter<T>` to the class `Wrapper<T>`:

 ```
  public interface Getter<T> {
    T get();
  }
 ```

then the interface you really have to add is `Getter` (the type parameters `<T>` drops off) and the method you also have to add to the `Wrapper` class is this simple one:

 ```
  public Object get() { return value; }
 ```

Note that no type parameters are necessary. Since `get` returns an `Object`, an explicit type cast is needed at the caller site if the source code is compiled by Javassist. For example, if the type parameter `T` is `String`, then `(String)` must be inserted as follows:

 ```
  Wrapper w = ...
  String s = (String)w.get();
 ```

The type cast is not needed if the source code is compiled by a normal Java compiler because it will automatically insert a type cast.

If you need to make type parameters accessible through reflection during runtime, you have to add generic signatures to the class file. For more details, see the API documentation (javadoc) of the `setGenericSignature` method in the `CtClass`.

# Varargs

Currently, Javassist does not directly support varargs. So to make a method with varargs, you must explicitly set a method modifier. But this is easy. Suppose that now you want to make the following method:

 ```
  public int length(int... args) { return args.length; }
 ```

The following code using Javassist will make the method shown above:

 CtClass cc = /* target class */; CtMethod m = CtMethod.make("public int length(int[] args) { return args.length; }", cc); m.setModifiers(m.getModifiers() | Modifier.VARARGS); cc.addMethod(m); 

 ```
  
 ```

The parameter type `int...` is changed into `int[]` and `Modifier.VARARGS` is added to the method modifiers.

To call this method in the source code compiled by the compiler embedded in Javassist, you must write:

 ```
  length(new int[] { 1, 2, 3 });
 ```

instead of this method call using the varargs mechanism:

 ```
  length(1, 2, 3);
 ```

# J2ME

If you modify a class file for the J2ME execution environment, you must perform preverification. Preverifying is basically producing stack maps, which is similar to stack map tables introduced into J2SE at JDK 1.6. Javassist maintains the stack maps for J2ME only if `javassist.bytecode.MethodInfo.doPreverify` is true.

You can also manually produce a stack map for a modified method. For a given method represented by a `CtMethod` object `m`, you can produce a stack map by calling the following methods:

 ```
  m.getMethodInfo().rebuildStackMapForME(cpool);
 ```

Here, `cpool` is a `ClassPool` object, which is available by calling `getClassPool()` on a `CtClass` object. A `ClassPool` object is responsible for finding class files from given class pathes. To obtain all the `CtMethod` objects, call the `getDeclaredMethods` method on a `CtClass` object.

# 装箱和拆箱

Boxing and unboxing in Java are syntactic sugar. There is no bytecode for boxing or unboxing. So the compiler of Javassist does not support them. For example, the following statement is valid in Java:

 ```
  Integer i = 3;
 ```

since boxing is implicitly performed. For Javassist, however, you must explicitly convert a value type from `int` to `Integer`:

 ```
  Integer i = new Integer(3);
 ```

# Debug

Set `CtClass.debugDump` to a directory name. Then all class files modified and generated by Javassist are saved in that directory. To stop this, set `CtClass.debugDump` to null. The default value is null.

For example,

 ```
  CtClass.debugDump = "./dump";
 ```

All modified class files are saved in `./dump`.