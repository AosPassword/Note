# AliasRegistry

​		先从一个摸得着看得见的例子：@AliasFor开始

## AliasFor

​		写了3000+，删了，直接去看https://www.jianshu.com/p/869ed7037833，比我写得好

​		算了，我再提供一个测试类你们试着跑一下,加深一下理解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
@Documented
public @interface MYTest {
    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};
}

```

​		还有测试类：

```java
@MYTest(path = "i am good")
public class Test implements InitializingBean {


    @Override
    public void afterPropertiesSet() throws Exception {
        MYTest myTest = AnnotationUtils.findAnnotation(getClass(),MYTest.class);
        System.out.println(myTest);
        String[] strings = myTest.value();
        System.out.println("i get the String!");
        for (String s:strings){
            System.out.println(s);
        }
    }
}
```

​		看过BeanFactory的应该知道啥意思。看看结果：

```java
@org.wuneng.web.springioc.study.alias.annotation.MYTest(value=[i am good], path=[i am good])
i get the String!
i am good
```

​		到这里应该就懂了。

## 回到AliasRegistry

> Common interface for managing aliases. Serves as super-interface for [`BeanDefinitionRegistry`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/support/BeanDefinitionRegistry.html).

​		AliasRegistry 用于管理别名的公共接口,作为BeanDefinitionRegistry的父接口。

​		我们先看源码：

```java
public interface AliasRegistry {

    /**
     * 注册表中给name注册一个别名alias
     */
    void registerAlias(String name, String alias);

    /**
     * 移除注册表中的别名alias
     */
    void removeAlias(String alias);

    /**
     * 校验注册表中是否存在别名name
     */
    boolean isAlias(String name);

    /**
     * 在注册表中获取给定name的所有别名信息
     */
    String[] getAliases(String name);
}
```

​		讲道理还是挺简单易懂的。

​		AliasRegistry有两个子类：

- SimpleAliasRegistry： AliasRegistry的简单实现类
- SimpleBeanDefinitionRegistry： BeanDefinitionRegistry的简单实现

​        我们先看它的子接口 BeanDefinitionRegistry

## BeanDefinitionRegistry

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    /**
     * 注册BeanDefinition到注册表
     */
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException;

    /**
     * 移除注册表中beanName的BeanDefinition
     */
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    /**
     * 获取注册表中beanName的BeanDefinition
     */
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    /**
     * 检查此注册表是否包含具有给定名称的BeanDefinitio。
     */
    boolean containsBeanDefinition(String beanName);

    /**
     * 返回此注册表中定义的所有bean的名称。
     */
    String[] getBeanDefinitionNames();

    /**
     * 返回注册表中定义的bean的数目
     */
    int getBeanDefinitionCount();

    /**
     * 确定给定bean名称是否已在该注册表中使用
     */
    boolean isBeanNameInUse(String beanName);
}
```

​		很明显这个接口的作用是用来注册BeanDefinition的。

## SimpleAliasRegistry

```java
public class SimpleAliasRegistry implements AliasRegistry {

    //别名-规范名称的映射MAP，用于存储注册信息（内存注册表）
    private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);


    //注册表中注册别名
    @Override
    public void registerAlias(String name, String alias) {
        //锁注册表
        //因为CurrentHashMap只有put和remove是线程安全的
        //此处要包装对CurrentHashMap的复合操作线程安全
        synchronized (this.aliasMap) {
            //判断别名与规范名称是否一样
            if (alias.equals(name)) {
                // 一样时，在注册表移除当前别名信息
                this.aliasMap.remove(alias);
            }
            else {
               //获取当前别名在注册表中的规范名称
                String registeredName = this.aliasMap.get(alias);
                
                if (registeredName != null) {
                    //规范名称存在，不需要注册，返回
                    if (registeredName.equals(name)) {
                        return;
                    }
                    //判断是否允许重写注册
                    if (!allowAliasOverriding()) {
                        throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" + name + "': It is already registered for name '" + registeredName + "'.");
                    }
                }
                // 校验规范名称是否指向当前别名的
                checkForAliasCircle(name, alias);
                // 注册表注册别名与规范名称的映射
                this.aliasMap.put(alias, name);
            }
        }
    }

    /**
     * 是否允许重写注册表别名信息，默认true
     */
    protected boolean allowAliasOverriding() {
        return true;
    }

    /**
     * 校验给定的name-alias映射是否已在注册表aliasMap中
     */
    public boolean hasAlias(String name, String alias) {
        //遍历注册表
        for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
            //注册表中单映射的规范名称name
            String registeredName = entry.getValue();
            //判断name是否与传入name一致
            if (registeredName.equals(name)) {
                //获取注册表单项的别名
                String registeredAlias = entry.getKey();
                
                // hasAlias(registeredAlias, alias)) 检测是否存在循环引用
                
                // 循环引用如下 
                // 注册表: A-B; C-A;D-C
                // B对应的别名有ACD
                // A对应的别名别名CD
                // C对应的别名有D
                // 是循环引用 此处需要校验
                return (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias));
            }
        }
        return false;
    }

    /**
     * 移除别名，在注册表aliasMap中
     */
    @Override
    public void removeAlias(String alias) {
        synchronized (this.aliasMap) {
            //移除别名，并判断是否移除成功
            String name = this.aliasMap.remove(alias);
            if (name == null) {
                throw new IllegalStateException("No alias '" + alias + "' registered");
            }
        }
    }

    /**
     * 校验是否包含给定的别名，在注册表中
     */
    @Override
    public boolean isAlias(String name) {
        return this.aliasMap.containsKey(name);
    }
    
    /**
     * 在注册表获取给定规范名称的所有别名信息
     */
    @Override
    public String[] getAliases(String name) {
        List<String> result = new ArrayList<>();
        synchronized (this.aliasMap) {
            retrieveAliases(name, result);
        }
        return StringUtils.toStringArray(result);
    }

    /**
     * 填充上面哪个方法
     */
    private void retrieveAliases(String name, List<String> result) {
        this.aliasMap.forEach((alias, registeredName) -> {
           //判断当前别名的规范名称是否为要查询的
            if (registeredName.equals(name)) {
                result.add(alias);
                //递归查询循环引用的别名
                retrieveAliases(alias, result);
            }
        });
    }

    /**
     *
     */
    public void resolveAliases(StringValueResolver valueResolver) {
        Assert.notNull(valueResolver, "StringValueResolver must not be null");
        synchronized (this.aliasMap) {
            Map<String, String> aliasCopy = new HashMap<>(this.aliasMap);
            aliasCopy.forEach((alias, registeredName) -> {
                String resolvedAlias = valueResolver.resolveStringValue(alias);
                String resolvedName = valueResolver.resolveStringValue(registeredName);
                if (resolvedAlias == null || resolvedName == null || resolvedAlias.equals(resolvedName)) {
                    this.aliasMap.remove(alias);
                }
                else if (!resolvedAlias.equals(alias)) {
                    String existingName = this.aliasMap.get(resolvedAlias);
                    if (existingName != null) {
                        if (existingName.equals(resolvedName)) {
                            // Pointing to existing alias - just remove placeholder
                            this.aliasMap.remove(alias);
                            return;
                        }
                        throw new IllegalStateException(
                                "Cannot register resolved alias '" + resolvedAlias + "' (original: '" + alias +
                                "') for name '" + resolvedName + "': It is already registered for name '" +
                                registeredName + "'.");
                    }
                    checkForAliasCircle(resolvedName, resolvedAlias);
                    this.aliasMap.remove(alias);
                    this.aliasMap.put(resolvedAlias, resolvedName);
                }
                else if (!registeredName.equals(resolvedName)) {
                    this.aliasMap.put(alias, resolvedName);
                }
            });
        }
    }

    /**
     * 校验给定的名称是否指向别名，不指向异常抛出
     */
    protected void checkForAliasCircle(String name, String alias) {
        if (hasAlias(alias, name)) {
            throw new IllegalStateException("Cannot register alias '" + alias +
                    "' for name '" + name + "': Circular reference - '" +
                    name + "' is a direct or indirect alias for '" + alias + "' already");
        }
    }

    /**
     * 根据给定的别名获取规范名称
     */
    public String canonicalName(String name) {
        String canonicalName = name;
        // Handle aliasing...
        String resolvedName;
        do {
           //获取给定别名的规范名称，获取到跳出循环
            resolvedName = this.aliasMap.get(canonicalName);
            if (resolvedName != null) {
                canonicalName = resolvedName;
            }
        }
        while (resolvedName != null);
        return canonicalName;
    }

}
```

## SimpleBeanDefinitionRegistry

```java
public class SimpleBeanDefinitionRegistry extends SimpleAliasRegistry implements BeanDefinitionRegistry {

    // name-BeanDefinition 映射关系表，ConcurrentHashMap实现的内存注册表，用于存储BeanDefinition
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(64);


    /**
     * 注册表中注册 name-BeanDefinition
     */
    @Override
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }

    /**
     * 移除注册表中的name-BeanDefinition
     */
    @Override
    public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
        if (this.beanDefinitionMap.remove(beanName) == null) {
            throw new NoSuchBeanDefinitionException(beanName);
        }
    }

    /**
     * 获取注册表中的name-BeanDefinition
     */
    @Override
    public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
        BeanDefinition bd = this.beanDefinitionMap.get(beanName);
        if (bd == null) {
            throw new NoSuchBeanDefinitionException(beanName);
        }
        return bd;
    }
    
    /**
     * 判断注册表中是否包含beanName的key
     */
    @Override
    public boolean containsBeanDefinition(String beanName) {
        return this.beanDefinitionMap.containsKey(beanName);
    }

    /**
     * 获取注册表beanName集合
     */
    @Override
    public String[] getBeanDefinitionNames() {
        return StringUtils.toStringArray(this.beanDefinitionMap.keySet());
    }
    /**
     * 获取注册表大小
     */
    @Override
    public int getBeanDefinitionCount() {
        return this.beanDefinitionMap.size();
    }

    /**
     * 判断beanName是否被使用，在BeanDefinition注册表中获取别名注册表中
     */
    @Override
    public boolean isBeanNameInUse(String beanName) {
        return isAlias(beanName) || containsBeanDefinition(beanName);
    }

}
```

​		可以看出SimpleBeanDefinitionRegistry维护了两个表，一个是从父类那里继承来的aliasMap<alias,name>，一个是自己所维护的beanDefinitionMap<name,beanDefinition>。