

## 平台无关性

write once, run anyway Java 编译生成的二进制文件能够不做任何改变运行于多个平台。

在计算机科学领域，有一句名言是“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决(All problems in computer science can be solved by another level of indirection)”，JVM 就是在这样的环境应运而生的。

![java_compile](/Users/baofengqi/Documents/java/jvm/pic/java_compile.png)

跨平台关键

1. 对下：屏蔽不同os，不同平台不同实现，linux，mac os，windows
2. 对上：jvm 和 字节码存储格式（二进制格式）统一

java编译工具：javac

### 字节码内容

Class文件是一组以8位字节为基础的二进制流，紧密排列，没有分隔符

class 文件的头四个字节称为魔数（Magic Number），可以看到 class 的**魔数**为 0xCAFEBABE。这个魔数是 JVM 识别 .class 文件的标志，虚拟机在加载类文件之前会先检查这四个字节，如果不是 0xCAFEBABE 则拒绝加载该文件

javap 反编译

```bash
最常用的选项是-c，可以对类进行反编译

默认情况下，javap 会显示访问权限为 public、protected 和默认（包级 protected）级别的方法，加上 -p 选项以后可以显示 private 方法和字段

javap 加上 -v 参数的输出更多详细的信息，比如栈大小、方法参数的个数。
javap 还有一个好用的选项 -s，可以输出签名的类型描述符。
```

字段描述符和方法描述符

**字段描述符**（Field Descriptor），是一个表示类、实例或局部变量的语法符号，它的表示形式是紧凑的，比如 int 是用 I 表示的。完整的类型描述符如下表

![descriptor](/Users/baofengqi/Documents/java/jvm/pic/descriptor.png)

**方法描述符**（Method Descriptor）表示一个方法所需参数和返回值信息，表示形式为`( ParameterDescriptor* ) ReturnDescriptor`。 ParameterDescriptor 表示参数类型，ReturnDescriptor表示返回值信息，当没有返回值时用`V`表示。比如方法`Object foo(int i, double d, Thread t)`的描述符为`(IDLjava/lang/Thread;)Ljava/lang/Object;`

java中存在大量的class，这里的class大多存在与硬盘上，因此用户如果需要使用这些类，jvm就需要把这些class 加载到内存中。

引申出下面的问题：jvm何时将何处的class如何加载到内存中。

## ClassLoader解决了什么问题？

将通过一个类的全限定名来获取描述此类的二进制字节流独立在jvm外部。

主要是类加载。

对于任意一个类，都需要有加载这个类的类加载器和这个类本身来确立其在jvm中的唯一性。这里的相等包括Class的equals，isassignablefrom。isinstance等等

ClassLoader主要解决从哪里如何加载 class 到内存中。jvm中依据类基础程度，采用了不同的ClassLoader来加载不同的类。

## ClassLoader分类

Java语言系统自带有三个类加载器：

- **Bootstrap ClassLoader**  - 最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如`java -Xbootclasspath/a:path`被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。
- **Extention ClassLoader** - 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载`-D java.ext.dirs`选项指定的目录。
- **Application ClassLoader也称为SystemAppClass** - 加载当前应用 classpath 下的所有类。

不同的ClassLoader会去不同路径下加载类。BootstrapClassLoader、ExtClassLoader、AppClassLoader 加载资源文件的路径实际是根据相应的环境属性，依次是：`sun.boot.class.path`、`java.ext.dirs`和`java.class.path`来。

## 加载顺序

Bootstrap ClassLoder  > Extention ClassLoader > Application ClassLoader

### Bootstrap ClassLoader

Bootstrap ClassLoader是由C/C++编写的，它本身是虚拟机的一部分，所以它并不是一个JAVA类，也就是无法在java代码中获取它的引用，JVM启动时通过Bootstrap类加载器加载rt.jar等核心jar包中的class文件，int.class，String.class等基础类都是由它加载。接下来的类加载才交给java的核心库来完成。

### Extention ClassLoader and Application ClassLoader 

有兴趣可以分析 %JRE_HOME%/lib/jre/lib/rt.jar!/sun/misc/Launcher.class 代码，它是一个java虚拟机的入口应用。

分析代码，可以得到相关的信息。

1. Launcher初始化了ExtClassLoader和AppClassLoader，同时将ExtClassLoader设置为AppClassLoader的父加载器，而ExtClassLoader的父加载器为null。类加载器之间的父子关系不是通过继承来实现的，而是通过组合来实现。
2. Launcher中并没有看见BootstrapClassLoader，但通过`System.getProperty("sun.boot.class.path")`得到了字符串`bootClassPath`，这个应该就是BootstrapClassLoader加载的jar包路径。





双亲委托

jdk 1.2 引入

一个类加载器查找class和resource时，是通过“委托模式”进行的，它首先判断这个class是不是已经加载成功，如果没有的话它并不是自己进行查找，而是先通过父加载器，然后递归下去，直到Bootstrap ClassLoader，如果Bootstrap classloader找到了，直接返回，如果没有找到，则一级一级返回，最后到达自身去查找这些对象。这种机制就叫做双亲委托。

如何自定义ClassLoader



破坏双亲委派模型

