Chapter 8. 依赖管理基础
===

依赖管理是很重要滴。

8.1. 依赖管理是什么?
---
大致的说，Gradle所谓广义的依赖管理涵盖两个方面：首先，Gradle需要知道你的工程编译或者运行需要依赖哪些东西。我们把这部分输入文件叫做工程的**依赖项dependencies**。其次，Gradle构建你的工程，得到一些产出并上传/发布到某地。我们把这部分输出文件叫做工程的***产出物publications**。因为gradle喜欢把依赖和产出都放在同一个库中，并且某工程的产出可能正好就是另一工程的依赖，二者之间本身就区别不大，所以Gradle把它们统一归类为“dependency management”。

大多数工程都不是自包含的。编译和运行需要很多外部资源，它们就是我们的工程的“依赖”。Gradle需要你告诉它依赖哪些东西，以及去哪里找（可能是来自于远程Maven库，或者一个本地目录，或者是某个被依赖工程产生出来的）。找到之后，编译或者运行才可以继续下去。我们把这个过程叫做**“依赖解决dependency resolution”**。

我们只需要声明我们项目的依赖项，但通常来说，依赖项自己也有它们的依赖。所以Gradle还需要挨个查找并获取依赖的依赖们。我们把这个叫做**“传递依赖transitive dependencies”**

大部分项目的目标都是构建一些文件扔出去供项目外某场合使用。这些输出文件就是我们项目的“产出物”。你只要声明好项目的产出，Gradle负责构建并发布这些东西到某个地方。我们把这个过程叫做**“发布publication”**。

废话一通，几个简单的概念名词解释完毕，开始实战。

8.2. 声明工程依赖
---
    apply plugin: 'java'
    
    repositories {
        mavenCentral()
    }
    
    dependencies {
        compile group: 'org.hibernate', name: 'hibernate-core', version: '3.6.7.Final'
        testCompile group: 'junit', name: 'junit', version: '4.+'
    }

这段代码声明了工程的两项依赖：Hibernate core（3.6.7.Final版）为工程的编译期依赖项，同时隐式的也就包含了Hibernate本身的所有依赖项们。还有一项junit的任意4.0以上版本为测试用例的依赖项。这些依赖项都会去MavenCentral仓库里去找。

8.3. 依赖配置
---
在Gradle中，依赖被分类存放在不同的配置里。换句话说，一个配置就是一个命名的依赖集合。以后我们将看到，配置也被用来管理项目的发布物。

java插件预定义了几个标准配置，不同的配置表现为java插件使用的不同的classpath。

- **compile** 编译项目源码需要的依赖项。
- **runtime** 项目在运行期需要的依赖。默认也包括所有编译期依赖项。
- **testCompile** 编译项目测试用例需要的依赖项。默认也包括项目的编译期依赖以及项目编译好的class文件。
- **testRuntime** 运行测试用例需要的依赖项。默认也包括项目编译期依赖、运行期依赖、测试用例编译期依赖。

很多插件都提供了更多的标准配置。我们也可以定义自己的配置。很久很久以后，51.3节会讲到这部分内容。

##8.4. 外部依赖
有很多类依赖可以声明。其中一类刚才见过，就是“外部依赖”。顾名思义是在我们的构建之外已经构建好的，存在某个仓库里，或者某个文件夹里。

    dependencies {
        compile group: 'org.hibernate', name: 'hibernate-core', version: '3.6.7.Final'
    }

一个外部依赖使用3个属性来标识：group、name、version。name是必须的，group和version可以作为可选项，取决于使用什么样的仓库。

省略写法是一个字符串"group:name:version"。

    dependencies {
        compile 'org.hibernate:hibernate-core:3.6.7.Final'
    }

##8.5. 仓库
外部依赖哪里找？Gradle需要我们指定一个仓库。仓库实际上就是用group/name/version组织起来的一堆文件。Gradle可以使用Maven、Ivy等几种仓库格式，并且可以以HTTP、本地文件夹等多种途径去存取仓库。

默认Gradle不会假定任何仓库。你需要给它至少指定一个仓库。最常见的就是Maven central库（这是由maven.org搭建的全球中心库，局域网访问极慢）：

    repositories {
        mavenCentral()
    }

或者一个maven远程库（一般由你公司内部自行搭建）：

    repositories {
        maven {
            url "http://repo.mycompany.com/maven2"
        }
    }

OSChina为了造福广大局域网用户，创建了一个maven镜像库：http://maven.oschina.net/content/groups/public/。1024好人一生平安！

也可以研究一下Ivy，一个Apache出品的依赖管理库，gradle的默认库。

    repositories {
        ivy {
            url "http://repo.mycompany.com/repo"
        }
    }

当然也可以在本地文件系统指定一个目录当做仓库。Maven和Ivy都可以这么干。

    repositories {
        ivy {
            // URL can refer to a local directory
            url "../local-repo"
        }
    }

一个工程可以使用多个仓库。Gradle会依次在每个库里找依赖，找到为止。是否可以使用本地目录、公司仓库、中心库做级联？留给同学们自己折腾。

##8.6. 发布artifacts
依赖配置也用于发布文件（Gradle开发者也觉得这么设定很绕人，所以正在尝试剥离这两个用途）。这些产出文件被叫做**publication artifacts**，经常也被简称为**artifacts**。（这货到底怎么翻译合适？愁煞人。）

关于生成什么怎么生成，插件通常都已经处理的很好了，不用我们操心太多。但我们还是要告诉Gradle，这些artifacts将要发布到哪里去。然后可以使用uploadArchives这个task来塞到库里去。举个塞到Ivy库的栗子：

    uploadArchives {
        repositories {
            ivy {
                credentials {
                    username "username"
                    password "pw"
                }
                url "http://repo.mycompany.com"
            }
        }
    }

执行`gradle uploadArchives`命令，Gradle自动编译打包并上传你的jar文件，顺便还生成并且上传了一个ivy.xml描述文件，可谓体贴入微。

你也可以发布到Maven库。语法和Ivy稍微有点不同，开发人员正在努力统一。记住发布到Maven库还需要使用`maven`插件。上传完成后当然也会生成并上传pom.xml文件。

    apply plugin: 'maven'
    
    uploadArchives {
        repositories {
            mavenDeployer {
                repository(url: "file://localhost/tmp/myRepo/")
            }
        }
    }

##8.7. 下一步干嘛？
前面讲的统统都是概述。51章依赖管理，52章artifacts发布，都会讲得更详细。

如果对上文中提到的一些DSL元素感兴趣，可以去Gradle DSL文档围观一下 Project.configurations{}，Project.repositories{} 以及 Project.dependencies{} 几个元素的详细文档。

