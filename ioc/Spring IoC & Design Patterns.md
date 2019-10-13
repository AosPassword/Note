# Spring IoC & Design Patterns

## 从一个简单的例子说起

​		先举个例子，在这个例子里面，我写了一个一个组件，这个组件可以提供一个特定导演导演过的所有电影的List：

```java
class MovieLister{
    public Movie[] moviesDirectedBy(String arg) {
      ColonDelimitedMovieFinder finder = new ColonDelimitedMovieFinder("movie.txt");
      List allMovies = finder.findAll();
      for (Iterator it = allMovies.iterator(); it.hasNext();) {
          Movie movie = (Movie) it.next();
          //如果不是这个导演的片子，就把这个片子从list中移除
          if (!movie.getDirector().equals(arg)) it.remove();
      }
      return (Movie[]) allMovies.toArray(new Movie[allMovies.size()]);
}
```

​		它让一个finder对象返回了所有的电影，然后他就遍历整个列表寻找他想要的导演的片子。

​		我们真正应该关注的点不是这个方法是如何实现的，而是这个lister对象和finder对象是如何配合的。

​		为什么lister和finder的配合才应该是我们关注的重点呢？因为我有个朋友也想用我的程序，但是他的电影名单不是储存在txt文件中的，而是储存在数据库的，还有一部分在xml文件里面。

​		这时候如果朋友想用的我的程序，他就不得不修改我的程序。

​		所以这很明显是违反对修改关闭的原则的。

> 开放封闭原则：软件实体应该是可扩展，而不可修改的。

​		所以我们编写了一个接口来符合这个开放封闭原则：

```java
public interface MovieFinder{
    List findAll();
}		
```

```java
class MovieLister{
    private MovieFinder finder;
    public MovieLister(MovieFinder finder){
        this.finder = finder;
    }
```



​		这样，让我们的Lister类中只依赖于MovieFinder接口，让MovieFinder调用findAll()方法，我们就实现了这个系统的解耦。

​		当然只有接口时不够的，所以得让我朋友实现MovieFinder接口，让他把自己的实现类**注入**到我的Lister中，这样我朋友就可以一使用我的代码了。

​		在Patterns of Enterprise Application Architecture一书中，我们把这种情况称为插件（plugin）：

​		MovieFinder的实现类不是在编译期连入程序之中的，因为我并不知道我的朋友会使用哪个实现类。我们希望MovieLister类能够与MovieFinder的任何实现类配合工作，并且允许在运行期插入具体的实现类，插入动作完全脱离我（原作者）的控制。这里的问题就是：如何设计这个连接过程，使MovieLister类在不知道实现类细节的前提下与其实例协同工作。

​		将这个例子推而广之，在一个真实的系统中，我们可能有数十个服务和组件。在任何时候，我们总可以对使用组件的情形加以抽象，通过接口与具体的组件交流（如果组件并没有设计一个接口，也可以通过适配器与之交流）。

​		而我们上面对于这个类的优化就称之为控制反转

##控制反转（Inversion of Control）

​		很明显这就是我们标题提到的IoC了。

​		但是有个问题随之而生：什么时控制反转，它反转了什么控制？

​		在我们最之前的案例中，整个Lister都是建立在finder对象之上的，换句话说，lister依赖于finder，就像汽车依赖于轮子。我们先有了findAll()方法，然后让lister去遍历寻找。就像我们有四个轮子，现在我们所做的事情是用这四个轮子去造一辆车。但是在我们决定使用控制反转之后，事情发生了变化。

​		我们的lister不再依赖于finder，而是依赖于MovieFinder接口，我们的lister用MovieList接口去命令其他的finder去实现MovieLIst接口，所有的finder都是基于MovieFinder接口编写的。现在时finder依赖于lister。

​		我们现在造车不再需要这四个指定的轮胎了，我们只需要画出整个汽车的工程图，然后让别人去根据我们所画的图来制造符合要求的轮胎。

​		只要别人所提供的轮胎符合我们所画的图纸，那么不管这个轮胎是中国造的还是日本造的，都能安装到我的车上——事实上我的车也不在乎是谁提供的轮子，只在乎这个轮子是否符合我所要求的尺寸。

> 依赖倒置原则：要依赖抽象，不要依赖具体类

让我们来看一个完完全全的反面典型：

```java
public class DependentPizzaStore {
 
	public Pizza createPizza(String type) {
		Pizza pizza = null;
		if (type.equals("cheese")) {
			pizza = new NYStyleCheesePizza();
		} else if (type.equals("veggie")) {
			pizza = new NYStyleVeggiePizza();
		} else if (type.equals("clam")) {
			pizza = new NYStyleClamPizza();
		} else if (type.equals("pepperoni")) {
			pizza = new NYStylePepperoniPizza();
		} else {
			System.out.println("Error: invalid type of pizza");
			return null;
		}
		pizza.prepare();
		pizza.bake();
		pizza.cut();
		pizza.box();
		return pizza;
	}
}
```

​		这个类教科书一般的依赖了大量的类，导致代码看起来混乱不堪。那么我们如何用依赖倒置原则去优化这个类呢？

​		依赖导致原则告诉我们，不要依赖于类，而是依赖于接口。但是这个类依赖了大量不同的Pizza对象，我们没法用一个Pizza接口去概括这么多的类。

​		并且这些披萨都是只现在所有的。一个PizzaStore不可能就只有这么一点披萨的种类，所以一我们现在最好的方法是使用工厂方法模式：

> 工厂方法模式定义了一个创建对象的接口，但是由子类决定要实例化的类是哪一个，工厂方法让类把实例化推迟到子类

​		定义一个这样的接口：

```java
public interface PizzaFactory {
    Pizza getPizza(String type); 
}
```

```java
public class InDependentPizzaStore {
 	PizzaFactory factory;
    DependentPizzaStore(PizzaFactory factory){
        this.factory = factory;
    }
    
	public Pizza createPizza(String type) {
		Pizza pizza = null;
		pizza = factroy.getPizza(type);
		pizza.prepare();
		pizza.bake();
		pizza.cut();
		pizza.box();
		return pizza;
	}
}
```

​		这样我们的PizzaStore就只与PizzaFactory产生了关系，并且并不关心PizzaFactory到底是怎么得到Pizza的。我们完成了解耦和依赖倒置。

​		那么这个时候你就可能想要问了，创建Pizza的代码其实还是那么的繁杂，因为毕竟我们还是要实现这个PizzaFactory的，而实现这个Pizza的代码其实还是要写入PizzaFactory的实现类中的。那么PizzaFactory的实现类还是要依赖于Pizza的种种子类。那这又有什么意义呢？

​		意义在于，当你使用了PizzaFactory，虽然工厂方法还是要依赖于大量的对象实例，但是你的PizzaStore是可以重用的，只要你给PizzaStore中注入了不同的PizzaFactory就可以获得不同的PizzaStore了！

​		但是我们还是没法避免另一个问题：PizzaFactory的实现类本身**依赖了**实现类。我们有没有办法**不使用new的方法来实例化指定的类并且返回Pizza的子类**呢？

​		要是我们可以直接给Factory一个type，让他自己加载我们所指定的类，然后将此类例化，返回实例化对象，那这不久皆大欢喜了嘛！

​		那么我们现在就有了新的问题：

​		如何加载类并使其实例化？

##如何加载类并使其实例化？

​		关于这个问题，我们先要解决一个更加基础的问题：

​		啥是个加载类啊？类咋加载的啊？

​		咱们先说说Java这个语言是怎么跑的。

​		这个大家应该都知道，从.java编译成.class文件，然后让JVM去处理.class文件。

​		你在跑Java程序的时候，JVM并不是一下子就把所有的.class文件全部都加载了（那Java启动得有多慢啊），类的加载实在程序运行时进行的。

​		所以说你在跑Java程序的时候，如果没有直接用到这个程序中的某个类，这个类的.class很可能是没有被JVM所加载的，直到你的程序跑到了要用到这个类的时候了，JVM才会读取这个类的.class文件，然后将类加载到虚拟机内存之中。

​		而我们现在要做的就是，当我们调用getPizza(String type)的时候，我们可以根据传入的type，找到对应的Pizza子类的.class文件，让JVM去加载它，让我们获得它的Class类对象，利用这个Class类对象，去实例化指定对象，然后将指定对象返回。

​		先从一个简单的demo开始：

```java
package org.wuneng.ioc.base;

public class Hello {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
public class Test {
    public static void main(String[] args) {
        testClass();
        cast();
    }

    public static void testClass() {
        Class<?> helloClass = Hello.class;
        System.out.println(helloClass.getName());
    }

    public static void cast() {
        Class<?> clazz = null;
        try {
            clazz = Class.forName("org.wuneng.ioc.base.Hello");
            Object o = clazz.getConstructor().newInstance();
            try {
                Hello student_one = (Hello) o;
                System.out.println("cast to Hello");
            } catch (ClassCastException e) {
                e.printStackTrace();
            }
            if (o.getClass().getCanonicalName().equals("org.wuneng.ioc.base.Hello")) {
                System.out.println("name is Hello");
                Hello student_two = (Hello) o;

            }
            if (o instanceof Hello) {
                Hello student_three = (Hello) o;
                System.out.println("is student");
            }

        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();

        }
    }
```

​		输出结果如下：

```java
org.wuneng.ioc.base.Hello
cast to Hello
name is Hello
is student
```

​		从上面的demo我们成功的使用了两种方式实现了类加载，用的函数分别是``Class.forName(String s)``和``Class<?> helloClass = Hello.class;``这样我们就成功解决了我们上面所提出来的问题：如何加载类并使其实例化？

​		我们可以根据传入是String字符串寻找到指定的类，然后将制定类实例化然后返回其实例。这样我们PizzaFactory的实现类就可以不具体写明要new哪个类，而是在运行的过程中决定要加载并实例化哪个类。

​		来让我们用这个理论写一个PizzaFactroy出来吧：

```java
public class NYStyleCheesePizzaFactroy implements PizzaFactory {
    @Override
    public Pizza getPizza(String type) {
        try {
            Class<?> clazz = Class.forName("org.wuneng.ioc.pizza.NYStyle"+
                    type.substring(0,1).toUpperCase()+
                    type.substring(1)
                    +"Pizza");
            Object o = clazz.getConstructor().newInstance();
            return (Pizza) o;
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

​		这样，只要是指定包下的所有按照命名模式命名的类，都可以被这个类所加载到。

​		但是这样的结构依旧存在问题，问题在于``return (Pizza) o;``我们在这串代码中有一个比较不安全的做法，就是我们直接将一个Object强转为Pizza返回了。

​		但是我们在这串代码中并不能保证这个Object一定是Pizza，万一有一个类没有实现Pizza接口，比如一个Pizza的模型：``NYStyleModelOfPizza``，当输入的时候失误输入了modelOf参数，那么我们这个方法就会将一个不是Pizza的东西转换为Pizza。这毫无疑问会抛出错误，然后返回null。

​		当然你可以说这个方法依旧可以执行下去，但是很明显这样实在是太过于粗暴了。

​		那么，我们如何在加载类的时候让我们的程序直到这个类倒是是不是我们真正想要的类呢？

​		我们可以在类上做一个记号，当我们加载类的时候，如果这个类是具有我们打的标签，那么这个类就是我们确实想要的类。

​		来看看改进之后的代码长什么样子：

```java
            if  (o instanceof Pizza) {
                return (Pizza) o;
            }else {
                System.out.println("this class is not a pizza!");
            }
```

​		很简单的逻辑。现在只有是Pizza的东西才可以被转为Pizza，问题解决了。

​		但是有的时候，我们不希望工厂方法给我只返回具有特定方法的类。比如说我们现在要当一个综合性的餐厅，客人可能点Pizza，也可能点一份汉堡。我希望我的Factory不仅可以返回Pizza，又可以返回汉堡，但是我又不希望它返回一个我们菜单上没有的东西，那我们如何去实现它呢？

​		其实这也很简单，我们只需要定义一个没有方法需要实现的接口，菜单上的类去实现这个接口，我们就可以完美实现这个需求了。

​		这个接口也有特殊的名字，叫：**标记接口**。这个接口仅仅是用来给类做一个标记，可以说是很恰当了。

​		当这个工厂用久了之后我们就会发现另外一个问题：每次我们调用getPizza(String type)方法的时候，我们都得调用类加载器搜索制定类，然后将其实例化，这是非常消耗性能的一系列操作。

​		那么如何降低这个方法所导致的性能开销呢？

​		很明显，如果我们可以把类，或者直接把类实例化的对象缓存起来，然后当再次使用的时候将其直接拿出来使用就好了。

​		但是，缓存类还可以明白，但是直接缓存对象。。。会不会导致什么不好的情况。比如多线程操作同一个对象会不会使得程序逻辑被破环？

​		事实上，这个时候如果学Servlet学的比较扎实的人就应该会回想起来当年学Servlet书上会经常提及到的一句话：

> “Deployment Descriptor”, controls how the servlet container provides instances of the servlet.For a servlet not hosted in a **distributed** environment (the default), the servlet container must use **only one instance** per servlet declaration. However, for a servlet implementing the SingleThreadModel interface, the servlet container may instantiate multiple instances to handle a heavy request load and serialize requests to a particular instance.

​		上面这条引用出自Servlet规范，肯定是足够权威了。

​		上面提及了，只要你的Servlert不是处于分布式环境（**distributed**）中的，并且没有实现``SingleThreadModel interface``,那么这个Servlet声明就是单例的（**only one instance** per servlet declaration）

​		当初学Servlet的时候，第一个要学的就是不要在Servlet中随意添加类属性（Field），因为Servlet是单例的，如果你的类方法中操作了类属性，那么它在多线程环境下就会产生难以预知的问题。

​		当然，方法中定义的局部变量并没有这个问题。因为在多线程中局部变量都是处于栈中的，而栈是线程独享的，所以局部变量不会被多线程所操作，自然不用关心它的多线程安全性。

​		当然提起Servlet的单例性质的主要原因是为了证明：**只要程序写得好，单例在多线程中是利大于弊的**。

​		当然，我们还有一些地方值得关注，那就是之前所提到的这个工厂方法：

```java
Pizza getPizza(String type)
```

​		我们先假设给用户设计的程序逻辑是这样的：

​		我们让用户来输入一个String字符除按，然后我们把这个String字符除按传入这个工厂方法中，这个工厂方法加载指定的类，缓存并返回它的实例对象。如果没找到指定类，那么就返回null。

​		但是每次在让用户输入的时候我们都要清楚明白的认识到：**用户输入的内容是不可信的**。

​		把类加载并实例化这样一个方法交给用户来控制其中一部分的变量，这无疑是十分危险的。

​		比如说，假如有个人掌握了向你的项目文件夹中传文件的方法，他写了个NYStyleHackerPizza类传入了你的程序目录下，然后使用你的getPizza方法加载它的NYStyleHackerPizza，并将其实例化。很明显，这是十分危险的。

​		所以我们可以选择另外一种方法：扫描指定包下的所有类，然后将这些类加载缓存，当用户调用指定类的时候，我们查看我们缓存的类，如果存在，就返回他的实例，如果不存在，那么我们就返回null。

​		这样就把类加载的能力掌握在了自己的手上，不会出现之前所说的问题。

​		让我们来看看这个进阶版的类加载器：

```java
import javax.mail.event.MailEvent;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Method;
import java.net.URL;
import java.util.*;

public class ControllerLoader {
    /**
     * 首先先将controller中的所有类加入到classSet之中
     * 然后将这些类进行实例化放入Map之中
     * @return Map<String   ,   Object>
     */
    public Map<String, Object> load() {
        Set<Class<?>> classSet = new HashSet<>();
        try {
            String packageName = "org.wuneng.ioc.workone.controller";
            Enumeration<URL> resources = ControllerLoader.class.getClassLoader().getResources(packageName.replaceAll("\\.", "/"));
            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                String protocol = resource.getProtocol();
                if (protocol.equals("file")) {
                    String packagePath = resource.getPath();
                    addClass(classSet, packagePath, packageName);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        Map<String, Object> urlControllersMap = new HashMap<>();
        for (Class<?> clazz : classSet) {
            String className = clazz.getSimpleName();
            String controllerName = className.substring(0, className.lastIndexOf("Controller")).toLowerCase();
            System.out.println("加载控制器--------->" + controllerName);
            try {
                urlControllersMap.put(controllerName, clazz.newInstance());
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            }
        }
        return urlControllersMap;
    }

    /**
     * @param classSet
     * @param packagePath
     * @param packageName
     * @throws ClassNotFoundException
     */
    private void addClass(Set<Class<?>> classSet, String packagePath, String packageName) throws ClassNotFoundException {
        File[] files = new File(packagePath).listFiles(pathname -> (pathname.isFile() && pathname.getName().endsWith(".class")) || pathname.isDirectory());
        for (File file : files) {
            if (file.isFile()) {//如果是文件则直接将文件加载之后放入classSet之中
                String fileName = file.getName().substring(0, file.getName().lastIndexOf("."));
                if (packageName != null && !packageName.equals("")) {
                    fileName = packageName + "." + fileName;
                }
                Class<?> clazz = loadClass(fileName, false);
                classSet.add(clazz);
            } else {
                String pathName = file.getName();
                if (packageName != null && !packageName.equals("")) {
                    packagePath = packagePath + "/" + pathName;
                    packageName = packageName + "." + pathName;
                }
                addClass(classSet, packagePath, packageName);
            }
        }
    }

    /**
     * 通过文件名称加载类
     *
     * @param fileName
     * @param b
     * @return
     * @throws ClassNotFoundException
     */
    private Class<?> loadClass(String fileName, boolean b) throws ClassNotFoundException {
        Class<?> clazz = null;
        try {
            clazz = Class.forName(fileName, b, getClass().getClassLoader());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return clazz;
    }
}

```

​		整个代码中可能会产生疑惑的地方可能有下面这个地方：

```java
Enumeration<URL> resources = ControllerLoader.class.getClassLoader().getResources(packageName.replaceAll("\\.", "/"));
```

​		首先我们看这个Enumeration，这其实就是以前版本的迭代器，当然现在用的比较多的是Iterator，之所以用Enumeration算是一个历史遗留问题。但是问题不大，我们把他当作迭代器用就可以了。

​		但是我们这里又将面临一个问题，我们有的时候并不像加载一个完整的对象，我们只想要调用一个方法。

​		其中Spring最明显的代表就是，你输入一个URL，通过与@RequestMapping的匹配，但会调用被该@RequestMapping修饰的方法。

​		我们如何想Spring那样通过URL获取所需要执行的对象以及它的方法，并且调用呢？

​		接下来让我们看看我们主程序是如何实现的：

```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
@WebServlet("/*")
public class DispatcherServlet extends HttpServlet{
    private Map<String,Object> urlControllerMaps =new HashMap<>();

    /**
     * 初始化controller的map实例化容器
     * @throws ServletException
     */
    @Override
    public void init() throws ServletException {
        ControllerLoader loader=new ControllerLoader();
        urlControllerMaps=loader.load();
    }
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String[] uri={"index","index"};

        //reqUri得到的第一个数字为请求的servlet的名字，第二个参数所要调用的方法
        System.out.println("reqUri-------------->"+req.getRequestURI().substring(1));
        String[] reqUri=req.getRequestURI().substring(1).split("/");

        if (reqUri.length>0){
            for (int i = 0; i < 2&&i<reqUri.length; i++) {
                uri[i]=reqUri[i];
            }
        }

        String controllerName=uri[0].toLowerCase();
        System.out.println("你所要请求的控制器名为-------->"+controllerName);

        String methodName=uri[1].toLowerCase();
        System.out.println("你所要请求的方法名为-------->"+methodName);

        //从urlControllerMaps中得到控制器的实例化对象，然后得到下属的所有方法
        Object object=urlControllerMaps.get(controllerName)==null?urlControllerMaps.get("index"):urlControllerMaps.get(controllerName);
        Method[] methods=object.getClass().getDeclaredMethods();
        //遍历控制器下的所有方法，将req的uri中指定的方法进行实例化
        for (Method method:methods){
            if (method.getName().equalsIgnoreCase(methodName)){
                try{
                    method.invoke(object,req,resp);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

​		至于这里所提到的IndexController类的index方法，很明显就是一个默认页面和默认方法，我们来看一看：

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class IndexController {
    public void index(HttpServletRequest request, HttpServletResponse response){
        try{
            response.getWriter().print("hello world");
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}

```

​		同时我们再写一个TestController和它的test方法来测试一下：

```java
package org.wuneng.ioc.workone.controller;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class TestController {
    public void test(HttpServletRequest request, HttpServletResponse response){
        try{
            response.getWriter().print("IOC");
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

![123](.\批注 2019-10-13 164057.png)

​		测试成功了，让我们再看一看Test：

![234](.\批注 2019-10-13 164307.png)

​		也在预料之中。

​		但是我们又回到了上面曾经提出过的一个问题，那就是：我们希望只加载我们**标记**加载的对象，而这里我们希望只加载我们标记加载的对象**以及被标记的方法**。

​		这意味着，即使是标记的类，也只有被标记的方法才可以被调用。

​		那么我们如何给方法加上标记？

​		我们之前给类加上标记的方式是让类加上标记接口，但是很明显，方法并不能加接口。

​		但是了解Spring的都应该知道（事实上我之前已经提到过了），Spring的方法是加注解。

​		在说这个之前，我们先来了解一下注解。

## 关于注解

###元注解

​	元注解是可以注解到注解上的注释，或者说，元注解是一种基本注解，但是他能够应用到其他的注解上面。

- @Retention
- @Documented
- @Target
- @Inherited
- @Repeatable

####@Retention

​	Retention意为保留期，当@Retention应用到一个注解上的时候，它解释说明了这个注解的存活时间。

- RetentionPolicy.SOURCE 注解只在源码阶段保留，在编译器进行编译时它将被丢弃忽视。
- RetentionPolicy.CLASS 注解只被保留到编译进行的时候，它并不会被加载到 JVM 中。
- RetentionPolicy.RUNTIME 注解可以保留到程序运行的时候，它会被加载进入到 JVM 中，所以在程序运行时可以获取到它们。

####@Documented

​	它的作用是能将注解中的元素包含到Javadoc中去。

####@Target

​	@Target制定了注解运用的地方

​	当一个注解被@Target注解时，这个注解就被限定了运用的场景。

- ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
- ElementType.CONSTRUCTOR 可以给构造方法进行注解
- ElementType.FIELD可以给属性进行注解
- ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
- ElementType.METHOD 可以给方法进行注解
- ElementType.PACKAGE 可以给一个包进行注解
- ElementType.PARAMETER 可以给一个方法内的参数进行注解
- ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举

####@Inherited

​	Inherited是继承的意思，他是说如果一个超类被@Inherited著结果的注解进行注释的话，那么如果它的子类没有被任何注解应用的话，那么这个子类就继承了超类的注解。

####@Repeatable

​	Repeatable是可重复的意思，@Repeatable是1.8之后才加入的。它的主要作用是给一个对象上加多个同类的注解

```java
@interface Persons {
	Person[]  value();
}

@Repeatable(Persons.class)
@interface Person{
	String role() default "";
}

@Person(role="artist")
@Person(role="coder")
@Person(role="PM")
public class SuperMan{
	
}
```

###注解的属性

​	注解的属性也叫做成员变量，注解只有成员变量，没有方法。注解的成员变量在注解的定义中以“无形参的方法”形式来声明，其方法名定义了该成员变量的名字，其返回值定义了该成员变量的类型。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
	int id();
	String msg() default "hello world";
}
```

​	比如说上面的注解中就拥有id和msg两个属性，在使用的时候需要对其赋值。

​	注解中的属性可以具有默认值，默认值需要用default关键值指定。就像上面缩写的hello world。

###注解与反射

​	注解可以通过反射进行获取，比如下列的Class对象的方法

```java
//判断类是否应用了某个注解
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
//获取Annotation对象
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
//获取类的所有Annotation对象
public Annotation[] getAnnotations() {}
```

## 说回之前

​		好了，在我们对注解有了一定的了解之后，我们开始处理之前我们所提出的问题：如何给方法加标志。事实上这个问题的解决已经很简单了。

​		自己写一个注解，标上也给元注解@Target(ElementType.METHOD)，把这个注解往方法上一加，大功告成。

​		众所周知，Spring还有一个非常常用的注解是@Autowried，可以将指定类的对象注入到Cotroller里面。那么我们该用什么方法去注入呢？

​		还是借助于反射，通过Field对象设置Cotroller的属性就可以解决问题。就像Setter一样。

​		康康下面的例子：

```java
import org.wuneng.ioc.worktwo.annotation.Component;
import org.wuneng.ioc.worktwo.annotation.Controller;

import java.io.File;
import java.io.FileFilter;
import java.io.IOException;
import java.net.URL;
import java.util.Enumeration;
import java.util.HashSet;
import java.util.Set;

/**
 * 类加载器
 * 加载worktwo下的所有类和方法
 */
public class ClassLoader {
    private static final String packageName = "org.wuneng.ioc.worktwo";

    private Set<Class<?>> classSet;
    private Set<Class<?>> controllerSet;
    private Set<Class<?>> componentSet;

    public ClassLoader() {
        load();
    }

    /**
     * 通过packageName调用下方的其他方法加载所有文件
     */
    private void load() {
        classSet = new HashSet<Class<?>>();
        try {
            Enumeration<URL> resouces = Thread.currentThread().getContextClassLoader().getResources(packageName.replaceAll("\\.", "/"));
            if (!resouces.hasMoreElements()){
                System.out.println(packageName.replace("\\.", "/")+"加载失败");
            }else{
                System.out.println(packageName.replace("\\.", "/")+"加载成功");
            }
            while ((resouces.hasMoreElements())) {
                URL url = resouces.nextElement();
                String protocol = url.getProtocol();
                if (protocol.equalsIgnoreCase("file")) {
                    String packagePath = url.getPath();
                    loadClass(classSet, packageName, packagePath);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        //将带有标志的类放到指定的Set中去
        loadComponentSet();
        loadControllerSet();
    }

    public void setControllerSet(Set<Class<?>> controllerSet) {
        this.controllerSet = controllerSet;
    }

    public void setComponentSet(Set<Class<?>> componentSet) {
        this.componentSet = componentSet;
    }

    public Set<Class<?>> getControllerSet() {
        return controllerSet;
    }

    public Set<Class<?>> getComponentSet() {
        return componentSet;
    }

    /**
     * 加载类的方法，像classset中加载类，三个参数，一个set集合，packagePath用于加载路径下的所有file类，packageName用于和file的name组合得到class类
     * @param classSet
     * @param packageName
     * @param packagePath
     */
    private void loadClass(Set<Class<?>> classSet, String packageName, String packagePath) {
        final File[] files = new File(packagePath).listFiles(new FileFilter() {

            public boolean accept(File pathname) {//只会加载class和文件夹
                return (pathname.isFile() && pathname.getName().endsWith(".class")) || pathname.isDirectory();
            }
        });
        for (File file : files) {
            String fileName = file.getName();
            if (file.isFile()) {//如果是文件则直接使用getClass加载
                if (packageName != null && !packageName.equals("")) {
                    fileName = packageName  + "." +  fileName.substring(0, fileName.lastIndexOf("."));;
                }
                Class<?> clazz = getClass(fileName);
                classSet.add(clazz);
            } else {//如果是文件夹，就将文件名称作为新的路径，递归本方法加载文件名下所有文件
                String subPackageName = fileName;
                System.out.println("\n"+"<----------"+fileName+"文件夹开始加载---------->");
                if (packageName != null && !packageName.equals("")) {
                    subPackageName = packageName + "." + subPackageName;
                }
                String subPackagePath = fileName;
                if (packagePath != null && !packagePath.equals("")) {
                    subPackagePath = packagePath + "/" + subPackagePath;
                }
                loadClass(classSet, subPackageName, subPackagePath);
            }
        }
    }

    /**
     * 调用forName方法通过文件路径名加载类
     * @param fileName
     * @return
     */
    private Class<?> getClass(String fileName) {
        Class<?> clazz = null;
        try {
            clazz = Class.forName(fileName);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return clazz;
    }

    /**
     * 从classSet中将带有component注释的.class文件取出来放入ComponentSet中
     */
    private void loadComponentSet() {
        componentSet = new HashSet<Class<?>>();
        if (classSet != null) {
            for (Class<?> clazz : classSet) {
                if (clazz.getAnnotation(Component.class) != null) {
                    componentSet.add(clazz);
                    System.out.println("已经将"+clazz.getName()+"加载到componentSet中");
                }
            }
        }
    }

    /**
     * 同上加载所有控制器
     */
    private void loadControllerSet() {
        controllerSet = new HashSet<Class<?>>();
        if (classSet != null) {
            for (Class<?> clazz : classSet) {
                if (clazz.getAnnotation(Controller.class) != null) {
                    System.out.println("准备将"+clazz.getName()+"加载到controllerSet中");
                    controllerSet.add(clazz);
                    System.out.println("已经将"+clazz.getName()+"加载到controllerSet中");
                }
            }
        }
    }
}
```

​		上面的代码通过递归调用扫描了包下所有的.class文件并将它们加载到了Set<Class<?>> classSet中，之后遍历这个classSet将含有指定注解的类放到指定的Set集合中。

​		我们先来看一下这几个指定注解：

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {

}
```

```java
package org.wuneng.ioc.worktwo.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Controller {
}

```

​		并没有什么特殊的地方，通过之前对于注解的了解应该明白这来俩注解就只是简单的给类做一个标记而已。

​		再来看看我们的工厂类的详细实现：

```java
import org.wuneng.ioc.worktwo.annotation.Autowried;
import org.wuneng.ioc.worktwo.annotation.RequestMapping;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

public class BeanFactory {
    private Map<Class<?>, Object> controllers;
    private Map<Class<?>, Object> components;
    private Map<String, Method> handlers;

    private ClassLoader classLoader;
    /**
     * 初始化上面那三个map
     * @param classLoader
     */
    public BeanFactory(ClassLoader classLoader) {
        this.classLoader = classLoader;
        initComponents();
        initControllersAndHandlers();
    }

    private void initControllersAndHandlers() {
        System.out.println("<----------BeanFactory开始实例化controllers以及其方法handlers---------->");
        Set<Class<?>> controllerSet = classLoader.getControllerSet();//先得到控制器的类
        controllers = new HashMap<>();
        handlers = new HashMap<>();
        for (Class<?> clazz : controllerSet) {//遍历所有控制器类
            try {
                String baseUri = clazz.getAnnotation(RequestMapping.class) != null ? clazz.getAnnotation(RequestMapping.class).value() : "";
                //将控制器类名下所有的有RequestMapping注释的类的名称
                Object controller =clazz.newInstance();
                //实例化控制器类
                Field[] fields=clazz.getDeclaredFields();
                //得到控制器名下的子类的属性
                for (Field field:fields){
                    if (field.getAnnotation(Autowried.class)!=null){
                        //遍历控制器属性中含有Autowried注释的属性
                        Class<?> fieldClazz =field.getType();
                        //得到属性的类
                        Object fieldValue =components.get(fieldClazz);
                        //从components这个map中得到属性的实例化对象
                        if (field.getAnnotation(Autowried.class)!=null){
                            field.setAccessible(true);
                            //设置属性的权限，如果是private也可以再之后更改属性
                        }
                        field.set(controller,fieldValue);//将控制器类的属性进行实例化
                    }
                }
                Method[] methods=clazz.getDeclaredMethods();
                // 得到控制器下的所有方法,
                // 让后将方法名和方法封装到handlers的集合中
                for (Method method:methods){
                    RequestMapping requestMapping=method.getAnnotation(RequestMapping.class);
                    //获得方法中的@RequestMapping中的参数，一个String，一个RequestMethod(获取请求方式)
                    if (requestMapping!=null){
                    //将String和RequestMethod中的请求方式用‘：’进行拼接，容易到DispatcherServlet中被检索
                        String requestUri = requestMapping.method().name() + ":" + baseUri + requestMapping.value();
                        handlers.put(requestUri,method);
                        System.out.println("已加载到方法名-------------->"+requestUri);
                    }
                }
                controllers.put(clazz,controller);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            }
        }
    }

    public Map<Class<?>, Object> getControllers() {
        return controllers;
    }

    public Map<String, Method> getHandlers() {
        return handlers;
    }

    /**
     * 通过classLoader得到componentSet，实例化components，
     * 然后将class类和其对应的实例化对象封装到components这个map容器中
     */
    private void initComponents() {
        Set<Class<?>> componentSet = classLoader.getComponentSet();
        components = new HashMap<>();
        for (Class<?> clazz : componentSet) {
            try {
                Object object = clazz.newInstance();
                components.put(clazz, object);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InstantiationException e) {
                e.printStackTrace();
            }
        }

    }
}
```

​		通过这个工厂模式，我们就实例化了指定类，并将所需的类通过Field属性设置的方法将指定对象注入了进去，这样我们的BeanFactroy就存储了我们运行时所需要的完整对象。

​		与此同时，我们还将Method对象缓存了下来，也节省了遍历Controller获得Method的过程。皆大欢喜。

​		至于@Autowried和@RequestMapping的实现其实也是十分简单，不言自明的：

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.FIELD,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
    String value();
    RequestMethod method() default RequestMethod.GET;
}
```

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowried {
}
```

​		至于RequestMethod，也只不过是一个简单的枚举类：

```java
public enum RequestMethod {
    GET,POST
}
```

​		我们来写两个测试类来康康效果如何

```java
import org.wuneng.ioc.worktwo.annotation.Component;

@Component
public class World {
    public String test(){
        return "hhhhhhhh";
    }
}
```

```java
package org.wuneng.ioc.worktwo.controller;

import org.wuneng.ioc.worktwo.annotation.Autowried;
import org.wuneng.ioc.worktwo.annotation.Controller;
import org.wuneng.ioc.worktwo.annotation.RequestMapping;
import org.wuneng.ioc.worktwo.annotation.RequestMethod;
import org.wuneng.ioc.worktwo.component.World;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Controller
public class IndexController {
    @Autowried
    World world;

    @RequestMapping(value = "/", method = RequestMethod.GET)
    public void test(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.getWriter().print(world.test());
    }
}
```

​		接下来是比较关键的负责转发所有请求的DispatcherServlet：

```java
package org.wuneng.ioc.worktwo.core;

import javax.servlet.GenericServlet;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Map;

@WebServlet("/*")
public class DispatcherServlet extends GenericServlet {
    private Map<Class<?>,Object> controllers;
    private Map<String,Method> handlers;

    /**
     * 初始化所有控制器对象和方法的实例化map
     * @throws ServletException
     */
    @Override
    public void init() throws ServletException {
        System.out.println("DispatcherServlet开始初始化");
        ClassLoader classLoader=new ClassLoader();
        BeanFactory beanFactory=new BeanFactory(classLoader);
        controllers =beanFactory.getControllers();
        handlers =beanFactory.getHandlers();
    }
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
        //第一步，转化req和resp
        HttpServletRequest request= (HttpServletRequest) servletRequest;
        HttpServletResponse response= (HttpServletResponse) servletResponse;

        //获得请求方法和req的uri并将其组合，然后通过两者进行组合，从BeanFactory中
        //早已将默认的方法通过这种方法组合，这样就可以得到控制器中的方法
        System.out.println("request.getMethod:"+request.getMethod()+
                "\trequest.getRequestURI:"+request.getRequestURI());
        String handlerKey=request.getMethod()+":"+request.getRequestURI();
        Method method =handlers.get(handlerKey);
        if (method!=null) {
            System.out.println("所寻找到的方法为----------->" + method.getName());
        }else {
            System.out.println("未找到方法名"+handlerKey);
        }
        //通过方法找到方法所属的控制器对象
        Object controller=controllers.get(method.getDeclaringClass());

        System.out.println("所寻找到的控制器为--------->"+controller);

        if (controller!=null){
            try{//调用方法处理请求
                method.invoke(controller,request,response);
                System.out.println(method.getName()+"调用成功");
            } catch (IllegalAccessException |InvocationTargetException e){
                Throwable cause = e.getCause();
                if (cause instanceof  IOException){
                    IOException exception= (IOException) cause;
                    exception.printStackTrace();
                }
                e.printStackTrace();
            }
        }
    }
}

```

​		其实就是利用BeanFactory，通过传入的URL获得指定的对象以及Method，调用Method的invoke方法就完成了。

​		让我们跑起来实验一下：

![hhhhhhh](.\捕获.PNG)

​		预料之中的结果，很开心。

​		那么让我们看看控制台打印了那么？

```java
DispatcherServlet开始初始化
org.wuneng.ioc.worktwo加载成功

<----------annotation文件夹开始加载---------->

<----------component文件夹开始加载---------->

<----------controller文件夹开始加载---------->

<----------core文件夹开始加载---------->
已经将org.wuneng.ioc.worktwo.component.World加载到componentSet中
准备将org.wuneng.ioc.worktwo.controller.IndexController加载到controllerSet中
已经将org.wuneng.ioc.worktwo.controller.IndexController加载到controllerSet中
<----------BeanFactory开始实例化controllers以及其方法handlers---------->
已加载到方法名-------------->GET:/
request.getMethod:GET	request.getRequestURI:/
所寻找到的方法为----------->test
所寻找到的控制器为--------->org.wuneng.ioc.worktwo.controller.IndexController@1a9d989b
test调用成功
request.getMethod:GET	request.getRequestURI:/
所寻找到的方法为----------->test
所寻找到的控制器为--------->org.wuneng.ioc.worktwo.controller.IndexController@1a9d989b
test调用成功
request.getMethod:GET	request.getRequestURI:/
所寻找到的方法为----------->test
所寻找到的控制器为--------->org.wuneng.ioc.worktwo.controller.IndexController@1a9d989b
test调用成功
request.getMethod:GET	request.getRequestURI:/favicon.ico
未找到方法名GET:/favicon.ico
```

​		我相信根据这个控制台的输出和程序，大家大概是可以理清这个IOC框架的原理了。

## 还有更多

### 标记

​		我们在之前提到了一个概念，就是标记。标记的方法有两种，一种是标记接口，一种是注解。那么我们来在这里探讨一下我们什么时候应该使用标记接口，什么时候应该使用注解？关于标记

​		我们在之前提到了一个概念，就是标记。标记的方法有两种，一种是标记接口，一种是注解。那么我们来在这里探讨一下我们什么时候应该使用标记接口，什么时候应该使用注解？

​		首先，标记接口定义了一个由标记类实例实现的类型，而注解标记不会。这意味着标记接口类型的存在允许在编译时捕获错误，如果使用标记注解，那么知道运行时才能捕获错误。

​		标记接口对于标记注解的另一个优点在于可以更为精确的定位目标。如果使用ElementType.TYPE来声明注解类型，那么这个接口就可以适用于任何类和接口。但是标记接口并没有这样的顾虑。

​		显然，如果标记适用于除了类或者接口以外的任何程序元素，就必须使用注解，因为只能使用类和接口来实现或者扩展接口。如果标记仅仅只适用于类和接口，那么就应该使用标记接口而不是注解。这将带来编译时检查的好处。

​		