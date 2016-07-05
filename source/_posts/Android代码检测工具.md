---
title: Android代码检测工具
date: 2016-06-28 15:05:00
categories:
- Android
- 代码检测
tags:
- Android
- 代码检测
---
> 引言：努力提高代码质量是每一个开发人员必须具备的素质和自我要求。
但是靠自己总会不小心遗漏一些细节，所以引入代码检测工具，应用到软件工程中，以此帮助大家养成良好的编程习惯，提升技能。
  接下来，和大家一起分析Android相关的代码检测工具的用法。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*IDE：Android Studio*<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*操作系统：Ubuntu*


### 代码检测工具分类
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前常用的Java代码检测工具有：AndroidLint、CheckStyle、FindBugs、PDM

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中，AndroidLint 已经作为插件被集成到Android Studio中；CheckStyle 可以作为插件安装使用；FindBugs 和 PDM 对Android Studio的支持较差，没有可用的插件或插件存在一些问题，但可以利用Gradle生成代码检测报告。

<!--more-->

### 代码检测工具的工作方式、功能

##### 一、什么是静态代码分析

<p>静态代码分析是指无需运行被测代码，仅通过分析或检查源程序的语法、结构、过程、接口等来检查程序的正确性，找出代码隐藏的错误和缺陷，如参数不匹配，有歧义的嵌套语句，错误的递归，非法计算，可能出现的空指针引用等等。
在软件开发过程中，静态代码分析往往先于动态测试之前进行，同时也可以作为制定动态测试用例的参考。统计证明，在整个软件开发生命周期中，30% 至 70% 的代码逻辑设计和编码缺陷是可以通过静态代码分析来发现和修复的。
但是，由于静态代码分析往往要求大量的时间消耗和相关知识的积累，因此对于软件开发团队来说，使用静态代码分析工具自动化执行代码检查和分析，能够极大地提高软件可靠性并节省软件开发和测试成本。</p>

##### 二、静态代码分析工具的优势

1. 帮助程序开发人员自动执行静态代码分析，快速定位代码隐藏错误和缺陷。
2. 帮助代码设计人员更专注于分析和解决代码设计缺陷。
3. 显著减少在代码逐行检查上花费的时间，提高软件可靠性并节省软件开发和测试成本。

##### 三、Java 静态代码分析理论基础和主要技术

* 缺陷模式匹配：缺陷模式匹配事先从代码分析经验中收集足够多的共性缺陷模式，将待分析代码与已有的共性缺陷模式进行模式匹配，从而完成软件的安全分析。这种方式的优点是简单方便，但是要求内置足够多缺陷模式，且容易产生误报。

* 类型推断：类型推断技术是指通过对代码中运算对象类型进行推理，从而保证代码中每条语句都针对正确的类型执行。这种技术首先将预定义一套类型机制，包括类 型等价、类型包含等推理规则，而后基于这一规则进行推理计算。类型推断可以检查代码中的类型错误，简单，高效，适合代码缺陷的快速检测。

* 模型检查：模型检验建立于有限状态自动机的概念基础之上，这一理论将被分析代码抽象为一个自动机系统，并且假设该系统是有限状态的、或者是可以通过抽象归 结为有限状态。模型检验过程中，首先将被分析代码中的每条语句产生的影响抽象为一个有限状态自动机的一个状态，而后通过分析有限状态机从而达到代码分析的 目的。模型检验主要适合检验程序并发等时序特性，但是对于数据值域数据类型等方面作用较弱。

* 数据流分析：数据流分析也是一种软件验证技术，这种技术通过收集代码中引用到的变量信息，从而分析变量在程序中的赋值、引用以及传递等情况。对数据流进行 分析可以确定变量的定义以及在代码中被引用的情况，同时还能够检查代码数据流异常，如引用在前赋值在后、只赋值无引用等。数据流分析主要适合检验程序中的 数据域特性。


### 四种工具简介 及其 特点

#### AndroidLint
###### &nbsp;&nbsp;1.简介
<p>
&nbsp;&nbsp;&nbsp;&nbsp;“Android lint工具是一个静态代码分析工具，它能检查安卓项目源文件的潜在缺陷和优化改进的正确性，安全性，性能，可用性，可访问性和国际化。”
正如官方网站所说，Android Lint是另一种静态分析工具，专门为Android服务。它是非常强大的，能给你大量的建议以提高你的代码质量。
</p>

###### &nbsp;&nbsp;2.使用方法
<div>
<p>&nbsp;&nbsp;&nbsp;&nbsp;点击Android studio工具栏 ——>Analyze ——>Inspect Code</p>
![](http://i2.piimg.com/567571/85bea3be27ae47b3.png)
<p>&nbsp;&nbsp;&nbsp;&nbsp;选择分析的文件范围</p>
![](http://i4.piimg.com/567571/c5cebf5f8a3f34e4.png)
<p>&nbsp;&nbsp;&nbsp;&nbsp;点击OK, 顶部会出现分析报告</p>
![](http://i2.piimg.com/567571/ec998783ca446c66.png)
</div>

---

#### CheckStyle
###### &nbsp;&nbsp;1.简介
<p>
&nbsp;&nbsp;&nbsp;&nbsp;“Checkstyle是一个开发工具用来帮助程序员编写符合代码规范的Java代码。它能自动检查Java代码为空闲的人进行这项无聊(但重要)的任务。”
正如Checkstyle的开发者所言，这个工具能够帮助你在项目中定义和维持一个非常精确和灵活的代码规范形式。当你启动CheckStyle，它会根据所提供的配置文件分析你的Java代码并告诉你发现的所有错误。
Checkstyle会发现大量的问题，特别是在你运用了大量的规则配置，如同你设置了一个非常精确的语法。尽管我通过Gradle使用checkstyle，例如在我进行推送之前，我仍然推荐你为IntellJ/Android Studio使用checkstyle插件(你可以通过Android Studio的工作面板文件/设置/插件直接安装插件)。这种方式下，你可以根据那些为Gradle配置的相同文件在你的工程中使用checkstyle，但是远不止这些，你可以直接在Android Studio中获取带有超链接结果，这些结果通过超链接在你的代码中对应，这是非常有用的(Gradle的这种方式仍然很重要的，因为你可以使用它自动构建系统，如Jenkins)。
</p>

###### &nbsp;&nbsp;2.使用方法
1) 首先进入设置页面进入Plugin页面，如图所示<br/>
![图一](http://i1.piimg.com/567571/9b310e60b1722877.png)

2) 点击Browse repositories进入选择页面，输入checkstyle即可选择安装，如图所示<br/>
![](http://i1.piimg.com/567571/be225c0df57aacc2.png)

3) 安装完成后点击Other Settings中的checkstyle进入选择文件页面，点击右上方的“+”选择你自己的checkstyle文件并应用即可。<br/>
![](http://i4.piimg.com/567571/4d7c24e4ceabc6e9.png)
<div>
&nbsp;&nbsp;&nbsp;&nbsp;这里可以选择本地的checkstyle配置文件，也可以选择SVN等远程的配置文件
![](http://i4.piimg.com/567571/d43084f5d5da26b1.png)
</div>

<div>
&nbsp;&nbsp;&nbsp;&nbsp;安装完成后点击你自己写的java文件，再点击左边绿色箭头,点击“运行”，插件会根据你的checkstyle文件指明你代码的一些不规范的地方，如下图所示
![](http://i4.piimg.com/567571/e6cf59c8f4668370.png)
<p>&nbsp;&nbsp;&nbsp;&nbsp;按照提示修改代码即可</p>
</div>

> 注意：如果找不到或者install失败的情况下，可以去 此网站等下载 CheckStyle的zip包到本地，然后让Android Studio 的 “install plugin from disk”获取本地插件<br/>![](http://i1.piimg.com/567571/2bf16b96058d09a4.png)


---

#### FindBugs
###### &nbsp;&nbsp;1.简介
<p>
&nbsp;&nbsp;&nbsp;&nbsp;Findbugs是否需要一个简介呢？我想它的名称已经让人顾名思义了。“FindBugs使用静态分析方法为出现bug模式检查Java字节码”。FindBugs基本上只需要一个程序来做分析的字节码，所以这是非常容易使用。它能检测到常见的错误，如错误的布尔运算符。FindBugs也能够检测到由于误解语言特点的错误，如Java参数调整（这不是真的有可能因为它的参数是传值）。
</p>

###### &nbsp;&nbsp;2.使用方法
<p>
&nbsp;&nbsp;&nbsp;&nbsp;详见下面的Gradle生成代码检测报告
</p>

---

#### PDM
###### &nbsp;&nbsp;1.简介
<p>
&nbsp;&nbsp;&nbsp;&nbsp;这个工具有个有趣的事实：PMD不存在一个准确的名称。(所以)在官网上你可以发现很有有趣的名称，例如:
* Pretty Much Done
* Project Meets Deadline

&nbsp;&nbsp;&nbsp;&nbsp;事实上，PMD是一个工作有点类似Findbugs的强大工具，但是(PMD)直接检查源代码而不是检查字节码(顺便说句，PMD适用很多语言)。 (PMD和Findbugs)的核心目标是相同的，通过静态分析方法找出哪些模式引起的bug。因此为什么同时使用Findbugs和PMD呢？好吧！尽管Findbugs和PMD拥有相同的目标，(但是)他们的检查方法是不同的。所以PMD有时检查出的bug但是Findbugs却检查不出来，反之亦然。
</p>

###### &nbsp;&nbsp;2.使用方法
<p>
&nbsp;&nbsp;&nbsp;&nbsp;详见下面的Gradle生成代码检测报告
</p>

---

#### Gradle 命令生成代码检测报告

> 相关Github示例 ：**我强烈建议你拷贝下这个[项目工程](https://github.com/vincentbrison/vb-android-app-quality)**，尽管我将介绍的案例都是来自它。与此同时，你将能够测试下自己对这些工具的了解情况。

##### &nbsp;&nbsp;1.关于Gradle任务
Gradle任务的概念(在Gradle中的含义)是理解该篇文章(以及如何以一种通用的方式写Gradle脚本)的基础。**我强烈建议你去看下这两篇关于Gradle任务的文档（[Gradle文档第14章](https://docs.gradle.org/current/userguide/tutorial_using_tasks.html)和[Gradle文档第17章](https://docs.gradle.org/current/userguide/writing_build_scripts.html)）。这个文档包含了大量的例子，因此它非常容易开始学习。如果不想看英文原版，这里我推荐极客学院wiki更新的[Gradle中文指南](http://wiki.jikexueyuan.com/project/gradleIn-action/install-gradle.html)**现在，我假定你拷贝了我的Repo，你导入这个工程到你的Android Studio，并且你熟悉Gradle任务。如果不是，别担心，我将尽我最大的努力让我的讲解更有意义。

##### &nbsp;&nbsp;2.示例各个检测工具的Gradle任务结构
<p>
&nbsp;&nbsp;&nbsp;&nbsp;在上面的示例项目中，这几个检测工具的Gradle形式如图所示：
![](http://i4.piimg.com/567571/2c9c9f4ab702e286.png)
&nbsp;&nbsp;&nbsp;&nbsp;这里的config文件完全与Android项目解耦，如果你的项目要用到Gradle进行项目的代码检测并生成错误报告，直接复制此文件夹到你的项目根目录下即可
</p>

##### &nbsp;&nbsp;3.生成文档流程
<p>
&emsp;显然，如果我们能同时使用这四个工具会更好。你可以添加你的gradle任务之间的依赖，比如当你执行一个任务，其他任务则是第一个完成后执行。通常在Gradle中，通过让工具具有“check”任务来达到工具之间的相互关系：
check.dependsOn 'checkstyle', 'findbugs', 'pmd', 'lint'
<br/>
&emsp;现在，当执行“check” 任务的时候，Checkstyle, Findbugs, PMD, and Android Lint将会同时执行。在你执行/ commiting / pushing / ask merge request 之前进行质量检查是一个很棒的方式。
</p>


<p>&nbsp;&nbsp;&nbsp;&nbsp;1)首先，将示例项目中的这个config文件夹拷贝到你的项目根目录（或者可以参考它的写法，自己定制一个Gradle任务集）</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;2）打开终端，命令行执行Gradle任务：
`gradle check`
![](http://i4.piimg.com/567571/e1565a327d606735.png)
</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;3）任务执行完后，就可以在指定的输出目录下找到文档：（示例是输出在<your_buildPath>/reports目录下）
![](http://i4.piimg.com/567571/f275090129e02125.png)
</p>


##### &nbsp;&nbsp;4.各个工具的具体Gradle表现形式以及使用技巧

###### &nbsp;&nbsp;&nbsp;&nbsp; CheckStyle
&nbsp;&nbsp;&nbsp;&nbsp;1) Gradle任务形式
<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;下面的代码向你展示了在你的项目中使用Checkstyle的最基本的配置(如Gradle任务):
```
task checkstyle(type: Checkstyle) {
  configFile file("${project.rootDir}/config/quality/checkstyle/checkstyle.xml")
  configProperties.checkstyleSuppressionsPath = file("${project.rootDir}/config/quality/checkstyle/suppressions.xml").absolutePath
  source 'src'
  include '**/*.java'
  exclude '**/gen/**'
  classpath = files()
}    
```
&nbsp;&nbsp;&nbsp;&nbsp;2) CheckStyle使用技巧
<p>
&emsp;&emsp;Checkstyle会发现大量的问题，特别是在你运用了大量的规则配置，如同你设置了一个非常精确的语法。尽管我通过Gradle使用 checkstyle，例如在我进行推送之前，我仍然推荐你为IntellJ/Android Studio使用checkstyle插件(你可以通过Android Studio的工作面板文件/设置/插件直接安装插件)。这种方式下，你可以根据那些为Gradle配置的相同文件在你的工程中使用 checkstyle，但是远不止这些，你可以直接在Android Studio中获取带有超链接结果，这些结果通过超链接在你的代码中对应，这是非常有用的(Gradle的这种方式仍然很重要的，因为你可以使用它自动构建系统，如Jenkins)。
</p>



###### &nbsp;&nbsp;&nbsp;&nbsp; FindBugs
&nbsp;&nbsp;&nbsp;&nbsp;1) Gradle任务形式
<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;下面的代码向你展示了在你的项目中使用Findbugs的最基本的配置(以Gradle任务为例):
```
task findbugs(type: FindBugs, dependsOn: assembleDebug) {
    ignoreFailures = false
    effort = "max"
    reportLevel = "high"
    excludeFilter = new File("${project.rootDir}/config/quality/findbugs/findbugs-filter.xml")
    classes = files("${project.rootDir}/app/build/intermediates/classes")
    source 'src'
    include '**/*.java'
    exclude '**/gen/**'
    reports {
        xml.enabled = false
        html.enabled = true
        xml {
            destination "$project.buildDir/reports/findbugs/findbugs.xml"
        }
        html {
            destination "$project.buildDir/reports/findbugs/findbugs.html"
        }
    }
    classpath = files()
}
```
&emsp;&emsp;&emsp;&emsp;它是如此的像一个Checkstyle任务。尽管Findbugs支持HTML和XML两种报告形式，我选择HTML形式，因为这种形式更具有可读性。而且，你只需要把报告的位置设置为书签就可以快速访问它的位置。这个任务也会失败如果发现Findbgus错误失败(同样生成报告)。执行 FindBugs任务，就像执行CheckStyle任务（除了任务的名称是“FindBugs”）。


&nbsp;&nbsp;&nbsp;&nbsp;2) FindBugs使用技巧
<p>
&emsp;&emsp;由于Android项目是从Java项目略有不同，我强烈推荐使用FindBugs过滤器(规则配置)。你可以在这一个例子（例如项目之一）。它基本上忽略了R文件和你的Manifest文件。顺便说一句，由于(使用)FindBugs分析你的代码，你至少需要编译一次你的代码才能够测试它。
</p>

###### &nbsp;&nbsp;&nbsp;&nbsp; PMD
&nbsp;&nbsp;&nbsp;&nbsp;1) Gradle任务形式
<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;下面的代码向你展示了在你的项目中使用PMD的最基本的配置(以Gradle任务为例):
```
task pmd(type: Pmd) {
    ignoreFailures = false
    ruleSetFiles = files("${project.rootDir}/config/quality/pmd/pmd-ruleset.xml")
    ruleSets = []
    source 'src'
    include '**/*.java'
    exclude '**/gen/**'
    reports {
        xml.enabled = false
        html.enabled = true
        xml {
            destination "$project.buildDir/reports/pmd/pmd.xml"
        }
        html {
            destination "$project.buildDir/reports/pmd/pmd.html"
        }
    }
}
```
&emsp;&emsp;&emsp;&emsp;就PMD来说，它几乎与Findbugs相同。PMD支持HTML和XML两种报告形式，所以我再次选择HTML形式。我强烈建议你使用自己的通用配置集文件，正如同我在这个例子(check this file)中一样。所以，你当然应该去看下这些通用配置集文件。我建议你，因为PMD可比FindBugs更有争议的很多，例如：如果你不声明”if statement”或”if statement”为空，它基本上会给你警告信息。如果这些规则是正确的，或这对于您的项目(来说是正确的)，我真的认可你和你队友的工作。我不希望程序因为”if statement”崩溃，我认为这样程序的可读性很差。执行PMD任务，就像是(执行)CheckStyle任务（除了任务的名称是“PMD”）。


&nbsp;&nbsp;&nbsp;&nbsp;2) FindBugs使用技巧
<p>
&emsp;&emsp;我建议你不要使用默认的规则配置集，你需要添加这行代码(已经加上)：
ruleSets = [ ]
<br/>
否则，因为默认值是这些基本的规则配置集，基本的规则配置集会和你定义的规则集一起执行。所以，如果你的自定义规则集不在那些基本配置集中，他们仍然会执行。
</p>

##### &nbsp;&nbsp;5.Gradle任务执行方式检测的总结
<p>
&emsp;配置好后，利用Gradle对Android使用代码质量检查工具是非常容易。比使用质量工具局部检查您的项目在您自己的计算机上，这些工具可以用于自动构建如Jenkins/Hudson这样的平台，让你自动进行质量检查，同时自动建立过程。执行所有我从CLI展现的测试，如同在 Jenkins/Hudson上执行，简单地执行：
gradle check
</p>

---

#### 参考文章

[常用 Java 静态代码分析工具的分析与比较](http://www.oschina.net/question/129540_23043?fromerr=C4u4jrna)<br/>
[如何提高你的代码质量](http://www.devtf.cn/?p=790)<br/>
[Checkstyle的检查项配置详解](http://blog.csdn.net/maritimesun/article/details/7668966)<br/>
[使用 CheckStyle 检查代码](http://gudong.name/2016/04/07/checkstyle.html)<br/>
优秀的开源项目的checkstyle配置文件：<br/>
https://gist.github.com/yoxin/d55c5d178290ed44763123a796de7ce6<br/>
https://github.com/JakeWharton/butterknife/blob/master/checkstyle.xml<br/>
https://github.com/square/picasso/blob/master/checkstyle.xml<br/>
使用Gralde集成错误报告的github地址：<br/>
https://github.com/vincentbrison/vb-android-app-quality
