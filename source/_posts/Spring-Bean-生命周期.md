---
title: Spring Bean 生命周期
top: false
cover: false
toc: true
mathjax: true
date: 2024-05-25 07:19:58
password:
summary:
tags: [spring]
categories: 后端
---



## Spring Bean 生命周期

### bean的4个阶段

1. Bean的**实例化**阶段
2. Bean的**设置属性**阶段
3. Bean的**初始化**阶段
4. Bean的**销毁**阶段



{% asset_img shengmingzhouqi.jpeg 生命周期-流程图 %}


#### 实例化Bean

1. Bean Definition Loading（bean定义加载）

   Spring容器从配置文件或者注解中读取Bean定义信息，这些定义信息包括Bean的类名、作用范围（scope）、依赖关系（循环依赖问题）、初始化方法。

2. Bean Instantiation（Bean实例化）

   Spring**通过反射机制创建Bean实例**，spring调用Bean的构造函数来实例化对象。

#### 设置属性

1. Spring容器在实例化Bean之后，会进行依赖注入，将Bean所依赖的其他Bean注入到该Bean中，此过程使用setter方法或者反射直接访问字段;

   - bean依赖注入3种方式

   - @Autowired, @Resource, @Qualifier, @Bean 几个自动注入注解区别

#### 实现Aware类接口

在`postProcessBeforeInitialization`方法执行前，会执行很多Aware类型的接口，这种接口的作用是：**允许bean获取到Spring容器的特定资源或信息**。由Spring容器自动调用相应的方法，**将相关资源或信息注入到bean中**。

常见Aware接口：

1. `BeanNameAware`：
   - 方法：`setBeanName(String name)`
   - 作用：将当前bean的名称注入到bean中。
   - 执行时机：bean实例化之后，但在依赖注入之前。
2. `BeanFactoryAware`：
   - 方法：`setBeanFactory(BeanFactory beanFactory)`
   - 作用：将`BeanFactory`注入到bean中。
   - 执行时机：bean实例化之后，但在依赖注入之前。
3. `ApplicationContextAware`：
   - 方法：`setApplicationContext(ApplicationContext applicationContext)`
   - 作用：将`ApplicationContext`注入到bean中。
   - 执行时机：在`BeanFactoryAware`之后调用。
4. `ResourceLoaderAware`：
   - 方法：`setResourceLoader(ResourceLoader resourceLoader)`
   - 作用：将`ResourceLoader`注入到bean中，允许bean加载资源文件。
   - 执行时机：bean实例化之后，但在依赖注入之前。
5. `EnvironmentAware`：
   - 方法：`setEnvironment(Environment environment)`
   - 作用：将`Environment`注入到bean中，允许bean访问环境信息。
   - 执行时机：bean实例化之后，但在依赖注入之前。

#### BeanPostProcessor(初始化前后置处理)

1. 解释：

   对新创建的bean实例进行自定义修改，通过实现该接口，可以**在初始化bean前后进行额外的处理**。

2. 实现方法

   ```java
   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
   ```

   - `postProcessBeforeInitialization`: 在bean初始化之前调用，可以对bean进行处理，然后放回处理后的bean实例。
   - `postProcessAfterInitialization`: 在bean初始化之后调用，可以对bean进行后续处理，然后返回处理后的bean实例。

3. 使用场景

   - 日志记录：可以在bean初始化前后记录日志，方便调试和监控；
   - 属性检查或设置：在bean初始化之前检查或设置某些属性；
   - 代理对象创建：额可以在bean初始化之后创建代理对象，用于AOP场景；

#### Bean初始化

- 使用@PostConstruct注解， 执行顺序 > 下面两种方式

- 继承`InitializingBean`实现afterPropertiesSet()方法
- 在@Bean注解上指定初始化方法（等同于使用spring中xml方法指定init-method属性的方法，但是现在谁还用spring xml啊）

#### Bean销毁

- `@PreDestory` 注解，用于在Spring Bean被销毁之前执行清理逻辑（释放资源，关闭连接等），可以与 `@PostConstruct` 注解结合使用，确保在 Bean 初始化和销毁时执行相应的逻辑。
- 实现`DisposableBean`接口destroy()方法
- 在@Bean注解上destroyMethod属性指定销毁方法



### demo

```java
@Slf4j
@Component
public class MyBean implements BeanNameAware, ApplicationContextAware, InitializingBean {

    private String beanName;

    public MyBean() {
        log.info("1.调用构造方法实例化");
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        log.info("2.实现BeanNameAware接口：{}", name);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        log.info("3.实现ApplicationContextAware接口：{}", applicationContext);
    }


    @Override
    public void afterPropertiesSet() {
        log.info("6.使用继承InitializingBean初始化：{}", beanName);
    }


    public void init(){
        log.info("7.使用init-method初始化：{}", beanName);
    }



    @PostConstruct
    public void init1(){
        log.info("5.使用@PostConstruct初始化：{}", beanName);
    }


    @PreDestroy
    public void destroyAnno(){
        log.info("9.@PreDestroy销毁bean：{}", beanName);
    }

    public void destroy(){
        log.info("10.使用destroy-method配置销毁bean：{}", beanName);
    }
}
```

```java
@Slf4j
@Component
public class BeanPostProcessorImpl implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        log.info("4.实例：{} 初始化前", beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        log.info("8.实例：{} 初始化后", beanName);
        return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
    }

}
```

```java
@Configuration
public class BeanConfig {

    @Bean(initMethod = "init", destroyMethod = "destroy")
    public MyBean myBean() {
        return new MyBean();
    }

}
```

```java
@Slf4j
@Component
public class DestroyService implements DisposableBean {

    @Override
    public void destroy() {
        log.info("11.实现DisposableBean接口执行销毁方法");
    }
}
```

```java
2024-05-24T18:18:58.548+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 1.调用构造方法实例化
2024-05-24T18:18:58.550+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 2.实现BeanNameAware接口：myBean
2024-05-24T18:18:58.550+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 3.实现ApplicationContextAware接口：org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@7e276594, started on Fri May 24 18:18:57 CST 2024
2024-05-24T18:18:58.550+08:00  INFO 18372 --- [springs] [           main] c.l.springBean.BeanPostProcessorImpl     : 4.实例：myBean 初始化前
2024-05-24T18:18:58.551+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 5.使用@PostConstruct初始化：myBean
2024-05-24T18:18:58.551+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 6.使用继承InitializingBean初始化：myBean
2024-05-24T18:18:58.552+08:00  INFO 18372 --- [springs] [           main] com.leejie.springBean.MyBean             : 7.使用init-method初始化：myBean
2024-05-24T18:18:58.552+08:00  INFO 18372 --- [springs] [           main] c.l.springBean.BeanPostProcessorImpl     : 8.实例：myBean 初始化后
2024-05-24T18:23:15.637+08:00  INFO 18372 --- [springs] [ionShutdownHook] com.leejie.springBean.MyBean             : 9.@PreDestroy销毁bean：myBean
2024-05-24T18:23:15.638+08:00  INFO 18372 --- [springs] [ionShutdownHook] com.leejie.springBean.MyBean             : 10.使用destroy-method配置销毁bean：myBean
2024-05-24T18:23:15.638+08:00  INFO 18372 --- [springs] [ionShutdownHook] com.leejie.springBean.DestroyService     : 11.实现DisposableBean接口执行销毁方法
```

