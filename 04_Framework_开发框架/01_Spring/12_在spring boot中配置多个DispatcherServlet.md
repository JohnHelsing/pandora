> 原文
>
> https://blog.csdn.net/mfkarj/article/details/107402266

spring boot为我们自动配置了一个开箱即用的DispatcherServlet，映射路径为‘/'，但是如果项目中有多个服务，为了对不同服务进行不同的配置管理，需要对不同服务设置不同的上下文，比如开启一个DispatcherServlet专门用于rest服务。

**传统springMVC项目**

在传统的[springMVC](https://so.csdn.net/so/search?q=springMVC&spm=1001.2101.3001.7020)项目中，配置多个DispatcherServlet很轻松，在web.xml中直接配置多个就行：

| 12345678910111213 | `<``servlet``>`` ``<``servlet-name``>restServlet</``servlet-name``>`` ``<``servlet-class``>org.springframework.web.servlet.DispatcherServlet</``servlet-class``>`` ``<``init-param``>``  ``<``param-name``>contextConfigLocation</``param-name``>``  ``<``param-value``>/WEB-INF/spring2.xml</``param-value``>`` ``</``init-param``>`` ``<``load-on-startup``>1</``load-on-startup``>``</``servlet``>``<``servlet-mapping``>`` ``<``servlet-name``>ModelRestServlet</``servlet-name``>`` ``<``url-pattern``>/service/*</``url-pattern``>``</``servlet-mapping``>` |
| ----------------- | ------------------------------------------------------------ |
|                   |                                                              |

通过指定init-param中的contextConfigLocation就能够为这个DispatcherServlet指定上下文。

**spring boot中注册Servlet的两种方式**

但spring boot把tomcat都给隐藏了，更别说web.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020)了。好在提供了另外的方式配置servlet。

**1.@WebServlet****注解：**

这个是javaee的注解，是servlet3.0以后提供的。spring boot会扫描这个注解，并将这个注解注解的类注册到web容器中作为一个[servlet](https://so.csdn.net/so/search?q=servlet&spm=1001.2101.3001.7020)。

但是DispatcherServlet并不是自定义的servlet，而是框架提供的servlet，所以此方法不行。

**2.ServletRegistrationBean：**

这个bean是由spring boot提供专门来注册servlet的，可以象注册bean一样去配置servlet。

 

| 123456789101112131415161718192021 | `@Bean``public` `ServletRegistrationBean restServlet(){`` ``//注解扫描上下文`` ``AnnotationConfigWebApplicationContext applicationContext``   ``= ``new` `AnnotationConfigWebApplicationContext();`` ``//base package`` ``applicationContext.scan(``"com.jerryl.rest"``);`` ``//通过构造函数指定dispatcherServlet的上下文`` ``DispatcherServlet rest_dispatcherServlet``   ``= ``new` `DispatcherServlet(applicationContext);` ` ``//用ServletRegistrationBean包装servlet`` ``ServletRegistrationBean registrationBean``   ``= ``new` `ServletRegistrationBean(rest_dispatcherServlet);`` ``registrationBean.setLoadOnStartup(``1``);`` ``//指定urlmapping`` ``registrationBean.addUrlMappings(``"/rest/*"``);`` ``//指定name，如果不指定默认为dispatcherServlet`` ``registrationBean.setName(``"rest"``);`` ``return` `registrationBean;``}` |
| --------------------------------- | ------------------------------------------------------------ |
|                                   |                                                              |

其中需要注意的是registration.setName("rest")，这个语句很重要，因为name相同的ServletRegistrationBean只有一个会生效，也就是说，后注册的会覆盖掉name相同的ServletRegistrationBean。

如果不指定，默认为“dispatcherServlet”而spring boot提供的DispatcherServlet的name就是“dispatcherServlet”。可以在spring boot的DispatcherServletAutoConfiguration类中找到：

| 1234567891011 | ` ``public` `ServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet) {``  ``ServletRegistrationBean registration = ``new` `ServletRegistrationBean(dispatcherServlet, ``new` `String[]{``this``.serverProperties.getServletMapping()});``  ``registration.setName(``"dispatcherServlet"``);``  ``registration.setLoadOnStartup(``this``.webMvcProperties.getServlet().getLoadOnStartup());``  ``if``(``this``.multipartConfig != ``null``) {``   ``registration.setMultipartConfig(``this``.multipartConfig);``  ``}` `  ``return` `registration;`` ``}``}` |
| ------------- | ------------------------------------------------------------ |
|               |                                                              |

所以为了不覆盖默认的dispatcherServlet，必须指定一个别的名称。

同时，在自定义的DispathcerServlet绑定的配置类上，要配置报扫描的话，必须要加上@EnableWebMvc注解，不然不会扫描@Contrller注解。

package com.jerryl.rest;

| 123456 | `@Configuration``@ComponentScan``(``"org.activiti.rest.service.api"``)``@EnableWebMvc``public` `class` `Cfg_Rest {``···``}` |
| ------ | ------------------------------------------------------------ |
|        |                                                              |

**屏蔽rest服务DispatcherServlet对静态资源的访问**

最后还有一个小问题，因为想让额外配置的一个DispatcherServlet专门用于提供rest服务，但是这样配置之后，访问localhost/rest/时会访问到页面等静态资源，感觉怪怪的。
因为spring boot默认是对静态资源做了映射的，但如果不想要访问到任何静态的资源，可以修改这个映射。

两种方式：

1.在application.yml中配置：

spring:

| 123456 | `mvc:`` ``#默认为/**`` ``static-path-pattern: /**``resources:`` ``#默认为classpath:/META-INF/resources/,classpath:/resources/,classpath:/static/,classpath:/public/ 。配置多个路径，中间用逗号隔开。`` ``static-locations:` |
| ------ | ------------------------------------------------------------ |
|        |                                                              |

如果在这里配置，就会影响整个springboot项目。但默认的DispatcherServlet是需要访问静态资源的，所以不能在这里配置。

**2.继承WebMvcConfigurerAdapter的java类中配置：**

| 12345678 | `@Configuration``@EnableWebMvc``public` `class` `Cfg_View ``extends` `WebMvcConfigurerAdapter{`` ``@Override`` ``public` `void` `addResourceHandlers(ResourceHandlerRegistry registry) {``  ``registry.addResourceHandler(``"/**"``);`` ``}``}` |
| -------- | ------------------------------------------------------------ |
|          |                                                              |

重写addResourceHandlers方法，只指定resourceHandler，不指定resourceLocation，这样写就能够使其拦截掉所有对静态资源的访问，并且不会返回任何静态资源。这里的配置是可指定的，只需要让负责rest服务的DispatcherServlet的上下文扫描这个配置类就可以了。不会影响默认的DispatcherServlet。





### 1.配置类中配置

启动class中加入该方法

```
@Bean  
 public ServletRegistrationBean dispatcherRegistration(DispatcherServlet dispatcherServlet) {  
     return new ServletRegistrationBean(dispatcherServlet,"/api/*");  
}  
1234
```

### 2.配置文件中配置

```
springBoot版本:2.0.7.RELEASE
已不支持在application.properties加入server.servlet-path=/api/*
目前使用:
spring.mvc.pathmatch.use-suffix-pattern=true
server.servlet.context-path=/
server.servlet.path=*.action
123456
```

## SpringMVC执行流程

![SpringMVC执行流程.png](https://img-blog.csdnimg.cn/20190603195914928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xlYXZlNDE3,size_16,color_FFFFFF,t_70)
1用户请求DispathcerServlet。
2DispatcherServlet接受到请求，将根据请求信息交给处理器映射器。
3处理器映射器（HandlerMapping）根据用户的url请求查找匹配该url的Handler，并返回一个执行链。
4DispacherServlet再根据执行链请求处理器适配器（HandlerAdapter）。
5处理器适配器调用相应的handle进行处理。
6对应的handler处理完成后返回ModelAndVIew给处理器适配器。
7处理器适配器将接受的ModelAndView返回给DispatcherServlet。
8DispatcherServlet请求视图解析器来解析视图。
9视图解析器处理完后返回View对象给DispacherServlet。
10最后前端控制器对View进行视图渲染（即将模型数据填充至视图中）。