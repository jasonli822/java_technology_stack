## 给你一份Spring Boot核心知识清单

<font size=2>CHEN川 [http://blog.51cto.com/luecsc/1964056](http://blog.51cto.com/luecsc/1964056)</font>

在过去两三年的Spring生态圈，最让人兴奋的莫过于Spring Boot框架。或许从命名上就能看出这个框架的设计初衷：快速的启动Spring应用。因而Spring Boot应用本质上就是一个基于Spring框架的应用，它是spring对“约定优于配置”理念的最佳实践产物，它能够帮助开发者更快速高效地构建基于Spring生态圈的应用。



那Spring Boot有何魔法？**自动装配、起步依赖、Actuator、命令行界面(CLI)**是Spring Boot最重要的4大核心特性，其中CLI是Spring Boot的可选特性，虽然它功能强大，但也引入了一套不太常规的模型，因而这个系统的文章仅关注其它3种特性。



如文章标题，本文是这个系列的第一部分，将为你打开Spring Boot的大门，重点为你剖析其启动流程以及自动配置实现原理。要掌握这部分核心内容，理解一些Spring框架的基础知识，将会让你事半功倍。



###  一、抛砖引玉：探索Spring IoC容器

如果看过`SpringApplication.run()`方法的源码，Spring Boot冗长无比的启动流程一定会让你抓狂，透过现象看本质，springAppplication只是将一个典型的Spring应用的启动流程进行了扩展，因此，透彻理解Spring容器是打开Spring Boot大门的一把钥匙。

#### 1.1、Spring IoC容器

可以把Spring IoC容器比作一间餐馆，当你来到餐馆，通常会直接招呼服务员：点菜！至于菜的原料是什么？如何用原料把菜做出来？可能你根本就不关心。IoC容器也是一样，你只需要告诉它需要某个bean，它就把对应的实例(instance)仍给你，至于这个bean是否依赖其他组件，怎样完成它的初始化，根本就不需要你关心。

作为餐馆，想要做出菜肴，得知道菜的原料和菜谱，同样地，IoC容器想要想要管理各个业务对象以及它们之间的依赖关系，需要通过某种途径来记录和管理这些信息。`BeanDefinition`对象就承担了这个责任：容器种的每一个bean都会有一个对应的BeanDefinition实例，该实例负责保存bean对象的所有必要信息，包括bean对象的class类型、是否是抽象类、构造方法和参数、其它属性等等。当客户端向容器请求相应对象时，容器就会通过这些信息为客户端返回一个完整可用的bean实例。



原材料已经准备好（把BeanDefinition看着原料），开始做菜吧，等等，你还需要一份菜谱，`BeanDefinitionRegistry`和`BeanFactory`就是这份菜谱，BeanDefinitionRegistry抽象出bean的注册逻辑，而BeanFactory则抽象出了bean的管理逻辑，而各个BeanFactory的实现类就具体承担了bean的注册以及管理工作。它们之间的关系就如下图：

![依赖关系图](https://github.com/jasonli822/java_technology_stack/blob/master/images/SpringBoot/%E6%A0%B8%E5%BF%83%E7%9F%A5%E8%AF%86%E6%B8%85%E5%8D%95/1.png)

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

这段代码仅为了说明BeanFactory底层的大致工作流程，实际情况会更加复杂，比如bean之间的依赖关系可能定义在外部配置文件(XML/Properties)中、也可能是注解方式。Spring IoC容器的整个工作流程大致可以分为两个阶段：

①、容器启动阶段

容器启动时，会通过某种途径加载 `ConfigurationMetaData`。除了代码方式比较直接外，在大部分情况下，容器需要依赖某些工具类，比如： `BeanDefinitionReader`，BeanDefinitionReader 会对加载的 `ConfigurationMetaData`进行解析和分析，并将分析后的信息组装为相应的 BeanDefinition，最后把这些保存了 bean 定义的 BeanDefinition，注册到相应的 BeanDefinitionRegistry，这样容器的启动工作就完成了。这个阶段主要完成一些准备性工作，更侧重于 bean 对象管理信息的收集，当然一些验证性或者辅助性的工作也在这一阶段完成。



来看一个简单例子吧，过往，所有的bean都定义在XML配置文件中，下面的代码将模拟BeanFactory如何从配置文件中加载bean的定义以及依赖关系：

```java
// 通常为BeanDefinitionRegistry的实现类，这里以DeFaultListabeBeanFactory为例
BeanDefinitionRegistry beanRegistry = new DefaultListableBeanFactory(); 
// XmlBeanDefinitionReader实现了BeanDefinitionReader接口，用于解析XML文件
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReaderImpl(beanRegistry);
// 加载配置文件
beanDefinitionReader.loadBeanDefinitions("classpath:spring-bean.xml");

// 从容器中获取bean实例
BeanFactory container = (BeanFactory)beanRegistry;
Business business = (Business)container.getBean("beanName");
```

②、Bean的实例化阶段

经过第一阶段，所有bean定义都通过BeanDefinition的方式注册到BeanDefinitionRegistory中，当某个请求通过容器的getBean方法请求某个对象，或者因为依赖关系容器需要隐士调用getBean时，就会触发第二阶段的活动：容器会首先检查所请求的对象之前是否已经实例化完成。如果没有，则会根据注册的BeanDefinition所提供的信息实例化被请求对象，并为其注入依赖。当该对象装配完毕后，容器会立即将其返回给请求方法使用。



BeanFactory只是Spring IoC容器的一种实现，如果没有特殊指定它采用延迟初始化策略:只有当访问容器中的某个对象时，才对该对象进行初始化和依赖注入操作。而在实际场景下，我们更多的使用另外一种类型的容器：`ApplicationContext`，它构建在BeanFactory之上，属于更高级的容器，除了具有BeanFactory的所有能力之外，还提供对事件监听机制以及国际化支持等。它管理的bean,在容器启动时全部完成初始化和依赖注入操作。



#### 1.2、Spring容器扩展机制

IoC容器负责管理容器中所有bean的生命周期，而在bean生命周期的不同阶段，Spring提供了不同的扩展点来改变bean的命运。在容器的启动阶段， BeanFactoryPostProcessor允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作，比如修改bean定义的某些属性或者增加其它信息等。



如果要自定义扩展类，通常要实现 `org.springframework.beans.factory.config.BeanFactoryPostProcessor`接口，与此同时，因为容器中可能有多个BeanFactoryPostProcessor，可能还需要实现 `org.springframework.core.Ordered`接口，以保证BeanFactoryPostProcessor按照顺序执行。Spring提供了为数不多的BeanFactoryPostProcessor实现，我们以 `PropertyPlaceholderConfigurer`来说明其大致的工作流程。



在Spring项目的XML配置文件中，经常可以看到许多配置项的值使用占位符，二将占位符所代表的值单独配置到独立的properties文件，这样可以将散落在不同XML文件中的配置集中管理，而且也方便运维根据不同的环境进行配置不同的值。这个非常实用的功能就是由PropertyPlaceholderConfigure负责实现的。



根据前文，当BeanFactory在第一阶段加载完所有配置信息时，BeanFactory中保存的对象的属性还是以占位符方式存在的，比如 `${jdbc.mysql.url}`。当PropertyPlaceholderConfigurer作为BeanFactoryPostProcessor被应用时，它会使用properties配置文件中的值来替换相应的BeanDefinition中占位符所表示的属性值。当需要实例化bean时，bean定义中的属性值就已经被替换成我们配置的值。当然其实现比上面描述的要复杂一些，这里仅说明其大致工作原理，更详细的实现可以参考其源码。



与之相似的，还有 `BeanPostProcessor`，其存在于对象实例化阶段。跟BeanFactoryPostProcessor类似，它会处理容器内所有符合条件并且已经实例化后的对象。简单的对比，BeanFactoryPostProcessor处理bean的定义，而BeanPostProcessor则处理bean完成实例化后的对象。BeanPostProcessor定义了两个接口：

```java
public interface BeanPostProcessor {
    // 前置处理
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    // 后置处理
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

为了理解这两个方法执行的时机，简单的了解下bean的整个生命周期：

![bean生命周期](https://github.com/jasonli822/java_technology_stack/blob/master/images/SpringBoot/%E6%A0%B8%E5%BF%83%E7%9F%A5%E8%AF%86%E6%B8%85%E5%8D%95/2.png)

`postProcessBeforeInitialization()`方法与 `postProcessAfterInitialization()`分别对应图中前置处理和后置处理两个步骤将执行的方法。这两个方法中都传入了bean对象实例的引用，为扩展容器的对象实例化过程提供了很大便利，在这儿几乎可以对传入的实例执行任何操作。注解、AOP等功能的实现均大量使用了 `BeanPostProcessor`，比如有一个自定义注解，你完全可以实现BeanPostProcessor的接口，在其中判断bean对象的脑袋上是否有该注解，如果有，你可以对这个bean实例执行任何操作，想想是不是非常的简单？

再来看一个更常见的例子，在Sprig中经常能看到各种各样的Aware接口，其作用就是在对象实例化完成以后将Aware接口定义中规定的依赖注入到当前实例中。比较常见的`ApplicationCtontextAware`接口，实现了这个接口的类都可以获取一个ApplicationContext对象。当容器中每个对象的实例化过程走到BeanPostProcessor前置处理这一步时，容器会检测到之前注册到容器的ApplicationContextAwareProcessor,然后就会调用其postProcessBeforeInitialization()方法，检查并设置Aware相关依赖。看看代码吧，是不是很简单：

```java
// 代码来自：org.springframework.context.support.ApplicationContextAwareProcessor
// 其postProcessBeforeInitialization方法调用了invokeAwareInterfaces方法
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
    // ......
}
```

最后总结一下，本小节内容和你一起回顾了Sprig容器的部分核心内容，限于篇幅不能写更多，但理解这部分内容，足以让你轻松理解Spring Boot的启动原理，如果在后续的学习过程中遇到一些晦涩难懂的知识，再回过头来看看Spring的核心知识，也许有意想不到的效果。也许Spring Boot的中文资料很少，但Spring中文资料和书籍有太多太多，总有东西能给你启发。



### 二、夯实基础：JavaConfig与常见Annotation

#### 2.1、JavaConfig

我知道bean是Spring IOC种非常核心的概念，Spring容器负责bean的生命周期的管理。在最初，Spring使用XML配置文件的方式来描述bean的定义以及相互之间的依赖关系，但随着 Spring的发展，越来越多的人对这种方式表示不满，因为Spring项目的所有业务类均以bean的形式配置在XML文件中，造成了大量的XML文件，使项目变得复杂且难以管理。

后来，基于纯Java Annotation依赖诸如胯关节`Guice`出世，其性能明显优于采用XML方式的Spring，甚至有部分人认为，`Guice`可以完全取代Spring（Guice仅是一个轻量级IOC框架，取代Spring还差的挺远）。正式这样的危机感，促使Spring及社区推出并持续完善了`JavaConfig`子项目，它基于Java代码和Annotation注解来描述 bean之间的依赖绑定关系。比如，下面是使用 XML 配置方式来描述bean的定义。

```xml
<bean id="bookService" class="cn.moondev.service.BookServiceImpl"></bean>
```

而基于JavaConfig的配置形式是这样的“

```java
@Configuration
public class MoonBookConfiguration {
    // 任何标志了@Bean的方法，其返回值将作为一个bean注册到Spring IOC容器中
    // 方法名默认成为该bean定义的id
    @Bean
    public BookService bookService() {
        return new BookServiceImpl();
    }
}
```

如果两个bean之间有依赖关系的话，在XML配置中应该是这样：

```xml
<bean id="bookService" class="cn.moondev.service.BookServiceImpl">
    <property name="dependencyService" ref="dependencyService"/>
</bean>
<bean id="otherService" class="cn.moondev.service.OtherServiceImpl">
    <property name="dependencyService" ref="dependencyService"/>
</bean>
<bean id="dependencyService" class="DependencyServiceImpl"/>
```

而在JavaConfig中则是这样：

```java
@Configuration
public class MoonBookConfiguration {
    // 如果一个bean依赖另一个bean,则直接调用对应JavaConfig类中依赖bean的创建方法即可
    // 这里直接调用dependencyService()
    @Bean
    public BookServcie bookServcie() {
        return new BookServiceImpl(dependencyService());
    }
    @Bean
    public OtherService othereServcie() {
        return new OtherServiceImpl(dependencyService);
    }
    @Bean
    public DependencyService dependencyService() {
        return new DependencyServiceImpl();
    }
}
```

#### 2.2、@ComponentScan

`@ComponentScan`注解对应XML配置形式中的`<context:component-scan>`元素，表示启用组件扫描，Spring会自动扫描所有通过注解配置的bean，然后将其注册到IOC容器中。我们可以通过`basePackages`等属性来指定`@ComponentScan`自动扫描的范围，如果不指定，默认从声明`@ComponentScan`所在类的`package`进行扫描。正因为如此，SpringBoot的启动类都默认在`src/main/java`下。

#### 2.3、@Import

`@Import`注解用于导入配置类，举个简单的例子：

```java
@Configuration
public class MoonBookConfiguration {
    @Bean
    public BookService bookService() {
        return new BookServiceImpl();
    }
}
```

现在有另外一个配置类，比如：`MoonUserConfiguration`，这个配置类中有一个bean依赖于 `MoonBookConfiguration`中的bookService，如何将这两个bean组合在一起？借助`@Import`即可：

```java
@Configuration
// 可以同时导入多个配置类，比如：@Import({A.class,B.class})
@Import(MoonBookConfiguration.class)
public class MoonUserConfiguration {
    @Bean
    public UserService userService(BookService bookService) {
        return new UserServiceImpl(bookService);
    }
}
```

需要注意的是，在4.2之前， `@Import`注解只支持导入配置类，但是在4.2之后，它支持导入普通类，并将这个类作为一个bean的定义注册到IOC容器中。

