---
title: Spring 加载XML文件的六种方式
date: 2019-12-06 
desc:
keywords: spring xml
categories: [Spring]
---

# 一: XmlBeanFactory 引用资源 
```
Resource resource = new ClassPathResource("appcontext.xml");
BeanFactory factory = new XmlBeanFactory(resource);
BaseEntity entity = (BaseEntity) factory.getBean("baseEntity1");
System.out.println("XmlBeanFactory 获取 bean ：" + entity.getId());
```

# 二: ClassPathXmlApplicationContext  编译路径 
```
// 单个文件
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:appcontext.xml");
// 多个文件
ApplicationContext multiApplicationContext = new ClassPathXmlApplicationContext("classpath:appcontext.xml", "classpath:appcontext2.xml");
BaseEntity entity = (BaseEntity) applicationContext.getBean("baseEntity");

BaseEntity entity2 = (BaseEntity) multiApplicationContext.getBean("baseEntity2");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity.getId());
System.out.println("ClassPathXmlApplicationContext 获取 bean2 ：" + entity2.getId());
```

# 三: 用文件系统的路径 
```
// classPath
ApplicationContext fileSystemXmlApplicationContext = new FileSystemXmlApplicationContext("classpath:appcontext.xml");
// 文件系统
ApplicationContext fileSystemXmlApplicationContext2 = new FileSystemXmlApplicationContext("spring-example-test/src/main/resources/appcontext.xml");
ApplicationContext fileSystemXmlApplicationContext3 = new FileSystemXmlApplicationContext("file:E:\\marcher\\spring-example\\spring-example-test\\src\\main\\resources/appcontext.xml");
ApplicationContext fileSystemXmlApplicationContext4 = new FileSystemXmlApplicationContext("E:\\marcher\\spring-example\\spring-example-test\\src\\main\\resources/appcontext.xml");

BaseEntity entity = (BaseEntity) fileSystemXmlApplicationContext.getBean("baseEntity");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity.getId());

BaseEntity entity2 = (BaseEntity) fileSystemXmlApplicationContext2.getBean("baseEntity");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity2.getId());

BaseEntity entity3 = (BaseEntity) fileSystemXmlApplicationContext3.getBean("baseEntity");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity2.getId());

BaseEntity entity4 = (BaseEntity) fileSystemXmlApplicationContext4.getBean("baseEntity");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity2.getId());
```

# 四: XmlWebApplicationContext是专为Web工程定制的。 
ServletContext servletContext = request.getSession().getServletContext(); 
ApplicationContext ctx = WebApplicationContextUtils.getWebApplicationContext(servletContext ); 

# 五: 使用BeanFactory 
```
BeanDefinitionRegistry reg = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(reg);
reader.loadBeanDefinitions(new ClassPathResource("appcontext.xml"));
reader.loadBeanDefinitions(new ClassPathResource("appcontext2.xml"));
BeanFactory bf = (BeanFactory) reg;
BaseEntity entity = (BaseEntity) bf.getBean("baseEntity");
BaseEntity entity2 = (BaseEntity) bf.getBean("baseEntity2");
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity.getId());
System.out.println("ClassPathXmlApplicationContext 获取 bean ：" + entity2.getId());
```

# 六：Web 应用启动时加载多个配置文件 
通过ContextLoaderListener 也可加载多个配置文件，在web.xml文件中利用 
<context-pararn>元素来指定多个配置文件位置，其配置如下: 

```
<context-param>  
    <!-- Context Configuration locations for Spring XML files -->  
       <param-name>contextConfigLocation</param-name>  
       <param-value>  
       ./WEB-INF/**/Appserver-resources.xml,  
       classpath:config/aer/aerContext.xml,  
       classpath:org/codehaus/xfire/spring/xfire.xml,  
       ./WEB-INF/**/*.spring.xml  
       </param-value>  
   </context-param>  
```

这个方法加载配置文件的前提是已经知道配置文件在哪里，虽然可以利用“*”通配符，但灵活度有限。 

# [获取配套代码](https://github.com/march1995/spring-example/tree/master/spring-example-test/src/main/java/com/wyb/test/spring/loadXml)
