---
title: java异常体系结构
date: 2018-08-08 
desc: 基于jdk1.8
keywords: java exception
categories: [JavaSE]
---
# java异常体系结构图
![java异常体系结构](/uploads/java/javase/异常结构体系.png)

从图中可以发现（列出要点，简要说明）：

- 所以异常类的父类为Throwable类（表示可抛出），直接继承为Error和Exception两个子类，Error也是异常的一种。
- Error是程序无法处理的错误，它是由JVM产生和抛出的，比如OutOfMemoryError、ThreadDeath等。这些异常发生时，Java虚拟机（JVM）一般会选择线程终止。Exception是程序本身可以处理的异常，这种异常分两大类运行时异常和非运行时异常。程序中应当尽可能去处理这些异常。
- RuntimeException（也称不检查异常）：即可以编译通过，一般由程序的逻辑错误引起，开发过程中应尽量避免。例如：NullPointerException，IndexOutOfBoundsException等。**自定义异常一般继承此类**。
- RuntimeException以外的异常（IOException）：编译器在编译阶段进行处理，程序必须处理此类异常否则无法通过编译。

# try-catch-finally-return

具体例子见(个人demo):
> https://github.com/march1995/spring-example/blob/master/spring-example-test/src/main/java/com/wyb/test/java/exception/ReturnFinallyTest.java

# 自定义全局异常处理（配套源码）

> https://github.com/march1995/spring-example/tree/master/spring-example-exception

# 常见的异常打印信息

- getMessage(): 返回此throwable的详细消息字符串，只会获得具体的异常名称，比如说NullPoint 空指针,就告诉你说是空指针。
- printStackTrace():提供对打印的堆栈跟踪信息的编程访问,会打出详细异常,异常名称,出错位置,便于调试用。
- toString():返回此throwable的简短描述。

## 注意要点：
- 多个catch模块，子类在先，为了保证所有的catch都有存在的意义
- finally模块在try-catch的return或者异常处理之前执行
- 在以下4种特殊情况下，finally块不会被执行： 
1）在finally语句块中抛出了异常且未处理。 
2）在前面的代码中用了System.exit()退出程序。 
3）程序所在的线程死亡。 
4）CPU出现异常被关闭。
