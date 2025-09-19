---
title: Spring Bean 生命周期
date: 2019-12-06 
desc:
keywords: spring Bean
categories: [Spring]
---

# 生命周期

![生命周期](/uploads/spring/springBeanLifeCycle.png)

# 单例模式

![单例模式生命周期](https://march1995.github.io/uploads/spring/单例模式springbean生命周期.jpg)

# 多例模式

需求主动调用才会创建对象，原因见下

```
LifeCycle lifeCycle = (LifeCycle) applicationContext.getBean("lifeCycle");
```


![多例模式生命周期](https://march1995.github.io/uploads/spring/多例模式springbean生命周期.jpg)

# 两者区别

## 单例模式

bean在单例模式下，spring容器启动时解析xml文件发现该bean标签后，直接创建该bean对象存入内部map中保存，
此后无论调用多少次getBean()获取该bean都是从map中获取该对象返回，一直是一个对象。
此对象一直被spring容器持有，直到容器推出时，随着容器的退出对象被销毁。

## 多例模式

bean在多例模式下，spring容器启动时解析xml发下该bean标签后，只是将该bean进行管理，并不会创建对象，
此后每层使用getBean()获取该bean时，spring都会重新创建该对象返回，每层都是一个新的对象。
这个对象spring容器并不会持有，什么时候销毁却决于该对象的用户自己什么时候销毁该对象。


# [相应测试源码](https://github.com/March1995/spring-example/blob/main/spring-example-test/src/main/java/com/wyb/test/spring/beanLifeCycle/LifeCycle.java)
