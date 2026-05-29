---
title: Spring ClassPathBeanDefinitionScanner
date: 2025-06-10 
desc:
keywords: Spring IOC
categories: [Spring]
---

在 `AnnotationConfigApplicationContext` 中，`ClassPathBeanDefinitionScanner` 主要用于在指定的类路径下扫描带有特定注解的类，并将这些类注册为 `BeanDefinition`。以下详细介绍其逻辑流程和作用：

### 1. 初始化
在 `AnnotationConfigApplicationContext` 的无参构造函数中，会初始化 `ClassPathBeanDefinitionScanner`：
```java
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
这里将 `AnnotationConfigApplicationContext` 自身作为 `BeanDefinitionRegistry` 传递给 `ClassPathBeanDefinitionScanner` 的构造函数，使得扫描到的 `BeanDefinition` 可以注册到该应用上下文中。

### 2. 扫描配置
在使用 `ClassPathBeanDefinitionScanner` 进行扫描之前，通常会对其进行一些配置，例如设置要扫描的基础包路径、添加或移除注解过滤器等。例如：
```java
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry);
scanner.setResourceLoader(resourceLoader);
scanner.addIncludeFilter(new AnnotationTypeFilter(Component.class));
scanner.scan("com.example.package");
```
 - **设置资源加载器**：`setResourceLoader` 方法设置用于加载类路径资源的 `ResourceLoader`，以便能够读取类文件。
 - **添加注解过滤器**：`addIncludeFilter` 方法添加一个过滤器，用于指定要扫描的注解类型。这里添加了对 `@Component` 注解的过滤，意味着只有带有 `@Component` 及其衍生注解（如 `@Service`、`@Repository`、`@Controller`）的类会被扫描到。也可以通过 `addExcludeFilter` 方法添加排除过滤器。

### 3. 扫描过程
`scan` 方法是扫描的入口：
```java
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidateComponents = findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidateComponent);
            candidateComponent.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidateComponent, this.registry);
            if (candidateComponent instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidateComponent, beanName);
            }
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidateComponent);
            }
            if (checkCandidate(beanName, candidateComponent)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidateComponent, beanName);
                definitionHolder =
                    AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```
 - **遍历基础包**：`scan` 方法接受一个或多个基础包路径作为参数，遍历每个基础包路径。
 - **查找候选组件**：调用 `findCandidateComponents` 方法在指定的基础包及其子包下查找所有可能的候选组件（即带有特定注解的类）。这个方法会使用 `ResourcePatternResolver` 来查找类路径下符合条件的类文件，并将其解析为 `BeanDefinition`。例如，会将类路径下所有带有 `@Component` 注解的类解析为 `ScannedGenericBeanDefinition`。
 - **解析作用域**：对于每个候选组件，通过 `ScopeMetadataResolver` 解析其作用域（如 `singleton`、`prototype` 等），并设置到 `BeanDefinition` 中。
 - **生成Bean名称**：使用 `BeanNameGenerator` 为候选组件生成一个唯一的Bean名称。
 - **后置处理Bean定义**：如果 `BeanDefinition` 是 `AbstractBeanDefinition` 的实例，调用 `postProcessBeanDefinition` 方法进行后置处理，该方法在子类中可以被重写以进行自定义处理。
 - **处理通用注解**：如果 `BeanDefinition` 是 `AnnotatedBeanDefinition` 的实例，调用 `AnnotationConfigUtils.processCommonDefinitionAnnotations` 方法处理一些通用的注解，如 `@Lazy`、`@Primary` 等。
 - **检查候选Bean**：调用 `checkCandidate` 方法检查该 `BeanDefinition` 是否可以注册，例如检查是否有同名的Bean已经注册等。
 - **注册Bean定义**：如果检查通过，将 `BeanDefinition` 和生成的Bean名称包装成 `BeanDefinitionHolder`，并根据作用域类型应用作用域代理（如果需要）。最后通过 `BeanDefinitionReaderUtils.registerBeanDefinition` 方法将 `BeanDefinitionHolder` 注册到 `BeanDefinitionRegistry` 中。

### 4. 作用总结
 - **自动发现组件**：通过在指定的类路径下扫描，`ClassPathBeanDefinitionScanner` 能够自动发现应用中的各种组件，无需手动逐个注册 `BeanDefinition`，极大地提高了开发效率。这符合Spring的约定优于配置的理念，使得开发人员可以专注于业务逻辑的实现。
 - **灵活的配置**：通过设置不同的注解过滤器，可以灵活地控制哪些类会被扫描并注册为 `Bean`。同时，对于扫描到的 `BeanDefinition`，还可以进行各种后置处理和通用注解的处理，满足不同的业务需求。
 - **集成到应用上下文**：`ClassPathBeanDefinitionScanner` 与 `AnnotationConfigApplicationContext` 紧密集成，扫描到的 `BeanDefinition` 可以直接注册到应用上下文中，为后续的Bean实例化和依赖注入做好准备。

通过上述逻辑流程，`ClassPathBeanDefinitionScanner` 在 `AnnotationConfigApplicationContext` 中扮演了自动扫描和注册 `BeanDefinition` 的重要角色，是Spring基于注解配置的重要组成部分。 