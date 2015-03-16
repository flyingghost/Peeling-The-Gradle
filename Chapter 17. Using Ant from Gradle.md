# Chapter 17. 在Gradle中使用Ant
Gradle提供了与Ant的完美集成。可以在Gradle构建中使用单独的Ant任务，或者直接使用整个Ant构建脚本。事实上你会发现，在Gradle中使用Ant比起以XML格式写Ant脚本要强到不知道哪里去了。甚至可以把Gradle当做Ant任务的一个强大的脚本工具。

Ant可以被分为两层。第一层是Ant脚本语言，为build.xml文件提供了语法基础，target处理，特殊的结构比如宏定义等，换句话说，除了Ant任务和类型之外的所有东西。Gradle理解这种语言，并且允许你直接导入Ant的build.xml到工程中。之后你可以把Ant的target当做Gradle的目标来使用。

Ant的第二层是其丰富的Ant任务和类型，例如javac，copy，jar等。在这一层，Gradle基于强大的Groovy语言和神奇的AntBuilder类也提供了集成能力。

最后，虽然构建脚本是Groovy脚本，你依然可以把一个Ant构建当做外部进程来执行。你的脚本可以包含像`"ant clean compile".execute()`这样的语句。这是使用了Groovy中“String可以直接当作命令行执行”的特性。

你可以把Gradle对Ant的集成作为把你家现有Ant脚本向Gradle平滑迁移的途径。例如，你可以从导入既有Ant构建脚本开始，然后把依赖声明从Ant脚本移植到Gradle脚本，最后，你可以把任务们从Ant脚本移到Gradle里，或者把任务们直接替换为某Gradle插件来实现。这个迁移过程可以随时间一点点的做，并且不管已迁移多少还剩多少，整个过程中你起码都有一个能用的Gradle构建。

## 17.1. 在构建中使用Ant的任务和类型
在构建脚本中存在一个Gradle提供的叫`ant`的属性，这是一个指向[AntBuilder](https://gradle.org/docs/current/javadoc/org/gradle/api/AntBuilder.html)实例的引用。AntBuilder可以用来在构建脚本中操作Ant的任务，类型以及属性。从Ant的build.xml到Groovy有一种很简单的映射方式，下文会讲到。

你可以通过调用AntBuilder实例的方法来运行一个Ant任务，直接使用任务名称当做方法名称就行。例如，可以通过调用`ant.echo()`方法来执行Ant的echo任务。Ant任务的属性以Map的形式作为参数传递给方法。下面是一个调用echo任务的栗子。注意我们可以混合Groovy代码和Ant的任务标记，非常强大。

Example 17.1. Using an Ant task

build.gradle

    task hello << {
        String greeting = 'hello from Ant'
        ant.echo(message: greeting)
    }

可以通过在调用任务方法的时候传递字符串参数，来给Ant任务传递一个嵌套文本。下例中，我们把消息字符串当做嵌套文本传递给echo任务。

Example 17.2. Passing nested text to an Ant task

build.gradle

    task hello << {
        ant.echo('hello from Ant')
    }

也可以通过闭包给Ant任务传递嵌套元素。嵌套元素和任务的定义方式相同，通过调用一个与我们想定义的元素同名的方法来完成。

Example 17.3. Passing nested elements to an Ant task

build.gradle

    task zip << {
        ant.zip(destfile: 'archive.zip') {
            fileset(dir: 'src') {
                include(name: '**.xml')
                exclude(name: '**.java')
            }
        }
    }

可以像操作任务一样的方式操作Ant的类型：用类型名称当做方法名。方法调用会返回Ant数据类型，可以在构建脚本中直接使用。下面的栗子中，我们创建一个Ant的path对象，然后遍历它的内容。

Example 17.4. Using an Ant type

build.gradle

    task list << {
        def path = ant.path {
            fileset(dir: 'libs', includes: '*.jar')
        }
        path.list().each {
            println it
        }
    }

更多关于AntBuilder的信息可以从《Groovy In Action》或者 Groovy Wiki文档里找到。

### 17.1.1. 在构建中使用自定义Ant任务
要在构建中使用自定义任务，你可以使用Ant任务taskdef（一般比较简单）或者typedef，就像在build.xml里那样用。然后，你可以像引用内置Ant任务一样引用自定义任务。

Example 17.5. Using a custom Ant task

build.gradle

    task check << {
        ant.taskdef(resource: 'checkstyletask.properties') {
            classpath {
                fileset(dir: 'libs', includes: '*.jar')
            }
        }
        ant.checkstyle(config: 'checkstyle.xml') {
            fileset(dir: 'src')
        }
    }

你可以使用Gradle的依赖管理来汇集classpath提供给自定义任务。要想这样做，你需要为classpath定义一个配置，然后给配置添加一些依赖项。51.4节会针对这个讲更详细。

Example 17.6. Declaring the classpath for a custom Ant task

build.gradle

    configurations {
        pmd
    }
    
    dependencies {
        pmd group: 'pmd', name: 'pmd', version: '4.2.5'
    }

要想使用classpath配置，可使用自定义配置的`asPath`属性。

Example 17.7. Using a custom Ant task and dependency management together

build.gradle

    task check << {
        ant.taskdef(name: 'pmd',
                    classname: 'net.sourceforge.pmd.ant.PMDTask',
                    classpath: configurations.pmd.asPath)
        ant.pmd(shortFilenames: 'true',
                failonruleviolation: 'true',
                rulesetfiles: file('pmd-rules.xml').toURI().toString()) {
            formatter(type: 'text', toConsole: 'true')
            fileset(dir: 'src')
        }
    }

## 17.2. 导入Ant构建
你可以使用`ant.importBuild()`方法来把Ant构建导入到Gradle工程里。当导入Ant构建的时候，每一个Ant目标都被当做Gradle的一个任务。这意味着你可以用与Gradle任务完全相同的方式操作和执行Ant目标。

Example 17.8. Importing an Ant build

build.gradle

    ant.importBuild 'build.xml'

build.xml

    <project>
        <target name="hello">
            <echo>Hello, from Ant</echo>
        </target>
    </project>

你可以添加一个依赖于Ant目标的Gradle任务。

Example 17.9. Task that depends on Ant target

build.gradle

    ant.importBuild 'build.xml'
    
    task intro(dependsOn: hello) << {
        println 'Hello, from Gradle'
    }

或者，你可以给Ant目标添加行为。

Example 17.10. Adding behaviour to an Ant target

build.gradle

    ant.importBuild 'build.xml'
    
    hello << {
        println 'Hello, from Gradle'
    }

也可以让一个Ant目标去依赖一个Gradle任务。

Example 17.11. Ant target that depends on Gradle task

build.gradle

    ant.importBuild 'build.xml'
    
    task intro << {
        println 'Hello, from Gradle'
    }

build.xml

    <project>
        <target name="hello" depends="intro">
            <echo>Hello, from Ant</echo>
        </target>
    </project>

有时候Ant目标和Gradle任务的名称冲突了，这时候就需要为导入Ant目标而自动生成的Gradle任务重新命名。要想这样做，可以使用`AntBuilder.importBuild()`方法。

Example 17.12. Renaming imported Ant targets

build.gradle

    ant.importBuild('build.xml') { antTargetName ->
        'a-' + antTargetName
    }

build.xml

    <project>
        <target name="hello">
            <echo>Hello, from Ant</echo>
        </target>
    </project>

注意这个方法的第二个参数应当是一个[Transfromer](https://gradle.org/docs/current/javadoc/org/gradle/api/Transformer.html)，但在Groovy里我们可以使用一个闭包来替代一个匿名内部类。可以看这里的文档：[Groovy支持将闭包自动强转为只有单一虚方法类型的实例](http://mrhaki.blogspot.ie/2013/11/groovy-goodness-implicit-closure.html)。

## 17.3. Ant属性和引用
有很多种方法来设置Ant属性以供Ant任务使用。可以直接在AntBuilder的实例上设置。也可以作为一个可修改的Map获取到Ant的属性们。还可以使用Ant的property任务。以下栗子展示如何做到这几种方式。

Example 17.13. Setting an Ant property

build.gradle

    ant.buildDir = buildDir
    ant.properties.buildDir = buildDir
    ant.properties['buildDir'] = buildDir
    ant.property(name: 'buildDir', location: buildDir)

build.xml

    <echo>buildDir = ${buildDir}</echo>

很多Ant任务执行的时候都会设置属性。有很多方式可以得到属性的值。可以直接通过AntBuilder实例获取属性值，也可以作为一个Map得到Ant的属性们。以下看栗子：

Example 17.14. Getting an Ant property

build.xml

    <property name="antProp" value="a property defined in an Ant build"/>

build.gradle

    println ant.antProp
    println ant.properties.antProp
    println ant.properties['antProp']

多种方式来设置一个Ant引用：

Example 17.15. Setting an Ant reference

build.gradle

    ant.path(id: 'classpath', location: 'libs')
    ant.references.classpath = ant.path(location: 'libs')
    ant.references['classpath'] = ant.path(location: 'libs')

build.xml

    <path refid="classpath"/>

多种方式得到一个Ant引用：

Example 17.16. Getting an Ant reference

build.xml

    <path id="antPath" location="libs"/>

build.gradle

    println ant.references.antPath
    println ant.references['antPath']

## 17.4. API
由[AntBuilder](https://gradle.org/docs/current/javadoc/org/gradle/api/AntBuilder.html)来提供Ant的集成工作。
