#Chapter 9. Groovy构建快速入门

要构建Groovy工程，可以使用Groovy插件。这插件在java插件上扩展而来，基本和java插件一致，额外添加了groovy支持。

##9.1. 一个基本的Groovy工程

    apply plugin: 'groovy'

同样的，简单一行代码引入groovy插件，你的工程就摇身一变成为groovy工程。插件扩展了compile任务，指定代码位于`src/main/groovy`，compileTest任务指定代码位于`src/test/groovy`。这些源码目录混合包含java和groovy代码都可以。

和java不同的是，要想编译groovy，你还得指定Groovy版本以及从哪里获取Groovy库（就像普通外部依赖一样）。

    repositories {
        mavenCentral()
    }
    
    dependencies {
        compile 'org.codehaus.groovy:groovy-all:2.3.6'
    }

##9.2. 综上
本章介绍了如何开始一个非常简单的groovy工程。简单的极其不厚道。实际上使用还不如仔细看24章，专门讲这个。要不是有完美综合症，这种没营养的章节我都不想译了。
