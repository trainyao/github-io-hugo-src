@(linux)

#Spring与SpringBoot的IoC和DI
###前言
spring的内容有很多，而springBoot对于spring的优化的内容也有非常多方面可讲。这次主要分享的是spring和springBoot中的IoC和DI，以及我所看到的在我们java项目中的体现。

##Ioc(控制反转)
控制反转即反转控制类实例的方式。
从**控制**反转为**被控制**
从**自己**生成/销毁反转为**别人**生成/销毁
控制反转的好处在于：
- 业务逻辑中不用考虑实例的生成/销毁，可以专注于业务逻辑(使用类实例)
- 对测试更加友好(mock实例行为)

##DI(依赖注入)
依赖注入是控制反转的一种实现方法。"别人"管理类实例的生成/销毁，并将业务逻辑需要用到的类实例注入进来。

##Spring的IoC
一个Spring应用有一个应用上下文（ApplicationContext），充当类实例的容器，管理着类实例的生成和销毁，并在应用需要时返回/注入对应的类实例。
Spring按类实例的种类管理容器中的类实例（种类有唯一的种类id：beanId）。每一种类实例称为Bean。
与很多IoC框架类似，类实例的注入方式有很多种(单例/非单例)，Spring中类实例存在方式可以是：
- 单例 (singleton)
- 每次注入返回新的实例 (prototype)
- 每次请求返回同一的实例 (request)
- 每次回话返回同一实例 (session/global session)

以下是spring注册Bean的其中某些特性：
- **1. xml可以配置BeanId，默认以类名为beanId。**
- **2. 注解为bean的时候默认以类名为beanId，也可以指定beanId**
- **3. 在获取bean时，如果无法查找到beanId，会查找对应的类是否有注册，如果不存在，会查找对应类的子类是否有注册**

另外，spring的IoC和DI利用了java的注解实现，也是面向切面编程的一个体现。在加载应用的时候，切换到应用bean管理切面，扫描类并注册Bean，在需要注入时，再次切换到bean管理切面，返回需要的bean。

##SpringBoot的IoC
springBoot简化了spring的配置部分
spring通过xml配置文件扫描/注解方式注册类实例
springBoot默认通过注解方式注册类实例，并默认扫描application包以及子包的所有类实例。默认以注解方式标记类实例。
springBoot可以沿用spring注解配置Bean时的规则
