---
title: JVM类加载机制
date: 2018-11-01
desc: jvm知识要点,jvm学习笔记
keywords: jvm
categories: [ JVM ]
---

# 类加载机制

类是在运行期间第一次使用时动态加载的，而不是编译时期一次性加载。因为如果在编译时期一次性加载，那么会占用很多的内存。

## 类的生命周期

![类生命周期](/uploads/java/jvm/类生命周期.jpg)

## 类加载过程

### 加载

查找并加载类的二进制数据加载时类加载过程的第一个阶段，在加载阶段，虚拟机需要完成以下三件事情：

- 通过一个类的全限定名来获取其定义的二进制字节流。
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- 在Java堆中生成一个代表这个类的 java.lang.Class对象，作为对方法区中这些数据的访问入口。

相对于类加载的其他阶段而言，加载阶段（准确地说，是加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，因为开发人员既可以使用系统提供的类加载器来完成加载，也可以自定义自己的类加载器来完成加载。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，而且在Java堆中也创建一个
java.lang.Class类的对象，这样便可以通过该对象访问方法区中的这些数据。

### 验证：确保被加载的类的正确性

验证是连接阶段的第一步，目的是保证加载的类符合当前虚拟机的要求。验证阶段大致分四步：

- 文件格式验证：验证字节流是否符合Class文件格式的规范；例如：是否以 0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
- 元数据验证：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了
  java.lang.Object之外。
- 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
- 符号引用验证：确保解析动作能正确执行。

验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用
-Xverifynone参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

### 准备：为**类的静态变量**分配内存，初始化默认值 类型默认值 不是程序赋值

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。对于该阶段有以下几点需要注意：

- 这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
- 这里所设置的初始值通常情况下是数据类型默认的零值（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。
- 如果类字段的字段属性表中存在 ConstantValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。

### 解析：把类中的符号引用转换为直接引用

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。

符号引用就是一组符号来描述目标，可以是任何字面量。

直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

### 初始化 <clinit>

初始化，为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化。在Java中对类变量进行初始值设定有两种方式：

- 声明类变量是指定初始值
- 使用静态代码块为类变量指定初始值

初始化步骤：

- 假如这个类还没有被加载和连接，则程序先加载并连接该类
- 假如该类的直接父类还没有被初始化，则先初始化其直接父类
- 假如类中有初始化语句，则系统依次执行这些初始化语句

类初始化时机（主动，被动）

只有当对类的主动使用的时候才会导致类的初始化，类的主动使用包括以下六种：

- 创建类的实例，也就是new的方式
- 反射（如 Class.forName(“com.Test”)
- 访问某个类或接口的静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 初始化某个类的子类，则其父类也会被初始化
- Java虚拟机启动时被标明为启动类的类（ JavaTest），直接使用 java.exe命令来运行某个主类

被动引用的常见例子包括：

- 通过子类引用父类的静态字段，不会导致子类初始化。

> System.out.println(SubClass.value); // value 字段在 SuperClass 中定义

- 通过数组定义来引用类，不会触发此类的初始化。该过程会对数组类进行初始化，数组类是一个由虚拟机自动生成的、直接继承自 Object
  的子类，其中包含了数组的属性和方法。

> SuperClass[] sca = new SuperClass[10];

- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

> System.out.println(ConstClass.HELLOWORLD);

### 结束生命周期

- 执行了System.exit()方法
- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而异常终止
- 由于操作系统出现错误而导致Java虚拟机进程终止

## 类加载器

站在Java虚拟机的角度来讲，只存在两种不同的类加载器：启动类加载器：它使用C++实现（这里仅限于Hotspot，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的），是虚拟机自身的一部分；
所有其它的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类 java.lang.ClassLoader，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。

站在Java开发人员的角度来看，类加载器可以大致划分为以下三类：

- 启动类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如 rt.jar、charsets.jar等 
- 扩展类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的JAR类包
- 应用程序类加载器：负责加载ClassPath路径下的类包，主要就是加载你自己写的那些类
- 自定义加载器：负责加载用户自定义路径下的类包

## 类加载机制

- 全盘负责，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入
- 父类委托，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类
- 缓存机制，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，
  只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效

### 双亲委派

![双亲委派机制](/uploads/java/jvm/双亲委派机制.jpg)

双亲委派模型的工作流程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，
只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

#### 双亲委派机制:

- 1、当 AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。
- 2、当 ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader```去完成。
- 3、如果 BootStrapClassLoader加载失败（例如在 $JAVA_HOME/jre/lib里未查找到该class），会使用 ExtClassLoader来尝试加载；
- 4、若ExtClassLoader也加载失败，则会使用 AppClassLoader来加载，如果 AppClassLoader也加载失败，则会报出异常 ClassNotFoundException

#### ClassLoader源码分析：

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}

synchronized (getClassLoadingLock(name)) {
    // 首先判断该类型是否已经被加载
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        // 如果没有被加载，就委托给父类加载或者委派给启动类加载器加载
        long t0 = System.nanoTime();
        try {
            if (parent != null) {
                //如果存在父类加载器，就委派给父类加载器加载
                c = parent.loadClass(name, false);
            } else {
                //如果不存在父类加载器，就检查是否是由启动类加载器加载的类，通过调用本地方法native Class findBootstrapClass(String name)
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // ClassNotFoundException thrown if class not found
            // from the non-null parent class loader
        }

        if (c == null) {
            // 如果父类加载器和启动类加载器都不能完成加载任务，才调用自身的加载功能
            long t1 = System.nanoTime();
            c = findClass(name);

            // this is the defining class loader; record the stats
            PerfCounter.getParentDelegationTime().addTime(t1 - t0);
            PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
            PerfCounter.getFindClasses().increment();
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

#### 双亲委派模型意义：

- 系统类防止内存中出现多份同样的字节码
- 保证Java程序安全稳定运行

## 自定义类加载器

通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java类的字节码，为保证安全性，
这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。自定义类加载器一般都是继承自 ClassLoader类，
从上面对 loadClass方法来分析来看，我们只需要重写 findClass 方法即可。下面我们通过一个示例来演示自定义类加载器的流程：

```java
package com.wyb.test.java.reflect;

import java.io.*;

public class WybClassLoader extends ClassLoader {

    private String root;


    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);

        }
    }

    private byte[] loadClassData(String className) {

        String fileName = root + File.separatorChar + className.replace('.', File.separatorChar) + ".class";
        try {
            InputStream ins = new FileInputStream(fileName);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0;
            while ((length = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, length);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;


    }

    public String getRoot() {
        return root;
    }

    public void setRoot(String root) {
        this.root = root;
    }


    public static void main(String[] args) {
        WybClassLoader wybClassLoader = new WybClassLoader();
        // class 需要用javac编译  class不能放在类路径下

        wybClassLoader.setRoot("E:\\marcher\\spring-example\\spring-example-test\\temp");
        try {
            Class clazz = wybClassLoader.loadClass("WybClass");
            Object obj = clazz.newInstance();
            System.out.println(obj.getClass().getClassLoader());
        } catch (ClassNotFoundException | InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
        }
    }

}
```

自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。由于这里只是演示，我并未对class文件进行加密，因此没有解密的过程。这里有几点需要注意：

1、这里传递的文件名需要是类的全限定性名称，即 com.wyb.test.java.reflect.WybClass，因为 defineClass 方法是按这种格式进行处理的。

2、最好不要重写loadClass方法，因为这样容易破坏双亲委托模式。

3、这类Test 类本身可以被 AppClassLoader类加载，因此我们不能把 E://marcher//spring-example/temp.class放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由 AppClassLoader加载，而不会通过我们自定义类加载器来加载。

### Tomcat打破双亲委派机制

以Tomcat类加载为例，Tomcat 如果使用默认的双亲委派类加载机制行不行？
我们思考一下：Tomcat是个web容器， 那么它要解决什么问题： 
1. 一个web容器可能需要部署两个应用程序，不同的应用程序可能会依赖同一个第三方类库的不同版本，不能要求同一个类库在同一个服务器只有一份，因此要保证每个应用程序的类库都是独立的，保证相互隔离。 
2. 部署在同一个web容器中相同的类库相同的版本可以共享。否则，如果服务器有10个应用程序，那么要有10份相同的类库加载进虚拟机。 
3. web容器也有自己依赖的类库，不能与应用程序的类库混淆。基于安全考虑，应该让容器的类库和程序的类库隔离开来。 
4. web容器要支持jsp的修改，我们知道，jsp 文件最终也是要编译成class文件才能在虚拟机中运行，但程序运行后修改jsp已经是司空见惯的事情， web容器需要支持 jsp 修改后不用重启。

再看看我们的问题：Tomcat 如果使用默认的双亲委派类加载机制行不行？ 
答案是不行的。为什么？
第一个问题，如果使用默认的类加载器机制，那么是无法加载两个相同类库的不同版本的，默认的类加器是不管你是什么版本的，只在乎你的全限定类名，并且只有一份。
第二个问题，默认的类加载器是能够实现的，因为他的职责就是保证唯一性。
第三个问题和第一个问题一样。
我们再看第四个问题，我们想我们要怎么实现jsp文件的热加载，jsp 文件其实也就是class文件，那么如果修改了，但类名还是一样，类加载器会直接取方法区中已经存在的，修改后的jsp是不会重新加载的。那么怎么办呢？我们可以直接卸载掉这jsp文件的类加载器，所以你应该想到了，每个jsp文件对应一个唯一的类加载器，当一个jsp文件修改了，就直接卸载这个jsp类加载器。重新创建类加载器，重新加载jsp文件。

#### Tomcat自定义加载器详解

![Tomcat自定义加载器](/uploads/java/jvm/Tomcat自定义加载器.png)

tomcat的几个主要类加载器：

- commonLoader：Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问；
- catalinaLoader：Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见；
- sharedLoader：各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见；
- WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，比如加载war包里相关的类，每个war包应用都有自己的WebappClassLoader，实现相互隔离，比如不同war包应用引入了不同的spring版本，这样实现就能加载各自的spring版本；

从图中的委派关系中可以看出：
CommonClassLoader能加载的类都可以被CatalinaClassLoader和SharedClassLoader使用，从而实现了公有类库的共用，而CatalinaClassLoader和SharedClassLoader自己能加载的类则与对方相互隔离。

WebAppClassLoader可以使用SharedClassLoader加载到的类，但各个WebAppClassLoader实例之间相互隔离。

而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的热加载功能。

tomcat 这种类加载机制违背了java 推荐的双亲委派模型了吗？答案是：违背了。 

很显然，tomcat 不是这样实现，tomcat 为了实现隔离性，没有遵守这个约定，每个webappClassLoader加载自己的目录下的class文件，不会传递给父类加载器，打破了双亲委派机制。

#### 模拟实现Tomcat的webappClassLoader加载自己war包应用内不同版本类实现相互共存与隔离 