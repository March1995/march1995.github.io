---
title: Spring AnnotationConfigApplicationContext
date: 2025-06-10 
desc:
keywords: Spring IOC
categories: [Spring]
---

`AnnotationConfigApplicationContext` 是Spring框架中用于基于注解配置的应用上下文实现。它的整体逻辑流程涵盖了从配置类的加载、Bean定义的注册与解析，到Bean的实例化与初始化等一系列关键步骤。以下详细阐述其逻辑流程：

### 1. 构造函数
`AnnotationConfigApplicationContext` 提供了多个构造函数，这些构造函数是整个流程的入口。常见的构造函数有：
 - **无参构造函数**：创建一个空的 `AnnotationConfigApplicationContext`，后续需要手动调用 `register(Class<?>... componentClasses)` 方法注册配置类，再调用 `refresh()` 方法启动上下文。
 - **带有配置类参数的构造函数**：例如 `AnnotationConfigApplicationContext(Class<?>... componentClasses)`，接收一个或多个配置类作为参数。在构造函数内部，首先调用无参构造函数进行初始化，然后调用 `register(componentClasses)` 方法注册配置类，最后调用 `refresh()` 方法启动上下文。

### 2. `register(Class<?>... componentClasses)`
 - **作用**：将一个或多个配置类注册到应用上下文中。这些配置类通常使用 `@Configuration` 注解标记，并且可能包含使用 `@Bean` 注解定义的Bean方法，或者使用其他Spring注解（如 `@Component`、`@Service` 等）的类。
 - **实现**：内部通过 `AnnotatedBeanDefinitionReader` 来完成配置类的注册。`AnnotatedBeanDefinitionReader` 会解析配置类上的注解，将其转换为 `BeanDefinition`，并注册到 `BeanDefinitionRegistry`（即 `AnnotationConfigApplicationContext` 本身，因为它实现了 `BeanDefinitionRegistry` 接口）中。

### 3. `refresh()`
`refresh()` 方法是 `AnnotationConfigApplicationContext` 的核心方法，继承自 `AbstractApplicationContext`，负责执行整个上下文的初始化和启动过程，其详细步骤如下：

#### 3.1 `prepareRefresh()`
 - **作用**：准备上下文环境，包括记录启动时间、标记上下文为已启动状态、验证和准备属性源（如 `Environment`）等操作。还会清除早期预警标志，为后续的刷新操作做准备。

#### 3.2 `obtainFreshBeanFactory()`
 - **作用**：创建一个新的 `BeanFactory`（具体为 `DefaultListableBeanFactory`），并从已注册的配置类中加载 `BeanDefinition`。
 - **实现**：通过 `AnnotatedBeanDefinitionReader` 对已注册的配置类进行解析，将其中定义的Bean转换为 `BeanDefinition` 并注册到 `BeanFactory` 中。例如，配置类中的 `@Bean` 方法定义的Bean，会被解析为对应的 `BeanDefinition`。

#### 3.3 `prepareBeanFactory(beanFactory)`
 - **作用**：对新创建的 `BeanFactory` 进行各种配置。设置类加载器为上下文的类加载器，添加 `BeanExpressionResolver` 用于解析表达式，注册一些特殊的 `BeanPostProcessor`，并注册一些与环境相关的 `Bean`（如 `environment`、`systemProperties` 等）。

#### 3.4 `postProcessBeanFactory(beanFactory)`
 - **作用**：该方法为空实现，留给子类扩展。在 `BeanFactory` 初始化完成后，子类可以重写此方法对 `BeanFactory` 进行进一步的自定义修改。

#### 3.5 `invokeBeanFactoryPostProcessors(beanFactory)`
 - **作用**：调用所有已注册的 `BeanFactoryPostProcessor`。这些处理器可以在 `BeanFactory` 标准初始化之后，实例化任何Bean之前，对 `BeanDefinition` 进行修改。例如，`PropertyPlaceholderConfigurer` 就是一个 `BeanFactoryPostProcessor`，它可以将配置文件中的占位符替换为实际的值。

#### 3.6 `registerBeanPostProcessors(beanFactory)`
 - **作用**：注册 `BeanPostProcessor`，这些处理器用于在Bean的初始化前后进行额外的处理。例如，`AutowiredAnnotationBeanPostProcessor` 用于处理 `@Autowired` 注解，在Bean实例化后，为其自动装配依赖的Bean。

#### 3.7 `initMessageSource()`
 - **作用**：初始化 `MessageSource`，用于国际化消息处理。如果上下文中定义了 `MessageSource`，则会将其初始化；否则，会创建一个默认的 `MessageSource`。

#### 3.8 `initApplicationEventMulticaster()`
 - **作用**：初始化事件广播器 `ApplicationEventMulticaster`，用于发布应用程序事件。如果上下文中定义了 `ApplicationEventMulticaster`，则会将其初始化；否则，会创建一个默认的事件广播器。

#### 3.9 `onRefresh()`
 - **作用**：该方法为空实现，留给子类扩展。在上下文刷新时，子类可以重写此方法进行额外的操作，比如 `AbstractRefreshableWebApplicationContext` 中重写此方法来初始化 `ServletContext`。

#### 3.10 `registerListeners()`
 - **作用**：注册监听器，将上下文中定义的监听器注册到事件广播器中。这些监听器会监听事件广播器发布的事件，例如 `ContextRefreshedEvent` 等。

#### 3.11 `finishBeanFactoryInitialization(beanFactory)`
 - **作用**：实例化所有剩余的（非懒加载）单例Bean。会对 `BeanFactory` 中的 `BeanDefinition` 进行实例化和初始化，填充属性，并应用 `BeanPostProcessor`。例如，会调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 和 `postProcessAfterInitialization` 方法。

#### 3.12 `finishRefresh()`
 - **作用**：完成上下文刷新，发布上下文刷新完成事件 `ContextRefreshedEvent`，初始化生命周期处理器（`LifecycleProcessor`）并启动所有实现了 `Lifecycle` 接口的Bean。

### 4. 获取Bean
 - **作用**：上下文启动完成后，用户可以通过 `getBean(String name)` 或 `getBean(Class<T> requiredType)` 等方法从上下文中获取所需的Bean实例。
 - **实现**：`AnnotationConfigApplicationContext` 委托内部的 `BeanFactory`（即 `DefaultListableBeanFactory`）来查找和返回Bean实例。`BeanFactory` 会根据 `BeanDefinition` 创建Bean实例，并处理依赖注入、生命周期回调等操作。

通过以上流程，`AnnotationConfigApplicationContext` 完成了从配置类加载到Bean实例化和应用上下文启动的全过程，为Spring应用程序提供了基于注解配置的运行环境。 