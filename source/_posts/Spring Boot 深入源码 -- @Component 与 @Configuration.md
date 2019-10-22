---
title: 'Spring Boot 深入源码 -- @Component 与 @Configuration'
date: 2019-10-22 08:56:05
categories:
- Spring
tags:
- Spring Boot
---

> 看过笔者之前的文章 [《Spring Boot 深入源码 -- @Import》](http://jpuneng.cn/Spring%20Boot%20%E6%B7%B1%E5%85%A5%E6%BA%90%E7%A0%81%20--%20@Import/)，大家会看到，一个例子中，我用`@Configuration`和`@Component` 好像效果没有什么区别呀~ `@Import`照样生效，`@Bean`方法也能生效呀。 那请问它们两者到底有什么区别呢？原理又是什么呢？ 本文尝试带你解读一番。

# 看看他们的源码注释
## @Component
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface Component {

	/**
	 * The value may indicate a suggestion for a logical component name,
	 * to be turned into a Spring bean in case of an autodetected component.
	 * @return the suggested component name, if any (or empty String otherwise)
	 */
	String value() default "";

}
```
## @Configuration
```
/*
 * ....................
 * <h2>Constraints when authoring {@code @Configuration} classes</h2>
 *
 * <ul>
 * <li>Configuration classes must be provided as classes (i.e. not as instances returned
 * from factory methods), allowing for runtime enhancements through a generated subclass.
 * <li>Configuration classes must be non-final.
 * <li>Configuration classes must be non-local (i.e. may not be declared within a method).
 * <li>Any nested configuration classes must be declared as {@code static}.
 * <li>{@code @Bean} methods may not in turn create further configuration classes
 * (any such instances will be treated as regular beans, with their configuration
 * annotations remaining undetected).
 * </ul>
 *
 * @author Rod Johnson
 * @author Chris Beams
 * @since 3.0
 * @see Bean
 * @see Profile
 * @see Import
 * @see ImportResource
 * @see ComponentScan
 * @see Lazy
 * @see PropertySource
 * @see AnnotationConfigApplicationContext
 * @see ConfigurationClassPostProcessor
 * @see org.springframework.core.env.Environment
 * @see org.springframework.test.context.ContextConfiguration
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

	/**
	 * Explicitly specify the name of the Spring bean definition associated with the
	 * {@code @Configuration} class. If left unspecified (the common case), a bean
	 * name will be automatically generated.
	 * <p>The custom name applies only if the {@code @Configuration} class is picked
	 * up via component scanning or supplied directly to an
	 * {@link AnnotationConfigApplicationContext}. If the {@code @Configuration} class
	 * is registered as a traditional XML bean definition, the name/id of the bean
	 * element will take precedence.
	 * @return the explicit component name, if any (or empty String otherwise)
	 * @see org.springframework.beans.factory.support.DefaultBeanNameGenerator
	 */
	@AliasFor(annotation = Component.class)
	String value() default "";

}
```
从源码来看，`Configuration`本质还是`@Component`，所以，`<context:component-scan/>` 或 `@ComponentScan` 都能处理`@Configuration`注解的类。


# @Configuration 与 @Component 的区别

一句话概括：`@Configuration` 中所有带 `@Bean` 注解的方法都会被**动态代理**，因此调用该方法返回的都是同一个实例。

* `@Component`可以注解在任何类上，但是`@Configuration`使用有条件。
* `@Configuration`中所有带`@Bean`注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。
* `@Configuration`本质上还是`@Component`，所以，`<context:component-scan/>`或者`@ComponentScan`都能处理`@Configuration`注解类。
* `@Configuration`标记的类必须符合一下要求：
  * 配置类必须以类的形式提供（不能是工厂方法返回的实例），允许通过生成子类在运行时增强（cglib 动态代理）。
  * 配置类不能是 final 类（没法动态代理）。
  * 配置注解通常为了通过 `@Bean` 注解生成 Spring 容器管理的类
  * 配置类必须是非本地的（即不能在方法中声明，不能是 private）。
  * 任何嵌套配置类都必须声明为static。
  * `@Bean` 方法可能不会反过来创建进一步的配置类（也就是返回的 bean 如果带有 `@Configuration`，也不会被特殊处理，只会作为普通的 bean）。


# 加载过程
Spring 容器在启动时，会加载默认的一些 `PostProcessor`，其中就有 `ConfigurationClassPostProcessor`，这个后置处理程序专门处理带有 @Configuration 注解的类，这个程序会在 bean 定义加载完成后，在 bean 初始化前进行处理。主要处理的过程就是使用 cglib 动态代理增强类，而且是对其中带有 @Bean 注解的方法进行处理。
> 这个加载过程，其实走的代码流程 和 我们之前分析 `@Import`时走的流程是十分相似的，同样会调用 `ConfigurationClassPostProcessor` 类，只是对于`@Component`和`@Configuration`调用的方法不同。


注释如下：
Prepare the Configuration classes for servicing bean requests at runtime by replacing them with CGLIB-enhanced subclasses.
> 通过用 CGLIB 增强的子类替换 Bean 请求，为运行时的 Bean 请求提供服务准备配置类。 

在 `ConfigurationClassPostProcessor` 中的 `postProcessBeanFactory` 方法主要流程如下：
1. 传入`ConfigurableListableBeanFactory`对象，并获取其factoryId。
2. 判断这个factoryId是否已经被处理过，如果已被处理，则抛异常。
3. 如果 BeanDefinitionRegistryPostProcessor hook apparently not supported，则 Simply call processConfigurationClasses lazily at this point then，调用`processConfigBeanDefinitions`方法，其实和`@Component`注解跑的逻辑一样，去开始注入bean。
4. **调用 `enhanceConfigurationClasses` 方法**：
   ```
    /**
	 * Post-processes a BeanFactory in search of Configuration class BeanDefinitions;
	 * any candidates are then enhanced by a {@link ConfigurationClassEnhancer}.
	 * Candidate status is determined by BeanDefinition attribute metadata.
	 * @see ConfigurationClassEnhancer
	 */
	public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
				// ...........
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		if (configBeanDefs.isEmpty()) {
			// nothing to enhance -> return immediately
			return;
		}

		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			try {
				// Set enhanced subclass of the user-specified bean class
				Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
				if (configClass != null) {
					Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
					if (configClass != enhancedClass) {
						// ..............
						beanDef.setBeanClass(enhancedClass);
					}
				}
			}
			catch (Throwable ex) {
				throw new IllegalStateException("Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
			}
		}
	}
   ```
   * 第一个for循环：查找所有带有 `@Configuration` 注解的bean 定义；
   * 第二个for循环：通过 `Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);` 方法 对类进行增强；
   * 使用增强后的类 替换原有的 `beanClass`：`beanDef.setBeanClass(enhancedClass);`；
   * 之后，所有带有`@Configuration`注解的 bean 都已经变成增强的类。

我们进一步看这个`enhance()` 方法：
```
/**
 * Loads the specified class and generates a CGLIB subclass of it equipped with
 * container-aware callbacks capable of respecting scoping and other bean semantics.
 * @return the enhanced subclass
 */
public Class<?> enhance(Class<?> configClass, @Nullable ClassLoader classLoader) {
    if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
        if (logger.isDebugEnabled()) {
            logger.debug(String.format("Ignoring request to enhance %s as it has " +
                    "already been enhanced. This usually indicates that more than one " +
                    "ConfigurationClassPostProcessor has been registered (e.g. via " +
                    "<context:annotation-config>). This is harmless, but you may " +
                    "want check your configuration and remove one CCPP if possible",
                    configClass.getName()));
        }
        return configClass;
    }
    Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
    if (logger.isTraceEnabled()) {
        logger.trace(String.format("Successfully enhanced %s; enhanced class name is: %s",
                configClass.getName(), enhancedClass.getName()));
    }
    return enhancedClass;
}
```
内部调用了`newEnhancer()`方法：
```
/**
 * Creates a new CGLIB {@link Enhancer} instance.
 * 创建一个新的 CGLIB 实例（Enhancer 对象）
 */
private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(configSuperClass);
    enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
    enhancer.setUseFactory(false);
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
    enhancer.setCallbackFilter(CALLBACK_FILTER);
    enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
    return enhancer;
}
```
从上面方法我们看到， 这个`enhancer.setCallbackFilter(CALLBACK_FILTER);` 实际上是将一个CallbackFilter接口实现类设置给到这个增强类。

通过 cglib 代理的类在调用方法时，会通过 `CallbackFilter` 调用，这里的 `CALLBACK_FILTER` 如下：
```
// The callbacks to use. Note that these callbacks must be stateless.
private static final Callback[] CALLBACKS = new Callback[] {
        new BeanMethodInterceptor(),
        new BeanFactoryAwareMethodInterceptor(),
        NoOp.INSTANCE
};

private static final ConditionalCallbackFilter CALLBACK_FILTER = new ConditionalCallbackFilter(CALLBACKS);
```
其实是一个`ConditionalCallbackFilter`类对象，它的构造器传入了对象，分别是：
* `new BeanMethodInterceptor()`
  * `ConfigurationClassEnhancer`的一个静态内部类
* `new BeanFactoryAwareMethodInterceptor()`
  * `ConfigurationClassEnhancer`的一个静态内部类
* `NoOp.INSTANCE`

其中，`BeanMethodInterceptor` 匹配方法如下：
```
@Override
public boolean isMatch(Method candidateMethod) {
    // 方法有 @Bean 注解，则返回true
    return BeanAnnotationHelper.isBeanAnnotated(candidateMethod);
}

//BeanAnnotationHelper
public static boolean isBeanAnnotated(Method method) {
    // 方法有没有 @Bean 注解
    return AnnotatedElementUtils.hasAnnotation(method, Bean.class);
}
```

另一个 `BeanFactoryAwareMethodInterceptor` 匹配的方法则：
```
@Override
public boolean isMatch(Method candidateMethod) {
    return (candidateMethod.getName().equals("setBeanFactory") &&
            candidateMethod.getParameterTypes().length == 1 &&
            BeanFactory.class == candidateMethod.getParameterTypes()[0] &&
            BeanFactoryAware.class.isAssignableFrom(candidateMethod.getDeclaringClass())); // 当前类需要实现 `BeanFactoryAware`接口
}
```
当前类还需要实现 `BeanFactoryAware` 接口，上面的 `isMatch` 就是匹配的这个接口的方法。


# @Bean 注解方法执行策略
先看看一个简单例子：
```
@Configuration
public class MyBeanConfig {

    @Bean
    public Country country(){
        return new Country();
    }

    @Bean
    public UserInfo userInfo(){
        return new UserInfo(country());
    }

}
```
> 相信大多数人第一次看到上面 userInfo() 中调用 country() 时，会认为这里的 Country 和上面 @Bean 方法返回的 Country 可能不是同一个对象，因此可能会通过下面的方式来替代这种方式：
> ```
> // 或者 @Resource
> @Autowired
> private Country country;
> ```
> 实际上不需要这么做（后面会给出需要这样做的场景），直接调用 `country()` 方法返回的是同一个实例。

下面看调用 `country()` 和 `userInfo()` 方法时的逻辑。

现在我们已经知道 `@Configuration` 注解的类是如何被处理的了，现在关注上面的 `BeanMethodInterceptor`，看看带有 `@Bean` 注解的方法执行的逻辑。下面分解来看 `BeanMethodInterceptor.intercept` 方法。
```
/**
 * Enhance a {@link Bean @Bean} method to check the supplied BeanFactory for the
 * existence of this bean object.
 * @throws Throwable as a catch-all for any exception that may be thrown when invoking the
 * super implementation of the proxied method i.e., the actual {@code @Bean} method
 */
@Override
@Nullable
public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
            MethodProxy cglibMethodProxy) throws Throwable {
    // 1. 首先通过反射从增强的 Configuration 注解类中获取 beanFactory
    ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
    
    // 2. 通过方法获取 beanName，默认是方法名，可通过 @Bean 注解指定
    String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

    // 3. 确定这个bean 是否指定了代理的范围（默认下面的if 条件为 false，不会执行）
    // Determine whether this bean is a scoped-proxy
    if (BeanAnnotationHelper.isScopedProxy(beanMethod)) {
        String scopedBeanName = ScopedProxyCreator.getTargetBeanName(beanName);
        if (beanFactory.isCurrentlyInCreation(scopedBeanName)) {
            beanName = scopedBeanName;
        }
    }

    // 省略部分 Factorybean 相关代码 ...........

    // 判断当前执行的方法是否为正在执行的 @Bean 方法
    // 因为存在在 userInfo() 方法中调用 country() 方法
    // 如果 country() 也有 @Bean 注解，那么这个返回值就是 false.
    if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
        // 判断返回值类型，如果是 BeanFactoryPostProcessor 就写警告日志
        if (logger.isInfoEnabled() &&
                BeanFactoryPostProcessor.class.isAssignableFrom(beanMethod.getReturnType())) {
            logger.info(String.format("@Bean method %s.%s is non-static and returns an object " +
                            "assignable to Spring's BeanFactoryPostProcessor interface. This will " +
                            "result in a failure to process annotations such as @Autowired, " +
                            "@Resource and @PostConstruct within the method's declaring " +
                            "@Configuration class. Add the 'static' modifier to this method to avoid " +
                            "these container lifecycle issues; see @Bean javadoc for complete details.",
                    beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName()));
        }
        // 直接调用原方法创建 bean
        return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
    }
    // 如果不满足上面 if，也就是在 userInfo() 中调用的 country() 方法
    return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
}
```
> **关于 `isCurrentlyInvokedFactoryMethod` 方法**:
>
> 可以参考 `SimpleInstantiationStrategy` 中的 `instantiate` 方法，这里先设置的调用方法：
> ```
> currentInvokedFactoryMethod.set(factoryMethod);
> return factoryMethod.invoke(factoryBean, args); // 反射
> ```
> 而通过方法内部直接调用 country() 方法时，不走上面的逻辑，直接进的代理方法，也就是当前的 intercept方法，因此当前的工厂方法和执行的方法就不相同了。

`obtainBeanInstanceFromFactory` 方法比较简单，就是通过 `beanFactory.getBean` 获取 `Country`，如果已经创建了就会直接返回，如果没有执行过，就会通过 `invokeSuper` 首次执行。

因此我们在 `@Configuration` 注解定义的 bean 方法中可以直接调用方法，不需要 `@Autowired` 注入后使用。


# 回头看看 @Component
`@Component` 注解并没有通过 cglib 来代理 `@Bean`方法的调用，因此像下面这样配置时，就是两个不同的 `country`：
```
@Component
public class MyBeanConfig {

    @Bean
    public Country country(){
        return new Country();
    }

    @Bean
    public UserInfo userInfo(){
        return new UserInfo(country());
    }

}
```
有些特殊情况下，我们不希望 `MyBeanConfig` 被代理（代理后会变成 `WebMvcConfig$$EnhancerBySpringCGLIB$$8bef3235293`）时，就得用 `@Component`，这种情况下，上面的写法就需要改成下面这样：
```
@Component
public class MyBeanConfig {

    @Autowired
    private Country country;

    @Bean
    public Country country(){
        return new Country();
    }

    @Bean
    public UserInfo userInfo(){
        return new UserInfo(country);
    }

}
```
就能保证使用的是同一个 `country` 实例。

# 总结
通过源码分析，我们知道了一些关于`@Configuration`的知识点：
1. 为什么`@Configuration`不能是 final？ 
   > 因为源码告诉我们，`@Configuration` bean需要被动态代理，进行运行时增强，而动态代理无法搞定final 类（所以我们在写unit test 的时候，想要mock 一些 final的类和成员属性，基本是无从下手的，比如，你想mock一个`String`对象，是不给的）
2. `@Configuration`和`@Component`都会为其下的`@Bean`方法 解析注入对象。 
   > 对的，源码告诉我们，因为`@Configuration`属于`@Component`，那么，就一定会走`ConfigurationClassPostProcessor.processConfigBeanDefinitions()`方法，去构建并验证其结构，最终都会调用`ConfigurationClassParser.parse`方法。
   >
   > `processConfigBeanDefinitions`方法做了以下关键两个步骤：
   > 1. `checkConfigurationClassCandidate()`：查看一个BeanDefinition是不是可以被parser解析，逻辑是如果有`@Configuration`的那么是full，如果是有`@Component`，`@ComponentScan`，`@Import`， `@ImportResource`四个注解，或者方法里面有`@Bean`注解，那么也可以被解析，属于lite。
   > 2. `ConfigurationClassParser.parse()`：用于分析`@Configuration`注解的配置类，产生一组`ConfigurationClass`对象。它的分析过程会接受一组种子配置类(调用者已知的配置类，通常很可能只有一个)，从这些种子配置类开始分析所有关联的配置类，分析过程主要是递归分析配置类的注解@Import，配置类内部嵌套类，找出其中所有的配置类，然后返回这组配置类。这个工具类自身的逻辑并不注册bean定义，它的主要任务是发现`@Configuration`注解的所有配置类并将这些配置类交给调用者(调用者会通过其他方式注册其中的bean定义)，而对于非`@Configuration`注解的其他bean定义，比如`@Component`注解的bean定义，该工具类使用另外一个工具`ComponentScanAnnotationParser`扫描和注册它们。
3. 为什么`@Configuration` 中的`@Bean`方法使用另一个bean对象时，可以直接调用另一个带有`@Bean`的方法来获取同一个bean， 而`@Component`却不行呢？
   > 源码告诉我们，因为`@Configuration`修饰的bean，在扫描验证过程结束后，会多一步 增强过程，会将自身的`@Bean`方法调用处理过程给到代理类是执行，从而控制调用方法时是否依然创建新的对象。

# 参考文章
* https://blog.csdn.net/isea533/article/details/78072133
* http://makaidong.com/chuliang/970048_20893468.html
* https://blog.csdn.net/andy_zhang2007/article/details/78549773