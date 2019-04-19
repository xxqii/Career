## SpringApplication

* 构造
  主要完成primarySources、webApplicationType、mainApplicationClass、initializers、listeners等内容的加载、解析和设置；其中initializers是ApplicationContextInitializer的实现类，主要用来初始化工作。listeners是ApplicationListener的实现，监听对应的事件。initializers和listeners都实现了Ordered接口，执行有先后顺序。
* 运行
  

### 程序入口main方法

```java
public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
}
```

### SpringApplication构造方法



```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   //设置primarySource
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //从ClassPath推断应用类型
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //从Spring.factories中加载初始化类：ApplicationContextInitializer实现
   setInitializers((Collection) getSpringFactoriesInstances(
         ApplicationContextInitializer.class));
    //从spring.factories中加载事件监听器
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //从线程栈中推断出main方法所在的类
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 应用类型推断

&emsp; 判断ClassPath中是否有对应的类Class来判断应用类型，判断是根据反射`Class.forName(className, false, classLoader);`来进行的。

```java
static WebApplicationType deduceFromClasspath() {
    //ClassPath中存在DispatcherHandler，且不存在DispatcherServlet、ServletContainer、
   if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
   }
    //如果ClassPath中没有Servlet或者ConfigurableWebApplicationContext，则不是Web应用
   for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
         return WebApplicationType.NONE;
      }
   }
    //否则是Servlet应用
   return WebApplicationType.SERVLET;
}
```

### 加载BeanFactories中的类

&emsp;通过ClassLoader加载ClassPath下所有的spring.factories文件里面的的配置信息。（应用程序中和spring-jars中的所有地方）

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
      Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   // Use names and ensure unique to protect against duplicates
    //加载所有spring.factories文件中定义的配置信息，获取类的名称
   Set<String> names = new LinkedHashSet<>(
         SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //构造类型实例
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
         classLoader, args, names);
    //对实例进行排序，Spring初始化和事件监听器都会有顺序。
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}

//加载SpringFactories中定义的配置类信息
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        //ClassLoader.getResource和加载类的过程相似，通过双亲委派加载ClassPath中所有spring.factories中的配置。
        Enumeration<URL> urls = (classLoader != null ?
                                 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryClassName = ((String) entry.getKey()).trim();
                for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryClassName, factoryName.trim());
                }
            }
        }
        //将配置信息加入cache中缓存。
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}

//创建SpringFactories中定义类的实例
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
			Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {	
            //反射取Class对象
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            //获取类的构造方法
            Constructor<?> constructor = instanceClass
                .getDeclaredConstructor(parameterTypes);
            //传递参数，创建实例
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException(
                "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

记载所有的spring.factories后的内容如下所示：

![SpringApplication-001](G:\Career\images\spring\SpringApplication-001.png)

加载后的`ApplicationContextInitializer`内容如下所示（排序后）：

![SpringApplication-002](G:\Career\images\spring\SpringApplication-002.png)

各初始化类的用途如下所示：

* DelegatingApplicationContextInitializer: 加载属性中context.initializer.classes中指定的类；

* ContextIdApplicationContextInitializer：contextId初始化；

