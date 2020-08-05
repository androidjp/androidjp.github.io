---
title: Spring Boot 通过类名找到类的踩坑之路
date: 2020-08-05 16:07:48
tags:
- spring
categories:
- java
---
> 通过全类名找类，估计大家都尝试过，但，通过 simple类名去找类，估计少用，这不，有个业务场景，尝试通过这个方式去解决，并进而踩坑的过程分享。
<!--more-->

# 需求
我通过MQ 传给你一个message，message上面有一堆的header，每个header都是一个json，我还传了一个headerType 列表，用于告诉 consumer，每个header到底是什么类型的，String？Integer？CustomObj？ 换言之，就是多了一份类型描述文档：

Message Header:
```json
{
    "breadcrumbId": "ID-76265358ec77-1594171418720-0-187147",
    "CamelJmsDeliveryMode": "2",
    "CARRIED_DATA_LIST": "{\"EVENT_ACTION\":\"java.lang.String\",\"EVENT_NUMBER_RELATIONSHIP\":\"com.example.entity.NumberRelationship\",\"EVENT_DATE_TIME\":\"java.time.LocalDateTime\"}",
    "EVENT_ACTION": "NEW",
    "EVENT_DATE_TIME": "\"2020-07-24T10:37:27\"",
    "EVENT_NUMBER_RELATIONSHIP": "{\"orderNumber\":\"Order001\",\"traceNumber\":\"TraceId01\",\"customerId\":\"user001\"}"
}
```
好，那consumer方要拿到这份json，然后，为了方便做一些处理，consumer方会尝试根据 这份message header里头的  `CARRIED_DATA_LIST` 这份类型清单，然后，根据这个类型去做一个 JSON to Java Object 的映射，然后方便我们以Java Object 的方式去进行后续的logic处理。

# 实现
于是，我们有了一个`OwnClassUtil.java`，在类加载过程中就扫描并加载好所有里里外外的类名：

```java
public class OwnClassUtil {
    private static List<String> ownClassList;

    static {
        try {
            ownClassList = getAllClassName();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static Class findDeclaredClassName(String className) throws ClassNotFoundException {
        Class clazz = findClassByCompleteName(className);
        if (clazz != null) {
            return clazz;
        }
        String[] splits = className.split("\\.");
        String simpleClassName = splits[splits.length - 1];
        return findClassBySimpleClassName(simpleClassName);
    }

    private static Class findClassByCompleteName(String className) {
        try {
            return Class.forName(className);
        } catch (ClassNotFoundException e) {
            return null;
        }
    }

    public static Class findClassBySimpleClassName(String simpleClassName) throws ClassNotFoundException {
        if ("String".equals(simpleClassName)) {
            return String.class;
        }
        if (ownClassList.contains(simpleClassName)) {
            return Class.forName(simpleClassName);
        }
        for (String className : ownClassList) {
            String[] strings = className.split("\\.");
            if (strings.length == 0) {
                continue;
            }
            if (strings[strings.length - 1].equals(simpleClassName)) {
                return Class.forName(className);
            }
        }

        throw new ClassNotFoundException("can not find class " + simpleClassName);
    }

    public static List<String> getAllClassName() throws IOException {
        List<String> names = new ArrayList<>();
        names.addAll(getClassName("com.example.myapp"));
        names.addAll(getClassName("com.example.common"));
        return names;
    }


    public static List<String> getClassName(String packageName) throws IOException {
        return getClassName(packageName, true);
    }


    public static List<String> getClassName(String packageName, boolean childPackage) throws IOException {
        List<String> fileNames = null;
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        String packagePath = packageName.replace(".", "/");
        Enumeration<URL> resources = loader.getResources(packagePath);
        URL url = loader.getResource(packagePath);
        while (resources.hasMoreElements()) {
            url = resources.nextElement();
            if (!url.toString().contains("test-classes")) {
                break;
            }
        }
        if (url != null) {
            String type = url.getProtocol();
            if ("file".equals(type)) {
                fileNames = getClassNameByFile(url.getPath(), null, childPackage);
            } else if ("jar".equals(type)) {
                fileNames = getClassNameByJar(url.getPath(), childPackage);
            }
        } else {
            fileNames = getClassNameByJars(((URLClassLoader) loader).getURLs(), packagePath, childPackage);
        }
        return fileNames;
    }


    private static List<String> getClassNameByFile(String filePath, List<String> className, boolean childPackage) {
        List<String> myClassName = new ArrayList<>();
        File file = new File(filePath);
        File[] childFiles = file.listFiles();
        for (File childFile : childFiles) {
            if (childFile.isDirectory()) {
                if (childPackage) {
                    myClassName.addAll(getClassNameByFile(childFile.getPath(), myClassName, childPackage));
                }
            } else {
                String childFilePath = childFile.getPath();
                if (childFilePath.endsWith(".class")) {
                    childFilePath = childFilePath.substring(childFilePath.indexOf("classes") + 8,
                            childFilePath.lastIndexOf("."));
                    childFilePath = childFilePath.replace("\\", ".");
                    childFilePath = childFilePath.replace("/", ".");
                    myClassName.add(childFilePath);
                }
            }
        }

        return myClassName;
    }


    private static List<String> getClassNameByJar(String jarPath, boolean childPackage) {
        List<String> myClassName = new ArrayList<>();
        String[] jarInfo = jarPath.split("!");
        String jarFilePath = jarInfo[0].substring(jarInfo[0].indexOf("/"));
        String packagePath = jarInfo[1].substring(1);
        try {
            JarFile jarFile = new JarFile(jarFilePath);
            Enumeration<JarEntry> entrys = jarFile.entries();
            while (entrys.hasMoreElements()) {
                JarEntry jarEntry = entrys.nextElement();
                String entryName = jarEntry.getName();
                if (entryName.endsWith(".class")) {
                    if (childPackage) {
                        if (entryName.startsWith(packagePath)) {
                            entryName = entryName.replace("/", ".").substring(0, entryName.lastIndexOf("."));
                            myClassName.add(entryName);
                        }
                    } else {
                        int index = entryName.lastIndexOf("/");
                        String myPackagePath;
                        if (index != -1) {
                            myPackagePath = entryName.substring(0, index);
                        } else {
                            myPackagePath = entryName;
                        }
                        if (myPackagePath.equals(packagePath)) {
                            entryName = entryName.replace("/", ".").substring(0, entryName.lastIndexOf("."));
                            myClassName.add(entryName);
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return myClassName;
    }


    private static List<String> getClassNameByJars(URL[] urls, String packagePath, boolean childPackage) {
        List<String> myClassName = new ArrayList<>();
        if (urls != null) {
            for (int i = 0; i < urls.length; i++) {
                URL url = urls[i];
                String urlPath = url.getPath();
                if (urlPath.endsWith("classes/")) {
                    continue;
                }
                String jarPath = urlPath + "!/" + packagePath;
                myClassName.addAll(getClassNameByJar(jarPath, childPackage));
            }
        }
        return myClassName;
    }
}
```

这样一来，就能够实现 当传入的 类型清单 里告诉我，这个 `EVENT_NUMBER_RELATIONSHIP` header 实际是 一个 `com.example.entity.NumberRelationship` 类型时，我能够match上我这边app的同名类，如 `com.backend.po.NumberRelationship`：

```java
String type = typeListMap.get("EVENT_NUMBER_RELATIONSHIP"); // com.example.entity.NumberRelationship
String stringValue = headerMap.get("EVENT_NUMBER_RELATIONSHIP"); // "{\"orderNumber\":\"Order001\",\"traceNumber\":\"TraceId01\",\"customerId\":\"user001\"}"
Class<?> clazz = OwnClassUtil.findDeclaredClassName(type);
JsonUtils.jsonToObject(jsonValue, clazz); //  com.backend.po.NumberRelationship 实例对象
```
其中，JsonUtils是用的 alibaba FastJson：
```java
public static <T> T jsonToObject(String json, Class<T> clazz) {
    if (StringUtils.hasText(json)) {
        try {
            return JSONObject.parseObject(json, clazz);
        } catch (JSONException e) {
            return JSONObject.parseObject("\"" + json + "\"", clazz);
        }
    } else {
        return null;
    }
}
```


# 抽common lib 之后
好，当业务logic多了之后，我们将一些common的 po类放入 common lib，供其他 app 引用。

然后，我们准备在本地 run 或者 debug 一个 引用了 common lib 的 app，我们称其为"**my-app**"，
![](/images/20200805/1.png)



此时 `OwnClassUtil.java` 发挥正常，其中的代码片段：
```java
ClassLoader loader = Thread.currentThread().getContextClassLoader(); 
String packagePath = packageName.replace(".", "/"); 
Enumeration<URL> resources = loader.getResources(packagePath); 
URL url = loader.getResource(packagePath);
//........
```
ContextClassLoader 能够正常 加载到 本app package 和 common-lib package 两边的 class：
```
com.example.myapp.entity.Student
com.example.myapp.entity.Teacher
com.example.myapp.entity.Address
com.example.common.lib.po.NumberRelationship
com.example.common.lib.po.Customer
com.example.common.lib.po.Order
```

所有的  "JSON to Java Object" 的logic 都完好运行。

# 打jar包之后
好，现在要将这个 **my-app** `mvn clean package` 一下了，因为它用的插件是：`spring-boot-maven-plugin`，所以打出来的包 就是 可以直接 通过 `java -jar` 命令启动的。

但是，我们发现，通过 `java -jar my-app.jar` 运行起来后，虽然spring boot 项目能正常运行，但是却发现， 我们的 `OwnClassUtil` 类并没有扫描到 common-lib 中的 类，换句话说，此时，线程上下文类加载器 ContextClassLoader 只扫描到了 当前项目的包`com.example.myapp`下的类，其他第三方库中的类并没有被扫描到：
```
com.example.myapp.entity.Student
com.example.myapp.entity.Teacher
com.example.myapp.entity.Address
```

# 扫描不到类的解决方案
## 方案一：ExtClassLoader帮忙扫多一个自定义目录

### 思路
我们知道，在整个JVM类加载过程中，遵循双亲委派机制的、我们最熟悉的三个ClassLoader：
* BootstrapClassLoader：【我爷爷】
   * JVM级别的，由C++撰写，jvm中某c++写的dll类
   * 负责加载java基础类，主要是 %JRE_HOME/lib/ 目录下的rt.jar、resources.jar、charsets.jar和class等
* ExtClassLoader：【我爸】
   * Java类
   * 继承自URLClassLoader超类
   * 负责加载java扩展类，主要是 %JRE_HOME/lib/ext 目录下的jar和class
   * 由 `sun.misc.Launcher` 初始化
* AppClassLoader：【我】
   * Java类
   * 继承自URLClassLoader超类
   * 负责加载当前java应用的classpath中的所有类。
   * 由 `sun.misc.Launcher` 初始化
* 自定义 ClassLoader：【我同辈】
   * Java类
   * 继承自URLClassLoader超类

其中，
* AppClassLoader 对应加载  `--classpath`，可以这样扩展：
   > JDK 9 弃用 -classpath  改用 --class-path
```sh
java -cp "./:./lib/a.jar:./lib/b.jar" -jar  my-app.jar

// 当 lib目录下jar过多，一般使用 shell帮忙
CLASS_PATH=./
for jar in ./lib/*.jar; do
   CLASS_PATH=$CLASS_PATH:$jar
done
java -cp $CLASS_PATH -jar my-app.jar
```
* ExtClassLoader 对应加载 `java.ext.dirs`，可以这样动态让ExtClassLoader多加载一个目录`./mylib`：
```sh
// Linux
java -Djava.ext.dirs=$JAVA_HOME/jre/lib/ext:/usr/java/packages/lib/ext:./mylib -jar  my-app.jar

// Windows (分号 和 冒号的区别)
java -Djava.ext.dirs=$JAVA_HOME/jre/lib/ext;/usr/java/packages/lib/ext;./mylib -jar  my-app.jar
```

所以，我们 可以利用这一点，让ExtClassLoader 帮 `OwnClassUtil` 一把，帮忙加载一些我们想扫描的jar包。


### 步骤
1. 首先，优化一下我们的`OwnClassUtil` 中的`getClassName` 方法，让ExtClassLoader 进来帮帮忙：
```java
public static List<String> getClassName(String packageName) throws IOException {
    List<String> names = getClassName(Thread.currentThread().getContextClassLoader(), packageName, true); // 上下文ClassLoader先尝试扫描
    if (CollectionUtils.isEmpty(names)) {
        try {
            ClassLoader extClassLoader = OwnClassUtil.class.getClassLoader().getParent().getParent(); // ExtClassLoader 再帮忙扫描
            names = getClassName(extClassLoader, packageName, true);
        } catch (Exception e) {
            System.err.println(e.getMessage());
        }
    }
    return names;
}
```
2. 然后，`mvn clean package`打包
3. 将打包好的`my-app.jar` 解压：`unzip my-app.jar`
4. 得到三个目录：`BOOT-INF`，`META-INF` 以及 `org`
5. 然后，在启动jar包时，执行这样一句命令：`java -jar -Djava.ext.dirs=$JAVA_HOME/jre/lib/ext;/usr/java/packages/lib/ext;./mylib -jar  my-app.jar`

这样一来，最终回归到 Dockerfile，是长这样的：
```dockerfile
FROM openjdk:8-alpine
COPY ./target/my-app.jar ./app/app.jar
WORKDIR ./app
RUN unzip app.jar

EXPOSE 8080
CMD ["java", "-Djava.ext.dirs=$JAVA_HOME/jre/lib/ext:/usr/java/packages/lib/ext:./BOOT-INF/lib",  "-jar", "app.jar"]
```
当然，如果你想要删除某些没必要再被ExtClassLoader扫描的jar，那可以删除它，如：我要删除除了`common-lib-1.0.0.jar`以外的所有jar包，可以这样：
```dockerfile
FROM openjdk:8-alpine
COPY ./target/my-app.jar ./app/app.jar
WORKDIR ./app
RUN unzip app.jar
WORKDIR ./BOOT-INF/lib
RUN ls | grep -v ^common-lib | xargs rm -rf
WORKDIR ./../../

EXPOSE 8080
CMD ["java", "-Djava.ext.dirs=$JAVA_HOME/jre/lib/ext:/usr/java/packages/lib/ext:./BOOT-INF/lib",  "-jar", "app.jar"]
```


## 方案二：解压后直接run JarLauncher

我们换个思路：其实spring有自己的专属启动方式，那我们就了解一下，它打包出来的jar包 `my-app.jar` 到底内部有什么蹊跷：
![](/images/20200805/2.png)

是的，spring 有自己的一套启动和类加载套路，它很通过 `org.springframework.boot.loader` 包下的 Launcher 和 ClassLoader 相关类，去很好地解决了依赖库依赖问题以及类查找问题。
![](/images/20200805/3.png)

而它的 `JarLauncher.java` 内部代码片段如下：
![](/images/20200805/4.png)
![](/images/20200805/5.png)
背后是让自定义类加载器`LuanchedURLClassLoader` 去做类加载的。

可以看到，它是指定扫描 `BOOT-INF/lib` 里头的依赖库，所以，我们索性是可以将这个jar包解压，然后直接 用 java 指令 直接跑 `JarLauncher`， 从而启动 `my-app` 的入口`Application.java`，进而启动整个spring项目。


首先解压你gen好的Spring Boot项目可执行jar包：
```
unzip your-app.jar
```
然后，能直接看到当前目录多了三个目录，分别是：
* BOOT-INF
* META-INF
* org

接着，直接运行 `JarLauncher` 即可启动项目：
```
java org.springframework.boot.loader.JarLauncher
```

最终，dockerfile这么写：
```docker
FROM openjdk:8-alpineCOPY ./target/BCCD_EXCEPTION_HANDLER.jar ./app/app.jarWORKDIR ./appRUN unzip app.jarEXPOSE 8080CMD ["java", "org.springframework.boot.loader.JarLauncher"]
```


# 分析
看看 `OwnClassUtil` 在扫描类和包的时候，使用的类加载器 以及 扫描得到所有类名：

其中，运行jar包 和 直接IDE运行，得到的 类加载器是：
![](/images/20200805/6.png)
都是 `TomcatEmbeddedWebappClassLoader` 其parent 是 `LaunchedURLClassLoader`:
```
TomcatEmbeddedWebappClassLoader
  context: ROOT
  delegate: true
----------> Parent Classloader:
org.springframework.boot.loader.LaunchedURLClassLoader@ad9418
```

但是，在jar包情况下，得到的classList如下：
```
[BOOT-INF.classes.com.example.myapp.entity.Teacher, .......]
```

而直接IDE运行，得到的classList如下：
```
[com.example.myapp.entity.Teacher, ......., com.example.common.po.Customer, ........]
```
可以看出，在jar包情况下运行，由于jar包环境没有了目录相关的概念，spring 的`LaunchedURLClassLoader` 并没有能力很好地走正常 扫描resource 类和jar包资源的logic，拿到的都是 `BOOT-INF.classes` 中的东西，并不能摸到 `BOOT-INF.lib` 中的jar包资源；而在IDE运行或解压运行的情况下，`JarLauncher` 则能够很好地扫描目录结构，因为文件结构是现成可见的。

# 总结
* 通过利用ClassLoader的装载特性，可以帮助扫描装载你想要的jar；
* spring-maven 插件打的jar包在 通过 `java -jar` 运行时，在类的可见性方面会有所限制，虽然背后的类加载器都是自定义Spring自家的`LaunchedURLClassLoader`，但由于jar包的目录概念弱化，正常的 `getResource` 仅能触及 `BOOT-INF/classes`即当前project本身的code；
* 通过解压 jar包，并运行 JarLauncher 类，能够打破类可见性方面的限制，文件系统再现，导致目录概念重回正轨，使得第三方类库中的类能够被当前类加载器通过`getResource` 扫描到；
* Spring 可执行jar包的启动是通过JarLauncher配合自定义URLClassLoader  `LaunchedURLClassLoader` 进行类加载和初始化的。一般情况下，如果我们不去定制 contextClassLoader，那么所有的线程将会默认使用 AppClassLoader，所有的类都将会是共享的， 但是在Spring 项目中，是用的自定义类加载器，所以线程上下文类加载器并非默认的AppClassLoader。