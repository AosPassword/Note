# SpringIoC 结构

​		我觉得当我们了解一项东西的时候，首先要从足够高的角度去看，这样可以帮助我们理解一个体系中各个部分的作用和联系，同时不会出现遗漏。

## 关于IoC

​		既然IoC的目标在于，把原来代码里需要实现的对象创建，依赖的代码，反转给容器来实现，那么我们需要创建一个容器，同时需要一种描述来让容器知道需要创建的对象和对象之间的关系。

​		那么我们就有了如下问题：

- 如何让框架知道对象之间的关系？

  可以使用xml，properties文件等表示

- 描述对象关系的文件可以放在那里？

  可能是在服务器上，也可能是网络资源

- 框架如何理解配置文件？

  我们需要有个东西可以将格式不同的配置文件描述转化为统一的对象定义，这样才能让框架的其他部分使用一个统一的对象定义。

  框架的其他部分不需要了解配置文件的格式，因为这些不同的配置文件都会被转化为统一的对象定义。

- 框架如何将不同的配置文件转化为统一的对象定义？

  对于不同的配置文件语法，使用不同的解析器。最终产出的对象定义是相同格式的就行。

​         Spring的创始人也想到了这些问题，所以他写下了如下的主要父接口：

###Resource

​		是对资源的抽象，每一个接口实现类都代表了一种资源类型，如ClasspathResource、URLResource，FileSystemResource等。每一个资源类型都封装了对**某一种特定资源的访问策略**。

### BeanDefinition

​		用来抽象和描述一个具体bean对象。是描述一个bean对象的基本数据结构（与bean一一对应）。

###BeanDefinitionReader

​		BeanDefinitionReader将外部资源对象描述的bean定义统一转化为统一的内部数据结构BeanDefinition。对应不同的描述需要有不同的Reader。如XmlBeanDefinitionReader用来读取xml描述配置的bean对象。

### BeanFactory

​		用来定义一个很纯粹的bean容器。它是一个bean容器的必备结构。同时和外部应用环境等隔离。BeanDefinition是它的基本数据结构。它维护一个BeanDefinitions Map,并可根据BeanDefinition的描述进行bean的创建和管理。

### ApplicationContext

​		从名字来看叫应用上下文，是和应用环境息息相关的。没错这个就是我们平时开发中经常直接使用打交道的一个类，应用上下文，或者也叫做spring容器。其实它的基本实现是会持有一个BeanFactory对象，并基于此提供一些包装和功能扩展。

​		为什么要这么做呢？因为BeanFactory实现了一个容器基本结构和功能，但是与外部环境隔离。那么读取配置文件，并将配置文件解析成BeanDefinition，然后注册到BeanFactory的这一个过程的封装自然就需要ApplicationContext。ApplicationContext和应用环境细细相关。

​		常见实现有``ClasspathXmlApplicationContext，FileSystemXmlApplicationContext，WebApplicationContext``等。Classpath、xml、FileSystem、Web等词都代表了应用和环境相关的一些意思，从字面上不难理解各自代表的含义。

当然ApplicationContext和BeanFactory的区别远不止于此，有：

1. 资源访问功能：在Resource和ResourceLoader的基础上可以灵活的访问不同的资源。

2. 支持不同的信息源（BeanFactory的信息源只有BeanDefinition）。

3. 支持应用事件：继承了接口ApplicationEventPublisher，这样在上下文中为bean之间提供了事件机制。

4. 。。。（还有很多）

5. ## 结构总览

   ![img](https://ask.qcloudimg.com/http-save/yehe-1655470/bwwknqti3q.jpeg?imageView2/2/w/1620)

   ​		该图为 ClassPathXmlApplicationContext 的类继承体系结构，虽然只有一部分，但是它基本上包含了 IOC 体系中大部分的核心类和接口。

   ​		其中左边黄色部分是ApplicationContext体系继承结构，右边是BeanFactory的结构体系。

   ​		这两个结构式典型的**模板方法设计模式**。

   ## 我们所要知道的要点

   - BeanFactory是一个bean工厂的最基本的定义。不关注资源，时间等。
   - ApplicationContext是一个容器的最基本的接口定义，继承了BeanFactory，拥有工厂的基本方法（注意没有实现AutowireCapableBeanFactory），同时继承了ApplicationEventPublisher，MessageSource， ResourcePatternResolver等接口，使其定义了一些额外的功能，比如资源（MessageSource， ResourcePatternResolver），事件（ApplicationEventPublisher）
   -  AbstractBeanFactory 和 AbstractAutowireCapableBeanFactory 是两个模板抽象工厂类。 AbstractBeanFactory 提供了 bean 工厂的抽象基类，同时提供了 ConfigurableBeanFactory 的完整实现。 AbstractAutowireCapableBeanFactory 是继承了 AbstractBeanFactory 的抽象工厂，里面提供了 bean 创建的支持，包括 bean 的创建、依赖注入、检查等等功能，是一个核心的 bean 工厂基类。
   - ClassPathXmlApplicationContext之 所以拥有 bean 工厂的功能是通过持有一个真正的 bean 工厂 DefaultListableBeanFactory 的实例，并通过**代理**该工厂完成。
   - ClassPathXmlApplicationContext 的初始化过程是对本身容器的初始化同时也是对其持有的 DefaultListableBeanFactory 的初始化。

## 举个例子吧！

​		我觉得没有什么比举个例子更容易让人理解的了。我们要举例子是：**ClasspathXmlApplicationContext** 

​		ClassPathXmlApplicationContext的refresh() 方法负责完成了整个容器的初始化。

​		为什么叫refresh？也就是说其实是刷新的意思，该IOC容器里面维护了一个单例的BeanFactory，如果bean的配置有修改，也可以直接调用refresh方法，它将销毁之前的BeanFactory，重新创建一个BeanFactory。所以叫refresh也是能理解的。

​		Refresh的基本步骤：

​		1.把配置xml文件转换成resource。Resource的转换是先通过ResourcePatternResolver来解析可识别格式的配置文件的路径(如"classpath\*:"等)，如果没有指定格式，默认会按照类路径的资源来处理。

​		2.利用XmlBeanDefinitionReader完成对xml的解析，将xml Resource里定义的bean对象转换成统一的BeanDefinition。

​		3.将BeanDefinition注册到BeanFactory，完成对BeanFactory的初始化。BeanFactory里将会维护一个BeanDefinition的Map。

​		上代码！

```java
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
        // Prepare this context for refreshing.  
        prepareRefresh();  
  
        // Tell the subclass to refresh the internal bean factory.  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
  
        // Prepare the bean factory for use in this context.  
        prepareBeanFactory(beanFactory);  
  
        try {  
            // Allows post-processing of the bean factory in context subclasses.  
            postProcessBeanFactory(beanFactory);  
  
            // Invoke factory processors registered as beans in the context.  
            invokeBeanFactoryPostProcessors(beanFactory);  
  
            // Register bean processors that intercept bean creation.  
            registerBeanPostProcessors(beanFactory);  
  
            // Initialize message source for this context.  
            initMessageSource();  
  
            // Initialize event multicaster for this context.  
            initApplicationEventMulticaster();  
  
            // Initialize other special beans in specific context subclasses.  
            onRefresh();  
  
            // Check for listener beans and register them.  
            registerListeners();  
  
            // Instantiate all remaining (non-lazy-init) singletons.  
            finishBeanFactoryInitialization(beanFactory);  
  
            // Last step: publish corresponding event.  
            finishRefresh();  
        }  
  
        catch (BeansException ex) {  
            // Destroy already created singletons to avoid dangling resources.  
            beanFactory.destroySingletons();  
  
            // Reset 'active' flag.  
            cancelRefresh(ex);  
  
            // Propagate exception to caller.  
            throw ex;  
        }  
    }  
}  
```

​		关于代码解析如下：![img](http://dl.iteye.com/upload/attachment/558068/2187e288-5c0b-313b-8053-82992267fab6.jpg)

​		以上的obtainFreshBeanFactory是很关键的一个方法，里面会调用loadBeanDefinition方法，如下：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws IOException {  
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.  
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);  
  
    // Configure the bean definition reader with this context's  
    // resource loading environment.  
    beanDefinitionReader.setResourceLoader(this);  
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));  
  
    // Allow a subclass to provide custom initialization of the reader,  
    // then proceed with actually loading the bean definitions.  
    initBeanDefinitionReader(beanDefinitionReader);  
    loadBeanDefinitions(beanDefinitionReader);  
}  
```

​		这里特定于整个IOC容器，实例化了一个XmlBeanDefinitionReader来解析Resource文件。关于Resource文件如何初始化和xml文件如何解析都在``loadBeanDefinitions(beanDefinitionReader)``中调用

## Bean的创建过程

​		通过上文其实我们可以知道一个容器的初始化过程大概是：

​		配置文件（XML）->BeanDefinitionMap->Beans

​		其中第一步过程是ApplicationContext的职责范围（用Resource和BeanDefinitionReader），而第二步就是BeanFactory的事情了。

​		我们上面大概分析了ApplicationContext做了什么事情，那么现在我们来分析分析BeanFactory做了什么？当然是创建Bean！那么我们就有了两个新问题：

- Bean什么时候被创建？
- Bean是怎么被创建的？

### Bean什么时候被创建的？

​		容器初始化的时候会预先对单例和非延迟加载的对象进行预先初始化，其他的都是延迟加载，在第一次调用getBean的时候被创建。

​		BeanFactory的"集大成"的实现类DefaultListableBeanFactory的preInstantiateSingletons可以看看到这个规则的实现：

```java
    public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Pre-instantiating singletons in " + this);
        }

        List<String> beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            String beanName;
            Object bean;
            do {
                while(true) {
                    RootBeanDefinition bd;
                    do {
                        do {
                            do {
                                if (!var2.hasNext()) {
                                    var2 = beanNames.iterator();

                                    while(var2.hasNext()) {
                                        beanName = (String)var2.next();
                                        Object singletonInstance = this.getSingleton(beanName);
                                        if (singletonInstance instanceof SmartInitializingSingleton) {
                                            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                            if (System.getSecurityManager() != null) {
                                                AccessController.doPrivileged(() -> {
                                                    smartSingleton.afterSingletonsInstantiated();
                                                    return null;
                                                }, this.getAccessControlContext());
                                            } else {
                                                smartSingleton.afterSingletonsInstantiated();
                                            }
                                        }
                                    }

                                    return;
                                }

                                beanName = (String)var2.next();
                                bd = this.getMergedLocalBeanDefinition(beanName);
                            } while(bd.isAbstract());
                        } while(!bd.isSingleton());
                    } while(bd.isLazyInit());

                    if (this.isFactoryBean(beanName)) {
                        bean = this.getBean("&" + beanName);
                        break;
                    }

                    this.getBean(beanName);
                }
            } while(!(bean instanceof FactoryBean));

            FactoryBean<?> factory = (FactoryBean)bean;
            boolean isEagerInit;
            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                SmartFactoryBean var10000 = (SmartFactoryBean)factory;
                ((SmartFactoryBean)factory).getClass();
                isEagerInit = (Boolean)AccessController.doPrivileged(var10000::isEagerInit, this.getAccessControlContext());
            } else {
                isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
            }

            if (isEagerInit) {
                this.getBean(beanName);
            }
        }
    }
```

​		wdnmd，这个while嵌套dowhile给我看呆了。

​		等等，别弃疗，我们还是看看以前版本的preInstantiateSingletons()，毕竟大概逻辑应该是一样的：

```java
public void preInstantiateSingletons() throws BeansException {  
        if (this.logger.isInfoEnabled()) {  
            this.logger.info("Pre-instantiating singletons in " + this);  
        }  
  
        synchronized (this.beanDefinitionMap) {  
            for (Iterator it = this.beanDefinitionNames.iterator(); it.hasNext();) {  
                String beanName = (String) it.next();  
                RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);  
                if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) { 
				//对非抽象、单例的和非延迟加载的对象进行实例化。  
                    if (isFactoryBean(beanName)) {  
                        FactoryBean factory = (FactoryBean) getBean(FACTORY_BEAN_PREFIX + beanName);  
                        if (factory instanceof SmartFactoryBean && ((SmartFactoryBean) factory).isEagerInit()) {  
                            getBean(beanName);  
                        }  
                    }  
                    else {  
                        getBean(beanName);  
                    }  
                }  
            }  
        }  
    }  
```

### Bean的创建过程

​		无论预先创建还是延迟加载都是调用getBean实现，AbstractBeanFactory 定义了 getBean 的过程：

​		（卧槽，大哥可别给我整上面那个while了，我真顶不住啊）

```java
    public Object getBean(String name) throws BeansException {
        return this.doGetBean(name, (Class)null, (Object[])null, false);
    }

    public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        return this.doGetBean(name, requiredType, (Object[])null, false);
    }

    public Object getBean(String name, Object... args) throws BeansException {
        return this.doGetBean(name, (Class)null, args, false);
    }

    public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args) throws BeansException {
        return this.doGetBean(name, requiredType, args, false);
    }
```

​		这代码总算正常点了。哎，这几个getBean其实都是调用了doGetBean方法，这点应该还是很清晰的。

​		来我们看看doGetBean干了些什么事情：

```java
	/**
     *
     * @param name          the name of the bean to retrieve
     * @param requiredType  the required type of the bean to retrieve
     * @param args          arguments to use when creating a bean instance using explicit arguments
     *                      (only applied when creating a new instance as opposed to retrieving an existing one)
     * @param typeCheckOnly whether the instance is obtained for a type check, not for actual use
     * @return              an instance, which may be shared or independent, of the specified bean.
     * @throws BeansException   if the bean could not be created
     */
	protected <T> T doGetBean(String name, 
                              @Nullable Class<T> requiredType, 
                              @Nullable Object[] args, 
                              boolean typeCheckOnly) throws BeansException {4
        //第一步我们会显得到Bean的名称，因为传入的name并不一定就是Bean的真正name
        //因为还有可能是aliasName(别名)，详细情况查看Alias相关内容
        String beanName = this.transformedBeanName(name);
        // Eagerly check singleton cache for manually registered singletons.
        // 先从单例缓存中尝试获取指定对象
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        if (sharedInstance != null && args == null) {
            //如果缓存的单例是存在的话
            if (this.logger.isTraceEnabled()) {
                if (this.isSingletonCurrentlyInCreation(beanName)) {
                    this.logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    this.logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
			// 我们对从单例缓存中获得的对象进行处理得到真正可以使用的对象
            	bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
            // Fail if we're already creating this bean instance:  
            // We're assumably within a circular reference.
            // 查看我们是否创建了这个实例，如果创建，则失败
            // 因为我们很可能在循环引用
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                //如果父工厂存在，并且本工厂不含指定的bean，那就尝试从父工厂获取bean
                String nameToLookup = this.originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                }

                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                if (requiredType != null) {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }

                return parentBeanFactory.getBean(nameToLookup);
            }

            if (!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
                //获取RootBeanDefinition
                //返回给定的Bean名称的“merged”的BeanDefinition，如有必要，将子bean定义与其父对象合并（merge）。
                RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
                //检查bean定义
                this.checkMergedBeanDefinition(mbd, beanName, args);
                // Guarantee initialization of beans that the current bean depends on.  
                // 生成这个bean的依赖对象
                String[] dependsOn = mbd.getDependsOn();
                String[] var11;
                // 校验依赖的Bean
                if (dependsOn != null) {
                    var11 = dependsOn;
                    int var12 = dependsOn.length;

                    for(int var13 = 0; var13 < var12; ++var13) {
                        String dep = var11[var13];
                        if (this.isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }

                        this.registerDependentBean(dep, beanName);

                        try {
                            this.getBean(dep);
                        } catch (NoSuchBeanDefinitionException var24) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "'" + beanName + "' depends on missing bean '" + dep + "'", var24);
                        }
                    }
                }
			   // Create bean instance.  
                // 根据Bean的scope做不同的处理
                if (mbd.isSingleton()) {
                    //getSingleton会使用到synchronized，确保线程安全
                    sharedInstance = this.getSingleton(beanName, () -> {
                        try {
                            return this.createBean(beanName, mbd, args);
                        } catch (BeansException var5) {
				// Explicitly remove instance from singleton cache: It might have been put there
                 // eagerly by the creation process, to allow for circular reference resolution.  
			    // Also remove any beans that received a temporary reference to the bean.  
                            this.destroySingleton(beanName);
                            throw var5;
                        }
                    });
                    //对bean再进行一些修饰
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                } else if (mbd.isPrototype()) {
                    var11 = null;

                    Object prototypeInstance;
                    try {
                        this.beforePrototypeCreation(beanName);
                        prototypeInstance = this.createBean(beanName, mbd, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                } else {
                    String scopeName = mbd.getScope();
                    Scope scope = (Scope)this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }

                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            this.beforePrototypeCreation(beanName);

                            Object var4;
                            try {
                                var4 = this.createBean(beanName, mbd, args);
                            } finally {
                                this.afterPrototypeCreation(beanName);
                            }

                            return var4;
                        });
                        bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    } catch (IllegalStateException var23) {
                        throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var23);
                    }
                }
            } catch (BeansException var26) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var26;
            }
        }

        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                } else {
                    return convertedBean;
                }
            } catch (TypeMismatchException var25) {
                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var25);
                }

                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        } else {
            return bean;
        }
    }
```

​		真是意料之中的长。。大概过程大概是这样的：

- 显示着从单例缓存对象中获取

  ```java
  Object sharedInstance = this.getSingleton(beanName);
  ```

- 如果自己这个容器找不见对象的定义，从父容器中尝试取得对象的定义，如果有，则由父容器创建

  ```java
  if (parentBeanFactory != null && 
      !this.containsBeanDefinition(beanName)) {
  	String nameToLookup = this.originalBeanName(name);
      if (parentBeanFactory instanceof AbstractBeanFactory) {
  		return ((AbstractBeanFactory)parentBeanFactory)
              .doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
      }
     	if (args != null) {
          return parentBeanFactory.getBean(nameToLookup, args);
     	}
    	if (requiredType != null) {
          return parentBeanFactory.getBean(nameToLookup, requiredType);
    	}
  	return parentBeanFactory.getBean(nameToLookup);
  }
  ```

  

- 如果是单例，则走单例对象的创建过程。

  在 spring 容器里单例对象和非单例对象的创建过程是一样的。都会调用父类 AbstractAutowireCapableBeanFactory 的 createBean 方法。

  不同的是单例对象只创建一次并且需要缓存起来。 DefaultListableBeanFactory 的父类 DefaultSingletonBeanRegistry 提供了对单例对象缓存等支持工作。

  所以是单例对象的话会调用 DefaultSingletonBeanRegistry 的 getSingleton 方法，它会间接调用 AbstractAutowireCapableBeanFactory 的 createBean 方法。

  如果是 Prototype 原例则直接调用父类 AbstractAutowireCapableBeanFactory 的 createBean 方法。

​        我们发现，在getBean的过程中，当我们初始化完需要注入的东西之后，我们调用了一个方法，从上面摘抄出来一段：

```java
if (mbd.isSingleton()) {
	sharedInstance = this.getSingleton(beanName, () -> {
		try {
			return this.createBean(beanName, mbd, args);
		} catch (BeansException var5) {
		// Explicitly remove instance from singleton cache: It might have been put there
		// eagerly by the creation process, to allow for circular reference resolution.  
		// Also remove any beans that received a temporary reference to the bean. 
		this.destroySingleton(beanName);
		throw var5;
		}
	});
	bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

​		这其中最为核心的代码在于：return this.createBean(beanName, mbd, args);

​		而AbstractBeanFactory并没有定义这个方法：

```java
    protected abstract Object createBean(String var1, RootBeanDefinition var2, @Nullable Object[] var3) throws BeanCreationException;
```

​		这个方法是由AbstractAutowireCapableBeanFactory来定义的，这是创建Bean的核心流程：

```java
    public <T> T createBean(Class<T> beanClass) throws BeansException {
        RootBeanDefinition bd = new RootBeanDefinition(beanClass);
        bd.setScope("prototype");
        bd.allowCaching = ClassUtils.isCacheSafe(beanClass, this.getBeanClassLoader());
        return this.createBean(beanClass.getName(), bd, (Object[])null);
    }

    public Object createBean(Class<?> beanClass, 
                             int autowireMode, 
                             boolean dependencyCheck) throws BeansException {
        RootBeanDefinition bd = new RootBeanDefinition(beanClass, autowireMode, dependencyCheck);
        bd.setScope("prototype");
        return this.createBean(beanClass.getName(), bd, (Object[])null);
    }

	//doGetBean真正调用的方法
   protected Object createBean(String beanName, 
                               RootBeanDefinition mbd,
                               @Nullable Object[] args) throws BeanCreationException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Creating instance of bean '" + beanName + "'");
        }

        RootBeanDefinition mbdToUse = mbd;
        // Make sure bean class is actually resolved at this point. 
        // 确认这个Bean的class
        Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }
		// Prepare method overrides. 
        try {
            mbdToUse.prepareMethodOverrides();
        } catch (BeanDefinitionValidationException var9) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var9);
        }

        Object beanInstance;
        try {
            // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.  
            // 给BeanPostProcessors一个返回代理对象的机会，是实现AOP的重要逻辑之一
            //这个方法的具体实现看附录
            beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
            if (beanInstance != null) {
                return beanInstance;
            }
        } catch (Throwable var10) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var10);
        }

        try {
            //creat的核心逻辑：doCreateBean
            beanInstance = this.doCreateBean(beanName, mbdToUse, args);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Finished creating instance of bean '" + beanName + "'");
            }

            return beanInstance;
        } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
            throw var7;
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
        }
    }
```

​		而createBean 会调用 doCreateBean 方法：``beanInstance = this.doCreateBean(beanName, mbdToUse, args);``

​		让我们再来看一下doCreateBean：

```java
    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        // Instantiate the bean. 
        //BeanWrapper时bean的一个包装类，核心还是Bean
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            //如果是单例，则先从factoryBeanInstanceCache尝试移除此BeanName的单例
            //ConcurrentMap<String, BeanWrapper> factoryBeanInstanceCache;
		   //不难看出其实就是一个map，remove成功返回对应value，失败返回null
            //说明如果缓存中创建过此单例了，那就拿着以前创建的单例用（严格保持单例）
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if (instanceWrapper == null) {
            //如果对象是单例，且instanceWrapper == null，说明从未创建过此对象，创建一个新对象
            //如果对象不是单例，也就是创建一个新对象
            //createBeanInstance，又是一个值得一讲的方法，后面会提到
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }
		//instanceWrapper即为创建Bean的包装，但是其中的bean是不完整的
        Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }
		// Allow post-processors to modify the merged bean definition.  
        Object var7 = mbd.postProcessingLock;
        synchronized(mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
            }
        }
        // Eagerly cache singletons to be able to resolve circular references  
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        
        if (earlySingletonExposure) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }
			//向SingletonFactory添加单例缓存
            this.addSingletonFactory(beanName, () -> {
                return this.getEarlyBeanReference(beanName, mbd, bean);
            });
        }
		// Initialize the bean instance.  
        Object exposedObject = bean;

        try {
            //属性注入，两种注入方法，根据name和更具type，附录里有源码
            this.populateBean(beanName, mbd, instanceWrapper);
            //初始化bean，激活Aware方法，后置处理器，自定义init
            exposedObject = this.initializeBean(beanName, exposedObject, mbd);
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        if (earlySingletonExposure) {
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }

        try {
            //注册DisposableBean
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
        }
    }
```

doCreateBean 的流程：

1. 会创建一个 BeanWrapper 对象 用于存放实例化对象。

2. 如果没有指定构造函数，会通过反射拿到一个默认的构造函数对象，并赋予 beanDefinition.resolvedConstructorOrFactoryMethod 。

3. 调用 spring 的 BeanUtils 的 instantiateClass 方法，通过反射创建对象。

4. applyMergedBeanDefinitionPostProcessors

5. populateBean(beanName, mbd, instanceWrapper); 根据注入方式进行注入。根据是否有依赖检查进行依赖检查。

 我们在此处在关注一个重要的方法：

``exposedObject = this.initializeBean(beanName, exposedObject, mbd);``

我们康康这个方法的详细情况

```java
    protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(() -> {
                this.invokeAwareMethods(beanName, bean);
                return null;
            }, this.getAccessControlContext());
        } else {
            this.invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
```

​		判断是否实现了 BeanNameAware 、 BeanClassLoaderAware 等 spring 提供的接口，如果实现了，进行默认的注入。同时判断是否实现了 InitializingBean 接口，如果是的话，调用 afterPropertySet 方法。

​		我们可以详细看一下invokeInitMethods的实现：

```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd) throws Throwable {
        boolean isInitializingBean = bean instanceof InitializingBean;
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
            }

            if (System.getSecurityManager() != null) {
                try {
                    AccessController.doPrivileged(() -> {
                        ((InitializingBean)bean).afterPropertiesSet();
                        return null;
                    }, this.getAccessControlContext());
                } catch (PrivilegedActionException var6) {
                    throw var6.getException();
                }
            } else {
                ((InitializingBean)bean).afterPropertiesSet();
            }
        }

        if (mbd != null && bean.getClass() != NullBean.class) {
            String initMethodName = mbd.getInitMethodName();
            if (StringUtils.hasLength(initMethodName) && (!isInitializingBean || !"afterPropertiesSet".equals(initMethodName)) && !mbd.isExternallyManagedInitMethod(initMethodName)) {
                this.invokeCustomInitMethod(beanName, bean, mbd);
            }
        }

    }
```

​		最终结果如下：

![img](https://ask.qcloudimg.com/http-save/yehe-1655470/f3hvn0pj2q.jpeg?imageView2/2/w/1620)

## 附录

resolveBeforeInstantiation：

```java
    @Nullable
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            //首先先判断是否有BeanPostProcessors，没有则直接跳过
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                //如果有InstantiationAwareBeanPostProcessors
                Class<?> targetType = this.determineTargetType(beanName, mbd);
                //AOP实现，返回代理
                if (targetType != null) {
                    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }

            mbd.beforeInstantiationResolved = bean != null;
        }

        return bean;
    }
```

populateBean:

```java
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
        if (bw == null) {
            if (mbd.hasPropertyValues()) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            }
        } else {
            boolean continueWithPropertyPopulation = true;
            //给与InstantiationAwareBeanPostProcessors最后一次改变Bean的机会
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Iterator var5 = this.getBeanPostProcessors().iterator();

                while(var5.hasNext()) {
                    BeanPostProcessor bp = (BeanPostProcessor)var5.next();
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                            continueWithPropertyPopulation = false;
                            break;
                        }
                    }
                }
            }

            if (continueWithPropertyPopulation) {
                PropertyValues pvs = mbd.hasPropertyValues() ? mbd.getPropertyValues() : null;
                if (mbd.getResolvedAutowireMode() == 1 || mbd.getResolvedAutowireMode() == 2) {
                    MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                    //查看注入类型是要按照名字注入还是按照类型注入
                    if (mbd.getResolvedAutowireMode() == 1) {
                        this.autowireByName(beanName, mbd, bw, newPvs);
                    }

                    if (mbd.getResolvedAutowireMode() == 2) {
                        this.autowireByType(beanName, mbd, bw, newPvs);
                    }

                    pvs = newPvs;
                }
				//检查属性值，并将属性值解析出来
                boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
                boolean needsDepCheck = mbd.getDependencyCheck() != 0;
                PropertyDescriptor[] filteredPds = null;
                if (hasInstAwareBpps) {
                    if (pvs == null) {
                        pvs = mbd.getPropertyValues();
                    }

                    Iterator var9 = this.getBeanPostProcessors().iterator();

                    while(var9.hasNext()) {
                        BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                        if (bp instanceof InstantiationAwareBeanPostProcessor) {
                            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                            PropertyValues pvsToUse = ibp.postProcessProperties((PropertyValues)pvs, bw.getWrappedInstance(), beanName);
                            if (pvsToUse == null) {
                                if (filteredPds == null) {
                                    filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                                }

                                pvsToUse = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
                                if (pvsToUse == null) {
                                    return;
                                }
                            }

                            pvs = pvsToUse;
                        }
                    }
                }

                if (needsDepCheck) {
                    if (filteredPds == null) {
                        filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }

                    this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
                }

                if (pvs != null) {
                    //真正注入属性值的方法
                    this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
                }

            }
        }
    }
    protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
        if (!pvs.isEmpty()) {
            if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
                ((BeanWrapperImpl)bw).setSecurityContext(this.getAccessControlContext());
            }

            MutablePropertyValues mpvs = null;
            List original;
            if (pvs instanceof MutablePropertyValues) {
                mpvs = (MutablePropertyValues)pvs;
                if (mpvs.isConverted()) {
                    try {
                        bw.setPropertyValues(mpvs);
                        return;
                    } catch (BeansException var18) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Error setting property values", var18);
                    }
                }

                original = mpvs.getPropertyValueList();
            } else {
                original = Arrays.asList(pvs.getPropertyValues());
            }

            TypeConverter converter = this.getCustomTypeConverter();
            if (converter == null) {
                converter = bw;
            }

            BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, (TypeConverter)converter);
            List<PropertyValue> deepCopy = new ArrayList(original.size());
            boolean resolveNecessary = false;
            Iterator var11 = original.iterator();

            while(true) {
                while(var11.hasNext()) {
                    PropertyValue pv = (PropertyValue)var11.next();
                    if (pv.isConverted()) {
                        deepCopy.add(pv);
                    } else {
                        String propertyName = pv.getName();
                        Object originalValue = pv.getValue();
                        Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
                        Object convertedValue = resolvedValue;
                        boolean convertible = bw.isWritableProperty(propertyName) && !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
                        if (convertible) {
                            convertedValue = this.convertForProperty(resolvedValue, propertyName, bw, (TypeConverter)converter);
                        }

                        if (resolvedValue == originalValue) {
                            if (convertible) {
                                pv.setConvertedValue(convertedValue);
                            }

                            deepCopy.add(pv);
                        } else if (convertible && originalValue instanceof TypedStringValue && !((TypedStringValue)originalValue).isDynamic() && !(convertedValue instanceof Collection) && !ObjectUtils.isArray(convertedValue)) {
                            pv.setConvertedValue(convertedValue);
                            deepCopy.add(pv);
                        } else {
                            resolveNecessary = true;
                            deepCopy.add(new PropertyValue(pv, convertedValue));
                        }
                    }
                }

                if (mpvs != null && !resolveNecessary) {
                    mpvs.setConverted();
                }

                try {
                    bw.setPropertyValues(new MutablePropertyValues(deepCopy));
                    return;
                } catch (BeansException var19) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Error setting property values", var19);
                }
            }
        }
    }
```

