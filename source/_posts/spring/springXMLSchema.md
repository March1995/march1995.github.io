---
title: springXMLSchema
date: 2017-07-05 
desc:
keywords: springmvc
categories: [Spring]
---

>出处 http://www.jianshu.com/p/1e35c15d0cb8

>参考资料 http://www.w3school.com.cn/schema/index.asp

# 自己理解整理：
- 基本概念：

XML Schema也被称为XML Schema定义（XML Schema Definition，XSD）。和DTD一样，Schema也是XML的约束，同样用于定义合法的XML文档构建模块。与DTD不同的是，XML Schema是用一套预先定义好的XML元素和属性创建的，这些元素和属性规定了XML文档的结构和内容模式，且XML Schema规定XML文档实例的结构和每一个元素或属性的数据类型。另外，Schema相对于DTD有一个明显的好处就是，Schema是基于XML编写的，自己本身也是一个XML文档（文件后缀名为.xsd），而不是像DTD有自成一套的语法，这也是Schema能比DTD更被广泛应用的原因。

- 名称空间：

在编写了一个XML Schema约束文档后，通常需要把这个文档定义的元素和属性绑定到一个URI地址上，这个地址就叫做名称空间。接着XML文档通过这个名称空间告诉解析引擎，文档中的元素和属性来自哪里。名称空间有什么用呢？就是用来唯一标识元素和属性来自哪个Schema。简单来说，当一个XML实例文档引用了多个Schema的时候，倘若这些Schema定义了同名的元素或属性，名称空间就可以将它们区分开来。

引用其他Schema

![引用其他Schema](http://upload-images.jianshu.io/upload_images/4942449-bd466d293a47249c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

自定义schema 给自己的文件命名方便其他文件调用

![](http://upload-images.jianshu.io/upload_images/4942449-575c97f367170ff5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4942449-09a4197ec412e170.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

xmlns后的属性即表示该文件中的context属性来源于这个ns

![](http://upload-images.jianshu.io/upload_images/4942449-09a4197ec412e170.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4942449-000dac2e6124f4d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 修改XML实例文档:

![](http://upload-images.jianshu.io/upload_images/4942449-e87ee2b5f0532d77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4942449-20e2d37ed4275474.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
