---
title: Spring AnnotatedBeanDefinitionReader
date: 2025-06-10 
desc:
keywords: Spring IOC
categories: [Spring]
---

在 `AnnotationConfigApplicationContext` 的无参构造函数中，`AnnotatedBeanDefinitionReader` 的主要作用是为后续手动注册基于注解的 `BeanDefinition` 做准备。以下详细介绍其逻辑流程和作用：

1. **`AnnotationConfigApplicationContext` 无参构造函数调用 `AnnotatedBeanDefinitionReader` 相关逻辑**
    ```java
    public AnnotationConfigApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    ```
    在无参构造函数中，创建了 `AnnotatedBeanDefinitionReader` 和 `ClassPathBeanDefinitionScanner` 实例。这里重点分析 `AnnotatedBeanDefinitionReader`。

2. **`AnnotatedBeanDefinitionReader` 的构造函数**
    ```java
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        Assert.notNull(environment, "Environment must not be null");
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    ```
    - **参数校验**：确保传入的 `BeanDefinitionRegistry` 和 `Environment` 不为空。`BeanDefinitionRegistry` 用于注册 `BeanDefinition`，在 `AnnotationConfigApplicationContext` 中，它自身实现了该接口。
    - **初始化条件评估器**：创建 `ConditionEvaluator`，用于评估 `@Conditional` 注解。`@Conditional` 注解允许根据特定条件决定是否注册某个 `Bean`。例如，如果系统属性满足特定条件，才注册某个 `Bean`。
    - **注册注解配置处理器**：调用 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 方法，注册一系列与注解处理相关的 `BeanDefinition`。这些处理器包括：
        - `ConfigurationClassPostProcessor`：处理 `@Configuration` 注解的配置类，解析配置类中的 `@Bean` 方法等。
        - `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired` 注解，实现自动装配功能。
        - `RequiredAnnotationBeanPostProcessor`：处理 `@Required` 注解，确保被注解的属性被设置。
        - `CommonAnnotationBeanPostProcessor`：处理 `@Resource`、`@PostConstruct` 和 `@PreDestroy` 等注解。
        - `PersistenceAnnotationBeanPostProcessor`（如果类路径中有相关的持久化依赖）：处理持久化相关的注解。

3. **`AnnotatedBeanDefinitionReader` 的 `register` 方法逻辑（用于注册Bean）**
    ```java
    public void register(Class<?>... componentClasses) {
        for (Class<?> componentClass : componentClasses) {
            registerBean(componentClass);
        }
    }
    ```
    - **遍历注册类**：`register` 方法接受多个 `Class` 参数，遍历这些类，对每个类调用 `registerBean` 方法。

    ```java
    public <T> void registerBean(Class<T> beanClass, @Nullable Supplier<T> supplier, @Nullable String... names,
                                 @Nullable Class<? extends Annotation>... qualifiers,
                                 BeanDefinitionCustomizer... definitionCustomizers) {

        AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
        if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
            return;
        }

        abd.setInstanceSupplier(supplier);
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        abd.setScope(scopeMetadata.getScopeName());
        String beanName = (names != null && names.length > 0)? names[0] : BeanDefinitionReaderUtils.generateBeanName(abd, this.registry);
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            } else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            } else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
        for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
            customizer.customize(abd);
        }

        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
        definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
        BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
    }
    ```
    - **创建 `AnnotatedGenericBeanDefinition`**：根据传入的 `beanClass` 创建 `AnnotatedGenericBeanDefinition`，它是 `BeanDefinition` 的一种实现，用于描述基于注解的Bean定义。
    - **条件评估**：使用 `ConditionEvaluator` 判断是否应跳过该 `BeanDefinition` 的注册，如果 `@Conditional` 注解条件不满足，则直接返回，不进行后续注册操作。
    - **设置实例供应商**：如果传入了 `Supplier`，则设置为 `BeanDefinition` 的实例供应商，用于创建Bean实例。
    - **解析作用域**：通过 `ScopeMetadataResolver` 解析 `BeanDefinition` 的作用域，如 `singleton`、`prototype` 等，并设置到 `BeanDefinition` 中。
    - **生成或指定Bean名称**：如果传入了自定义的Bean名称数组，则使用第一个名称；否则，通过 `BeanDefinitionReaderUtils.generateBeanName` 方法生成一个唯一的Bean名称。
    - **处理限定符**：遍历传入的限定符注解数组，根据不同的限定符注解设置 `BeanDefinition` 的相关属性，如 `@Primary` 注解设置 `BeanDefinition` 为主要的，`@Lazy` 注解设置为懒加载等。
    - **应用自定义器**：遍历 `BeanDefinitionCustomizer` 数组，对 `BeanDefinition` 进行自定义修改。
    - **创建 `BeanDefinitionHolder` 并应用作用域代理**：将 `BeanDefinition` 和Bean名称包装成 `BeanDefinitionHolder`，并根据作用域类型应用作用域代理（如果需要）。
    - **注册 `BeanDefinition`**：最后通过 `BeanDefinitionReaderUtils.registerBeanDefinition` 方法将 `BeanDefinitionHolder` 注册到 `BeanDefinitionRegistry` 中。

综上，`AnnotatedBeanDefinitionReader` 在 `AnnotationConfigApplicationContext` 的无参构造函数中初始化，并在后续手动注册基于注解的 `BeanDefinition` 时，负责解析注解、评估条件、设置属性和注册 `BeanDefinition` 等关键操作。 