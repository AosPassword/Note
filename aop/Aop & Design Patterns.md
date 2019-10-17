#AOP & Design Patterns

## 先从一个简单的例子讲起

​		有很多时候我们的对象信息需要一定的权限管理，比如说，我们要搞一个约会系统，现在你要为每个人设置一个属性，就是“HOT”和“NOT”，用这个属性来表示这个人是否受欢迎。

​		我们先写一个PersonBean：

```java
package org.wuneng.web.springioc.study.proxytest;

public interface PersonBean {
    String getName();
    String getGender();
    String getInterests();
    int getHotOrNotRating();

    void setName(String name);
    void setGender(String gender);
    void setInterests(String interests);
    void setHotOrNotRating(int rating);
}

```

​		再写一个实现类：

```java
package org.wuneng.web.springioc.study.proxytest.impl;

import org.wuneng.web.springioc.study.proxytest.PersonBean;

public class PersionBeanImpl implements PersonBean {
    String name;
    String gender;
    String interests;
    //总分
    int rating;
    //评价次数
    int ratingCount = 0;
    @Override
    public String getName() {
        return name;
    }
    @Override
    public void setName(String name) {
        this.name = name;
    }
    @Override
    public String getGender() {
        return gender;
    }
    @Override
    public void setGender(String gender) {
        this.gender = gender;
    }
    @Override
    public String getInterests() {
        return interests;
    }

    /**
     * @return 平均分
     */
    @Override
    public int getHotOrNotRating() {
        if (ratingCount == 0) return 0;
        return rating / ratingCount;
    }
    @Override
    public void setInterests(String interests) {
        this.interests = interests;
    }

    @Override
    public void setHotOrNotRating(int rating) {
        this.rating += rating;
        ratingCount++;
    }
}
```

​		系统再运行的过程中发现了一个严重的bug，那就是有人可以更改别人的属性，与此同时，有些人可以自己更改自己的评分。这样以来我们这个用户系统就完全么发用了。

​		所以我们要使用保护代理为我们的PersonBean提供访问权限保护，只有拥有权限的人才可以更改指定的属性。

>代理模式：为另一个对象提供一个**替身**或者**占位符**以访问这个对象

​		我们希望客户可以设置自己的信息。同时又可以防止他人更改这些消息。而HotOrNot恰恰相反，我们并不希望客户可以设置自己的值。

​		我们决定为此创建两个代理，一个访问自己的PersonBean，一个访问其他人的PersonBean，而你是否可以修改属性是由这两个代理所决定的。

​		我们来看一下这两个代理类：

```java
public class OwnerPersonBeanImpl implements PersonBean {
    private PersonBean personBean;

    OwnerPersonBeanImpl(PersonBean personBean){
        this.personBean = personBean;
    }

    @Override
    public void setHotOrNotRating(int rating) throws IllegalAccessException {
        throw new IllegalAccessException("就是你小子每天想着该自己的分？");
    }
    @Override
    public String getName() throws IllegalAccessException {
        return personBean.getName();
    }

    @Override
    public String getGender() {
        return personBean.getGender();
    }
    ...//其他的事情都是personBean干的
}
```

```java
public class OtherPersonBeanImpl implements PersonBean {
    private PersonBean personBean;
    OtherPersonBeanImpl(PersonBean personBean){
        this.personBean = personBean;
    }

    @Override
    public String getName() {
        return personBean.getName();
    }

    @Override
    public void setName(String name) throws IllegalAccessException {
        throw new IllegalAccessException("就是你小子每天想着给别人改属性？？");
    }

    @Override
    public void setHotOrNotRating(int rating) throws IllegalAccessException {
        personBean.setHotOrNotRating(rating);
    }
    ...//其他也差不多这样子
}
```

​		这个结构长得其实挺像是装饰模式（不懂得自己下去看看），但是我们这里最主要的功能是访问权限的控制。当然，从某种意义上来说，代理模式确实也就是特别的装饰模式，只不过装饰之处是访问权限控制。

​		其实这么写代理的代码挺累的，花费大量的时间在重复代码上，还得往接口上加``throws IllegalAccessException``，而代码的阅读者看着也不是很直观。

​		事实上Java提供了专门的代理的类和接口，我们来用一用。

##使用Java的Proxy

​		使用Java的Proxy的第一步是创建InvocationHandler。

​		什么是InvocationHandler？你可以这么理解，当代理的方法被调用时，代理就会把这个方法转发给InvocationHandler。

​		我们先看一下具体的实现：

```java
public class OwnerInvocationHandler implements InvocationHandler {
    PersonBean personBean;

    public OwnerInvocationHandler(PersonBean personBean){
        this.personBean = personBean;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (method.getName().startsWith("get")){
                return method.invoke(personBean,args);
            }else if (method.getName().equals("setHotOrNotRating")){
                throw  new IllegalAccessException("就是你小子每天想着怎么改自己评分？");
            }else if (method.getName().startsWith("set")){
                return method.invoke(personBean,args);
            }
        }catch (InvocationTargetException e){
            e.printStackTrace();
        }
        return null;
    }
}public class OwnerInvocationHandler implements InvocationHandler {
    PersonBean personBean;

    public OwnerInvocationHandler(PersonBean personBean){
        this.personBean = personBean;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            if (method.getName().startsWith("get")){
                return method.invoke(personBean,args);
            }else if (method.getName().equals("setHotOrNotRating")){
                throw  new IllegalAccessException();
            }else if (method.getName().startsWith("set")){
                return method.invoke(personBean,personBean);
            }
        }catch (InvocationTargetException e){
            e.printStackTrace();
        }
        return null;
    }
}
```

​		同时再新建一个工厂方法：

```java
public class PersonBeanProxyFactory {
    public static PersonBean getOwnerProxy(PersonBean personBean){
        return (PersonBean) Proxy.newProxyInstance(
                personBean.getClass().getClassLoader(),
                personBean.getClass().getInterfaces(),
                new OwnerInvocationHandler(personBean));
    }
}
```

​		这样当我们调用工厂方法获得来了对应的实例的Owner代理之后，我们调用proxy的任意一个方法，就会调用OwnerInvocationHandler的invoke方法。

​		我们来看下InvocationHandler这个接口：

```java
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
```

​		只有一个invoke方法，参数是Object，Method，Object[]。分别是代理对象proxy，指定方法method，指定方法的参数args。

​		在本例中，我们在实例化InvocationHandler的时候直接注入了原始对象PersonBean，之后使用method.invoke(personBean,args)完成了方法的真正调用。

​		我们来写一个测试类来测试一下我们的代码：

```java
package org.wuneng.web.springioc.study.proxytest.test;

import org.wuneng.web.springioc.study.proxytest.PersonBean;

import org.wuneng.web.springioc.study.proxytest.factorys.PersonBeanProxyFactory;
import org.wuneng.web.springioc.study.proxytest.impl.PersonBeanImpl;

public class Test {
    public static void main(String[] args)  {
        PersonBean personBean = new PersonBeanImpl();
        personBean.setName("张三");
        personBean.setGender("男");
        personBean.setInterests("吃饭睡觉打豆豆");
        PersonBean ownerProxy = PersonBeanProxyFactory.getOwnerProxy(personBean);
        ownerProxy.setName("李四");
        ownerProxy.setHotOrNotRating(10);
    }
}
```

​		运行后的结果：

```java
Exception in thread "main" java.lang.reflect.UndeclaredThrowableException
	at com.sun.proxy.$Proxy0.setHotOrNotRating(Unknown Source)
	at org.wuneng.web.springioc.study.proxytest.test.Test.main(Test.java:16)
Caused by: java.lang.IllegalAccessException: 就是你小子每天想着怎么改自己评分？
	at org.wuneng.web.springioc.study.proxytest.invocationhandlers.OwnerInvocationHandler.invoke(OwnerInvocationHandler.java:23)
	... 2 more
```

​		嗯，和我们之前预测的结果是相同的。

​		现在这个代码。。写的挺垃圾的。因为我的invoke方法中为了匹配指定的方法，使用了按照方法名字符串匹配的方式，这种方式难以扩展，依赖性强，可读性差，一不小心把String输错还容易出大问题，我们有什么办法将这个匹配指定方法的方式改一下。

​		比如说：注解？

​		设计两个注解，一个注解是@Owner，一个注解是@Other，如果一个方法只能Owner调用，那么就在方法上面标一个@Owner，否则反之。在OwnerInvocationHandler的Invoke方法中，我们只需要判断这个方法是否有@Other标记，如果有就抛出错误，没有就执行，皆大欢喜。

​		当然，关于如何去具体实现这个过程，我觉得你们看了ioc应该都懂了，你们自己写一个，真的挺简单的。

​		如果实在不会写。。。

​		。。。

​		。。。

​		。。。

​		就去把我那篇IOC再看一遍！

​		现在你有什么疑问么？

​		。。。

​		。。。

​		。。。

​		算了还是我来提个问题吧。

​		我们看见invoke方法有三个参数，事实上我们只用到了两个参数（Method & Object[])，我们注入了原来的对象使用args调用了Method的invoke，那么第一个参数Object proxy到底有什么用处，我们为什么不用这个对象呢？

​		这个问题挺简单的，应该一下就想到了。

​		好了让我们再整理一下思路看看有什么我们错过的问题没有。。。

​		没有么？

​		不，有问题！

​		我们来仔细观察一下这个生成了Proxy的工厂方法：

```java
public class PersonBeanProxyFactory {
    public static PersonBean getOwnerProxy(PersonBean personBean){
        return (PersonBean) Proxy.newProxyInstance(
                personBean.getClass().getClassLoader(),
                personBean.getClass().getInterfaces(),
                new OwnerInvocationHandler(personBean));
    }
}
```

​		我们可以看到，这个方法返回了一个PersonBean的实现类。这个实现类是由Proxy的newProxyInstance静态工厂方法产生的一个Object类转换过去的。

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
```

​		这个Object是一个PersonBean的实现类！那么问题来了，这个类是什么？

​		事实上，在这个使用代理的代码框架中，我们从始至终就只是创建了一个PersonBean的实现类，那就是PersonBeanImpl，但是Proxy.newProxyInstance()方法返回的却绝对不是PersonBeanImpl类，因为PersonBeanImpl的方法和OwnerInvocationHandler的invoke方法八竿子打不着，PersonBeanImpl甚至都不知道OwnerInvocationHandler的存在。当然，更不可能是OwnerInvocationHandler类，因为OwnerInvocationHandler实现的是InvocationHandler接口，怎么可能是PersonBean的实现类。

​		那么返回的实现类是什么？

​		我们在测试代码里康康：

```java
public class Test {
    public static void main(String[] args)  {
        PersonBean personBean = new PersonBeanImpl();
        personBean.setName("张三");
        personBean.setGender("男");
        personBean.setInterests("吃饭睡觉打豆豆");
        PersonBean ownerProxy = PersonBeanProxyFactory.getOwnerProxy(personBean);
        System.out.println(ownerProxy.getClass().getName());
//        ownerProxy.setName("李四");
//        ownerProxy.setHotOrNotRating(10);
    }
}
```

​		输出结果：

```java
com.sun.proxy.$Proxy0
```

​		这是个我们从来没见过的类。事实上它所处的包也和我们的包大相径庭（在sun的包里），这肯定不是Sun公司预知到了我么要写这么个接口（如果是那才吓人），所以只有一个可能了：

​		这个类是自动动态生成的。

## Proxy类是如何炼成的

​		如果我们不难么细致的来分析.java文件的执行过程，我们可以将它分成两个步骤，一个是把.java文件转变成含有字节码指令的.class文件（javac命令），之后就是.class文件被加载到jvm中。

​		Proxy类的生成，就发生在.java文件转变为字节码指令这样一个过程中。

​		我们先来看一个简单的例子：

```java
package nine;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxyTest {
    public interface IHello{
        void sayHello();
    }
    static class Hello implements IHello{

        @Override
        public void sayHello() {
            System.out.println("hello world");
        }
    }
    static class DynamicProxy implements InvocationHandler{
        Object object;

        Object bind(Object o){
            this.object = o;
            return Proxy.newProxyInstance(object.getClass().getClassLoader(),
                                          object.getClass().getInterfaces(),
                                          this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println(proxy.getClass().getName());
            System.out.println("welcome");
            return method.invoke(object,args);
        }
    }

    /**
     * 输出结果：
     * com.sun.proxy.$Proxy0
     * welcome
     * hello world
     */
    public static void main(String[] args) {
        IHello hello = (IHello) new DynamicProxy().bind(new Hello());
        hello.sayHello();
    }
}
```

​		我们可以在main()方法加入下面这句：

```java
 System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
```

​		这样磁盘上就会产生一个名为"$Proxy0.class"的Class文件，将其反编译后结果如下：

```java
package nine;

import nine.DynamicProxyTest;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements DynamicProxyTest.IHello{
    private static Method m3;
    private static Method m1;
    private static Method m0;
    private static Method m2;

    static{
        try{
            m3 = Class.forName("nine.DynamicProxyTest$IHello").getMethod("sayHello",new Class[0]);
            //hashCode,equals之类的方法
        }catch (NoSuchMethodException e){
            throw new NoSuchMethodError(e.getMessage());
        } catch (ClassNotFoundException e) {
            throw new NoClassDefFoundError(e.getMessage());
        }
    }

    /**
     * Constructs a new {@code Proxy} instance from a subclass
     * (typically, a dynamic proxy class) with the specified value
     * for its invocation handler.
     *
     * @param h the invocation handler for this proxy instance
     * @throws NullPointerException if the given invocation handler, {@code h},
     *                              is {@code null}.
     */
    protected $Proxy0(InvocationHandler h) {
        super(h);
    }

    @Override
    public void sayHello() {
        try{
            this.h.invoke(this,m3,null);
            return;
        }catch (RuntimeException e){
            throw e;
        }catch (Throwable throwable){
            throw new UndeclaredThrowableException(throwable);
        }
    }
}
```

​		这个代理类的实现代码也很简单。我相信你们一看就懂了。

​		至于它是怎么更具Class文件的格式规范去瓶装字节码的,可以去看sun.misc.ProxyGenerator的源码。

​		别看我，我也没看过这源码。我也没法讲。

## 这样的Proxy就满足了么？

​		Java自带包的Proxy好用么？比起之前的代码确实要好用不少。但是我们这样就知足了么？

​		Java自带的Proxy实现有个非常严重的问题。那就是得有接口JVM才能自动帮我们生成$Proxy0。就像PersonBean明明只是一个用户对象我们还非得给他添加一个接口。

​		有没有什么方法可以让我们代理没有接口的类？？

## CGlib动态代理

​		我们先写一个CGLibProxy:

```java
public class CGLibProxy implements MethodInterceptor {
    public <T> T getProxy(Class<T> clazz){
        return (T) Enhancer.create(clazz,this);
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object o1 = methodProxy.invokeSuper(o,objects);
        after();
        return o1;
    }
    
    private void before(){
        System.out.println("before");
    }
    
    private void after(){
        System.out.println("after");
    }
}
```

​		我们不难发现这个intercept方法比起之前JDK的InvocationHandler的invoke对了一个参数：MethodProxy。并且我们并没有给这个CGLibProxy提供一个原始对象。

​		不难从代码上发现这个o就是原始对象。那为什么有两个Method?

​		事实上第一个Mehtod指的是代理类的指定方法，而后米娜的MethodProxy才是原始方法。如果你调用Method.invoke，分分钟堆栈溢出（自己调用自己可不就堆栈溢出了么）。

​		这是CGLib给我们提供的方法级别的代理，可以理解为对**方法**的拦截。

​		而我们写的getProxy方法就是获得指定类的代理对象。

​		随便写一个测试类来实践一下：

```java
public class Test {
    public static void main(String[] args) {
        CGLibProxy cgLibProxy = new CGLibProxy();
        Hello hello = cgLibProxy.getProxy(Hello.class);
        hello.sayHello();
    }
}
```

​		输出结果:

```java
before
hello!
after
```



## 开始AOP之路

​		AOP的意思是面向切面编程，听上去还是挺玄学的。其实就是流程图里先处理啥，后处理啥，从上到下的执行的逻辑。Filter就挺像AOP（先执行FIlter后执行请求的逻辑代码）。不过AOP是利用预编译方法和运行期动态代理实现程序功能的统一维护的一种技术。

​		讲真的这句话说的挺有B格（让人摸不着头脑）的，莫慌，让我慢慢给你讲。

​		啥是个预编译方法，其实就是.java -> .class的过程，在这个过程中我们其实是可以利用特定框架的代码对生成的.class文件进行一定的改变的。这个具体怎么实现，我们之后再说。

​		而运行期动态代理程序，拿我们那个Hello来说，当JVM加载Hello类之后，我们要得到它的代理类，然后JVM内部就自动生成了$Proxy0，这就是运行期动态代理。

​		代码方面大部分的编程思想的专有名词的定义并不像理科名词那么严谨，你只要大概理解它的意思就可以了，并不需要要硬扣其中的定义。

​		我们再介绍AOP的核心概念，当然，就和上面我说的一样，你只要领会了那个意思就可以了。

## AOP核心概念

### 横切关注点

​		其实就是告诉程序:

​		对哪些方法进行拦截：``if (method.getName().startsWith("get"))``

​		拦截后怎么处理：``return method.invoke(personBean,args);``

​		这些关注点称之为横切关注点。

### 切面

​		切面就是对于横切关注点的抽象，你可以理解为就是流程图中的一个块，比如我们之前写的那个OwnerInvocationHandler，就是一个切面。

​		就像上面我们所提到的OwnerInvocationHandler就是一个切面，负责查看是否有调用方法的权限。

### 连接点

​		被拦截到的点，因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器。

​		放到PersonBeanImpl中，它的所有方法都是连接点。因为所有的方法都被InvocationHandler拦截了，当然你也可以注释掉``if (method.getName().startsWith("get"))``，让这些方法不被拦截。

​		假如你使用注释的话，那么被你注释的方法就是连接点。

### 切入点

​		对连接点进行拦截的定义。就是告诉程序要拦截哪些程序。比如说``if (method.getName().startsWith("get"))``就是一个切入点，它告诉程序所有以get开头的方法都会成为连接点（被拦截之后处理）。

### 通知 

​		通知就是拦截到连接点之后要执行的代码，在Spring通知可以分为前置，后置，异常，返回，环绕五类类。

​		它们大致的逻辑是像这样子的：

```java
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("before");//前置
            return method.invoke(object,args);
        }
```

```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object object = method.invoke(object,args);
        System.out.println("after");//后置
        return object;
    }
```
```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("around");//环绕
        Object object = method.invoke(object,args);
        System.out.println("around");//环绕
        return object;
    }
```

```java
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object obj = null;
        try{
        	obj = method.invoke(object,args);
            System.out.println("success");//返回
        }catch （Exception e){
            System.out.println("error");//异常
        }
        return obj;
    }
```

### 目标对象

​		OwnerInvocationHandler里的PersonBeanImpl

### 织入

​		其实就是用Proxy.newProxyInstance()获得代理对象。

​		就是我们将PersonBeanImpl塞到了OwnerInvocationHandler里面，再把OwnerInvocationHandler塞到这方法里这个过程。

### 引入

​		在不修改代码的前提下，引入可以在**运行期**为类动态地添加一些方法或字段。

​		我觉得引入这个东西值得我去详细讲解一下，用AOP的行话来说，对方法的增强称之为织入，对类的增强称之为引入。

​		这里又提到了一个新的东西叫做**增强**，其实我特别不喜欢代码书的一点，就是它们总能写一些奇奇怪怪的专有词去形容一些明明很简单的东西，并且还不想给你解释这个词是啥意思（而且这种词的定义往往还毫无卵用）。

​		增强其实就是对这个东西进行一些额外的操作。比如之前代理的类调用函数，就是在原来类的函数周围加了一层新代码。这个函数的功能变多了，这就叫做增强。

​		之前的例子都是织入，我们来试试引入怎么搞。

​		看一下这个道歉的接口。

```java
public interface Apology {
    void saySorry(String name);
}
```

​		我们想让一个已经写好的类（假设这个类叫GreetingImpl）中实现这个接口，但是我们不想在代码中让GreetingImpl直接实现这个接口（因为这个类可能十分复杂而且非常容易出Bug），而是象再程序运行的时候动态实现它。

​		再说的直白一点，我想给这个已经写好的类里面加一个新的方法，但是我又不想把这个方法写到这个已经写好的类里面去，我想把这个新方法写在别的地方。

​		当然这个的逻辑并不复杂，我们只需要创建一个新的类去实现这个Apology接口，然后再写一个类，同时代理这个新的类和GreetingImpl这个旧的类，那么这个一下子代理了两个类的新类完成了引入。

​		这其实也就是调一调API的事情，但是你要从字节码这个层面来自己写一个，我是不太行，谁行谁上吧。

## 给咱也整一个AOP框架

​		首先我们先定一个需求，比如说，我们我们老板让测试一下，我们的代码的执行时间。

​		我们就想，哎，给代理方法加一个注解，value里加上其他注解，这样**被**value中添加的那些其他注解所注解的类，就可以使用我们写的代理了。原本类连注解都不用加！

​		讲道理上面这句话，给十个人看，十个人都看不懂。莫慌，看代码。

​		来来来开工了！

​		首先我们需要一个注解来帮我们表示我们需要代理的类。就@Aspect的吧，大家都用这个。

```java
@Target(ElementType.TYPE)//修饰类，接口
@Retention(RetentionPolicy.RUNTIME)
public @interface Aspect {
    Class<? extends Annotation> value();
}
```

​		这时候我们就要想了，我这个业务呢，肯定不可能只写一个代理方法。因为我肯定是不能只是写一层的切面（就是一层的代理），以后肯定要加需求的。所以啊，在使用CGLib生成代理类的时候，就不能只是像以前一样调用一个method.invoke就走那么简单了。毕竟你不能代理类套代理类，因为代理类起码也是在字节码层面生成的。.java文件里可没有这个Class让你传到参数里面去。

​		所以我们要一口气把所有**匹配**的，**我们自己写的**代理方法都放到一起然后让CGLib生成一个代理类。

​		我们怎么把所有的代理方法都放到一起去？那肯定是就要出动链表了啊！

​		看看我们的实现：

​		先定义一个接口，这个接口用来定义代理方法：

```java
public interface Proxy {
    Object doProxy(ProxyChain proxyChain) throws Throwable;
}
```

​		因为我们需要链式调用每一个Proxy，所以我们肯定需要一个链。而这个链就是用这个参数ProxyChain实现的。这样我们就可以在调用doProxy方法的同时链式调用其他的Proxy的方法。

```java
public class ProxyChain {
    private final Class<?> targetClass;
    private final Object targetObject;//目标对象
    private final Method targetMethod;//目标方法
    private final MethodProxy methodProxy; //代理方法
    private final Object[] methodParams;
    private List<Proxy> proxyList; //代理对象列表
    private int proxyIndex = 0;

    public ProxyChain(Class<?> targetClass, Object targetObject, Method targetMethod, MethodProxy methodProxy,
                      Object[] methodParams, List<Proxy> proxyList) {
        this.targetClass = targetClass;
        this.targetObject = targetObject;
        this.targetMethod = targetMethod;
        this.methodProxy = methodProxy;
        this.methodParams = methodParams;
        this.proxyList = proxyList;
    }

    public Class<?> getTargetClass() {
        return targetClass;
    }

    public Method getTargetMethod() {
        return targetMethod;
    }

    public Object[] getMethodParams() {
        return methodParams;
    }

    public Object doProxyChain() throws Throwable {
        Object methodResult;
        if (proxyIndex < proxyList.size()) {
            methodResult = proxyList.get(proxyIndex++).doProxy(this);
        } else {
            methodResult = methodProxy.invokeSuper(targetObject, methodParams);
        }
        return methodResult;
    }
}
```

​		通过proxyIndex++，我们就可以遍历调用List中的每一个Proxy的方法。

​		但是我们要注意，我们必须确保所有的before()方法调用都必须在after()前（忘记啥意识的去前面看看五类通知），你不能已经调用了mehtod.invoke方法再调用第二个Proxy的before()方法啊，那我这还before()个锤子。

​		来看看我们的默认Proxy是如何实现的：

```java
public class AbstractAspectProxy implements Proxy {
    @Override
    public Object doProxy(ProxyChain proxyChain) throws Throwable {
        Object result = null;
        Class<?> cls = proxyChain.getTargetClass();
        Method method = proxyChain.getTargetMethod();
        Object[] params = proxyChain.getMethodParams();
        begin();
        try {
            if (intercept(cls, method, params)) {//看看是不是我想要代理的方法
                before(cls, method, params);
                around(cls,method,params);
                result = proxyChain.doProxyChain();
                around(cls,method,params);
                after(cls, method, params);
            } else {
                result = proxyChain.doProxyChain();
            }
        } catch (Exception e) {
            error(cls, method, params, e);
            throw e;
        } finally {
            end();
        }
        return result;
    }
    public void begin() {

    }

    public boolean intercept(Class<?> cls, Method method, Object[] params) throws Throwable {
        return true;
    }

    public void before(Class<?> cls, Method method, Object[] params) throws Throwable {

    }

    public void after(Class<?> cls, Method method, Object[] params) throws Throwable {

    }

    public void error(Class<?> cls, Method method, Object[] params, Throwable e) throws Throwable {
        e.printStackTrace();
    }

    public void end() {

    }

    public void around(Class<?> cls, Method method, Object[] params)throws Throwable{

    }
}
```

​		我们首先要关注的地方在于，``result = proxyChain.doProxyChain();``没错我们又调用回去了！通过这个方法，我们用proxyChain调用Proxy，Proxy再调用proxyChain，就这样不断地来回调用（压栈）的过程中，因为index++方法实现了链式调用。同时我们还保证了在执行完methodProxy.invokeSuper之前绝对不可能调用后面的after方法。

​		那为什么剩下的详细方法我都是空着的呢？

​		这是因为我这里使用了模板设计方法。

>模板方法模式：在一个方法中定义一个算法的骨架，而将这一些步骤延迟到子类中。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤

​		当然我原来是想详细讲一讲模板方法模式的，但是我实在是懒得讲了，自己下去多看看，回来看这个代码会有比较大的收获。

​		我专门把这个类的命名为Abstract，但是事实上这个定义是class，这并不是我对名字命名错了，而是我想让使用者知道，我希望使用者继承我这个类，并且重写我的方法。

​		比如我们之前说了，想要给程序计时，就这么写：

```java
@Aspect(value=Controller.class)
public class ControllerAspect extends AbstractAspectProxy {
    private static final Logger LOGGER = LoggerFactory.getLogger(ControllerAspect.class);
    private long begin;

    @Override
    public void before(Class<?> cls, Method method, Object[] params) throws Throwable {
        LOGGER.debug("-----begin-----");
        LOGGER.debug(String.format("class: %s", cls.getName()));
        LOGGER.debug(String.format("method: %s", method.getName()));
        begin = System.currentTimeMillis();
    }

    @Override
    public void after(Class<?> cls, Method method, Object[] params) throws Throwable {
        LOGGER.debug(String.format("time:%dms", System.currentTimeMillis() - begin));
        LOGGER.debug("-----end-----");
    }
}
```

​		我们再来写一个创建代理对象的方法：

```java
public class ProxyFactory {

    @SuppressWarnings("unchecked")
    public static <T> T createProxy(final Class<?> targetClass, final List<Proxy> proxyList) {
        return (T) Enhancer.create(targetClass, new MethodInterceptor() {
            public Object intercept(Object targetObject, Method targetMethod, Object[] methodParams,
                                    MethodProxy methodProxy) throws Throwable {
                return new ProxyChain(targetClass, targetObject, targetMethod, methodProxy, methodParams, proxyList)
                        .doProxyChain();
            }
        });
    }
}
```

​		好了现在工具都有了，现在我们的目标就是把现在已有的工具和之前我们曾经写过的Ioc的BeanFactory结合在一起，

​		我们来捋一下逻辑：

- 类加载所有的我们程序的类
- 找出所有的代理类（即有@Aspect注解的类）
- 看看这些代理类的目标分别是哪些注解（比如上面写的@Controller），为每一个注解它们对应的代理链，记住这些注解，我们暂时称之为目标注解
- 从所有的加载的类里面获得了拥有目标注解的类，然后将类与对应的代理链传入ProxyFactory创建代理对象，放到``Map<Class<?>, Object>``里面去

​        当当当当，这样我们的Bean工厂所实例化的类就是具有对应代理链的对象的代理对象！每当使用这个对象的方法的时候，都会执行代理！

​		下面是具体代码实现：

​		。。。

​		。。。

​		。。。

​		算了你们学了这么多了，自己来实现一个吧，哎，嘿嘿嘿。

## 参考资料

《架构探险》

《深入理解Java虚拟机》

《headfirst设计模式》