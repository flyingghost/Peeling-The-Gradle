# Chapter 11. 使用Gradle命令行

## 11.1. 运行多个task
你可以使用gradle命令行一次运行多个task。例如，`gradle compile test`命令将按照列出的顺序依次执行compile和test两个任务，并且包含它们的依赖项。不管task是如何参与到执行序列中的（命令行指定、某一个依赖、或二者都是），每个task只会被运行一次。来看个栗子。

如图4个task，dist和test都会依赖compile，但运行`gradle dist test`只会导致compile被运行一次。

![](img/commandLineTutorialTasks.png)

    task compile << {
        println 'compiling source'
    }
    
    task compileTest(dependsOn: compile) << {
        println 'compiling unit tests'
    }
    
    task test(dependsOn: [compile, compileTest]) << {
        println 'running unit tests'
    }
    
    task dist(dependsOn: [compile, test]) << {
        println 'building the distribution'
    }

每个task都只会被运行一次，所以`gradle test test`就跟`gradle test`是一样一样的。

## 11.2. 排除task

可以通过 -x 参数后跟task名称来排除掉某个task被执行（即使它被依赖了）。例如上一个栗子，执行`gradle dist -x test`将只会执行compile和dist两个task，dist的依赖项test没有执行，当然test的依赖compileTest也没有执行。当然compile作为dist的依赖项还是存在的。

## 11.3. 当失败时继续构建

默认来说，当任意task出错时，Gradle都将中止执行过程并立即宣布构建失败。这种机制可以让已经发生错误的构建尽快收工，但实际上更多可能发生的问题并没有暴露出来。为了在一次构建执行过程中发现尽可能多的问题，你可以使用`--continue`选项。

当执行中使用`--continue`选项时，Gradle将在每一个需要执行的task的所有依赖项都正确执行完毕后开始执行这个task，而不是在遇到第一个错误发生时就立即停止整个构建过程。执行过程中遇到的每一个失败都将在构建结束时被报告。

从安全角度而言，如果一个task失败了，所有依赖于它的tasks都将不被执行。例如测试task直接或非直接依赖于编译task，如果一段代码编译失败，覆盖这段代码的测试用例将不会被执行。

## 11.4. task名的简略写法
某些task名称很长，而懒惰作为程序员的天性是不会放过这个施展机会的。我们可以只提供task名的部分前缀，gradle只要足够唯一确定某task，就可以正确执行。例如上面的例子，运行`gradle d`和运行`gradle dist`是等效的。

当然如果提供的信息不足以唯一确认task，gradle不会擅自猜测，而是直接出错提示后结束运行。可以试试`gradle c`的效果。

java世界的习惯是驼峰命名法：首单词小写，以后每单词首字母大写。task也习惯于这样命名。这样的话，可以提供每个单词的首字母，也是一种支持的简略写法。例如：`gradle cT`等效于`gradle compileTest`。

## 11.5. 选择运行build脚本
到目前为止，我们从来没指定过gradle需执行的目标脚本，这样它会默认寻找当前目录下的build.gradle文件。我们也可以使用-b参数指定一个脚本文件名称。**注意，如果使用了-b参数，多工程需要的settings.gradle文件也不会自动使用。**

另外，我们也可以使用-p选项来指定工程目录。

## 11.6. 获取构建信息
gradle内置了一些简单的命令来获取一个构建的信息，在不爬源代码的情况下大致了解一下工程结构什么的还是比较方便的。当然默认就代表粗略，更详细的信息可以使用41章介绍的project report plugin来看看。或者，我等还是看源码比较实在。

### 11.6.1. 工程列表
`gradle projects`命令能以树形形式列出**当前**工程的所有子工程。如果处在某子工程目录里，又想回头看看全景，可使用`gradle :projects`命令。":"号表示了根工程。

列出的工程树干巴巴的没多少信息。实际上我们可以给每个工程至少加上点简述信息，只需在工程build文件里增加一行属性赋值：

    description = 'The shared API for the application'

### 11.6.2. 任务列表
`gradle tasks`命令则可以列出当前工程的所有task，以及它们的简述信息，还会指出工程的默认task是谁。

`gradle tasks -all`则会输出更详细的信息，包括依赖关系等属性。

### 11.6.3. 显示task的使用信息细节
`gradle help --task someTask`可以显示指定task的细节信息。包括在工程结构中的路径、类型、描述等。

### 11.6.4. 工程依赖列表
`gradle dependencies`命令可以列出当前工程的依赖关系，每个配置都输出。如果嫌长，可以使用`--configuration`参数选择只看某个配置。

### 11.6.5. 深入查看某个依赖
`gradle dependencyInsight`可以深入查看某一个具体依赖项的细节。在分析某个依赖从哪里引入，引入了什么版本等信息时非常有用，尤其是工程使用了大量外部依赖项，而不清楚它们背后的子依赖项具体情况时。

### 11.6.6. 工程属性列表
对我等初等菜鸟来说，`gradle properties`的用处才是真的大。它可以列出工程的所有属性。大部分是只读的。但据此我们可以明确在脚本中有哪些属性可以直接利用而不用绞尽脑汁去创造。例如：`buildDir`和`buildFile`属性可以得知构建脚本目录和文件，然后工程内所有文件都可以相对路径得出来了。

### 11.6.7. 度量一个构建过程
`--profile`参数可以在运行构建过程中度量构建过程的执行时间，并在结束后输出到`build/reports/profile`目录。不仅仅是运行阶段的task执行，脚本的配置阶段也会被记录下来，并且按最耗时程度排序（以方便你检讨到底是哪个task写的不合理）。狗屎的android构建真应该检讨一下到底是哪个过程那么坑爹，太愧对google的英名了。

## 11.7. 模拟演习
有时候我们仅仅是有兴趣了解一下过程中到底有哪些task将会以怎样的顺序被执行。老老实实的去跑一遍就太土了。可以使用`-m`选项来搞个模拟演习。例如，运行`gradle -m assembleRelease`可以用来分析创建一个发布包的全部过程，顺藤摸瓜起来比较方便。

## 11.8. 综上
应该已经习惯了吧，本章依然是一个“简介”。详细信息可以查看**附录D：Gradle命令行**深入学习。


























