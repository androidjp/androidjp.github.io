---
title: Spring Boot 深入源码 -- @Import
date: 2019-10-15 08:17:24
categories:
- Spring
tags:
- Spring Boot
---

# @Import的作用
允许通过它引入 `@Configuration` 注解的类 (java config)， 引入`ImportSelector`接口(这个比较重要， 因为要通过它去判定要引入哪些`@Configuration`) 和 `ImportBeanDefinitionRegistrar` 接口的实现，也包括 `@Component`注解的普通类。

看看其源码注释：
```
/**
 * Indicates one or more {@link Configuration @Configuration} classes to import.
 *
 * <p>Provides functionality equivalent to the {@code <import/>} element in Spring XML.
 * Allows for importing {@code @Configuration} classes, {@link ImportSelector} and
 * {@link ImportBeanDefinitionRegistrar} implementations, as well as regular component
 * classes (as of 4.2; analogous to {@link AnnotationConfigApplicationContext#register}).
 *
 * <p>{@code @Bean} definitions declared in imported {@code @Configuration} classes should be
 * accessed by using {@link org.springframework.beans.factory.annotation.Autowired @Autowired}
 * injection. Either the bean itself can be autowired, or the configuration class instance
 * declaring the bean can be autowired. The latter approach allows for explicit, IDE-friendly
 * navigation between {@code @Configuration} class methods.
 *
 * <p>May be declared at the class level or as a meta-annotation.
 *
 * <p>If XML or other non-{@code @Configuration} bean definition resources need to be
 * imported, use the {@link ImportResource @ImportResource} annotation instead.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.0
 * @see Configuration
 * @see ImportSelector
 * @see ImportResource
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

    /**
     * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
     * or regular component classes to import.
     */
    Class<?>[] value();

}
```
## 说说其背景
在Spring 3.0 以前，创建Bean可通过xml配置文件和扫描特定包下的类 来达到类注入Spring IoC容器的效果。而在Spring 3.0 之后，提供了所谓了 Java Config 的方式，也就是可以通过java代码的形式去注入类，将IOC容器里Bean的元信息以java代码的方式进行描述。我们可以通过`@Configuration`与`@Bean`这两个注解配合使用来将原来配置在xml文件里的bean通过java代码的方式进行描述。

# 从源码看@Import的处理原理
