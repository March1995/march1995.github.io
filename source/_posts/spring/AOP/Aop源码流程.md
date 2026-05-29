---
title: Spring AOP模块 
date: 2025-06-06 
desc:
keywords: Spring AOP
categories: [Spring]
---

# 类图

![类图](https://march1995.github.io/uploads/spring/aop/aop类图.png)

# aop功能代码分析图

![aop功能代码分析图](https://march1995.github.io/uploads/spring/aop/aop功能代码分析图.png)

### 重点关注 InstantiationAwareBeanPostProcessor 和 postProcessBeforeInstantiation() 后置处理。

# 1.InstantiationAwareBeanPostProcessor(类的实例化之前执行)

```java
try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    // 给bean的后置处理器一个机会来生成一个代理对象返回,在aop模块进行详细讲解
    // 由于bean未实例化，所以并未实际创建代理对象，只是如果对象有切面的话，找到对象切面并缓存
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
}
try {
    // 真正进行主要的业务逻辑方法来进行创建bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isTraceEnabled()) {
        logger.trace("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
}
```

## AbstractAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor接口

### 主要作用 

1.预测 Bean 类型：
- predictBeanType()方法允许在 Bean 实际创建之前预测其类型

2.确定候选构造函数：
- determineCandidateConstructors()方法可以影响 Spring 选择哪个构造函数来实例化 Bean

3.取早期 Bean 引用：
- getEarlyBeanReference()方法在 Bean 完全初始化前提供引用，用于解决循环依赖问题

### 典型应用场景

- AOP 代理：Spring AOP 使用它来预测代理对象的类型
- 循环依赖解决：在构造器注入循环依赖时发挥作用
- 自定义实例化逻辑：当需要特殊处理 Bean 的实例化过程时

### 执行流程 postProcessBeforeInstantiation

# 2.postProcessBeforeInstantiation(类的初始化之前或者之后执行)
