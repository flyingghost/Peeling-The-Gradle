# Chapter 14. 教程 - 其他零碎知识点
## 14.1. 创建目录
编译过程中多个task依赖于某目录是否存在是一个很常见的情况。当然你可以在每个task前添加mkdir，但这不是一个好办法，不应该重复任何只需要做一次的代码（DRY原则）。更好的解决方案是为task们定义`dependsOn`依赖关系，以达到重用创建目录task的目的。

    def classesDir = new File('build/classes')
    
    task resources << {
        classesDir.mkdirs()
        // do something
    }
    task compile(dependsOn: 'resources') << {
        if (classesDir.isDirectory()) {
            println 'The class directory exists. I can operate'
        }
        // do something
    }

## 14.2. Gradle属性和系统属性
Gradle提供很多方式来为你的构建添加一个属性。可以使用命令行参数 **-D** 来为运行gradle的JVM提供系统属性。在`gradle`命令使用 **-D** 参数和在`java`命令使用 **-D** 参数用法和效果都是一样的。

也可以使用properties文件来给gradle提供属性。可以在`gradle.properties`文件中定义属性值，放在Gradle的用户目录（由环境变量`GRADLE_USER_HOME`定义，默认值为`USER_HOME/.gradle`目录）会对所有使用gradle的工程生效；放在工程目录会对当前工程生效。在多工程项目中，可以把`gradle.properties`放在任意子工程目录中。在`gradle.properties`文件中定义的属性和值可以通过project对象来访问。放在用户目录的properties文件优先级比放在工程目录中的要高（有点反直觉？）

你也可以通过 **-P** 命令行参数来添加属性。

当Gradle发现有特殊名字的系统属性或者环境变量存在，它也会设置project属性。如果你在一个持续集成服务器上没有admin权限，而且还想设置一个不容易被直接看到的属性（比如说密码），这个特性就可以发挥作用了。在这种情况下，你不能使用 -P 参数，也无法改变系统级配置文件。正确姿势是去修改持续集成的job的配置，添加一个特殊模式的环境变量，普通用户是无法看到这些配置的，Jenkins/Teamcity/Bamboo等CI服务都可以支持这个特性。

如果环境变量名称类似`ORG_GRADLE_PROJECT_prop=somevalue`，Gradle会给project设置一个叫`prop`的属性，值为`somevalue`。Gradle也支持在系统属性上这么玩，但名称模式稍有不同：类似于org.gradle.project.prop这样。

你还可以在gradle.properties文件中设置系统属性。如果一个属性的名称以"systemProp."前缀开头，例如`systemProp.propName`，这个属性和它的值将被作为系统属性设置，当然去掉了前缀，只取"propName"部分。在一个多工程项目中，"systemProp."属性只能在主工程设置，子工程设置的都会被忽略。也就是说，只有根工程的gradle.properties里的"systemProp."打头的属性才会被设置或添加。

### 14.2.1. 检查project属性
你可以在脚本中像使用普通变量一样只使用名字来引用project的属性。但如果属性不存在，脚本会抛出一个异常，然后宣告构建失败。如果你的脚本依赖于一个用户可能会配置的属性（例如在 gradle.properties 文件中），你需要在引用属性之前先检查它是否存在。可以使用`hasProperty('propertyName')`来检查属性是否存在。


## 14.3. 引入外部build脚本
可以使用一个外部脚本来配置当前工程的构建。Gradle构建语言的所有内容都可以在外部脚本中使用。甚至可以在外部脚本中再导入另一个脚本。

build.gradle

    apply from: 'other.gradle'
    
other.gradle

    println "configuring $project"
    task hello << {
        println 'hello from other script'
    }

## 14.4. 配置任意对象
可以通过以下非常易读的方式“配置”任意对象

    task configure << {
        def pos = configure(new java.text.FieldPosition(10)) {
            beginIndex = 1
            endIndex = 5
        }
        println pos.beginIndex
        println pos.endIndex
    }

## 14.5. 引入外部脚本来配置任意对象
你也可以导入一个外部脚本来得到一个配置对象。

先创建一个other.gradle来定义所有属性和值。

    // Set properties.
    beginIndex = 1
    endIndex = 5

然后在另一个脚本中引入。

    task configure << {
        def pos = new java.text.FieldPosition(10)
        // Apply the script
        apply from: 'other.gradle', to: pos
        println pos.beginIndex
        println pos.endIndex
    }

## 14.6. 缓存
Gradle缓存了所有已被编译的脚本来提高构建速度。包括所有构建脚本、初始化脚本和其他脚本。当你第一次运行工程的构建脚本时，Gradle把编译后的脚本缓存在当前目录下的`.gradle`子目录中。下一次再运行构建，如果脚本本身没有更改，Gradle会直接运行缓存的编译后的脚本。如果脚本本身更新过，那Gradle会重新编译脚本并缓存在`.gradle`目录中。如果使用`--recompile-scripts`选项，不管脚本有没有更新，缓存结果都将会被忽略，整个脚本重编译然后重新进缓存。
