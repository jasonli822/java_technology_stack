## 给你一份Spring Boot核心知识清单

<font size=2>CHEN川 [http://blog.51cto.com/luecsc/1964056](http://blog.51cto.com/luecsc/1964056)</font>

在过去两三年的Spring生态圈，最让人兴奋的莫过于Spring Boot框架。或许从命名上就能看出这个框架的设计初衷：快速的启动Spring应用。因而Spring Boot应用本质上就是一个基于Spring框架的应用，它是spring对“约定优于配置”理念的最佳实践产物，它能够帮助开发者更快速高效地构建基于Spring生态圈的应用。



那Spring Boot有何魔法？**自动装配、起步依赖、Actuator、命令行界面(CLI)**是Spring Boot最重要的4大核心特性，其中CLI是Spring Boot的可选特性，虽然它功能强大，但也引入了一套不太常规的模型，因而这个系统的文章仅关注其它3种特性。



如文章标题，本文是这个系列的第一部分，将为你打开Spring Boot的大门，重点为你剖析其启动流程以及自动配置实现原理。要掌握这部分核心内容，理解一些Spring框架的基础知识，将会让你事半功倍。



### 抛砖引玉：探索Spring IoC容器

如果看过`SpringApplication.run()`方法的源码，Spring Boot冗长无比的启动流程一定会让你抓狂，透过现象看本质，springAppplication只是将一个典型的Spring应用的启动流程进行了扩展，因此，透彻理解Spring容器是打开Spring Boot大门的一把钥匙。

#### 1.1、Spring IoC容器

可以把Spring IoC容器比作一间餐馆，当你来到餐馆，通常会直接招呼服务员：点菜！至于菜的原料是什么？如何用原料把菜做出来？可能你根本就不关心。IoC容器也是一样，你只需要告诉它需要某个bean，它就把对应的实例(instance)仍给你，至于这个bean是否依赖其他组件，怎样完成它的初始化，根本就不需要你关心。

作为餐馆，想要做出菜肴，得知道菜的原料和菜谱，同样地，IoC容器想要想要管理各个业务对象以及它们之间的依赖关系，需要通过某种途径来记录和管理这些信息。`BeanDefinition`对象就承担了这个责任：容器种的每一个bean都会有一个对应的BeanDefinition实例，该实例负责保存bean对象的所有必要信息，包括bean对象的class类型、是否是抽象类、构造方法和参数、其它属性等等。当客户端向容器请求相应对象时，容器就会通过这些信息为客户端返回一个完整可用的bean实例。



原材料已经准备好（把BeanDefinition看着原料），开始做菜吧，等等，你还需要一份菜谱，`BeanDefinitionRegistry`和`BeanFactory`就是这份菜谱，BeanDefinitionRegistry抽象出bean的注册逻辑，而BeanFactory则抽象出了bean的管理逻辑，而各个BeanFactory的实现类就具体承担了bean的注册以及管理工作。它们之间的关系就如下图：



【图片1】



`DefaultListableBeanFactory`作为一个比较通用的BeanFactory实现，它同时也实现了BeanDefinitionRegistry接口，因此它就承担了Bean的注册管理工作。从图中可以看出，BeanFactory接口中主要包含getBean、containBean、getType、getAliases等管理bean的方法，而BeanDefinitionRegistry接口则包含registerBeanDefinition、removeBeanDefinition、getBeanDefinition等注册管理BeanDefinition的方法。



下面通过一段简单的代码来模拟BeanFactory底层是如何工作的：

```java
// 默认容器实现
DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
// 根据业务对象构造相应的BeanDefinition
AbstractBeanDefinition definition = new RootBeanDefinition(Business.class,true);
// 将bean定义注册到容器中
beanRegistry.registerBeanDefinition("beanName",definition);
// 如果有多个bean，还可以指定各个bean之间的依赖关系
// ........

// 然后可以从容器中获取这个bean的实例
// 注意：这里的beanRegistry其实实现了BeanFactory接口，所以可以强转，
// 单纯的BeanDefinitionRegistry是无法强制转换到BeanFactory类型的
BeanFactory container = (BeanFactory)beanRegistry;
Business business = (Business)container.getBean("beanName");
```





