---
title: 用Gradle从零搭建 Spring Cloud Greenwich.SR4  微服务项目（二）-- 配置中心
date: 2019-12-11 20:26:56
tags:
- gradle
- spring cloud
categories:
- spring cloud
- java
---

这篇，重点关注我们的配置中心，在各种场景下该如何运用，才会最妙！

<!--more-->

# 原理
怎么验证是否能够读取到git上面的配置呢？
> 答：直接启动配置中心，然后，访问对应你想要访问的git上面的目录或文件，如：
> * `http://localhost:9010/micro-financial/manager`
> * `http://localhost:9010/micro-financial-manager/dev`
> * `http://localhost:9010/micro-financial-manager.yml`

修改了git上面的配置，配置中心是如何收到更新通知的？
> 答：同样，通过访问上述的几个API，此时，配置中心就会自动读取最新提交内容：
> * `http://localhost:9010/micro-financial/manager`
> * `http://localhost:9010/micro-financial-manager/dev`
> * `http://localhost:9010/micro-financial-manager.yml`

仓库中的配置文件会被转换为web接口，访问参照以下规则：
```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

## 相关配置参数

属性|默认值|说明
---|---|---
spring.cloud.config.allow-override|true|外部源配置是否可被覆盖
spring.cloud.config.override-none|false|外部源配置是否不覆盖任何源
spring.cloud.config.override-system-properties|true|外部源配置是否可覆盖本地属性

针对相应的属性的值对应的外部源在Environment对象中的读取优先级，罗列如下

属性|读取优先级
---|---
spring.cloud.config.allow-override=false|最高
spring.cloud.config.override-none=false&&spring.cloud.config.override-system-properties=true|最高(默认)
spring.cloud.config.override-none=true|最低
spring上下文无systemEnvironment属性|最低
spring上下文有systemEnvironment属性 && spring.cloud.config.override-system-properties=false|在systemEnvironment之后
spring上下文有systemEnvironment属性 && spring.cloud.config.override-system-properties=false|在systemEnvironment之前

* 即默认情况下，外部源的配置属性的读取优先级是最高的。
* 且除了`spring.cloud.config.override-none=true`的情况下，其他情况下外部源的读取优先级均比本地配置文件高。

> Note:值得注意的是，如果用户想复写上述的属性，则放在`bootstrap.yml`|`application.yml`配置文件中是无效的，根据源码分析只能是自定义一个`PropertySourceLocator`接口实现类并放置在相应的`spring.factories`文件中方可生效。

## 自定义PropertySourceLocator接口
目的：想要将部分的参数优先读本地配置

假设有这么一个场景，远程仓库的配置都是公有的，我们也不能修改它，我们只在项目中去复写相应的配置以达到兼容的目的。那么用户就需要自定义去编写接口了。

1. 编写PropertySourceLocator接口实现类
    ```
    /**
     * @description 自定义的PropertySourceLocator的顺序应该要比远程仓库读取方式要优先
     * @see org.springframework.cloud.config.client.ConfigServicePropertySourceLocator
     */
    @Order(value = Ordered.HIGHEST_PRECEDENCE + 1)
    public class CustomPropertySourceLocator implements PropertySourceLocator {
    
        private static final String OVERRIDE_ADD_MAPPING = "spring.resources.add-mappings";
    
        @Override
        public PropertySource<?> locate(Environment environment) {
    
            Map<String, Object> customMap = new HashMap<>(2);
            // 远程仓库此配置为false，本地进行复写
            customMap.put(OVERRIDE_ADD_MAPPING, "true");
    
            return new MapPropertySource("custom", customMap);
        }
    }
    ```
2. 编写BootstrapConfiguration
    ```
    @Configuration
    public class CustomBootstrapConfiguration {
    
        @Bean("customPropertySourceLocator")
        public CustomPropertySourceLocator propertySourceLocator() {
            return new CustomPropertySourceLocator();
        }
    }
    ```
3. 在src\main\resources目录下创建META-INF\spring.factories文件
    ```
    # Bootstrap components
    org.springframework.cloud.bootstrap.BootstrapConfiguration=\
    com.example.configdemo.propertysource.CustomBootstrapConfiguration
    ```

# 应用
## 直接读取本地配置
好，对于本地环境，或者默认环境，直接读取本地配置文件会是一个OK的选择：

`basic-config`的`application-dev.yml`这么配：
```
spring:
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
```
然后，在`resources/config/application-dev.yml`作为给到别的微服务的配置：
```
test: Mike
```
最后，`micro-financial-manager`的`bootstrap.yml`这么配：
```
spring:
  application:
    name: micro-financial-manager
  cloud:
    config:
      discovery:
        enabled: true
        service-id: basic-config
      profile: ${JP_CLOUD_CONFIG_PROFILE:dev}
```

## 配置放在git仓库
首先，新建github仓库，把你的配置扔上去：

github上面配置文件的目录结构：
```
|_ qa
  |_ micro-financial-manager.yml
|_ prod
```
其中，文件`micro-financial-manager.yml`配置：
```
jpcloud:
  test: Jay
```

然后，`basic-config`的`application-qa-git.yml`这么配：
```
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:jpuneng/jp-cloud-gradle-config-repo.git
          search-paths: qa
      label: master
```
这里的`uri`也可以使用HTTPs方式：
```
https://github.com/jpuneng/jp-cloud-gradle-config-repo.git
```

最后，`miro-financial-manager`的`boostrap.yml`：
```
spring:
  application:
    name: micro-financial-manager  # 对应{application}部分
  cloud:
    config:
      discovery:                   # 通过eureka发现配置中心
        enabled: true
        service-id: basic-config   # 指定配置中心的service-id，便于扩展为高可用配置集群。
      label: master                # 对应git的分支。如果配置中心使用的是本地存储，则该参数无用
      profile: dev                 # 对应{profile}部分
      fail-fast: true
```
> 特别注意：上面这些与spring-cloud相关的属性必须配置在bootstrap.properties中，config部分内容才能被正确加载。因为config的相关配置会先于application.properties，而bootstrap.properties的加载也是先于application.properties。

在微服务启动时，读取配置中心相关配置的过程涉及到三个类：
```
ConfigServicePropertySourceLocator ---- 调用 locate() 方法，真的去请求获取远程配置
ConfigClientHealthProperties ---------- 相关配置（如：时间间隔）
PropertySourceBootstrapConfiguration -- 真的将远程配置更新到微服务自己的各个属性
```

只要有注入`spring-cloud-starter-config`并且有配置这个`spring.cloud.config.discovery.enabled=true`，那么，就会默认每5分钟去config service上面读取配置。注意：是去配置中心上面拿配置，此时，配置中心回去看看git上面的配置的version对不对，有没有被修改过，并最终将version信息返回给到微服务。定时读取配置的相关类：
```
ConfigServerHealthIndicator ---------- Scheduler调用的方法入口
ConfigClientHealthProperties --------- 相关配置（如：时间间隔）
ConfigServicePropertySourceLocator --- 调用 locate() 方法，真的去请求获取远程配置
```
如果要自定义定时获取配置的时间间隔，比如：我要10s获取一次配置，则在微服务的`bootstrap.yml`里头加上：
```
health:
  config:
    time-to-live: 10000
```
即可。

但注意，虽然微服务已经读到了更新后的version，发现了配置已经是被更新了的，但是，此时并不会触发`PropertySourceBootstrapConfiguration.initialize()`方法，那么，就不会真正地去使用这个已更新的配置。


## 其他服务想要的配置，都只能从我这里拿
我的其他微服务，例如：路由网关服务`basic-gateway`的一些routing配置，我是不是也可以配在我自己的git仓库当中呢？

其实可以！

1. 首选我们将原本在`basic-gateway/bootstrap.yml`中的配置，放到github 上面的`basic-gateway.yml`文件上：
    ```
    spring:
      cloud:
        gateway:
          routes:
            - id: micro-financial-manager-route
              uri: http://localhost:9200/
              predicates:
                - Path=/financial/**
    ```
2. 然后，将原来code里头的config删除；
3. 接着，我们引入config相关依赖库；
    ```
    compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-config'
    ```
4. 以及相关配置`bootstrap.yml`：
    ```
    spring:
      application:
        name: basic-gateway
      cloud:
        config:
          discovery:
            enabled: true
            service-id: basic-config
          label: master
          fail-fast: true
    ```
5. 这样一来，就能读取到git上的routing配置了。


## 主动推送更新配置给到配置使用方
我的git中配置更新了，我又不想通过重启微服务的方式去拿这个更新后的值，怎么样才能动态、主动地将更新后的配置告诉给各大微服务爸爸呢？

### 方式一：actuator/refresh刷新微服务配置
* 原理：通过调用`/actuator/refresh`POST请求，让微服务主动去找配置中心刷新最新的配置；
* 缺点：配置更新了，但也无法主动通知到微服务，需要微服务自己handle；
* 优点：不需要重启微服务和配置中心；

步骤：
1. 首先，如果没有配`actuator`相关依赖，需要配上，当然，eureka-client库,config库,甚至admin-client库里头本身依赖着`actuator`相关库，所以只要你有依赖这几个库的其中一个，都不需要单独引入`org.springframework.boot:spring-boot-starter-actuator`：
    ```
    compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-netflix-eureka-client'
    compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-config'
    compile group: 'de.codecentric', name: 'spring-boot-admin-starter-client', version: '2.1.6'
    ```
2. 然后，`micro-financial-manager`的`application.yml`配置暴露refresh相关接口：
    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: '*'
      endpoint:
        health:
          show-details: ALWAYS
    ```
3. 然后，我们给需要上去配置中心读取的配置，所在的类，头上，加上一个注解：`@RefreshScope`；
4. 最后，我们主动帮`micro-financial-manager`调用`/actuactor/refresh`POST方法，即可刷新配置（因为这个方法会trigger `PropertySourceBootstrapConfiguration.initialize`方法）；
5. 当然，我们也可以通过github 的 webhook，来监听每次git配置的push请求，然后自动请求我们这个`/actuator/refresh`方法。

### 方式二：集成event-bus，并调用actuator/bus-refresh刷新微服务配置

这里应该需要配合 事件总线 来实现。

流程：![UTOOLS1575938691182.png](https://i.loli.net/2019/12/10/nFxj32iRBtYgKpX.png)

* 缺点：比起方式一，更加繁琐，需要依赖`event-bus`和MQ服务的配合；
* 优点：能够通过通配符一键trigger所有的微服务更新配置，比起一个个调用，好一点；

步骤：
1. 首先，保证你的MQ服务已经启动，例如这里我们选用rabbitMQ（`spring-cloud`默认支持RabbitMQ和Kafka两种event-bus集成方式）【详细搭建步骤可看另一篇文章】
2. 然后，给你的微服务引入依赖：
    ```
    compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-bus-amqp'
    ```
    spring-cloud-starter-bus-amqp 内部依赖：spring-cloud-starter-stream-rabbit 和 spring-cloud-bus
3. 配置`application.yml`：
    ```
    spring:
      rabbitmq:
        host: 192.168.204.132
        port: 5672
        username: guest
        password: guest
      cloud:
        bus:
          enabled: true
          trace:
            enabled: true
    ```
4. 最后，调用 POST `/actuator/bus-refresh?destination=customers:**`即刷新服务名为customers的所有服务。
   > 注意： 这个接口也可能是： `/bus/refresh`

打开rabbitmq管理界面，看到它自己建了相关的queue，就是为了发一个更新的数据给到各个微服务：
![UTOOLS1575899420222.png](https://i.loli.net/2019/12/09/sfH6k1hyIiqv482.png)

## 源码分析
具体看这篇文章：https://vitzhou.top/2018-02-spring-cloud-refresh-context/