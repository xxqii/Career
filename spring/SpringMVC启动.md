### SpringMVC

HttpServletBean -> init

FrameworkServlet -> initServletBean -> initWebApplicationContext -> createWebApplicationContext -> configureAndRefreshWebApplicationContext 

AbstractApplicationContext -> refresh -> obtainFreshBeanFactory

AbstractRefreshableApplicationContext -> refreshBeanFactory

XmlWebApplicationContext -> loadBeanDefinitions



AbstractApplicationContext的refresh创建的ApplicationContext后会调用finishRefresh方法发送ContextRefreshedEvent，

FrameworkServlet中的ContextRefreshListener监听了ContextRefreshedEvent事件，调用DispatcherServlet的onRefresh方法。



DispatcherServlet -> initStrategies （initMultipartResolver、initLocaleResolver、initThemeResolver、initHandlerMapping）

initHandlerMapping：通过查找org.springframework.web.servlet.HandlerMapping的配置信息找到默认配置

BeanNameUrlHandlerMapping、**DefaultAnnotationHandlerMapping**

**AnnotationMethodHandlerAdapter**

### SpringBoot

SpringApplication 初始化类型推断（applicationContext类型、primaryResource、main方法）从spring.factories中获取initializer、listener。

run方法：初始化ApplicationContext



接口第一次调用的时候初始化servlet，执行FrameworkServlet的init方法。