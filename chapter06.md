Chapter 6. 构建脚本基础
===
6.1 Project和Task
---
Gradle中万事万物都建立在两个基本概念之上：project和task。

project就是工程，它的目标取决于你想做什么，不一定就是构建。也许是发布、自动化测试、检查等。

每一个projct都是由多个task组成。一个task一般就是一个原子的构建步骤或者动作。例如编译、创建jar、生成javadoc、发布到某仓库等。

6.2 Hello world
---
使用gradle命令来运行，当然前提是依照第四章做好环境配置。命令默认会执行当前目录下的build.gradle文件（也可以使用 `gradle -b xxx.gradle` 来指定一个脚本文件来执行）。

#### Example 6.1. Hello world


    task hello {
        doLast {
            println 'Hello world!'
        }
    }
    
执行命令 `gradle hello` 。发生这么几件事：

- 定义一个task，名称为hello。
- 每个task在本身之外还可以追加动作，有两个插入点：doFirst在task本身之前执行，doLast会在task本身之后执行。这里在doLast阶段插入新的动作：一个闭包。闭包的概念请参见Groovy语法。
- 闭包内部直接输出Hello world，这正是Gradle的强大之处：随时随机插入真正的脚本。（当然前面说过，前面看起来像声明或者配置的玩意，本质也是脚本）。

6.3. doLast的快捷方式
---

    task hello << {
        println 'Hello world!'
    }

可以用 << 来快捷表达doLast。

6.4. 脚本即代码
---
    task upper << {
        String someString = 'mY_nAmE'
        println "Original: " + someString 
        println "Upper case: " + someString.toUpperCase()
    }

已解释过。随时写代码。

    task count << {
        4.times { print "$it " }
    }

顺便再来点Groovy语法介绍：在Groovy中，所有东西都是对象，包括`4`这种数值常量也是。它有一个方法`times`，接受一个闭包，把它执行`4`次。**闭包可以简单的理解为一段代码。后续会针对闭包有更详细的介绍。**

6.5. Task依赖
---
    task hello << {
        println 'Hello world!'
    }
    task intro(dependsOn: hello) << {
        println "I'm Gradle"
    }

声明intro的执行依赖于task的执行。运行`gradle -q intro`可看到效果。

添加依赖的时候，对应的task不一定已经存在。

    task intro(dependsOn: 'hello') << {
        println "I'm Gradle"
    }
    
    task hello << {
        println 'Hello world!'
    }

这个特性在多工程依赖、动态task依赖的时候特别有用：你在配置当前task依赖子工程某task的时候，子工程build脚本也许还没加载进来呢。

但是这个例子没有讲彻底。如果你胆敢去掉两个引号，那就挂了：

    task intro(dependsOn: hello) << {    ①
        println "I'm Gradle"
    }
    
    task hello << {                        ②
        println 'Hello world!'
    }

②实质上是为当前脚本添加了一个名叫hello的属性。

①如果写`'hello'`，意为**“以字符串'hello'为名称查找一个task并依赖之”**，运行这个task的时候自然会找。但如果写`hello`，意为**“找到属性hello引用的task对象并依赖之”**，这时候hello并未声明。于是卒。

6.6. 动态生成Task
---
    4.times { counter ->
        task "task$counter" << {
            println "I'm task number $counter"
        }
    }

这个例子表现如何运行时动态的生成task。依然夹带讲点Groovy语法：

- times刚才讲过，整形对象的方法，接受一个闭包。这个闭包其实会携带一个参数：当前是第几次。如果像6.4例子中那样不明确声明闭包参数的名字，它将会有一个默认的名字`it`。本例中给它声明了一个名字`counter`，将会从0变化到3（就像普通的for循环）。闭包参数之后跟随`->`和后续的代码部分分隔开来。

- `"task$counter"`：groovy中字符串可以用单引号（对应Java中的String）或者双引号（对应一种扩展字符串GString来自于groovy类库）。双引号字符串中可以包含变量，在运行中被替换为变量的实际值（非常类似于c中的字符串format）。标准写法为`"字符串${varName}字符串"`，当替换动作不会产生二义性的时候（例如`$变量名`之后是空格或者字符串结束等情况下）甚至可以省略{ }。

明白了这两项语法，例子代码不言自明。

6.7. 操作已存在的Task
---
    4.times { counter ->
        task "task$counter" << {
            println "I'm task number $counter"
        }
    }
    task0.dependsOn task2, task3

task们一旦声明，就可以像普通对象一样操作它们的方法了。

或者添加点行为。

    task hello << {
        println 'Hello Earth'
    }
    hello.doFirst {
        println 'Hello Venus'
    }
    hello.doLast {
        println 'Hello Mars'
    }
    hello << {
        println 'Hello Jupiter'
    }

doFrist和doLast都可以被调用多次。其实first和last是两个链表，每次调用都会添加一个新的action到链表上。doFirst每次添加到first链表头，doLast每次添加到last链表尾巴。执行的时候会把两个链表里的素有action依次执行。

6.8. 引用task
---
    task hello << {
        println 'Hello world!'
    }
    hello.doLast {
        println "Greetings from the $hello.name task."
    }
刚才已经说过，当一个task创建之后，它就顺便成为脚本的一个属性。这时候就可以直接拿这个属性引用task了。没啥可说的。


6.9. 额外的task属性
---
    task myTask {
        ext.myProperty = "myValue"
    }
    
    task printTaskProperties << {
        println myTask.myProperty
    }
可以为task增加自定义的属性：只需在task范围内，在ext中声明自己的属性，以后就可以当task自带属性一样使用了。

这一招不仅仅可以用于task。对于script、project等也都可以这么玩。

6.10. Using Ant Tasks
---

    task loadfile << {
        def files = file('../antLoadfileResources').listFiles().sort()
        files.each { File file ->
            if (file.isFile()) {
                ant.loadfile(srcFile: file, property: file.name)
                println " *** $file.name ***"
                println "${ant.properties[file.name]}"
            }
        }
    }

如果你有庞大的ant历史遗产无法迁移，gradle也提供了集成ant使用的能力。ant的target和property都可以很方便的引用。

不过在我看来，如果ant遗产很小，不如重写。如果ant遗产庞大，那引入gradle就变得更可怕。除非万不得已，给一套庞大稳定成熟的系统再引入额外的异构技术都是不理智的。所以gradle要对历史致敬也属无奈，但我是对这部分没多大兴趣的。后续我们可以介绍一些使用ant自带的任务来作为gradle类库的补充，以简化我们的脚本开发工作。但随着gradle自身的完善，需要”借用“ant来补充的情况一定会越来越少的。

所以我们不如来关注脚本本身吧。脚本详析部分非常基础，熟悉groovy的同学尽可略过。这段面对的是大批从未了解过groovy，为了gradle才不得不接触的同学。

- file()方法得到一个File对象。listFiles()方法得到目录里的所有文件的集合。
- 对于所有集合，都可以调用each方法来遍历。each接受一个闭包参数，对每一个集合成员都执行闭包。
- 每一个元素同样被当做闭包的参数传递进来并命名为file，和之前不同的是，你可以顺手指定这个参数的类型。这样就可以享受更准确的代码提示，副作用是万一集合中有不同类型元素，这里就直接转型失败。
- ant是project的属性，可以通过ant.task_name调用所有ant任务。
- loadfile是ant自带任务。[戳这里查看文档](https://ant.apache.org/manual/Tasks/loadfile.html)。
- ant任务的参数使用map传递。这里有srcFile和property两个参数。**参数的类型约束没有找到，请知道的同学指点。**

6.11. 方法的定义和使用
---
    task checksum << {
        fileList('../antLoadfileResources').each {File file ->
            ant.checksum(file: file, property: "cs_$file.name")
            println "$file.name Checksum: ${ant.properties["cs_$file.name"]}"
        }
    }
    
    task loadfile << {
        fileList('../antLoadfileResources').each {File file ->
            ant.loadfile(srcFile: file, property: file.name)
            println "I'm fond of $file.name"
        }
    }
    
    File[] fileList(String dir) {
        file(dir).listFiles({file -> file.isFile() } as FileFilter).sort()
    }

gradle里可以定义方法（废话，说了很多遍了是groovy脚本）。

所以我们还是穿插点groovy语法（诡异，此文有沦为groovy教程的嫌疑）。

- ant的checksum文档可[戳这里](http://ant.apache.org/manual/Tasks/checksum.html)
- 字符串中使用${}嵌入变量的时候，大括号内可以视为另一个闭包，内部当然可以使用新的""定义字符串。所以行4看起来会有""嵌套的错觉。groovy正确的嵌套姿势是""内使用'，或者''内使用"。
- `{file -> file.isFile() } as FileFilter`，as是一个非常强大的操作符，可以动态的做很多类型转换。**但这个闭包是如何转换成为FileFilter的，这点还不明白。希望了解的同学能告诉我**。

6.12. 默认task
---
    defaultTasks 'clean', 'run'
    
    task clean << {
        println 'Default Cleaning!'
    }
    
    task run << {
        println 'Default Running!'
    }
    
    task other << {
        println "I'm not a default task!"
    }
可以为当前Project指定默认任务，如果在命令行没有指定运行哪个task，这个设置就生效了。在多project的项目中，子工程可以指定自己的默认task。如果没指定，那就继承父工程的运行任务。

6.13. 无环有向图(DAG)式配置
---
**Gradle构建脚本的运行分为两个阶段：配置阶段和执行阶段。**配置阶段之后，gradle就以DAG的形式把所有需要执行的task组织起来，并提供了API可以让你获取task信息并据此做一些逻辑。比如说：看看DAG里有没有release任务来得知是否是一次release发布。

    task distribution << {
        println "We build the zip with version=$version"
    }
    
    task release(dependsOn: 'distribution') << {
        println 'We release now'
    }
    
    gradle.taskGraph.whenReady {taskGraph ->
        if (taskGraph.hasTask(release)) {
            version = '1.0'
        } else {
            version = '1.0-SNAPSHOT'
        }
    }

whenReady方法接受一个闭包，并在配置阶段结束task们已经准备就绪的时候执行。注意这里只是检查DAG中是否有release，至于它是否最终目标倒无所谓。

6.14. 下一步?
---
诏曰可。












