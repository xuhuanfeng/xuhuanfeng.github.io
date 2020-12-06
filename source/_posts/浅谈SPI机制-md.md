---
title: 浅谈SPI机制.md
date: 2020-03-08 14:56:12
tags: SPI
categories: Java
---

  文章主要分析的Java中的SPI机制，SPI机制是一个基于约定的形式来加载对应的信息，实现者只需要按照约定，将对应的信息保存在指定文件中，应用启动之后就会自行去读取、发现，通过这种方式了提高了应用的可扩展性

<!--more-->

# 浅谈SPI机制

## SPI

SPI全称是Service Provider Interface，翻译过来是服务提供者接口，这个翻译其实不那么形象，理解起来也不是很好理解，至少不那么见名知意。

其实SPI是一种机制，一种类似于服务发现的机制，什么叫做服务发现呢，就是能够根据情况发现已有服务的机制，好像说了跟没说一样，对吧，下面我们逐个来理解。

首先是**服务**，英文叫做Service，服务可以理解为就是某一种或者某几种功能，比如日常生活中的医生，提供看病的服务；家政公司，提供家政服务；房产中介公司，提供，这样子的话，关于服务，应该是理清楚了。

接下来是**服务的发现**，英文是Service Discovery，理解了服务，那么服务的发现就应该很好理解了，用大白话讲就是具有某种能力，可以发现某些服务，比如生活中的房产中介公司(服务发现)，他们就能够发现很多的拥有空闲房子并且愿意出租的人(服务)。

SPI机制的作用就是服务发现，也就是说，我们有一些服务，然后通过SPI机制，就能让这些服务被需要的人所使用，而我们这些服务被发现的过程就是SPI的任务了。

说到这里，可能你还是不太理解SPI是什么，接下来我们通过具体的例子分析来理解SPI。

在JDBC4.0之前，我们使用JDBC去连接数据库的时候，通常会经过如下的步骤

1. 将对应数据库的驱动加到类路径中
2. 通过`Class.forName()`注册所要使用的驱动，如`Class.forName(com.mysql.jdbc.Driver)`
3. 使用驱动管理器`DriverManager`来获取连接
4. 后面的内容我们不关心了。

这种方式有个缺点，加载驱动是由用户来操作的，这样就很容易出现加载错驱动或者更换驱动的时候，忘记更改加载的类了。

在JDBC4.0，现在我们使用的时候，上面的第二步就不需要了，并且能够正常使用，这个就是SPI的功劳了。

接下来我们先来看下为什么不需要第二步。

熟悉反射的同学应该知道，第二步其实就是将对应的驱动类加载到虚拟机中，也就是说，现在我们没有手动加载，那么对应的驱动类是如何加载到虚拟机中的呢，我们通过`DriverManger`的源码的了解SPI是如何实现这个功能的。

> DriverManager.java

在DriverManager中，有一段静态代码(静态代码在类被加载的时候就会执行)

```java
static {
    // 在这里加载对应的驱动类
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```

接下来我们来具体看下其内容

> loadInitialDrivers()

```java
private static void loadInitialDrivers() {
    String drivers;
    try {
        // 先获取系统变量
        drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }

    // SPI机制加载驱动类
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            
            //  通过ServiceLoader.load进行查找，我们的重点也是这里，后面分析
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            // 获取迭代器，也请注意这里
            Iterator<Driver> driversIterator = loadedDrivers.iterator();

            try{
                // 遍历迭代器
                // 这里需要这么做，是因为ServiceLoader默认是延迟加载
                // 只是找到对应的class，但是不加载
                // 所以这里在调用next的时候，其实就是实例化了对应的对象了
                // 请注意这里 --------------------------------------------------------------------  1
                while(driversIterator.hasNext()) {
                    // 真正实例化的逻辑，详见后面分析
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });

    println("DriverManager.initialize: jdbc.drivers = " + drivers);

    if (drivers == null || drivers.equals("")) {
        return;
    }
    // 同时加载系统变量中找到的驱动类
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            // 由于是系统变量，所以使用系统类加载器，而不是应用类加载器
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```

从上面的代码中并没有找到对应的操作逻辑，唯一的一个突破点就是`ServiceLoader.load(Driver.class)`方法，该方法其实就是SPI的核心啦

接下来我们来分析这个类的代码(代码可能有点长哦，要有心理准备)

> ServiceLoader.java

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{
    /**
    *  由于是调用ServiceLoader.load(Driver.class)方法，所以我们先从该方法分析
    */ 
    public static <S> ServiceLoader<S> load(Class<S> service) {
        // 获取当前的上下文线程
        // 默认情况下是应用类加载器，具体的内容稍后分析
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        // 调用带加载器的加载方法
        return ServiceLoader.load(service, cl);
    }

    /**
    *  带类加载器的加载方法
    */
    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        // 只是返回一哥ServiceLoader对象，调用自己的构造函数嘛
        return new ServiceLoader<>(service, loader);
    }

    /**
    *  私有构造函数
    */
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        // 目标加载类不能为null
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        // 获取类加载器，如果cl是null，则使用系统类加载器
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        // 调用reload方法
        reload();
    }

    // 用于缓存加载的服务提供者
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 真正查找逻辑的实现
    private LazyIterator lookupIterator;

    /**
    *  reload方法
    */
    public void reload() {
        // 先清空内容
        providers.clear();
        // 初始化lookupIterator
        lookupIterator = new LazyIterator(service, loader);
    }
}
```

> LazyIterator.class

LazyIterator是ServiceLoader的私有内部类

```java
private class LazyIterator
        implements Iterator<S>
{

    Class<S> service;
    ClassLoader loader;

    /**
    *  私有构造函数，用于初始化参数
    */
    private LazyIterator(Class<S> service, ClassLoader loader) {
        this.service = service;
        this.loader = loader;
    }
}
```

到了上面的内容，其实ServiceLoader.load()方法就结束了，并没有实际上去查找具体的实现类，那么什么时候才去查找以及加载呢，还记得上面的`Iterator<Driver> driversIterator = loadedDrivers.iterator();`这一行代码吗，这一行代码用于获取一个迭代器，这里同样也没有进行加载，但是，其后面还有遍历迭代器的代码，上面标注为1的部分。

迭代器以及遍历迭代器的过程如下所示

> ServiceLoader.java

```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        // 注意这里的providers，这里就是上面提到的用于缓存
        // 已经加载的服务提供者的容器。
        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();

        // 底层其实委托给了providers
        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            // 如果没有缓存，则查找及加载
            return lookupIterator.hasNext();
        }

        // 同上
        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }
    };
}
```

上面已经分析过了，ServiceLoader.load()方法执行到LazyIterator的初始化之后就结束了，真正地查找直到调用`lookupIterator.hasNext()`才开始。

> LazyIterator.java

```java
// 希望你还记得他
private class LazyIterator
        implements Iterator<S>
{

    Class<S> service;
    ClassLoader loader;
    Enumeration<URL> configs = null;
    Iterator<String> pending = null;
    String nextName = null;

    //检查 AccessControlContext，这个我们不关系
    // 关键的核心是都调用了hasNextService()方法
    public boolean hasNext() {
        if (acc == null) {
            return hasNextService();
        } else {
            PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                public Boolean run() { return hasNextService(); }
            };
            return AccessController.doPrivileged(action, acc);
        }
    }

    private boolean hasNextService() {
        // 第一次加载
        if (nextName != null) {
            return true;
        }
        // 第一次加载
        if (configs == null) {
            try {
                // 注意这里，获取了的完整名称
                // PREFIX定义在ServiceLoader中
                // private static final String PREFIX = "META-INF/services/"
                // 这里可以看到，完整的类名称就是 META-INF/services/CLASS_FULL_NAME
                // 比如这里的 Driver.class，完整的路径就是
                //                  META-INF/services/java.sql.Driver，注意这个只是文件名，不是具体的类哈
                String fullName = PREFIX + service.getName();
                // 如果类加载器为null，则使用系统类加载器进行加载
                // 类加载会加载指定路径下的所有类
                if (loader == null)
                    configs = ClassLoader.getSystemResources(fullName);
                else // 使用传入的类加载器进行加载，其实就是应用类加载器
                    configs = loader.getResources(fullName);
            } catch (IOException x) {
                fail(service, "Error locating configuration files", x);
            }
        }
        // 如果pending为null或者没有内容，则进行加载，一次只加载一个文件的一行
        while ((pending == null) || !pending.hasNext()) {
            if (!configs.hasMoreElements()) {
                return false;
            }
            // 解析读取到的每个文件，高潮来了
            pending = parse(service, configs.nextElement());
        }
        nextName = pending.next();
        return true;
    }

    /**
    *  解析读取到的每个文件
    */
    private Iterator<String> parse(Class<?> service, URL u)
        throws ServiceConfigurationError
    {
        InputStream in = null;
        BufferedReader r = null;
        ArrayList<String> names = new ArrayList<>();
        try {
            in = u.openStream();
            // utf-8编码
            r = new BufferedReader(new InputStreamReader(in, "utf-8"));
            int lc = 1;
            // 一行一行地读取数据
            while ((lc = parseLine(service, u, r, lc, names)) >= 0);
        } catch (IOException x) {
            fail(service, "Error reading configuration file", x);
        } finally {
            try {
                if (r != null) r.close();
                if (in != null) in.close();
            } catch (IOException y) {
                fail(service, "Error closing configuration file", y);
            }
        }
        // 返回迭代器
        return names.iterator();
    }

    // 解析一行行的数据
    private int parseLine(Class<?> service, URL u, BufferedReader r, int lc,
                          List<String> names)
        throws IOException, ServiceConfigurationError
    {
        String ln = r.readLine();
        if (ln == null) {
            return -1;
        }
        // 查找是否存在#
        // 如果存在，则剪取#前面的内容
        // 目的是防止读取到#及后面的内容
        int ci = ln.indexOf('#');
        if (ci >= 0) ln = ln.substring(0, ci);
        ln = ln.trim();
        int n = ln.length();
        if (n != 0) {
            // 不能包含空格及制表符\t
            if ((ln.indexOf(' ') >= 0) || (ln.indexOf('\t') >= 0))
                fail(service, u, lc, "Illegal configuration-file syntax");
            int cp = ln.codePointAt(0);
            // 检查第一个字符是否是Java语法规范的单词
            if (!Character.isJavaIdentifierStart(cp))
                fail(service, u, lc, "Illegal provider-class name: " + ln);
            // 检查每个字符
            for (int i = Character.charCount(cp); i < n; i += Character.charCount(cp)) {
                cp = ln.codePointAt(i);
                if (!Character.isJavaIdentifierPart(cp) && (cp != '.'))
                    fail(service, u, lc, "Illegal provider-class name: " + ln);
            }
            // 如果缓存中没有，并且当前列表中也没有，则加入列表。
            if (!providers.containsKey(ln) && !names.contains(ln))
                names.add(ln);
        }
        return lc + 1;
    }

    /**
    *  上面解析完文件之后，就开始加载文件的内容了
    */
    private S nextService() {
        if (!hasNextService())
            throw new NoSuchElementException();
        String cn = nextName;
        nextName = null;
        Class<?> c = null;
        try {
            // 这一行就很熟悉啦
            c = Class.forName(cn, false, loader);
        } catch (ClassNotFoundException x) {
            fail(service,
                    "Provider " + cn + " not found");
        }
        if (!service.isAssignableFrom(c)) {
            fail(service,
                    "Provider " + cn  + " not a subtype");
        }
        try {
            // 实例化并且将其转化为对应的接口或者父类
            S p = service.cast(c.newInstance());
            // 将其放入缓存中
            providers.put(cn, p);
            // 返回当前实例
            return p;
        } catch (Throwable x) {
            fail(service,
                    "Provider " + cn + " could not be instantiated",
                    x);
        }
        throw new Error();          // This cannot happen
    }
}
```

到此，解析的步骤就完成了，在一开始的DriverManager中，我们也看到了在DriveirManager中一直在调用next方法，也就是持续地加载找到的所有的Driver的实现类了，比如MySQL的驱动类，Oracle的驱动类啦。

这个例子有点长，但我们收获还是很多，我们知道了JDBC4不用手动加载驱动类的实现原理，其实就是通过ServiceLoader去查找当前类加载器能访问到的目录下的`WEB-INF/services/FULL_CLASS_NAME`文件中的所有内容，而这些内容由一定的规范，如下

- 每行只能写一个全类名
- `# `作为注释
- 只能使用utf-8及其兼容的编码
- 每个实现类必须提供一个无参构造函数，因为是直接使用`class.newInstance()`来创建实例的嘛

由此我们也明白了SPI机制的工作原理，那么这个东西有什么用呢，其实JDBC就是个最好的例子啦，这样用户就不需要知道到底是要加载哪个实现类，一方面是简化了操作，另一方面避免了操作的错误，当然，这种一般是用于写框架之类的用途，用于向框架使用者提供更加便利的操作，比如上面的引导我看到SPI的例子，其实是来自一个RPC框架，通过SPI机制，让我们可以直接编写自定义的序列化方式，然后由框架来负责加载即可。

## SPI实战小案例

上面学习完了SPI的例子，也学习完了JDBC是如何实现的，接下来我们来通过一个小案例，来动手实践一下SPI是如何工作的。

新建一个接口，内容随便啦

> HelloServie.java

```java
public interface HelloService {
    void sayHello();
}
```

然后编写其实现类

> HelloServiceImpl.java

```java
public class HelloServiceImpl implements HelloService {
    @Override
    public void sayHello() {
        System.out.println("hello world");
    }
}
```

关键点来了，既然是学习SPI，那么我们肯定不是手动new一个实现类啦，而是通过SPI的机制来加载，如果认真地看完上面的分析，那么下面的内容应该很容易看懂啦，如果没看懂，再回去看一下啦。

1. 在实现类所在项目(这里是同个项目哈)的类路径下，如果是maven项目，则是在resources目录下

   1. 建立目录`META-INF/services`
   2. 建立文件cn.xuhuanfeng.spi.HelloService（接口的全限定名哈）

2. 内容是实现类的类名:cn.xuhuanfeng.spi.impl.HelloServiceImpl(注意这里我们直接放在同个项目，不是同个项目也可以的！！！)

3. 自定义一个加载的类，并且通过ServiceLoader.load()方法进行加载，如下所示

   ```java
   public class HelloServiceFactory {
   
       public HelloService getHelloService() {
           ServiceLoader<HelloService> load = ServiceLoader.load(HelloService.class);
           return load.iterator().next();
       }
   }
   ````

4. 测试一下，enjoy :)

5. 如果你有兴趣的话，可以尝试将实现放在另一个项目中，然后打包成jar包，再放置在测试项目的classpath中，enjoy :)

## 总结

本小节我们主要学习了SPI，主要包括了SPI是什么，JDBC4中不需要手动加载驱动类的原理，并且详细看了DriverManager中的代码实现，最后，通过一个简单的小案例来实现我们自己的SPI服务，通过这个小节，应该说，SPI的大部分内容我们是掌握了，当然，里面管理类加载器部分我们还没有学习，这里先挖个坑，后面有时间再分析一下。