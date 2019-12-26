# Spring BeanFactory

​		之前我们已经模仿Spring的注解格式实现了一个IoC框架，之后以此为基础讲了AOP，但是很明显我们写的代码肯定是没有人家Spring写得好的。现在我们来看看Spring到底是怎么实现IoC的。

​		我们首先要意识到Spring并不是像我们之前所写的那样，一启动容器就开始了bean的实例化进程，只有当客户端通过显示或者隐式的方式调用BeanFactory的``getBean``方法来请求某个实例对象的时候，他才会出发相应的bean的实例化进程，当然也可以选择直接使用ApplicationContext容器，因为该容器启动的时候会立即调用注册到改容器的

## Bean的生命周期

​		我们先来看看Bean的生命周期，当然会很多看不懂的地方，但是我们需要做的是在脑子里面有一个印象，哦，bean在实例化的时候有这那么一个过程，就足够了。当我们后面具体学习的时候知道，哦，是有这么个东西，好像有些用处，就可以了。

​		下面是Spring Bean完整生命周期的一系列关键节点：

![img](https://images0.cnblogs.com/i/580631/201405/181453414212066.png)

![img](https://images0.cnblogs.com/i/580631/201405/181454040628981.png)

​		我们看到这个生命周期涉及到了一系列的接口。

​		我们可以自己定义这些接口的实现并将其注册（加一个@Component的事情），那么程序会按照上面的流程把你自己自定义的接口的实现加载进去，然后给其他bean做手脚。

## 接口方法整理

​		上面提到的接口这么多，我们来给他们分个类好记忆一点。

- Bean自身的方法：包括Bean本身调用的方法和通过配置文件中的``<bean>``的``init-method``和``destroy-method``指定的方法，当然你也可以通过``@PostConstruct``和``@PreDestroy``来指定这个方法（自己尝试一下就懂了）

- Bean级生命周期接口方法：包括了BeanNameAware，BeanFactoryAware，InitializingBean和DiposableBean这些接口的方法。

  这又可以分成两类，一类是BeanNameAware和BeanFactoryAware，Aware的意思为知道，它们俩个的作用分别是让Bean知道自己在Spring容器中的名字是什么，以及创建自己的工厂名字叫什么。

  那怎么让bean知道自己的名字呢？听起来挺高大上的。

  哎，你给Bean的类里加一个Field：String beanName，然后实现接口：``implements BeanNameAware``，实现方法：

  ```java
     public void setBeanName(String arg0) {
          this.beanName = arg0;
  }
  ```

  就完成了，在容器里会自动把Bean名字传进去。

  而InitializingBean，DisposableBean两个接口的作用则和上面我们提到的``@PostConstruct``与``@PreDestroy``类似。当你的对象实现了这两个接口，那肯定要实现这两个接口指定的方法：

  ```java
  public interface InitializingBean {    
      void afterPropertiesSet() throws Exception;
  }
  ```

  ```java
  public interface DisposableBean {
      void destroy() throws Exception;
  }
  ```

  当你的Bean实现了这俩接口之后，你的Bean在实例化之后会按照上面的流程，在指定的地方自动调用这两个方法。

- 容器级生命周期接口方法：包括InstantiantionAwareBeanPostProcessor和BeanPostProcessor这两个接口的实现。一般称它们的实现类为“后处理器”

  这两个接口的实现类会在非常非常早就实例化了，只要你实现了，他就会被实例化，然后在每个类的生命周期里作妖。

- 工厂后处理器接口方法：包括了AspectJWeavingEnabler，ConfigurationClassPostProcessor还有CustomAutowireConfigurer等等很有用的工厂后处理器接口。这些玩意也和上面一样是一样的，只要你往容器里面一放，你加载别的类的时候他就会自动搞事情。

​     我们先从Bean的配置与管理说起：BeanFactory

## BeanFactory

​		Spring Bean的创建是典型的工厂模式。在Spring中有很多IOC容器的实现供用户选择和使用。

![img](https://images0.cnblogs.com/blog/400827/201409/172219470349285.x-png)

​		我们可以看到最顶层的是BeanFactory，一个接口类，定义了IOC容器的基本功能规范。

​		我们来瞧一下BeanFactory定义了哪些方法：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.beans.factory;

import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;
import org.springframework.lang.Nullable;

public interface BeanFactory {
    
    String FACTORY_BEAN_PREFIX = "&";
	
    Object getBean(String var1) throws BeansException;
	
    <T> T getBean(String var1, Class<T> var2) throws BeansException;
	
    Object getBean(String var1, Object... var2) throws BeansException;
	
    <T> T getBean(Class<T> var1) throws BeansException;
	
    <T> T getBean(Class<T> var1, Object... var2) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> var1);

    <T> ObjectProvider<T> getBeanProvider(ResolvableType var1);

    boolean containsBean(String var1);

    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, ResolvableType var2) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, Class<?> var2) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String var1) throws NoSuchBeanDefinitionException;

    String[] getAliases(String var1);
}
```

​		看着这方法稍微有点多啊，但是我们搞清楚它们是用来干什么的就可以了。

- FACTORY_BEAN_PREFIX：一个用来找Bean的前缀
- getBean：通过各种参数获取一个Bean
- getBeanProvider：获得指定Bean的提供者
- containsBean：通过各种参数判断IoC中是否有你想要的Bean
- isSingletion：判断Bean的类型是否是Singletion（单例），在Spring中Bean的默认属性是Singletion
- isProtoType：判断Bean的类型是否是一个ProtoType类型，该属性会导致每次对该Bean的请求都会创建一个新的Bean实例。而对于具有ProtoType作用域的Bean，Spring不会对该Bean的声明周期负责，由调用者负责销毁对象回收资源。
- isTypeMatch：查询指定**名字**的Bean的Class类是否是指定的Class类
- getType：获得指定名字的Bean的Class类
- getAliases：获得指定名字的bean'的别名

​        我们先来看看最可能让人疑惑的一个方法：isProtoType。当然并不是这个方法的定义让人疑惑，而是什么是ProtoType。

​		众所周知，Spring中IoC的工厂方法提供的对象都是默认单例的。有“默认”这个词就说明还是有不是单例的时候的。

​		@Scope可以更改对应Bean在IoC的存在模式，分别如下：

- singleton单例模式

  全局有且仅有一个实例

- prototype原型模式

  每次获取Bean的时候会有一个新的实例

- request

  request表示该针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP request内有效

- session　

  session作用域表示该针对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP session内有效

- global session

  global session作用域类似于标准的HTTP Session作用域，不过它仅仅在基于portlet的web应用中才有意义。Portlet规范定义了全局Session的概念，它被所有构成某个 portlet web应用的各种不同的portlet所共享。在global session作用域中定义的bean被限定于全局portlet Session的生命周期范围内。如果你在web中使用global session作用域来标识bean，那么web会自动当成session类型来使用。

​        上面的内容大概知道是怎么回事就可以了，并不影响我们之后的探究。

​		除此之外我们还有一个更为重要的方法还在等待着我们探究：``getBeanProvider()``

​		我们可以看到它的返回值是一个ObjectProvider，其父级接口是ObjectFactory：

```java
//先忽略下迭代器，那不是我们讲Spring的重点
public interface ObjectProvider<T> extends ObjectFactory<T>, Iterable<T> 
```

​		ObjectFactory是个什么东西？再看看源码：

```java
@FunctionalInterface
public interface ObjectFactory<T> {
    T getObject() throws BeansException;
}
```

​		真的算得上是顾名思义了，就是一个获取对象的方法，甚至没有参数，全靠泛型作为返回对象的类。

​		调用这个方法返回的是一个对象的实例，通常用于封装也给泛型工厂，在每次调用的时候返回一个目标对象的新实例。

​		所以ObjectProvider就是一个实现了迭代器的，可以通过泛型返回对象的一个工厂。

​		除此之外，我们还有一个类你可能不怎么见过：``ResolvableType``。

​		ResolvableType是什么？

​		众所周知，泛型在编译的时候会被擦除。这导致如果我们想获得指定泛型类型的Bean是不能直接通过Class来获得的。（事实上也没有``List<Interger>.class``，你最多写个``List.class``）。但是使用ResolvableType就可以解决这个问题。比如说：

```java
ResolvableType type = ResolvableType.forClassWithGenerics(List.class, Integer.class);
ObjectProvider<List<Integer>> op = applicationContext.getBeanProvider(type);
List<Integer> bean = op.getIfAvailable()
```

​		其实这里实例化的type就可以等价于``List<Interger>.class``。

​		简单来说，ResolvableType是对于``java.lang.reflect.Type``的封装，并且提供了一些访问该类型的其他信息的方法（比如父类，泛型）。

​		在BeanFafctory下有三个子类（忘记的翻上去看图）：ListableBeanFactory，HierarchicalBeanFactory和AutowireCapableBeanFactory直接继承了BeanFactory，但是事实上这三个也都是接口。而后面跟着的花里胡哨的不是接口就是抽象类，BeanFacory真正的默认实现类是DefaultListableBeanFactory

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
```

​		那么问题来了，为什么要定义这么多接口呢？

​		那是因为这些接口的功能不同。

###HierarchicalBeanFactory

​		HierarchicalBeanFactory接口是在继承BeanFactory的基础上，实现BeanFactory的父子关系。

​		啥叫个实现了BeanFactory的父子关系？我们来瞧一瞧源码：

```java

/**
 * Sub-interface implemented by bean factories that can be part
 * of a hierarchy.
 * 
 * 可以被作为分层结构中的一部分的bean工厂实现
 *
 * <p>The corresponding {@code setParentBeanFactory} method for bean
 * factories that allow setting the parent in a configurable
 * fashion can be found in the ConfigurableBeanFactory interface.
 * 
 * 那些允许以配置的方式设置其父工厂的bean工厂对应的方法setParentBeanFactory可以在接口setParentBeanFactory
 * 中找到
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 07.07.2003
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#setParentBeanFactory
 */
public interface HierarchicalBeanFactory extends BeanFactory {
    /**
     * Return the parent bean factory, or {@code null} if there is none.
     * 返回其父工厂，如果没有返回Null
     */
    @Nullable
    BeanFactory getParentBeanFactory();

    /**
     * Return whether the local bean factory contains a bean of the given name,
     * ignoring beans defined in ancestor contexts.
     * 
     * 返回当前bean工厂上下文是否存在给定bean名字的bean，忽略定义在其继承层次中的工厂上下文。
     * 
     * <p>This is an alternative to {@code containsBean}, ignoring a bean
     * of the given name from an ancestor bean factory.
     * 
     * containsBean方法与此方法是二选一的，都忽略其继承层次中的bean定义，只在当前层次中查找 
     * 
     * @param name the name of the bean to query
     * @return whether a bean with the given name is defined in the local factory
     * @see BeanFactory#containsBean
     */
    boolean containsLocalBean(String var1);
}
```

​		我们在注释里面看到很重要的两个单词，一个是parent，一个是local，这说明了BeanFactory是有分层的。且子类具有指向父类的指针。

​		当然有分层肯定就不只是一个BeanFactory（要不分个锤子的层），并且这些BeanFactory是有着父类和子类之间的联系的。

​		我们来瞧一瞧这个接口的一个重要的子接口：ConfigurableBeanFactory

```java
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory, SingletonBeanRegistry 
```

> Configuration interface to be implemented by most bean factories. Provides facilities to configure a bean factory, in addition to the bean factory client methods in the [`BeanFactory`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html)
>
> This bean factory interface is not meant to be used in normal application code: Stick to [`BeanFactory`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html) or [`ListableBeanFactory`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/ListableBeanFactory.html) for typical needs. This extended interface is just meant to allow for framework-internal plug'n'play and for special access to bean factory configuration methods.
>
> Configuration interface将会被大多数的bean factories实现。除了BeanFactory接口中的bean工厂客户端方法之外，本接口还提供了配置一个bean工厂的功能。
>
> 此bean工厂接口并不打算在常规应用程序代码中使用：用BeanFactory或ListableBeanFactory以满足典型需求。此扩展接口仅用于允许在框架内部进行即插即用，并允许对bean工厂配置方法的特殊访问。

​		以上就是Spring官方对于ConfigurableBeanFactory的注解。

​		我们先来看看这个子接口继承的另外一个接口：

```java
package org.springframework.beans.factory.config;


public interface SingletonBeanRegistry {

    //在给定的bean名称下，在bean注册表中将给定的现有对象注册为单例。
    void registerSingleton(String beanName, Object singletonObject);

	//返回一个单例类
    Object getSingleton(String beanName);

    //判断容器红是否存在这个单例的bean
    boolean containsSingleton(String beanName);
    
	//返回这个单例bean的所有名字
    String[] getSingletonNames();

    //统计单例Bean的个数
    int getSingletonCount();
    //返回此注册表使用的单例互斥体（对于外部协作者）。
    Object getSingletonMutex();

}
```

​		不难看出它的主要功能就是注册并管理单例bean。

​		再看看源码：

```java
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory, SingletonBeanRegistry {
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";

    void setParentBeanFactory(BeanFactory var1) throws IllegalStateException;

    void setBeanClassLoader(@Nullable ClassLoader var1);

    @Nullable
    ClassLoader getBeanClassLoader();
	/*
	* Specify a temporary ClassLoader to use for type matching purposes.
	* A temporary ClassLoader is usually just specified if load-time weaving is involved, to make sure that actual bean classes are loaded as lazily as possible. The temporary loader is then removed once the BeanFactory completes its bootstrap phase.
	* 如果类加载过程中需要织入（Proxy.newProxyInstance()），则指定这个临时类加载器去加载。以保证尽可能延迟的加载实际的bean类。beanFactory完成其引导阶段后，会删除临时加载程序
	*/
    void setTempClassLoader(@Nullable ClassLoader var1);

    @Nullable
    ClassLoader getTempClassLoader();

    /*
    * Set whether to cache bean metadata such as given bean definitions (in merged fashion) and resolved bean classes.
    * 设置是否有bean相关的数据的缓存（given bean definitions (in merged fashion) and resolved bean classes.）
    * Turn this flag off to enable hot-refreshing of bean definition objects and in particular bean classes. If this flag is off, any creation of a bean instance will re-query the bean class loader for newly resolved classes.
    * 如果这个flag是false，则任何创建bean实例的操作都会重新查询bean类加载器以获取新解析的类。
    */
    void setCacheBeanMetadata(boolean var1);

    boolean isCacheBeanMetadata();

    void setBeanExpressionResolver(@Nullable BeanExpressionResolver var1);

    @Nullable
    BeanExpressionResolver getBeanExpressionResolver();

    void setConversionService(@Nullable ConversionService var1);

    @Nullable
    ConversionService getConversionService();

    void addPropertyEditorRegistrar(PropertyEditorRegistrar var1);

    void registerCustomEditor(Class<?> var1, Class<? extends PropertyEditor> var2);

    void copyRegisteredEditorsTo(PropertyEditorRegistry var1);

    void setTypeConverter(TypeConverter var1);

    TypeConverter getTypeConverter();
	//Add a String resolver for embedded values such as annotation attributes.
    void addEmbeddedValueResolver(StringValueResolver var1);
	
    boolean hasEmbeddedValueResolver();

    @Nullable
    String resolveEmbeddedValue(String var1);

    //Add a new BeanPostProcessor that will get applied to beans created by this factory.
    void addBeanPostProcessor(BeanPostProcessor var1);

    int getBeanPostProcessorCount();

    void registerScope(String var1, Scope var2);

    String[] getRegisteredScopeNames();

    @Nullable
    Scope getRegisteredScope(String var1);

    AccessControlContext getAccessControlContext();

    void copyConfigurationFrom(ConfigurableBeanFactory var1);

    void registerAlias(String var1, String var2) throws BeanDefinitionStoreException;

    void resolveAliases(StringValueResolver var1);

    BeanDefinition getMergedBeanDefinition(String var1) throws NoSuchBeanDefinitionException;

    boolean isFactoryBean(String var1) throws NoSuchBeanDefinitionException;

    void setCurrentlyInCreation(String var1, boolean var2);

    boolean isCurrentlyInCreation(String var1);

    void registerDependentBean(String var1, String var2);

    String[] getDependentBeans(String var1);

    String[] getDependenciesForBean(String var1);

    void destroyBean(String var1, Object var2);

    void destroyScopedBean(String var1);

    void destroySingletons();
}

```

​		总之这就是一个可以初始化



###ListableBeanFactory

​		让我们来康康这个Factory给BeanFactory整了些什么好功能，来康康源码：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.beans.factory;

import java.lang.annotation.Annotation;
import java.util.Map;
import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;
import org.springframework.lang.Nullable;

public interface ListableBeanFactory extends BeanFactory {
    // 是否包含给定名字的bean的定义
    boolean containsBeanDefinition(String var1);
	// 工厂中bean的定义的数量
    int getBeanDefinitionCount();
	// 工厂中所有定义了的bean的名字
    String[] getBeanDefinitionNames();
	// 获取指定类型的bean的名字
    String[] getBeanNamesForType(ResolvableType var1);
    String[] getBeanNamesForType(@Nullable Class<?> var1);
    String[] getBeanNamesForType(@Nullable Class<?> var1, boolean var2, boolean var3);
    // 根据指定的类型来获取所有的bean
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> var1) throws BeansException;
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> var1, boolean var2, boolean var3) throws BeansException;
	//获得所有拥有指定注解的Bean的名字
    String[] getBeanNamesForAnnotation(Class<? extends Annotation> var1);
	// 获取所有拥有指定注解的Bean
    Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> var1) throws BeansException;
	// 查找指定bean中的所有指定的注解（会考虑接口和父类中的注解）
    @Nullable
    <A extends Annotation> A findAnnotationOnBean(String var1, Class<A> var2) throws NoSuchBeanDefinitionException;
}
```

​		上面的方法都不考虑父工厂中的Bean，只会考虑当前工厂中所定义的Bean。

​		我觉得这些方法的具体作用我都写到了注释上，应该可以看懂吧。

​		这个接口要求实现的功能是列出工厂中所有的bean的能力，很算是对得起Listable这个名字了。

###AutowireCapableBeanFactory

​		咱们可以先望文生义一波，有Autowire，这肯定是和注入有关系，又有Capable（有能力的），说明肯定是给BeanFactory加上注入的功能。当然不能只望文生义，还是得看看源码：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//
package org.springframework.beans.factory.config;

import java.util.Set;
import org.springframework.beans.BeansException;
import org.springframework.beans.TypeConverter;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.lang.Nullable;

public interface AutowireCapableBeanFactory extends BeanFactory {
     // 常量，用于标识外部自动装配功能是否可用。但是此标识不影响正常的（基于注解的等）自动装配功能的使用
    int AUTOWIRE_NO = 0;
    //标识按名装配的常量
    int AUTOWIRE_BY_NAME = 1;
    //标识按类型自动装配的常量
    int AUTOWIRE_BY_TYPE = 2;
    //标识按照贪婪策略匹配出的最符合的构造方法来自动装配的常量
    int AUTOWIRE_CONSTRUCTOR = 3;
    //标识自动识别一种装配策略来实现自动装配的常量
    /** @deprecated */
    @Deprecated
    int AUTOWIRE_AUTODETECT = 4;
    String ORIGINAL_INSTANCE_SUFFIX = ".ORIGINAL";

  	/**
     * 创建一个给定Class的实例。
     * 执行此Bean所有的关于h的接口方法如BeanPostProcessor
     * 此方法用于创建一个新实例，它会处理各种带有注解的域和方法，并且会调用所有Bean初始化时所需要调用的回调函数
     * 此方法并不意味着by-name或者by-type方式的自动装配，如果需要使用这写功能，可以使用其重载方法
     */
    <T> T createBean(Class<T> var1) throws BeansException;
    /**
     * Populate the given bean instance through applying after-instantiation callbacks
     * 通过调用给定Bean的after-instantiation及post-processing接口，对bean进行配置。
     * 此方法主要是用于处理Bean中带有注解的域和方法。
     * 此方法并不意味着by-name或者by-type方式的自动装配，如果需要使用这写功能，可以使用其重载方法autowireBeanProperties
     */
    void autowireBean(Object var1) throws BeansException;
    /**
     * Configure the given raw bean: autowiring bean properties, applying
     * 配置参数中指定的bean，包括自动装配其域，对其应用如setBeanName功能的回调函数。
     * 并且会调用其所有注册的post processor.
     * 此方法提供的功能是initializeBean方法的超集，会应用所有注册在bean definenition中的操作。
     * 不过需要BeanFactory 中有参数中指定名字的BeanDefinition。
     */
    Object configureBean(Object var1, String var2) throws BeansException;
    /**
     * 创建一个指定class的实例，通过参数可以指定其自动装配模式（by-name or by-type）.
     * 会执行所有注册在此class上用以初始化bean的方法，如BeanPostProcessors等
     */
    Object createBean(Class<?> var1, int var2, boolean var3) throws BeansException;
    /**
     * 通过指定的自动装配策略来初始化一个Bean。
     * 此方法不会调用Bean上注册的诸如BeanPostProcessors的回调方法
     */
    Object autowire(Class<?> var1, int var2, boolean var3) throws BeansException;
    /**
     * 通过指定的自动装配方式来对给定的Bean进行自动装配。
     * 不过会调用指定Bean注册的BeanPostProcessors等回调函数来初始化Bean。
     * 如果指定装配方式为AUTOWIRE_NO的话，不会自动装配属性，但是依然会调用BeanPiostProcesser等回调方法。
     */
    void autowireBeanProperties(Object var1, int var2, boolean var3) throws BeansException;
    /**
     * 将参数中指定了那么的Bean，注入给定实例当中
     * 此方法不会自动注入Bean的属性，它仅仅会应用在显式定义的属性之上。如果需要自动注入Bean属性，使用
     * autowireBeanProperties方法。
     * 此方法需要BeanFactory中存在指定名字的Bean。除了InstantiationAwareBeanPostProcessor的回调方法外，
     * 此方法不会在Bean上应用其它的例如BeanPostProcessors
     * 等回调方法。不过可以调用其他诸如initializeBean等方法来达到目的。
     */
    void applyBeanPropertyValues(Object var1, String var2) throws BeansException;
    /**
     * 初始化参数中指定的Bean，调用任何其注册的回调函数如setBeanName、setBeanFactory等。
     * 另外还会调用此Bean上的所有postProcessors 方法
     */
    Object initializeBean(Object var1, String var2) throws BeansException;
     // 调用参数中指定Bean的postProcessBeforeInitialization方法
    Object applyBeanPostProcessorsBeforeInitialization(Object var1, String var2) throws BeansException;
	//  调用参数中指定Bean的postProcessAfterInitialization方法
    Object applyBeanPostProcessorsAfterInitialization(Object var1, String var2) throws BeansException;
    /**
     * 销毁参数中指定的Bean，同时调用此Bean上的DisposableBean和DestructionAwareBeanPostProcessors方法
     * 在销毁途中，任何的异常情况都只应该被直接捕获和记录，而不应该向外抛出。
     */
    void destroyBean(Object var1);
    /**
     * 查找唯一符合指定类的实例，如果有，则返回实例的名字和实例本身
     * 和BeanFactory中的getBean(Class)方法类似，只不过多加了一个bean的名字
     */
    <T> NamedBeanHolder<T> resolveNamedBean(Class<T> var1) throws BeansException;
    /**
     * 解析出在Factory中与指定Bean有指定依赖关系的Bean
     * 参数建下一个方法
     */
    Object resolveBeanByName(String var1, DependencyDescriptor var2) throws BeansException;
    /**
     * 解析指定Bean在Factory中的依赖关系
     * @param descriptor 依赖描述 (field/method/constructor)
     * @param requestingBeanName 依赖描述所属的Bean
     * @param autowiredBeanNames 与指定Bean有依赖关系的Bean
     * @param typeConverter 用以转换数组和连表的转换器
     * @return the 解析结果，可能为null
     */
    @Nullable
    Object resolveDependency(DependencyDescriptor var1, @Nullable String var2) throws BeansException;

    @Nullable
    Object resolveDependency(DependencyDescriptor var1, @Nullable String var2, @Nullable Set<String> var3, @Nullable TypeConverter var4) throws BeansException;
}
```

​		这个类的注解看起来好长啊，莫慌莫慌，我们来对其解释一下。

​		我们先看一个在注释里经常出现的玩意：BeanPostProcessor，呦呦呦，老熟人了。还记得不，容器级别的那个，加载顺序及其靠前的，其他类实例化的时候就出来搞事的那玩意。

​		BeanPostProcessor是一个Spring IoC容器给我们提供的一个扩展接口，接口声明如下：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

​		下面是spring官方对第一个方法的说明

> Apply this `BeanPostProcessor` to the given new bean instance *before* any bean initialization callbacks (like InitializingBean's `afterPropertiesSet` or a custom init-method). The bean will already be populated with property values. The returned bean instance may be a wrapper around the original.

​		其实没什么新货，看看前面的流程图比这个好懂多了。

​		我们来举个例子：

```java
@Component
public class TestProcessors implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("before:"+beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("after:"+beanName);
        return bean;
    }
}
```

​		把这玩意随便放到什么Spring里面跑一下就懂了。

​		你在看接口的注释就应该大概明白这个工厂接口的方法就是为了管理Bean的生命周期。

​		对了Bean生命周期里还有一个BeanFactoryPostProcessor和InstantiationAwareBeanPostProcessorAdapter，其实也都类似于`BeanPostProcessor`，都属于Spring的钩子（查过模板设计模式的应该知道是啥意思）

​		



## 参考文献

https://www.cnblogs.com/zhangfengxian/p/11086695.html

https://blog.csdn.net/weixin_39165515/article/details/77096774

https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/

https://www.jianshu.com/p/7b77b05407ae

https://www.jianshu.com/p/869ed7037833