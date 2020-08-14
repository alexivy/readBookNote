
# Spring  

## FactoryBean  

FactoryBean是spring中提供的工厂类接口，用户可以通过实现该接口定制实例化bean的逻辑。  

接口中定义了以下3个方法：
  T getObject() 返回由FactoryBean创建的bean实例，如果isSingleton()返回true ，则该实例会放到Spring中单实例缓存池中。   
  boolean isSingleton() 返回由FactoryBean创建的bean 实例的作用域是singleton还是prototype。   
  Class\<T\> getObjectType() 返回FactoryBean创建的 bean 类型。   

当配置文件中\<bean\>的class属性配置的实现类TFB实现了FactoryBean接口时，获取此bean时获取到的实例不是TFB的实例，而是TFB实现的FactoryBean#getObject()方法所返回的对象。   

在类TFB中设置相应属性接收bean配置中的信息，在getObject()中对信息进行处理即可便捷的配置生成bean。  

## 循环依赖  

循环依赖分为三种：  

1. 构造器循环依赖  
无法完成依赖注入，抛出BeanCurrentlyInCreationException异常表示循环依赖。  
Spring容器将每一个正在创建的bean标识符放在一个“当前创建bean池”中，bean标识符在创建过程中将一直保持在这个池中，因此如果在创建bean过程中发现自己已经在“当前创建bean池”里时，即表示存在通过构造器注入构成的循环依赖。  
2. setter循环依赖  
对于setter注入造成的依赖是通过Spring容器提前引用刚完成构造器注入但未完成其他步骤（如setter注入）的bean来完成的，而且只能解决单例作用域的bean循环依赖。  
3. prototype范围的依赖处理  
无法完成依赖注入，Spring容器不缓存prototype作用域的bean，因此无法提前引用一个创建中的bean。  

## 创建实例  

1.选择构造方法  

选择创建实例的构造方法，分为两种情况，一种是通用的实例化，另一种是带有参数的实例化。并且根据选择的构造方法转换对应的参数类型。  

2.调用实例化策略进行实例化  

先判断如果beanDefinition.getMethodOverrides()为空，也就是用户没有使用replace或者lookup的配置方法，那么直接使用反射的方式，但是如果使用了这两个特性，就必须要使用动态代理的方式将包含两个特性所对应的逻辑的拦截增强器设置进去，以保证在调用方法的时候会被相应的拦截器增强，返回的实例为包含拦截器的代理实例。  

## 依赖注入  

依赖注入的几种类型  

1. 设值注入：IoC容器使用成员变量的setter方法来注入被依赖对象。  
2. 构造注入：IoC容器使用构造器来注入被依赖对象。  
3. 接口注入：让调用者Bean实现特定接口，IoC容器检测到该Bean实现该接口后会自动调用该Bean特定的setter方法，调用setter方法时将被依赖对象注入调用者Bean。典型的，Bean获取Spring容器、Bean获取自身的配置ID分别要实现ApplicationContextAware、BeanNameAware 接口，这就是接口注入。  

整体流程：根据两种注入类型（autowireByName/autowireByType），查找依赖的bean并统一存入PropertyValues中。在applyPropertyValues中进行依赖注入。之后会初始化（init-method）bean。  

## AOP  
AOP的实现核心是使用代理，主要步骤是：  
1. 创建代理  
Spring的代理中JDKProxy的实现和CglibProxy的实现，具体选择方法：  
<!-- JDKProxy的实现也是用了 Proxy和InvocationHandler -->
   - 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP，也可以强制使用CGLIB实现AOP。  
   - 如果目标对象没有实现了接口，必须采用CGLIB库，Spring会自动在JDK动态代理和CGLIB之间转换。  
2. 获取代理  

## spring事务传播行为  
事务传播行为是spring提供的事务增强特性，它用来描述一个被事务传播行为修饰的方法被嵌套进另一个方法时事务如何传播。  
```@Transactional(propagation = Propagation.REQUIRED ) ```
共7种。如图所示：![image](/image/202008110941spring%20transaction.png)  
在REQUIRED的情况下，处在1个事务中的多个方法只要有一个抛出异常回滚，那么这个事务就会回滚，即使异常在事务中其他方法中被catch。  
方法a调用方法b，当两个方法不在同一个事务中时，b抛出异常，b的事务回滚，a种catch了b抛出的异常，a的事务就不会回滚。  
以上两点可以总结为：方法抛出了异常，方法所在的事务就会回滚；事务b中的异常在事务a种被catch，事务a不会回滚。  

事务中有子事务时，外围事务回滚，子事务同样回滚；但可以对单独对子事务单独回滚（外围事务catch子事务中的异常），此时外围事务不回滚。  

参考[此文](https://segmentfault.com/a/1190000013341344)。  

# Spring MVC  

## DispatcherServlet  
DispatcherServlet是Spring MVC框架的核心控制器，Spring MVC框架用它来处理所有的HTTP请求，DispatcherServlet收到用户请求之后，再将用户请求转发给特定的业务控制器（controller），当Controller将请求处理结果放入到ModelAndView中以后，DispatcherServlet会根据ModelAndView选择合适的视图进行渲染。  

