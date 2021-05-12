#### 背景

SpringBoot启动方式有两种，

方式一：使用内置的tomcat容器

   默认的application启动，在创建项目时自动生成application启动类，直接run执行即可。

方式二：使用外置的tomcat启动

   默认的启动类要继承SpringBootServletInitiailzer类，并复写configure()方法。



#### 步骤

###### 注意事项

```
1.必须创建war项目，需要创建好web项目的目录。
2.嵌入式Tomcat依赖scope指定provided。
3.编写SpringBootServletInitializer类子类,并重写configure方法。
```

###### 步骤1 创建SpringBoot项目

![](https://www.yunsom.com/storage/api/file/200014202102201200580-6231b4068a)

![](https://www.yunsom.com/storage/api/file/200014202102201201280-4e7197a57f)

![](https://www.yunsom.com/storage/api/file/200014202102201209320-18f6a5cf3a)

![](https://www.yunsom.com/storage/api/file/200014202102201202170-50963b442f)

![](https://www.yunsom.com/storage/api/file/200014202102201205580-637b65a3f1)

###### 步骤2 添加本地tomcat并进行配置

![](https://img-blog.csdn.net/20180827110849991?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbnl1YW4xOTkz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![](https://img-blog.csdn.net/20180827110917298?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbnl1YW4xOTkz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

###### 步骤3 启动服务器



#### 原理分析

###### 1 **SpringBootServletInitializer**的执行过程

嵌入式Servlet容器默认是不支持jsp。

简单来说就是通过*SpringApplicationBuilder*构建并封装SpringApplication对象，并最终调用SpringApplication的run方法的过程。

spring boot就是为了简化开发的，也就是用注解的方式取代了传统的xml配置。

SpringBootServletInitializer就是原有的web.xml文件的替代。



###### 2 jar包和war包启动方式的区别

  jar包:执行SpringBootApplication的run方法,启动IOC容器,然后通过调用重写的refresh方法创建嵌入式Servlet容器

  war包: 先是启动Servlet服务器,服务器启动Springboot应用(springBootServletInitizer),然后启动IOC容器



```
SpringServletContainerInitializer implements ServletContainerInitializer
SpringBootServletInitializer implements WebApplicationInitializer 
```



Tomcat容器启动后根据Servlet3.0规范SPI机制，会实例化SpringServletContainerInitializer，配合@HandlesTypes({WebApplicationInitializer.class})这个注解，通过收集classpath下实现了WebApplicationInitializer接口的非抽象实现类，作为的方法参数调用onStartup。

SpringBootServletInitializer子类（WebApplicationInitializer接口实现类之一）实例执行onStartup方法的时候会通过createRootApplicationContext方法来执行run方法，接下来的过程就和以jar包形式启动的应用的run过程一样了，在内部会创建IOC容器并返回，只是以war包形式的应用在创建IOC容器过程中没有创建Servlet容器的过程（因为pom文件中排除了相关嵌入式容器，推断出的应用类型为`NONE`而非`SERVLET`）。