---
title: 【SpringBoot】SpringBoot 自动装配原理
date: 2020-03-08 15:00:59
tags: 
  - SpringBoot
  - Spring Starter
  - 自动装配
categories:
  - Java
  - SpringBoot

---

SpringBoot中很酷的一个功能是自动装配

装配的对象是依赖，也即Spring中的Bean，而所谓的自动装配，就是尽量不需要人工配置Bean信息，比如说在没有SpringBoot之前，我们装配一个Spring MVC，需要在配置文件中配置DispatcherServlet、视图解析器等等的依赖，而在SpringBoot中则不需要，只需要引入一个Starter就能做到

原理呢，就是通过某种形式来实现自动加载依赖，而不再通过程序员配置的形式来注入

这篇文章，就来详细分析SpringBoot中是自动装配的原理

<!--more-->

# 【SpringBoot】SpringBoot 自动装配原理

## 一个简单的Starter

SpringBoot中的Starter是一中模块机制，通过引入该模块，就能实现该模块相关Bean的自动注入，为了弄清楚其原理，这里我们先写一个简单的Starter

新建一个maven项目，引入如下依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
        <version>2.2.4.RELEASE</version>
    </dependency>
</dependencies>
```

然后新建一个配置类

```java
@Configuration
@EnableConfigurationProperties
public class HelloServiceAutoConfig {

    @ConditionalOnMissingBean
    @Bean("helloService")
    public HelloService helloService() {
        return new HelloService();
    }
}
```

上面有几个注解

- `@Configuration`，是将该Bean声明为配置Bean
- `@EnableConfigurationProperties`，是开启自动配置配置文件功能
- `@ConditionalOnMissingBean`，用于来实现条件注册Bean，当没有该Bean时才注册
- `@Bean`，表示该方法返回的对象需要注册到IoC容器中

``` java
@ConfigurationProperties(prefix = "cn.xuhuanfeng.hello")
public class HelloService {

    private String prefix;
    private String suffix;

    // 省了set、get方法
    public String sayHello(String name) {
        return prefix + " " + name + " " + suffix;
    }
}
```

上面的`@ConfigurationProperties`是用于将配置文件中的配置自动注入到Bean中，如里头的：`prefix`、`suffix`等信息

之后，在`resources`目录中新建一个`META-INF`目录，并且在里头建一个名为：`spring.factories`的文本文件，内容如下

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
cn.xuhuanfeng.config.HelloServiceAutoConfig
```

键是：`org.springframework.boot.autoconfigure.EnableAutoConfiguration`，值则是上面的配置类的全路径名称

编译，打包，然后安装到本地仓库中：`mvn install`

此时，在本地就可以引用该模块了，可以建一个SpringBoot项目测试一下，这里就不演示了

## 为什么是Starter

为什么需要打包成Starter呢？不能直接打包成一个普通的jar包，然后标注上Spring的Bean注解如：`@Component`，然后通过Spring的组件扫描机制自动装载进去吗？

很遗憾，很多时候是不行的，原因在于，Spring的组件扫描机制是基于包路径的，也就是说，如果引入的jar包的包路径不符合类扫描路径的时候，是不会被Spring自动发现的，此时，又回到了手工创建Bean的时代了

增加包扫描路径，可行，但是不太现实，毕竟依赖包那么多，写不完的

## Starter是如何工作的

如果之前接触过SPI机制，那么从上面的`spring.factories`文件中，大概是可以猜得出来，Starter的运行原理了，对的，Starter通过类似于SPI的机制来实现配置类的加载，然后通过配置类加载来加载对应的Bean，从而实现自动装配的原理的

下面详细分析具体的实现细节

一个最基础的SpringBoot应用大概如下所示

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

其中，最重要的是`@SpringBootApplication`注解，点击去可以看到如下信息：

```java
// ....省略无关注解
@EnableAutoConfiguration
public @interface SpringBootApplication {
    // ....
}
```

上面省略了一些不相干的注解，只留下了`@EnableAutoConfiguration`，该注解是实现Starter核心的核心

```java
// ....
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ....
}
```

这里，最重要的是：`@Import`注解，以及其引入的：`AutoConfigurationImportSelector`

`@Import`JavaConfig项目中的一个组件，目的是用于将对应的bean添加到容器中，该注解可以支持以下几种类型的值

- 普通的类，这里的普通是指不承担其他的功能
- 配置类：也就是`@Configuration`标注的类
- ImportSelector：导入选择器的实现类，容器会加载其方法返回的bean
- ImportBeanDefinitionRegistrar：beanDefinitionRegistrar的实现类

这里的`AutoConfigurationImportSelector`就是`ImportSelector`的实现，从下面的类图中可以看出

![AutoConfigurationImportSelector.png](http://ww1.sinaimg.cn/large/b162e9f4gy1gcmhwon8jfj20xg06iwem.jpg)

`ImportSelector`中只有一个方法：`String[] selectImports(AnnotationMetadata importingClassMetadata)`，容器会将该方法返回的全类名类加载到容器中

所以我们直接看`AutoConfigurationImportSelector`的`selectImports`方法即可

> AutoConfigurationImportSelector#selectImports

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // ....
    // 这一行代码是关键
    AutoConfigurationEntry autoConfigurationEntry = 
        getAutoConfigurationEntry(autoConfigurationMetadata,annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

> getAutoConfigurationEntry

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(
      AutoConfigurationMetadata autoConfigurationMetadata,
      AnnotationMetadata annotationMetadata) {
   
   // ...
   // 这一行是关键
   List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
   // 后面的可以不看了
   configurations = removeDuplicates(configurations);
   Set<String> exclusions = getExclusions(annotationMetadata, attributes);
   checkExcludedClasses(configurations, exclusions);
   configurations.removeAll(exclusions);
   configurations = filter(configurations, autoConfigurationMetadata);
   fireAutoConfigurationImportEvents(configurations, exclusions);
   return new AutoConfigurationEntry(configurations, exclusions);
}
```

> getCandidateConfigurations

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   // 这一行是关键
   List<String> configurations = 
       SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
         getBeanClassLoader());
    
   Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
         + "are using a custom packaging, make sure that file is correct.");
   return configurations;
}
```

看到这里有木有一点点惊喜：`SpringFactoriesLoader`，上面我们写的文件是：`spring.factories`

SpringFactoriesLoader是Spring中一个通用的加载器，用于加载指定目录文件：`FACTORIES_RESOURCE_LOCATION`，也就是：`"META-INF/spring.factories"`中的配置信息

其中的方法：`loadFactoryNames`可以根据类型名称来获取指定配置项的内容

上面代码中的`getSpringFactoriesLoaderFactoryClass()`返回值是：`EnableAutoConfiguration.class`

这个是前面`spring.factories`中的key：`org.springframework.boot.autoconfigure.EnableAutoConfiguration`，也就是说，加载这个key中配置的bean，也就是我们自己写的配置Bean了

接着分析下去

> loadFactoryNames

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
   String factoryTypeName = factoryType.getName();
    // 根据key来获取对应的value，这里key既是：EnableAutoConfiguration
   return loadSpringFactories(classLoader)
       .getOrDefault(factoryTypeName, Collections.emptyList());
}
```

> loadSpringFactories

```java
private static Map<String, List<String>> loadSpringFactories(
    @Nullable ClassLoader classLoader) {
    // 先从缓存中获取，如果有，直接返回
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }

   try {
       // 通过类加载加载指定的文件：META-INF/spring.factories
      Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
       /*
       * public LinkedMultiValueMap() {
	   *   this.targetMap = new LinkedHashMap<>();
	   *  }
       */
      result = new LinkedMultiValueMap<>();
       
      while (urls.hasMoreElements()) {
         // 其中的一行，即类路径中的一个：spring.factories
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
          // 将其内容作为properties加载该文件
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
          //　遍历每一项
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            // key
            String factoryTypeName = ((String) entry.getKey()).trim();
            // value，将value按照,切分
            for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                // 添加到map中，key就是对应的key
               result.add(factoryTypeName, factoryImplementationName.trim());
            }
         }
      }
      // 放入缓存
      cache.put(classLoader, result);
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load factories from location [" +
            FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```

到这里就一目了然了，SpringBoot的Starter其实是通过类似于SPI的机制

通过`@Import`导入：`AutoConfigurationImportSelector`，在`AutoConfigurationImportSelector`的`selectImports`中通过：`SpringFactoriesLoader`读取配置文件：`spring.factories`中的配置信息，然后将

对应的信息，也就是我们配置的自动配置类信息返回，并且容器会加载返回的自动配置类信息，而在自动类配置信息加载的时候，容器会解析其标注了`@Bean`的方法，从而实现了依赖的自动注入

## 实例

虽然通过前面的简单的Starter我们已经知道了怎么实现了，不过为了说明我们的分析是正确的，我们通过springboot官方的一个AutoConfiguration来验证

这里选择：`DispatcherServletAutoConfiguration`，该配置信息位于：`spring-boot-autoconfigure-2.2.4.RELEASE.jar/META-INF/spring.factories`中，点开该文件之后，搜索该文件名就可以找到了

该类的基本信息如下：

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration {
    
    // ....
    
    @Configuration(proxyBeanMethods = false)
	@Conditional(DefaultDispatcherServletCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties({ HttpProperties.class, WebMvcProperties.class })
	protected static class DispatcherServletConfiguration {
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet(HttpProperties httpProperties,
                                                   WebMvcProperties webMvcProperties) {
            // 注意这里哦
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
			// ....
			return dispatcherServlet;
		}

		@Bean
		@ConditionalOnBean(MultipartResolver.class)
		@ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
		public MultipartResolver multipartResolver(MultipartResolver resolver) {
			return resolver;
		}
	}
}
```

## 总结

这篇文章主要是分析SpringStarter的实现原理，首先是通过一个简单的Starter来说明Starter的实现方式，然后通过分析SpringBoot加载Starter的源码，详细地分析了加载的机制

