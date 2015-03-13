# Chapter 13. 编写构建脚本
光阴似箭日月如梭！转眼间gradle就升级2.3了。。。从本章开始2.3走起，追不上官方我心拔凉啊。这是第一个升级断层，但一定不是最后一个。。。

## 13.1. Gradle构建语言
Gradle是一个基于Groovy语言的DSL（领域专用语言），专注构建过程的描述。一个构建脚本其实也就是一个Groovy脚本，可以包含任意的groovy语言元素。Gradle默认脚本是UTF-8编码，不要问我怎么样支持GBK编码的脚本，还在用GBK编码的程序员简直就是自作孽。

## 13.2. Project相关API
第七章里我们使用了`apply`“命令”，其实它是个方法。这货来自于哪里？我们曾经说过，build脚本定义了一个工程，Gradle为每一个工程创建了一个[Project](http://gradle.org/docs/current/dsl/org.gradle.api.Project.html)类型的实例，并把它和build脚本关联起来。脚本运行过程中：

- 被调用的任何方法，如果没有在脚本中定义，将被委托给这个Project对象。
- 所访问的任何属性，如果没有在脚本中定义，将被委托给这个Project对象。

试试访问name属性。

    println name
    println project.name
    
第一行name并未定义过，自动委托给project对象。所以和第二行是一样的效果，所以这样写比较省力。只有你定义的方法/属性与project内置的方法/属性名称相同，才必须使用project对象去获取Project对象自带的那一个。

### 13.2.1. 标准Project属性
Project对象提供了一些标准属性以便脚本使用。见下表：

| 属性名 | 类型 | 默认值 |
|-------|-----|--------|
|project|Project|Project实例|
|name|String|项目名称|
|path|String|项目绝对路径|
|description|String|项目描述|
|projectDir|File|包含构建脚本的目录|
|buildDir|File|构建生成目录，一般都在工程目录的build子目录下|
|group|Object|未指定|
|version|Object|未指定|
|ant|AntBuilder|AntBuilder对象引用，可以直接使用ant任务|

## 13.3. Script类
当Gradle执行脚本的时候，首先会把整个脚本编译为[Script](https://gradle.org/docs/current/dsl/org.gradle.api.Script.html)的实例。也就是说Script的所有属性和方法也可以在脚本中使用。

## 13.4. 声明变量
两种变量可以声明：局部变量和附加属性。

### 13.4.1. 局部变量
使用`def`关键字声明的是局部变量，有效作用域范围就是它们声明的地方。来看一下groovy的变量作用域：

- 方法屏蔽外部局部变量

        def local = 'asdf'
        void foo(){
        	assert 'asdf' == local    //报错
        }
        foo()

- 块内可以访问局部变量

        def local = 'asdf'
        def aClosure = {
        	assert 'asdf' == local    //可行
        }
        aClosure()

所以，userguide里给出的以下代码可以正常运行就可以理解了：task是方法调用，后面的{}是一个闭包。

    def dest = "dest"
    
    task copy(type: Copy) {
        from "source"
        into dest
    }

### 13.4.2. 自定义属性
在Gradle的领域模型中，所有被扩展的对象都可以添加用户自定义属性。所谓被扩展对象，包括但不限于project、task、sourceSet等。扩展的自定义属性可以通过所属对象的ext属性进行添加、读取、修改。或者，直接放在ext块里一次性添加多个属性。

    apply plugin: "java"
    
    ext {
        springVersion = "3.1.0.RELEASE"
        emailNotification = "build@master.org"
    }
    
    sourceSets.all { ext.purpose = null }
    
    sourceSets {
        main {
            purpose = "production"
        }
        test {
            purpose = "test"
        }
        plugin {
            purpose = "production"
        }
    }
    
    task printProperties << {
        println springVersion
        println emailNotification
        sourceSets.matching { it.purpose == "production" }.each { println it.name }
    }

本例中，ext代码块（给当前project对象）添加了2个自定义属性。此外，一个名叫purpose的属性添加到了每一个源码集，并设置了初始值null。一旦属性被添加，就可以像默认预定义属性一样去操作它们。

如上所示，添加一个自定义属性需要特殊的语法。所以如果在不合理的场合试图设置预定义或者自定义属性，但属性并不存在（或者只是拼写错误），gradle将会立刻失败。相比局部变量，自定义属性的作用域范围要大的多，任何能存取某对象的场合，都能操作它的自定义属性。子工程也可以访问父工程的自定义属性。

关于自定义属性的更多细节，可以参考API文档 [ExtraPropertiesExtension](https://gradle.org/docs/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html)。

## 13.5. Groovy 基础
Groovy提供了大量用于创建DSL的灵活语法特色，Gradle把这些特点用的是炉火纯青。理解gradle是如何工作的，有利于我们写自己的构建脚本，尤其是写自定义plugin和自定义task的时候。

### 13.5.1. Groovy JDK
Groovy在标准java类库的基础上添加了大量有用的方法。例如，Iterable新增的each方法将对Iterable中的所有元素进行遍历：

    // Iterable gets an each() method
    configurations.runtime.each { File f -> println f }

看看[http://groovy.codehaus.org/groovy-jdk/](http://groovy.codehaus.org/groovy-jdk/)基本上会被吓到。太丧病了。

### 13.5.2. 属性访问器
Groovy会自动地把一个属性的引用转换为对适当的 getter 或 setter 方法的调用。

    // Using a getter method
    println project.buildDir
    println getProject().getBuildDir()
    
    // Using a setter method
    project.buildDir = 'target'
    getProject().setBuildDir('target')

### 13.5.3. 方法调用时括号是可选的
调用方法时括号是可选的。

    test.systemProperty 'some.prop', 'value'
    test.systemProperty('some.prop', 'value')

### 13.5.4. list 和 map 常量
Groovy提供了定义list和map的字面常量实例的快捷方式，写法简洁明快，map字面常量的使用还有个有趣的小技巧。

举个栗子，当apply一个插件的时候，apply关键字实际上调用了project的apply方法，使用一个map作为参数。然而，当你输入`apply plugin:'java'`时，实际上并不是使用map字面常量，而是使用了“命名参数”（一个参数命名为plugin，值为'java'），只是长得比较像不带方括号的map字面常量而已。当方法被调用时，这个命名参数列表才被自动转化为一个map传递给apply方法。

    // List 字面常量
    test.includes = ['org/gradle/api/**', 'org/gradle/internal/**']
    
    List<String> list = new ArrayList<String>()
    list.add('org/gradle/api/**')
    list.add('org/gradle/internal/**')
    test.includes = list //list变量赋值，和第一行是等效的
    
    // Map 字面常量
    Map<String, String> map = [key1:'value1', key2: 'value2']
    
    // Groovy把命名参数列表强制转化为一个map
    apply plugin: 'java'

### 13.5.5. 闭包作为方法的最后一个参数
Groovy DSL在很多地方都使用闭包。[Groovy文档](http://groovy.codehaus.org/Closures)中有一个专题来讲闭包。如果闭包是方法的最后一个参数，可以直接把闭包放在方法调用之后。

    // 方法名不用写括号也可以调用
    repositories {
        println "in a closure"
    }
    // 最后一个参数闭包可以写在括号之外
    repositories() { println "in a closure" }
    // 当然常规写法也是可以的。这三种写法完全等效
    repositories({ println "in a closure" })

### 13.5.6. 闭包委托
每个闭包都有一个委托对象，如果一个变量或者方法引用不是闭包的局部变量或参数，Groovy将在委托对象里查找它。Gradle把这种闭包称为**配置闭包**，需要配置的对象就扔给它一个委托对象。

    dependencies {
        //thisClosure.delegate == project.getDependencies
        assert delegate == project.dependencies
        // 这一步如何变成add('testCompile','junit')？不明哎
        testCompile('junit:junit:4.11')
        delegate.testCompile('junit:junit:4.11')
    }

先讲这么多。不懂的回头再补。
