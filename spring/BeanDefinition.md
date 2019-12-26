# BeanDefinition

## 定义

>A BeanDefinition describes a bean instance, which has property values, constructor argument values, and further information supplied by concrete implementations.
>
>This is just a minimal interface: The main intention is to allow a [`BeanFactoryPostProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanFactoryPostProcessor.html) to introspect and modify property values and other bean metadata.
>
>- **Since:**
>
>  19.03.2004
>
>- **Author:**
>
>  Juergen Hoeller, Rob Harrop

​		这个注释有以下重点：

- 一个BeanDefinition描述一个bean实例，并且有这个实例的很多信息（which has property values, constructor argument values, and further information supplied by concrete implementations）
- 它的主要作用是让BeanFactoryPostProcesser**自省**和修改**属性值和其他的对象元数据**

### 自省

> In [computing](https://en.wikipedia.org/wiki/Computing), **type introspection** is the ability of a program to *examine* the type or properties of an [object](https://en.wikipedia.org/wiki/Object_(computer_science)) at [runtime](https://en.wikipedia.org/wiki/Run_time_(program_lifecycle_phase)). Some [programming languages](https://en.wikipedia.org/wiki/Programming_language) possess this capability.
>
> Introspection should not be confused with [reflection](https://en.wikipedia.org/wiki/Reflection_(computer_programming)), which goes a step further and is the ability for a program to *manipulate* the values, meta-data, properties and/or functions of an object at runtime. Some programming languages - e.g. Java, Python and Go - also possess that capability.（维基百科）

​	在计算中，自省是程序在运行过程中检查一个object的类和属性的能力，一些编程语言拥有这个功能。

​	自省不应当与反射互相混淆，反射要比自省更进一步，反射让一个程序具有可以在运行时操作values，meta-data，properties （和/或）程序的能力。一些编程语言，比如java，python，和Go，也具备这种能力。

## 继承关系

​		``BeanDefinition``继承了``AttributeAccessor``和`BeanMetadataElement`接口。

###AttributeAccessor

```java
public interface AttributeAccessor {
    void setAttribute(String var1, @Nullable Object var2);

    @Nullable
    Object getAttribute(String var1);

    @Nullable
    Object removeAttribute(String var1);

    boolean hasAttribute(String var1);

    String[] attributeNames();
}
```

​		很明显就是用来访问bean的属性的。

### BeanMetadataElement

​		看看doc和源码可以得到：

> Interface to be implemented by bean metadata elements that carry a configuration source object.

```java
public interface BeanMetadataElement {
 //Return the configuration source Object for this metadata element (may be null).
    @Nullable
    Object getSource();
}
```

​		哦，获取指定对象的数据源对象！

##BeanDefinition源码及官方注释

​		现在来看看BeanDefinition源码以及官方doc给的注释

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	/**
	 * Scope identifier for the standard singleton scope: "singleton".
	 * <p>Note that extended bean factories might support further scopes.
	 *
	 * @see #setScope
	 */
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

	/**
	 * Scope identifier for the standard prototype scope: "prototype".
	 * <p>Note that extended bean factories might support further scopes.
	 *
	 * @see #setScope
	 */
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	/**
	 * Role hint indicating that a {@code BeanDefinition} is a major part
	 * of the application. Typically corresponds to a user-defined bean.
	 * 角色表示一个BeanDefinition是一个应用程序的主要部分，通常对应于用户定义的bean。
	 */
	int ROLE_APPLICATION = 0;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is a supporting
	 * part of some larger configuration, typically an outer
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 * 角色表示一个BeanDefinition是一些larger configuration的一个支持部分，通常是一个外部的ComponentDefinition
	 * SUPPORT beans are considered important enough to be aware
	 * of when looking more closely at a particular ComponentDefinition,
	 * but not when looking at the overall configuration of an application.
	 * 当更仔细地查看一个特定的ComponentDefinition对象时，
	 * SUPPORT　bean被认为是足够重要被注意的，
	 * 但是在查看整个应用程序的时候你可以忽略SUPPORT bean
	 */
	int ROLE_SUPPORT = 1;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is providing an
	 * entirely background role and has no relevance to the end-user. This hint 	 
	 * is used when registering beans that are completely part of 
	 * the internal workings of a ComponentDefinition.
	 * 这种角色表明一个BeanDefinition在提供一个完整的幕后角色，与最终用户无关。
	 * 当注册的beans完全是一个ComponentDefinition内部工作的一部分的时候，
	 * 这种角色被调用
	 */
	int ROLE_INFRASTRUCTURE = 2;


	// Modifiable attributes

	/**
	 * Set the name of the parent definition of this bean definition, if any.
	 */
	void setParentName(@Nullable String parentName);

	/**
	 * Return the name of the parent definition of this bean definition, if any.
	 */
	@Nullable
	String getParentName();

	/**
	 * Specify the bean class name of this bean definition.
	 * <p>The class name can be modified during bean factory post-processing,
	 * typically replacing the original class name with a parsed variant of it.
	 * 这个方法用来指定这个BeanDefinition的bean的类名
	 * 可以在bean工厂的post-processing期间修改类名，
	 * 通常将原始类名改为其解析的变体（比如类的代理）
	 * @see #setParentName
	 * @see #setFactoryBeanName
	 * @see #setFactoryMethodName
	 */
	void setBeanClassName(@Nullable String beanClassName);

	/**
	 * Return the current bean class name of this bean definition.
	 * Note that this does not have to be the actual class name used at runtime,
     * incase of a child definition overriding/inheriting the class name from its parent.
     * Also, this may just be the class that a factory method is called on, or it mayeven be empty in case of a factory bean reference that a method is called on.Hence, do <i>not</i> consider this to be the definitive bean type at runtime butrather only use it for parsing purposes at the individual bean definition level.
	 *
	 * @see #getParentName()
	 * @see #getFactoryBeanName()
	 * @see #getFactoryMethodName()
	 */
	@Nullable
	String getBeanClassName();

	/**
	 * Override the target scope of this bean, specifying a new scope name.
	 *
	 * @see #SCOPE_SINGLETON
	 * @see #SCOPE_PROTOTYPE
	 */
	void setScope(@Nullable String scope);

	/**
	 * Return the name of the current target scope for this bean,
	 * or {@code null} if not known yet.
	 */
	@Nullable
	String getScope();

	/**
	 * Set whether this bean should be lazily initialized.
	 * <p>If {@code false}, the bean will get instantiated on startup by bean
	 * factories that perform eager initialization of singletons.
	 * 设置是否延迟初始化此Bean，如果false，
	 * 这个bean将会在启动时被（执行单例初始化的bean工厂）实例化
	 */
	void setLazyInit(boolean lazyInit);

	/**
	 * Return whether this bean should be lazily initialized, i.e. not
	 * eagerly instantiated on startup. Only applicable to a singleton bean.
	 */
	boolean isLazyInit();

	/**
	 * Set the names of the beans that this bean depends on being initialized.
	 * 设置这些beans初始化时需要依赖的bean
	 * The bean factory will guarantee that these beans get initialized first.
	 * bean工厂将会先实例化这些依赖的bean
	 */
	void setDependsOn(@Nullable String... dependsOn);

	/**
	 * Return the bean names that this bean depends on.
	 */
	@Nullable
	String[] getDependsOn();

	/**
	 * Set whether this bean is a candidate for getting autowired into some other bean.
	 * 设置这个类时候会被注入到其他的bean中
	 * <p>Note that this flag is designed to only affect type-based autowiring.
	 * 这个标志位被设计仅用来影响基于类型的注入
	 * It does not affect explicit references by name, which will get resolved even
	 * 它并不会影响基于名字的注入，基于名字的注入会在之后解决
	 * if the specified bean is not marked as an autowire candidate. As a consequence,
	 * autowiring by name will nevertheless inject a bean if the name matches.
	 * 如果这个特殊的bean被没有被标记上是一个 autowire candidate，作为代价，如果bean的名字匹配的话，
	 * 将会按照名字注入一个bean
	 */
	void setAutowireCandidate(boolean autowireCandidate);

	/**
	 * Return whether this bean is a candidate for getting autowired into some other bean.
	 */
	boolean isAutowireCandidate();

	/**
	 * Set whether this bean is a primary autowire candidate.
	 * <p>If this value is {@code true} for exactly one bean among multiple
	 * matching candidates, it will serve as a tie-breaker.
	 * 设置此bean是否为自动装配的主要候选对象。
	 * 如果多个matching candidates中的一个bean的这个value是true，那么这个bean会被保存为一个tie-breaker
	 */
	void setPrimary(boolean primary);

	/**
	 * Return whether this bean is a primary autowire candidate.
	 */
	boolean isPrimary();

	/**
	 * Specify the factory bean to use, if any.
	 * This the name of the bean to call the specified factory method on.
	 * 指定要用哪个factory bean来加载这个bean，（如果这个factory bean存在）。使用其指定工厂方法加载
	 * @see #setFactoryMethodName
	 */
	void setFactoryBeanName(@Nullable String factoryBeanName);

	/**
	 * Return the factory bean name, if any.
	 */
	@Nullable
	String getFactoryBeanName();

	/**
	 * Specify a factory method, if any. This method will be invoked with
	 * constructor arguments, or with no arguments if none are specified.
	 * 指定一个工厂方法，如果存在，将使用构造函数参数调用此方法，如果没有方法被指定，则不适用参数
	 * The method will be invoked on the specified factory bean, if any,
	 * or otherwise as a static method on the local bean class.
	 * 这个方法将会在一个指定的工厂bean中被调用（如果这个工厂bean存在），
	 * 或者作为一个静态方法在本地bean的类中被调用
	 * @see #setFactoryBeanName
	 * @see #setBeanClassName
	 */
	void setFactoryMethodName(@Nullable String factoryMethodName);

	/**
	 * Return a factory method, if any.
	 */
	@Nullable
	String getFactoryMethodName();

	/**
	 * Return the constructor argument values for this bean.
	 * The returned instance can be modified during bean factory post-processing.
	 * 返回这个bean的构造参数的值。
	 * 这个方法的返回值可以在bean factory post-processing中被修改
	 * @return the ConstructorArgumentValues object (never {@code null})
	 */
	ConstructorArgumentValues getConstructorArgumentValues();

	/**
	 * Return if there are constructor argument values defined for this bean.
	 * 如果这个bean的构造函数的参数被定义了的话，返回构造函数的参数
	 * @since 5.0.2
	 */
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	/**
	 * Return the property values to be applied to a new instance of the bean.
	 * <p>The returned instance can be modified during bean factory post-processing.
	 * 返回要应用到Bean的新实例的属性值。
	 * 这个方法的返回值可以在bean factory post-processing中被修改
	 * @return the MutablePropertyValues object (never {@code null})
	 */
	MutablePropertyValues getPropertyValues();

	/**
	 * Return if there are property values values defined for this bean.
	 * 
	 * @since 5.0.2
	 */
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	/**
	 * Set the name of the initializer method.
	 * 
	 * @since 5.1
	 */
	void setInitMethodName(@Nullable String initMethodName);

	/**
	 * Return the name of the initializer method.
	 *
	 * @since 5.1
	 */
	@Nullable
	String getInitMethodName();

	/**
	 * Set the name of the destroy method.
	 *
	 * @since 5.1
	 */
	void setDestroyMethodName(@Nullable String destroyMethodName);

	/**
	 * Return the name of the destroy method.
	 *
	 * @since 5.1
	 */
	@Nullable
	String getDestroyMethodName();

	/**
	 * Set the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * 设定这个BeanDefinition的角色，
	 * 角色提示为框架和工具提供了特定{@code BeanDefinition}的角色和重要性的指示。
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 * @since 5.1
	 */
	void setRole(int role);

	/**
	 * Get the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * 
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	int getRole();

	/**
	 * Set a human-readable description of this bean definition.
	 *
	 * @since 5.1
	 */
	void setDescription(@Nullable String description);

	/**
	 * Return a human-readable description of this bean definition.
	 */
	@Nullable
	String getDescription();


	// Read-only attributes

	/**
	 * Return whether this a <b>Singleton</b>, with a single, shared instance
	 * returned on all calls.
	 *
	 * @see #SCOPE_SINGLETON
	 */
	boolean isSingleton();

	/**
	 * Return whether this a <b>Prototype</b>, with an independent instance
	 * returned for each call.
	 *
	 * @see #SCOPE_PROTOTYPE
	 * @since 3.0
	 */
	boolean isPrototype();

	/**
	 * Return whether this bean is "abstract", that is, not meant to be instantiated.
	 */
	boolean isAbstract();

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors).
	 */
	@Nullable
	String getResourceDescription();

	/**
	 * Return the originating BeanDefinition, or {@code null} if none.
	 * 返回原始的BeanDefinition，如果没有，则返回null。
	 * Allows for retrieving the decorated bean definition, if any.
	 * 允许获取修饰的bean定义（如果有）。
	 * <p>Note that this method returns the immediate originator. Iterate through the
	 * originator chain to find the original BeanDefinition as defined by the user.
	 * 这个方法返回直接的originator，遍历originator寻找用户定义的原始的BeanDefinition
	 */
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();
}
```

## BeanDefinitionBuilder与BeanDefinitionRegistryPostProcessor

​		BeanDefinitionBuilder是Builder模式的应用，我么可以方便的构建BeanDefinition的实例对象。

​		`BeanDefinitionRegistryPostProcessor`继承了`BeanFactoryPostProcessor`接口，同时又增加了一个新的方法`postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)`，这个方法可以让我们通过代码手动向程序中注册BeanDefinition

​		举个例子：

```java
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(OrderService.class)
				//这里的属性名是根据setter方法
				.addPropertyReference("dao", "orderDao")
				.setInitMethodName("init")
				.setScope(BeanDefinition.SCOPE_SINGLETON)
				.getBeanDefinition();

		registry.registerBeanDefinition("orderService", beanDefinition);

	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		//do nothing
	}
}
```



## 参考

https://cloud.tencent.com/developer/article/1512598