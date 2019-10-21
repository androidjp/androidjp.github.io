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

# 写例子，用一下@Import
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

## （一）SpringBoot程序启动过程
以下过程是SpringBoot程序启动过程的代码执行过程：

1. `SpringApplication.run(String... args)`
   1. `stopWatch.start();`：启动秒表
   2. `System.setProperty("java.awt.headless", true/false);`：设置headless 为`true`or`false`。
      > Headless模式是系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标。后端开发者往往需要这个模式，因为服务器（如提供Web服务的主机）往往可能缺少前述设备，但又需要使用他们提供的功能，生成相应的数据，以提供给客户端（如浏览器所在的配有相关的显示设备、键盘和鼠标的主机）。
   3. 获取`SpringApplicationRunListeners`并让`listeners.starting()`
      1. 实际上，是找到`EventPublishingRunListener`等bean 对象，并遍历让这些listener执行其`starting()`方法。
      2. `SpringApplicationRunListener`接口专门监听`SpringApplication.run()`，每次程序启动都会创建一个其实现类（如：`EventPublishingRunListener`）。
      3. 背后是让`EventPublishingRunListener`里头的`initialMulticaster`去分发一个`ApplicationStartingEvent`，这个event 是在 `Environment`和`ApplicationContext`可用之前，且`ApplicationListener`注册之后，发出的。
   4. `prepareEnvironment()`
      1. 创建environment（`StandardServletEnvironment` or `StandardReactiveWebEnvironment` or `StandardEnvironment`）
      2. 发送`ApplicationEnvironmentPreparedEvent`
   5. 创建`ConfigurableApplicationContext`，同样有三种选择：Servlet、Reactive、default。
   6. .....
   7. `prepareContext()`
      1. `postProcessApplicationContext(context)`
   8. `refreshContext(context)`
      1. `refresh(context)`
	     1. 判断applicationContext 属于 AbstractApplicationContext 类，并 调用其 `refresh()`
		    1. 进入同步块（this.startupShutdownMonitor 的对象锁）
			2. `prepareRefresh();`
			3. Tell the subclass to refresh the internal bean factory
			4. Prepare the bean factory for use in this context. 此时， beanFactory 中的 beanDefinedMap 中，只有几个bean。
			5. `postProcessBeanFactory();`   Allows post-processing of the bean factory in context subclasses.
			6. `invokeBeanFactoryPostProcessors();`     Invoke factory processors registered as beans in the context.
			   1. **`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`**
   9. `afterRefresh(context);`
   10. `stopWatch.stop();`
   11. `listeners.started(context);`
   12. `callRunners(context, {});`

## （二）ConfigurationClassPostProcessor
好，然后我们看看`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`里头到底做了什么？
> 答： for 循环 遍历 `postProcessors`，拿到每一个 `BeanDefinitionRegistryPostProcessor`或其子类，然后，对每一个postProcessoer item 都调用其`postProcessBeanDefinitionRegistry()` 方法，把 `registry` 传给它。


其中，就会找到`ConfigurationClassPostProcessor`类对象，并调用其方法。

看看这个 `ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()` 方法：
* 作用：看方法源码注释：Derive further bean definitions from the configuration classes in the registry.【从注册表中的配置类派生更多的bean定义。】
* 内部方法注释： Build and validate a configuration model based on the registry of 带有`@Configuration`的classes. 【基于带有`@Configuration`的类的注册表构建和验证配置模型。】
* 内部方法过程：
   1. 首先查看是否是配置类，如果是就加入`configCandidates`候选者列表中。
   2. 如果`configCandidates`为空，则直接返回。
   3. 否则，对这些配置类根据`@Order`排序(如果有配`@Order`)，如果一个有`@Order`，另一个没有配，那么，有`@Order`的排在前面。
   4. Detect any custom bean name generation strategy supplied through the enclosing application context 【检测通过封闭应用程序上下文提供的任何自定义bean名称生成策略。】
   5. 创建 `ConfigurationClassParser` （配置类的解析类）
   6. 调用`ConfigurationClassParser.parse()`方法，开始解析`@Configuration`配置类。

## （三）ConfigurationClassParser.parse()
好，再去看，到底这个`ConfigurationClassParser.parse()`方法会做什么事情？
```
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();
    
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }
    processDeferredImportSelectors();
}
```
1. 遍历每一个 `BeanDefinitionHolder` 候选者 （实际上只有一个候选者，那就是我们程序的入口类`Application.java`的对象，名叫`application`）
2. 看看这个候选者的`BeanDefinition`属于三种类型中的哪一种（一般会进第一种），但背后都是调用一个`parse()`方法。
3. 然后，在方法内部调用`processConfigurationClass()`方法，进行处理。

```
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    //在这里处理Configuration重复import
    //如果同一个配置类被处理两次，两次都属于被import的则合并导入类，返回。如果配置类不是被导入的，则移除旧使用新的配置类
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            return;
        }
        else {
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // 递归地处理配置类及其超类层次结构。
    SourceClass sourceClass = asSourceClass(configClass);
    do {
      //接着往下看吧
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}



protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// 处理递归类
			processMemberClasses(configClass, sourceClass);
		}

    // 处理@PropertySource注解
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // 处理 @ComponentScan 注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }
    //处理Import注解，这个是咱们的菜
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // 处理@ImportResource 注解
    if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    //处理包含@Bean注解的方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // 处理普通方法
    processInterfaces(configClass, sourceClass);

   
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    return null;
}
```

然后，我们就会发现，实际上，上面的代码就会处理`@Bean`、`@ImportSource`、`@Import`、`@ComponentScan`、`@PropertySource`。

我们focus 看 `@Import`：
```
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                // 首先， 判断如果被import的是 ImportSelector.class 接口的实现， 那么初始化这个被Import的类， 然后调用它的selectImports方法去获得所需要的引入的configuration， 然后递归处理
                if (candidate.isAssignable(ImportSelector.class)) {
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                // 其次， 判断如果被import的是 ImportBeanDefinitionRegistrar 接口的实现， 那么初始化后将对当前对象的处理委托给这个ImportBeanDefinitionRegistrar
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                     // 最后，将import引入的类作为一个正常的类来处理
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

从这里我们知道， 如果你引入的是一个正常的component， 那么会作为 `@Compoent` 或者 `@Configuration` 来处理， 这样在`BeanFactory`里边可以通过`getBean()`拿到， 但如果你是 `ImportSelector` 或者 `ImportBeanDefinitionRegistrar` 接口的实现， 那么spring并不会将他们注册到`beanFactory`中，而只是调用他们的方法。

好，由此可见`@Import`导入流程和导入顺序。

