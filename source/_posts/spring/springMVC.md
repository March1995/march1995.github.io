---
title: Spring MVC入门
date: 2017-07-05 
desc:
keywords: springmvc
categories: [Spring]
---
# 1.架构

- 1.1架构图
![架构天](http://images2015.cnblogs.com/blog/932062/201609/932062-20160909153624488-530274633.png)

- 1.2架构流程
  
1.用户发送请求至前端控制器DispatcherServlet

2.DispatcherServlet收到请求调用HandlerMapping处理器映射器。

3.处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet。

4.DispatcherServlet通过HandlerAdapter处理器适配器调用处理器

5.执行处理器(Controller，也叫后端控制器)。

6.Controller执行完成返回ModelAndView

7.HandlerAdapter将controller执行结果ModelAndView返回

8.DispatcherServlet

DispatcherServlet将ModelAndView传给ViewReslover视图解析器

9.ViewReslover解析后返回具体View

10.DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）。

11.DispatcherServlet响应用户
