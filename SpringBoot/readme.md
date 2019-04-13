## SpringBoot

### Spring Boot为什么突然如此备受关注与推崇呢？主要有以下几点：  

* 简化依赖管理：在Spring Boot中提供了一系列的Starter POMs，将各种功能性模块进行了划分与封装，让我们可以更容易的引入和使用，有效的避免了用户在构建传统Spring应用时维护大量依赖关系而引发的JAR冲突等问题。  
* 自动化配置：Spring Boot为每一个Starter都提供了自动化的Java配置类，用来替代我们传统Spring应用在XML中繁琐且并不太变化的Bean配置；同时借助一系列的条件注解修饰，使得我们也能轻松的替换这些自动化配置的Bean来进行扩展。  
* 嵌入式容器：除了代码组织上的优化之外，Spring Boot中支持的嵌入式容器也是一个极大的亮点（此处仿佛又听到了Josh Long的那句：“Deploy as a Jar, not a War”），借助这个特性使得Spring Boot应用的打包运行变得非常的轻量级。  
* 生产级的监控端点：spring-boot-starter-actuator的推出可以说是Spring Boot在Spring基础上的另一个重要创新，为Spring应用的工程化变得更加完美。该模块并不能帮助我们实现任何业务功能，但是却在架构运维层面给予我们更多的支持，通过该模块暴露的HTTP接口，我们可以轻松的了解和控制Spring Boot应用的运行情况。  

### 事件监听  
&emsp; ApplicationStartingEvent -> ApplicationEnvironmentPreparedEvent -> ApplicationPreparedEventListener -> ApplicationStartedEventListener -> ApplicationReadyEventListener  

* ApplicationStartingEvent: spring-boot开始启动时执行的事件；  
* ApplicationEnvironmentPreparedEvent: spring boot 对应Enviroment已经准备完毕，但此时上下文context还没有创建;  
* ApplicationPreparedEventListener: spring boot上下文context创建完成，但此时spring中的bean是没有完全加载完成的;  
* ApplicationStartedEventListener: spring-boot启动完成时执行的事件；  
* ApplicationFailedEvent: spring boot启动异常时执行事件;  
