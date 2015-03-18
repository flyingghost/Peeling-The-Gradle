Chapter 7. Java构建快速入门
===
7.1. Java插件
---
Gradle是一个通用的构建工具。我们先来编译一下Java。

大部分Java工程都比较类似，编译java源文件、运行单元测试、创建一个JAR包等。Gradle当然不会像ant一样需要你为每个工程都写脚本去做这些事。Gradle使用**插件**的形式来解决所有问题。

和很多小内核架构的设计类似（比如eclipse），Gradle也是由一个什么活都不会做的内核，以及众多负责具体工程编译的插件组成。一个插件就是一个gradle的扩展，它以某种结构组织你的工程，然后为你的工程添加一些预定义好的task。Gradle自带了很多常用插件，你也可以扩展实现自己的插件。我们这章要使用的就是**Java plugin**，主要负责编译、单元测试、jar打包等java工程的常见任务。

前面我们已经提过，gradle是基于约定的。Java插件也是，也就是说插件为java工程的很多常见属性都做了预定义配置，比如说java源文件在哪个目录。如果你了解并遵守了这些约定，那根本就不用写什么配置代码就可以完成构建工作下班回家了。当然如果你不愿意，也可以随便改这些配置项。这就是所谓的**“约定优于配置”**，说白了就是**”所有配置项都有默认值“**而已。

7.2. 一个最基础的java工程
---
这一节我们来搞个大项目：一个典型java工程的编译脚本。

    apply plugin: 'java'   

额。。。完了？我加班泡面和熬夜香烟都准备好了，就一行就搞完了？

就是这么简单。这行代码声明使用java插件。然后插件内的一大波task就向你涌来。你可以在当前目录下使用`gradle tasks`命令行查看所有任务们。随着task涌来的还有默认配置们。java插件约定：

- `src/main/java` 产品源代码
- `src/test/java` 测试源代码
- `src/main/resources` 资源，会被打入jar包
- `src/test/resources` 测试资源，运行测试的时候自动使用
- `build` 所有编译输出文件都会在这个目录里
- `build/libs` jar文件会到这里

所以如果你的工程结构“恰好”也是这么组织的（事实上这个工程结构继承自Maven，并且得到了大量java工程的使用），那最省事，apply一个插件就万事俱备了。

###7.2.1. 构建工程
要想构建工程，只需执行java插件带来的任务`build`即可，`build`将会对工程做一次全量构建，包括编译、执行单元测试、打一个包含了class和资源的jar包等。

可以命令行运行`gradle build`看看输出，会详细列出被执行过的task名称。一次复杂的编译就一行脚本完成了。

还有一些比较有用的task：

- clean，清除所有构建输出（默认为build目录）。注意，基于以下假设**”如果所有输入不变，操作也不变，那么输出一定不会发生变化“**，gradle并不会默认帮你每次清除已经存在的构建成果，它只检查每一个task，如果这个task需要的所有输入（比如compileJava的输入就是所有java源代码）没有发生任何变化，那就无需再来一次重复工作，直接输出`UP-TO-DATE`就返回了。如果需要请自行执行clean。
- assemble，编译并打包，但不执行测试。
- check，编译并执行测试。很多插件会扩展这个任务，比如说常见的”代码风格检查“类插件。

###7.2.2. 外部依赖
一般来说java工程都会有一些外部依赖jar包，你得告诉gradle依赖哪些东西，去哪里找。在gradle中，这些jar包都会被放在仓库中。仓库不仅可以放依赖jar包，还可以放工程发布物。例如，我们可以使用java圈最著名的依赖管理仓库Maven：

    repositories {
        mavenCentral()
    }

然后加一些依赖项：

    dependencies {
        compile group: 'commons-collections', name: 'commons-collections', version: '3.2'
        testCompile group: 'junit', name: 'junit', version: '4.+'
    }

以上表明工程代码将编译期依赖commons-collections库，测试代码将依赖junit库。

###7.2.3. 定制工程属性
Java插件添加了一大堆自带默认值的属性，一般情况下可以省了我们的大量重复配置工作。当某属性的默认值不合适时，也可以定点调整。

    sourceCompatibility = 1.5
    version = '1.0'
    jar {
        manifest {
            attributes 'Implementation-Title': 'Gradle Quickstart',
                       'Implementation-Version': version
        }
    }

以上修改了源码兼容性、工程version，并在jar的manifest里增加了两项值。有多少属性可以供你定制？可以运行`gradle properties`命令来查看清单。

由java插件添加进来的task就是标准的task，和我们自己声明的task没有任何不同。所以我们也可以使用之前介绍过的任何方法来操作这些插件导入带来的task。例如，设置task的属性，为task添加行为，改变task的依赖关系，或者把task整个换掉。下例将给test任务添加一个属性。

    test {
        systemProperties 'property': 'value'
    }

###7.2.4. 发布JAR包
当你需要把工程发布的时候，你得告诉gradle发布到哪里去。一般在gradle中jar这种产出物都会发布到仓库里。举个栗子，发布到本地仓库：

    uploadArchives {
        repositories {
            flatDir {
                dirs 'repos'
            }
        }
    }

运行`gradle uploadArchives`就可以发布到本地的`repos`目录下。打开目录可以看到，这是一种“平的”ivy仓库。不建议手动管理这个目录的内容。

###7.2.5. 创建一个Eclipse工程
本小节教你如何使用gradle脚本创建一个Eclipse的工程配置文件(.project)。但是！

理论上一个工程应当只有gradle构建脚本，而不关注开发人员到底是用什么IDE打开/导入了工程。当你用这种方法生成eclipse工程文件之后，就不得不注意.project文件和gradle编译脚本之间的同步问题。因为gradle和.project文件总有一个是源，一个是产出物。产出物怎么会被保存、被上传呢？所以决定不讨论本节内容。给出的替代方案是：你应该给你的IDE添加gradle配置导入能力。IDEA自带，Eclipse可以安装插件。

###7.2.6. 综上
本小节把7.2节所有代码片段做了一个综合。有骗字数之嫌。有节操的我保持掠过姿态。

7.3. 多工程的java项目构建
---
典型的项目都是由多个工程组合而成的。假设我们有这样一个工程结构：

    multiproject/
      api/
      services/webservice/
      shared/
      services/shared/

这里有4个工程：
1. api工程产生一个jar文件，webservice用到。
2. webservice是一个webapp。
3. shared是一些基础类库，webservice和api都用到。
4. services/shared也依赖shared。

###7.3.1. 定义一个多工程构建
一个多工程的构建需要在工程根目录存在一个`settings.gradle`文件来声明需要导入哪些工程。此文件名固定不能改。

    include "shared", "api", "services:webservice", "services:shared"

默认的工程树结构和文件夹结构保持一致。根工程使用名称":"，其后工程名称之间使用:作为分隔符。所以你可以写`include "shared"`也可以写`include ":shared"`，效果一样。

###7.3.2. 通用配置
大部分多工程项目，各子工程都有一些通用的配置。在我们的例子中，我们将在根工程中定义这些通用配置项，这叫做“配置注入”。这时主工程就像一个容器，通过subprojects方法得到它的所有子工程，并注入一些通用配置项。

    subprojects {
        apply plugin: 'java'
        apply plugin: 'eclipse-wtp'
    
        repositories {
           mavenCentral()
        }
    
        dependencies {
            testCompile 'junit:junit:4.11'
        }
    
        version = '1.0'
    
        jar {
            manifest.attributes provider: 'gradle'
        }
    }

如以上脚本，给所有的子工程都设置了maven库和依赖项，并且在jar中设置了通用的manifest项。注意，apply一堆插件是给各子工程的，主工程并不受影响。

###7.3.3. 各工程间的依赖
如7.3所述，各工程之间有依赖关系，意味着api工程在编译时需要一个前提：它的依赖工程shared已经被编译完毕。通过设定工程间依赖，gradle可以自动的做到这一点。

    dependencies {
        compile project(':shared')
    }

当然如前面所述，如果检查shared工程发现已经编译完毕并且没有任何变更，那它只会输出`UP-TO-DATE`然后就pass这一步了。这个规则对于任何task都有效。

###7.3.4. 创建一个发布

    task dist(type: Zip) {
        dependsOn spiJar
        from 'src/dist'
        into('libs') {
            from spiJar.archivePath
            from configurations.runtime
        }
    }
    
    artifacts {
       archives dist
    }

这段代码槽点多多，待日后功力大增后再来回顾。

7.4. Where to next?
---
本章大概讲了java插件的基础使用。很不透彻，23章将会专篇讲更多关于java插件的使用。并且在gradle发布包的sample/java目录下有大量栗子可以学习。
