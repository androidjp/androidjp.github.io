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

看看其源码以及注释（摘自 spring-context-5.1.8.RELEASE）：
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
通过上面贴出来的`@Import` 源码，可以了解到它是配合`Configuration`,`ImportSelector`以及`ImportBeanDefinitionRegistrar`来使用的，也可以把`@Import`修饰的类当做普通的Bean来使用。

这是例子：
```
class Service01 {}
```
以上代码，bean容器没有service01。
```
@Component
class Service01 {}
```
以上代码，bean容器存在bean `service01`。
```
@Component
@Bean(Service02.class)
class Service01 {}

class Service02 {}
```
以上代码，bean容器存在bean `service01` 和 `com.xxx.Service02`。
```
@Configuration
@Bean(Service02.class)
class Service01 {}

class Service02 {}
```
以上代码，bean容器存在bean `service01` 和 `com.xxx.Service02`。

```
@Bean(Service02.class)
class Service01 {}

class Service02 {}
```
以上代码，bean容器不存在bean。



```
@Configuration
@Bean(Service02.class)
class Service01 {}

class Service02 {
    Service02(String id) {}
}
```
以上代码，报错。由于`@Import`注入的bean 只能默认调用其无参构造器。


下面我们来看看`ImportBeanDefinitionRegistrar`接口的源码：
```
/**
 * Interface to be implemented by types that register additional bean definitions when
 * processing @{@link Configuration} classes. Useful when operating at the bean definition
 * level (as opposed to {@code @Bean} method/instance level) is desired or necessary.
 *
 * <p>Along with {@code @Configuration} and {@link ImportSelector}, classes of this type
 * may be provided to the @{@link Import} annotation (or may also be returned from an
 * {@code ImportSelector}).
 *
 * <p>An {@link ImportBeanDefinitionRegistrar} may implement any of the following
 * {@link org.springframework.beans.factory.Aware Aware} interfaces, and their respective
 * methods will be called prior to {@link #registerBeanDefinitions}:
 * <ul>
 * <li>{@link org.springframework.context.EnvironmentAware EnvironmentAware}</li>
 * <li>{@link org.springframework.beans.factory.BeanFactoryAware BeanFactoryAware}
 * <li>{@link org.springframework.beans.factory.BeanClassLoaderAware BeanClassLoaderAware}
 * <li>{@link org.springframework.context.ResourceLoaderAware ResourceLoaderAware}
 * </ul>
 *
 * <p>See implementations and associated unit tests for usage examples.
 *
 * @author Chris Beams
 * @since 3.1
 * @see Import
 * @see ImportSelector
 * @see Configuration
 */
public interface ImportBeanDefinitionRegistrar {

	/**
	 * Register bean definitions as necessary based on the given annotation metadata of
	 * the importing {@code @Configuration} class.
	 * <p>Note that {@link BeanDefinitionRegistryPostProcessor} types may <em>not</em> be
	 * registered here, due to lifecycle constraints related to {@code @Configuration}
	 * class processing.
	 * @param importingClassMetadata annotation metadata of the importing class
	 * @param registry current bean definition registry
	 */
	void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry);

}
```
参数：
* `importingClassMetadata`: 正在被导入的类的元数据信息；
* `registry`: 中文翻译：“登记处”，可通过它来操作IOC容器。

例子：
```
public class Service02 {}

public class Service02BeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder service02 = BeanDefinitionBuilder.rootBeanDefinition(Service02.class);
        registry.registerBeanDefinition("service00000002", service02.getBeanDefinition());
    }
}

@Configuration
@Import(Service02BeanDefinitionRegistrar.class)
public class Service01 {}
```
最终能够得到`service01` 和 `service00000002`。

实际上，其作用是：在项目启动时，全局扫描`@Import`，将其中的类进行Bean注入，注入过程中发现是`ImportBeanDefinitionRegistrar`的实现类，则会给到xxx 执行，以注入满足我们自定义注入规则的类。


下面再看看`ImportSelector`源码：
```
/**
 * Interface to be implemented by types that determine which @{@link Configuration}
 * class(es) should be imported based on a given selection criteria, usually one or
 * more annotation attributes.
 *
 * <p>An {@link ImportSelector} may implement any of the following
 * {@link org.springframework.beans.factory.Aware Aware} interfaces,
 * and their respective methods will be called prior to {@link #selectImports}:
 * <ul>
 * <li>{@link org.springframework.context.EnvironmentAware EnvironmentAware}</li>
 * <li>{@link org.springframework.beans.factory.BeanFactoryAware BeanFactoryAware}</li>
 * <li>{@link org.springframework.beans.factory.BeanClassLoaderAware BeanClassLoaderAware}</li>
 * <li>{@link org.springframework.context.ResourceLoaderAware ResourceLoaderAware}</li>
 * </ul>
 *
 * <p>{@code ImportSelector} implementations are usually processed in the same way
 * as regular {@code @Import} annotations, however, it is also possible to defer
 * selection of imports until all {@code @Configuration} classes have been processed
 * (see {@link DeferredImportSelector} for details).
 *
 * @author Chris Beams
 * @since 3.1
 * @see DeferredImportSelector
 * @see Import
 * @see ImportBeanDefinitionRegistrar
 * @see Configuration
 */
public interface ImportSelector {

	/**
	 * Select and return the names of which class(es) should be imported based on
	 * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
	 */
	String[] selectImports(AnnotationMetadata importingClassMetadata);

}
```
它的作用也是可以给我们自定义想要注入哪些bean 类型，配置起来更简单，如以下例子：
```
public class AAAImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{Service02.class.getName()};
    }
}
```

如果混用三者，加以分析Spring Boot启动上下文，其加载顺序为：
1. Application的 `main`方法中的 `SpringApplication.run(...)`；
2. 所有`@Import({xxx})`中的`ImportSelector`实现类；
3. 所有`@Import({xxx})`中的`ImportBeanDefinitionRegistrar`实现类；
4. 所有 `ApplicationContextAware` 的实现类的 `setApplicationContext(...)`方法；
5. 所有`@Import({xxx})`中的`ImportSelector`实现类中想要注入的类的无参构造器；
6. 所有`@Import({Service02.class})`中的 `Service02`类的构造器；
7. 类内部的带有`@Bean`的方法；
8. 所有`@Import({xxx})`中的`ImportBeanDefinitionRegistrar`实现类中想要注入的类的无参构造器；
9.  对下一个`@Component/@Configuration`修饰的类循环第5、6、7、8步。（注意：这里不是优先执行`@Configuration`后执行`@Component`, 而是按照类名称字典排序来顺序扫描的。）

# 跟着源码的步伐
`refresh` --> `ConfigurationClassParser`