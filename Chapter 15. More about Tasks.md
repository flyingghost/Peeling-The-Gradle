# Chapter 15. 深入Task
在前面的章节中我们已经学到如何创建一个简单的task，以及如何为task添加其他行为，以及如何为task们声明依赖关系。以上都是task的简单使用，但gradle的task概念可以更牛掰。gradle支持增强的task，意味着task可以有自己属性和方法。这和Ant的target有很大的不同。这种增强task可以由我们来自己创建，gradle也自带一些供使用。

## 15.1. 定义task
在第六章中我们见过如何用“关键字”风格创建task。这种风格有几种变体供某些特殊情况下使用。例如，关键字风格在表达式里就不能用。

Example 15.1. 

build.gradle

    task(hello) << {
        println "hello"
    }
    
    task(copy, type: Copy) {
        from(file('srcDir'))
        into(buildDir)
    }

也可以使用字符串作为task的名称：

Example 15.2. 

build.gradle

    task('hello') <<
    {
        println "hello"
    }
    
    task('copy', type: Copy) {
        from(file('srcDir'))
        into(buildDir)
    }

还有一种定义方式可能有人喜欢用（不是我）：

Example 15.3. Defining tasks with alternative syntax

build.gradle

    tasks.create(name: 'hello') << {
        println "hello"
    }
    
    tasks.create(name: 'copy', type: Copy) {
        from(file('srcDir'))
        into(buildDir)
    }

这里我们把新定义的task直接加入了task容器。可以看看[TaskContainer](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/TaskContainer.html)文档关于create()方法的更多信息。

## 15.2. 查找task
我们经常需要在build文件中查找已经定义的task，例如，对它们进行配置或者作为依赖使用它们。这时候有很多种方法来定位或者查找task。首先，每个task都是project的一个属性，直接拿task名做属性名就可以了。

Example 15.4. Accessing tasks as properties

build.gradle

    task hello
    
    println hello.name
    println project.hello.name

task也可以在project的tasks容器里找到。

Example 15.5. Accessing tasks via tasks collection

build.gradle

    task hello
    
    println tasks.hello.name
    println tasks['hello'].name

我们还可以使用project的tasks.getByPath()方法，传入task的路径来查找。其实不限于路径，task的名称、绝对路径、相对路径都可以作为查找参数。

Example 15.6. Accessing tasks by path

build.gradle

    project(':projectA') {
        task hello
    }
    
    task hello
    
    println tasks.getByPath('hello').path
    println tasks.getByPath(':hello').path
    println tasks.getByPath('projectA:hello').path
    println tasks.getByPath(':projectA:hello').path

执行 gradle -q hello

    > gradle -q hello
    :hello
    :hello
    :projectA:hello
    :projectA:hello

瞄一眼[TaskContainer](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/TaskContainer.html)文档可以找到更多关于查找task的信息。

## 15.3. 配置 task
作为栗子，我们来看看gradle提供的Copy task。

Example 15.7. Creating a copy task

build.gradle

    task myCopy(type: Copy)

一行代码创建毫无用处的拷贝task，但可以通过API来配置。后续栗子展示了实现同样配置的几种不同姿势。

要清楚一点，本task的名称是"myCopy"，而类型是"Copy"。我们有很多个名称不同但类型相同的task。后续你会发现可以针对某一具体类型来对所有task做处理的强大能力。

Example 15.8. Configuring a task - various ways

build.gradle

    Copy myCopy = task(myCopy, type: Copy)
    myCopy.from 'resources'
    myCopy.into 'target'
    myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')

这种风格有点像java里配置对象，声明一个对象，然后每一行重复一次对象的引用名称。相当罗嗦而且不方便维护。万一我们想改名叫yourCopy，算算要改几行代码？

其实可以使用另一种风格：用大括号保持一段配置的上下文，可读性就好多了。

Example 15.9. Configuring a task - with closure

build.gradle

    task myCopy(type: Copy)
    
    myCopy {    //第3行
       from 'resources'
       into 'target'
       include('**/*.txt', '**/*.xml', '**/*.properties')
    }

这种方式适用于任何task。第3行其实是tasks.getByName('myCopy')方法调用的快捷写法。需要注意的是，如果你给getByName()方法传一个闭包参数，这个闭包是用来配置task的，而不是运行task。

也可以在定义task的同时就使用闭包来配置。这才是大爱的风格。

Example 15.10. Defining a task with closure

build.gradle

    task copy(type: Copy) {
       from 'resources'
       into 'target'
       include('**/*.txt', '**/*.xml', '**/*.properties')
    }

## 15.4. 为task添加依赖关系
定义task的依赖关系有几种方法。在6.5节"Task依赖"中已经介绍了使用task名称来定义依赖。task名称既可以指向同一project的task也可以其他工程的task。但指向其他工程的task需要把它所属工程的路径名作为task名的前缀。以下栗子展示了如何给projectA:taskX添加依赖projectB:taskY。

Example 15.11. Adding dependency on task from another project

build.gradle

    project('projectA') {//连project都顺便定义了
        task taskX(dependsOn: ':projectB:taskY') << { //记得前文讲过":"作为/一样的路径分隔符
            println 'taskX'
        }
    }
    
    project('projectB') {
        task taskY << {
            println 'taskY'
        }
    }

也可以使用task对象来定义依赖。

Example 15.12. Adding dependency using task object

build.gradle

    task taskX << {
        println 'taskX'
    }
    
    task taskY << {
        println 'taskY'
    }
    
    taskX.dependsOn taskY

更高端一些，还可以用闭包来定义依赖。在脚本的配置阶段计算某个task依赖关系的时候，可以使用一个闭包。这个闭包应当返回一个Task实例或者Task实例的集合，然后把它们添加为当前计算task的依赖。以下栗子展示了如何把所有以"lib"打头的task都添加为taskX的依赖。

Example 15.13. Adding dependency using closure

build.gradle

    task taskX << {
        println 'taskX'
    }
    
    taskX.dependsOn {
        tasks.findAll { task -> task.name.startsWith('lib') }
    }
    
    task lib1 << {
        println 'lib1'
    }
    
    task lib2 << {
        println 'lib2'
    }
    
    task notALib << {
        println 'notALib'
    }

点[Task文档](https://gradle.org/docs/current/dsl/org.gradle.api.Task.html)解锁更多依赖管理知识。

## 15.5. 定义task执行顺序
*task排序是一个发展中特性，请留意此特性在未来的gradle版本中可能有所调整。*

有时候我们想控制两个task的执行顺序，但不想给它们声明强制性的依赖关系。task的执行顺序和依赖关系区别在于：执行顺序规则不会影响到哪些task必须被执行，只影响task们的执行顺序；相比之下依赖关系会导致被依赖task一定会被执行（当然顺序也确定了）。

按顺序执行在很多情景下都很有用：

- 强制task们的执行顺序。例如：'build'不一定要求必须'clean'，但一定不会在'clean'之前运行。
- 在构建中尽早验证先决条件。例如：在创建一个release之前验证是否有正确的证书。
- 在运行耗时验证之前先通过快速验证任务来尽快得到反馈。例如：单元测试应当在集成测试之前运行。
- 一个任务聚合了某一类所有任务的结果。例如：测试报告task组合了所有被执行过的测试任务的输出结果。

目前有两条规则可以使用：“必须在xx之后运行(mustRunAfter)”和“应当在xx之后运行(shouldRunAfter)”。

通过使用“必须在xx后运行”规则，我们可以指定taskB总是在taskA之后执行，而不管taskA和taskB是否真的都会被执行。这个小需求可以通过一句`taskB.mustRunAfter(taskA)`来实现。“应当在xx之后运行”规则也比较类似，但相对不那么严格，在两种情况下会被忽略：一个情况是，如果使用规则导致执行顺序有死循环。另一个情况是，如果使用了并行执行，而且一个task除了shouldRunAfter“依赖”之外所有dependsOn依赖都已就绪，那么这个task就会被执行，而放弃等待它的shouldRunAfter“依赖”们。如果期待task的执行顺序有所帮助但也不是那么严格强制，那可以使用“应当在xx之后运行”特性。

**说了半天，目前使用这些规则仍有可能造成taskA执行但taskB未执行，或者taskB执行但taskA未执行。所以未完成feature请自己评估风险，慎用慎用。**

Example 15.14. Adding a 'must run after' task ordering

build.gradle

    task taskX << {
        println 'taskX'
    }
    task taskY << {
        println 'taskY'
    }
    taskY.mustRunAfter taskX

Example 15.15. Adding a 'should run after' task ordering

build.gradle

    task taskX << {
        println 'taskX'
    }
    task taskY << {
        println 'taskY'
    }
    taskY.shouldRunAfter taskX

以上例子中，运行`gradle -q taskY`即可看到，只执行taskY而不引起taskX的执行也是可以的。

在栗子中可以看到，要为两个task指定执行顺序规则，可使用 [Task.mustRunAfter()](https://gradle.org/docs/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:mustRunAfter(java.lang.Object[])) 和 [Task.shouldRunAfter()](https://gradle.org/docs/current/javadoc/org/gradle/api/Task.html#shouldRunAfter(java.lang.Object[])) 两个方法。这俩方法接受task实例引用、task名称以及其他任何 [Task.dependsOn()](https://gradle.org/docs/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:dependsOn(java.lang.Object[])) 方法所接受的参数。

注意`B.mustRunAfter(A)`或者`B.shouldRunAfter(A)`并不会意味着两个任务有任何依赖关系。

- 可以独立的运行A或者B。顺序规则只生效于它俩一起被运行的时候。
- 如果使用`--continue`参数，A失败与否B都会在A之后执行。

如前所述，如果“应该在xx之后运行”规则引入了顺序循环，那么它将会被忽略。

Example 15.17. A 'should run after' task ordering is ignored if it introduces an ordering cycle

build.gradle

    task taskX << {
        println 'taskX'
    }
    task taskY << {
        println 'taskY'
    }
    task taskZ << {
        println 'taskZ'
    }
    taskX.dependsOn taskY
    taskY.dependsOn taskZ
    taskZ.shouldRunAfter taskX

## 15.6. 为task添加描述信息
可以给task添加描述信息，这个信息将会在执行`gradle tasks`列出task列表时显示在task名之后。

Example 15.18. Adding a description to a task

build.gradle

    task copy(type: Copy) {
       description 'Copies the resource directory to the target directory.'
       from 'resources'
       into 'target'
       include('**/*.txt', '**/*.xml', '**/*.properties')
    }

## 15.7. 替换task
有时候我们想偷天换日替换掉某个task。例如，如果java插件提供了某task但我们对此表示非常不满，想用一个自定义的不同的task来替换掉它。可以这样实现：

Example 15.19. Overwriting a task

build.gradle

    task copy(type: Copy)
    
    task copy(overwrite: true) << {
        println('I am the new one.')
    }

这将导致一个Copy类型的task被我们后来提供的另一个同名task所替换。定义新task的时候需要指定`overwrite`属性为true，否则gradle将抛出一个异常告诉你同名的task已经存在。

## 15.8. 跳过task
Gradle提供多种方式来跳过某task不执行。

### 15.8.1. 使用断言
我们可以使用`onlyIf()`方法给task增加一个断言。只有断言为true的时候task才会被执行。`onlyIf()`方法需要我们提供一个闭包作为参数，闭包返回true表示应当执行这个task，返回false表示应当跳过这个task。断言的执行时机是方法即将被执行之前。以下例子使用命令行` gradle hello -PskipHello`查看效果。

Example 15.20. Skipping a task using a predicate

build.gradle

    task hello << {
        println 'hello world'
    }
    
    hello.onlyIf { !project.hasProperty('skipHello') }

### 15.8.2. 使用 StopExecutionException
如果跳过一个task的逻辑无法使用闭包来表达，可以使用 [StopExecutionException](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/StopExecutionException.html) 。如果task的某个action抛出了这个异常，当前action的剩余代码以及未来所有需要执行的action都将被跳过。构建脚本将按原计划继续执行下一个task。

Example 15.21. Skipping tasks with StopExecutionException

build.gradle

    task compile << {
        println 'We are doing the compile.'
    }
    
    compile.doFirst {
        // Here you would put arbitrary conditions in real life.
        // But this is used in an integration test so we want defined behavior.
        if (true) { throw new StopExecutionException() }
    }
    task myTask(dependsOn: 'compile') << {
       println 'I am not affected'
    }

如果我们使用Gradle提供的task，这个功能就非常有用。它允许我们向一个task的内建action里添加执行条件。

附带一提，也许你想知道为什么既没见着导入StopExecutionException也没见着使用全限定名org.gradle.api.tasks.StopExecutionException，就可以直接用？其实就像java自动导入java.lang包一样，Gradle也会自动导入很多包。而且还可以定制自动导入哪些包，可以学习附录E来看如何配置。

### 15.8.3. 激活和禁用task
每一个task都有一个enabled标志，默认为true。设为false则可以直接废掉整个task。

Example 15.22. Enabling and disabling tasks

build.gradle

    task disableMe << {
        println 'This should not be printed if the task is disabled.'
    }
    disableMe.enabled = false

## 15.9. 跳过处于最新状态的task
如果我们使用Gradle提供的task，例如java插件引入的那些task，经常会看到Gradle执行某个task，在后面注明 **UP-TO-DATE** 然后就迅速跳过整个task。这是gradle提供的一项优化，当一个task的输入没有发生任何改变，那么它的输出也不应该有任何改变。栗子：在已经执行过编译task产生class文件输出的情况下，再次执行编译task，Gradle会检查它的输入（所有源码）是否发生了改变，如果没有源码被修改，那就没必要再重编译一次，gradle会把编译task标记为 UP-TO-DATE 并且直接跳过执行。这个行为非常棒，我们的自定义task也可以实现。

### 15.9.1. 声明task的输入和输出
看一个栗子。这里我们的task根据源xml文件产生多个输出文件。先来重复运行几次。

Example 15.23. A generator task

build.gradle

    task transform {
        ext.srcFile = file('mountains.xml')
        ext.destDir = new File(buildDir, 'generated')
        doLast {
            println "Transforming source file."
            destDir.mkdirs()
            def mountains = new XmlParser().parse(srcFile)
            mountains.mountain.each { mountain ->
                def name = mountain.name[0].text()
                def height = mountain.height[0].text()
                def destFile = new File(destDir, "${name}.txt")
                destFile.text = "$name -> ${height}\n"
            }
        }
    }

默认情况下，即使什么都没有发生改变，gradle第二次执行这个task的时候也没有选择跳过。我们的栗子使用一个action闭包定义，Gradle也不知道这个闭包具体干了点啥，也不知道根据什么来指出task是不是处于最新状态。要使用Gradle的up-to-date检查机制，我们需要声明task的输入和输出是什么。

每一个task都有inputs和outputs属性可以让我们声明task的输入和输出是什么。接下来我们修改一下栗子，声明task使用源xml文件作为输入，产生出来的目标目录作为输出。然后再试着多次运行task。

Example 15.24. Declaring the inputs and outputs of a task

build.gradle

    task transform {
        ext.srcFile = file('mountains.xml')
        ext.destDir = new File(buildDir, 'generated')
        inputs.file srcFile
        outputs.dir destDir
        doLast {
            println "Transforming source file."
            destDir.mkdirs()
            def mountains = new XmlParser().parse(srcFile)
            mountains.mountain.each { mountain ->
                def name = mountain.name[0].text()
                def height = mountain.height[0].text()
                def destFile = new File(destDir, "${name}.txt")
                destFile.text = "$name -> ${height}\n"
            }
        }
    }

现在当我们第二次执行task的时候Gradle已经知道通过检查哪些文件来判断task是否处于最新状态了。

task的inputs属性是 [TaskInputs](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/TaskInputs.html) 的实例。，outputs属性是 [TaskOutputs](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/TaskOutputs.html) 的实例。

一个task如果没有指定outputs属性，它永远不会被判定为up-to-date。某些场合下task的输出并不是文件，或者也许有更复杂的场合，可以使用 [TaskOutputs.upToDateWhen()](https://gradle.org/docs/current/javadoc/org/gradle/api/tasks/TaskOutputs.html#upToDateWhen(groovy.lang.Closure)) 方法来计算task是否应当被认为处于最新状态。

如果一个task只指定了output属性，那只要上次构建后输出文件没有发生变化，task就会认为是up-to-date。

### 15.9.2. 怎么实现的？
在task第一次被执行之前，Gradle会对task的输入进行快照。这份快照包括所有输入的文件列表以及每个文件的hash。然后执行task本身。成功执行之后，再对所有输出来个快照，同样包括所有输出文件列表以及每个文件的hash。Gradle会保存这两份快照。

之后每一次执行task之前，Gradle都为输入输出产生一份新的快照。如果新的快照和上次保存的快照完全相同，gradle就认为task处于最新状态，不需要重新执行。如果有不同，那就执行task，然后为输入输出取得新的快照并保存下来以供下次判断。

注意如果一个task的输出包括一个目录，在上次执行之后任何文件添加到这个目录都会被忽略，当然也不会导致task被判为状态过期。这个特性可以让不同的task使用同一个输出目录而不至于互相影响。如果你觉得这个行为不爽，请考虑使用TaskOutputs.upToDateWhen()方法自己写判断逻辑。

## 15.10. 根据规则动态创建task
有时候你想让一个task的行为依赖一个巨量甚至根本就是无限的参数集。一种更优雅的方式是使用规则来创建一堆task。

Example 15.25. Task rule

build.gradle

    tasks.addRule("Pattern: ping<ID>") { String taskName ->
        if (taskName.startsWith("ping")) {
            task(taskName) << {
                println "Pinging: " + (taskName - 'ping')    //groovy语法：字符串可以相减，用来去除相匹配的子串
            }
        }
    }
    
可以使用命令行`gradle -q pingServer1`来查看效果。当脚本被执行的时候，查找名叫pingServer1的task当然不会直接找到，这时候就会去查找规则列表。符合以'ping'前缀的条件，一个task就被动态的创建出来。addRules方法的字符串参数用来描述这条规则。`gradle tasks`时候会显示出来。

规则不仅仅可以用于命令行执行task。建立task间依赖关系的时候也可以使用基于规则的task。运行`gradle -q groupPing`收看效果。

Example 15.26. Dependency on rule based tasks

build.gradle

    tasks.addRule("Pattern: ping<ID>") { String taskName ->
        if (taskName.startsWith("ping")) {
            task(taskName) << {
                println "Pinging: " + (taskName - 'ping')
            }
        }
    }
    
    task groupPing {
        dependsOn pingServer1, pingServer2
    }

如果使用`gradle tasks`命令是看不到pingServer1和pingServer2的，但脚本的执行逻辑确实用到了这两个“动态”task。

## 15.11. 清道夫task
**警告，这又是一个孵化中特性。看看就好不要太当真。**

如果需要被清理的task会被执行，它的清道夫task也会自动添加到task执行计划表中。

Example 15.27. Adding a task finalizer

build.gradle

    task taskX << {
        println 'taskX'
    }
    task taskY << {
        println 'taskY'
    }
    
    taskX.finalizedBy taskY

使用`gradle -q taskX`执行taskX之后，不管taskX成功还是失败，它的清道夫taskY都会被自动执行。

Example 15.28. Task finalizer for a failing task

build.gradle

    task taskX << {
        println 'taskX'
        throw new RuntimeException()
    }
    task taskY << {
        println 'taskY'
    }
    
    taskX.finalizedBy taskY

另一方面，如果被清理task在执行计划中，但没有做任何事情（比如up-to-date跳过了，或者它的依赖任务失败导致它还没机会运行），它的清道夫task也不会执行。

某些场合清道夫task会很有用，比如说无论构建成功还是失败都得有人来清理一下产生出来的狼藉现场。一个更具体的栗子：测试task跑完以后，不管测试用例执行成功还是失败，为了测试而启动的web容器都应该关闭掉。

Task.finalizedBy()方法需要指定一个task作为清道夫。方法可接受task实例引用、task名称字符串等所有和Task.dependsOn()方法一样的参数。

## 15.12. 小结
如果你刚从Ant转移过来，像Copy这种Gradle的增强型task有点类似于Ant里target（目标）和task（任务）概念的混合体。尽管Ant的task和target实在是不同的玩意，Gradle还是把这两者概念结合了起来。普通的task类似于Ant的target，而增强的task还包含了Ant里task的某些方面。所有Gradle的task共享一套通用的API，你可以创建它们之间的依赖关系。相比Ant的task，Gradle的task更容易配置和使用，它们充分利用了Groovy的类型系统，更具有表现力也更易于维护。
