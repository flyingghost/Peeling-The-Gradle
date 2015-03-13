# Chapter 16. 文件操作

大多数构建都是在一堆文件上折腾。Gradle添加了很多文件相关的概念和API来支持。

## 16.1. 定位文件
我们可以使用`Project.file()`方法传入相对于工程目录的相对路径定位一个文件。

Example 16.1. Locating files

build.gradle

    // Using a relative path
    File configFile = file('src/config.xml')
    
    // Using an absolute path
    configFile = file(configFile.absolutePath)
    
    // Using a File object with a relative path
    configFile = file(new File('src/config.xml'))

你可以给file()方法传入任何对象，它会尝试将其转换为一个绝对路径的File对象。通常情况下我们会传入一个String或者File的实例。如果传入path是一个绝对路径，直接产生对应File对象。如果是相对路径，会基于工程目录来处理相对路径并得到最终File对象。file()方法也可以处理形如`file:/some/path.xml/`的URL。

使用这种方法可以很方便的把某些用户输入转换为绝对路径的File对象。file()方法总是基于工程目录（这个是固定的）来计算相对路径，而“当前工作目录”会取决于用户如何执行gradle而改变。所以如果相对工程目录固定可以使用file()，否则最好使用File(somePath)。

## 16.2. 文件集合
简单的说文件集合就是一堆文件，它通过[FileCollection](https://gradle.org/docs/current/javadoc/org/gradle/api/file/FileCollection.html)接口来表示。Gradle API里很多对象都实现了这个接口。例如：[依赖配置](https://gradle.org/docs/current/userguide/dependency_management.html#sub:configurations)实现了FileCollection接口。

获取一个FileCollection实例的一种方法是使用[Project.files()](https://gradle.org/docs/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:files(java.lang.Object[]))方法。我们可以向这个方法传入任意个对象，而它们会被转换为一组File对象。files()方法接受任意类型的对象作为参数。它们会被作为相对于工程目录的相对路径来处理并得到File对象，就像上一节对file()方法描述的一样。也可以把集合、可迭代对象、map、数组等传递给files()方法。它们会被展开，每一项分别转为File对象。

Example 16.2. Creating a file collection

build.gradle

    FileCollection collection = files('src/file1.txt',
                                      new File('src/file2.txt'),
                                      ['src/file3.txt', 'src/file4.txt'])

一个文件集合可以被迭代枚举，并且可以使用`as`操作符转为各种其他类型。还可以使用 + 操作符取得两个集合的并集，或者使用 - 操作符取得两个集合的交集。以下是一些栗子，展示我们可以对文件集合做哪些操作。

Example 16.3. Using a file collection

build.gradle

    // Iterate over the files in the collection
    collection.each {File file ->
        println file.name
    }
    
    // Convert the collection to various types
    Set set = collection.files
    Set set2 = collection as Set
    List list = collection as List
    String path = collection.asPath
    File file = collection.singleFile
    File file2 = collection as File
    
    // Add and subtract collections
    def union = collection + files('src/file3.txt')
    def different = collection - files('src/file3.txt')

还可以给files()方法传递一个闭包或一个Callable对象的实例，返回一个集合。当集合的内容被查询时，闭包会被真正执行，然后返回值被转化为File的集合。返回值可以是一个对象，或者任何files()方法支持的玩意。这是一个“实现”FileCollection接口的简单方法。这段话比较绕，读不懂没关系，看代码。

Example 16.4. Implementing a file collection

build.gradle

    task list << {
        File srcDir        //只定义了srcDir的类型，为下面的闭包打基础，这里并没有真正赋值
    
        // Create a file collection using a closure
        collection = files { srcDir.listFiles() }    //返回了集合，但这个集合的内容并没有真正确定，只是挂接了一个闭包
    
        srcDir = file('src')        //确定了srcDir的值
        println "Contents of $srcDir.name"
        collection.collect { relativePath(it) }.sort().each { println it }    //collect方法导致内容被查询，才执行闭包，返回了srcDir下的所有文件列表，然后依次由relativePath求得相对路径，形成真正的内容集合（一堆相对路径），再sort，再each遍历输出。Groovy的灵活性体现。
    
        srcDir = file('src2')
        println "Contents of $srcDir.name"
        collection.collect { relativePath(it) }.sort().each { println it }
    }

还可以给files()方法喂食其他类型的对象：

- FileCollection，会被展开，内容放进文件集合。
- Task，task定义的输出文件们放进文件集合。
- TaskOutputs，同上，输出文件们放进文件集合。

注意，在上一个栗子可以看到，一个file集合的实际内容将尽可能晚的计算，直到需要的时候才去计算到真正的内容。这意味着我们可以创建一个FileCollection来代表某个File集合的逻辑上的概念，而File集合在未来才可能由某个task创建出来，然后才计算出FileCollection的真正内容。

## 16.3. 文件树
文件树是按照层次结构组织的一组文件集合。例如，一个文件树可以表达一个目录树结构，或者一个zip包的内容。文件树由 [FileTree](https://gradle.org/docs/current/javadoc/org/gradle/api/file/FileTree.html) 接口表达。FileTree接口继承自FireCollection接口，所以可以在FileCollection上做的事情也可以在FileTree上操作。Gradle里有很多对象实现了FileTree接口，例如 [source sets源文件集](https://gradle.org/docs/current/userguide/java_plugin.html#sec:source_sets)。

获取FileTree实例的途径之一是调用Project.fileTree()方法，它将创建一个基于某目录的FileTree对象，并且支持Ant风格的include和exclude表达模式。

Example 16.5. Creating a file tree

build.gradle

    // Create a file tree with a base directory
    FileTree tree = fileTree(dir: 'src/main')
    
    // Add include and exclude patterns to the tree
    tree.include '**/*.java'
    tree.exclude '**/Abstract*'
    
    // Create a tree using path
    tree = fileTree('src').include('**/*.java')
    
    // Create a tree using closure
    tree = fileTree('src') {
        include '**/*.java'
    }
    
    // Create a tree using a map
    tree = fileTree(dir: 'src', include: '**/*.java')
    tree = fileTree(dir: 'src', includes: ['**/*.java', '**/*.xml'])
    tree = fileTree(dir: 'src', include: '**/*.java', exclude: '**/*test*/**')

使用文件树的方式和使用文件集合的方式相同，还可以用Ant风格的表达模式访问树的内容，或者选择一个子树。

Example 16.6. Using a file tree

build.gradle

    // Iterate over the contents of a tree
    tree.each {File file ->
        println file
    }
    
    // Filter a tree
    FileTree filtered = tree.matching {
        include 'org/gradle/api/**'
    }
    
    // Add trees together
    FileTree sum = tree + fileTree(dir: 'src/test')
    
    // Visit the elements of the tree
    tree.visit {element ->
        println "$element.relativePath => $element.file"
    }

## 16.4. 把压缩包内容直接当做文件树来用
可以使用Proejct.zipTree()或Project.tarTree()方法，把ZIP或者TAR等格式的压缩包所有内容作为一个文件树。这些方法返回一个FileTree实例，你可以把它当做普通的文件树或者文件集合，就像来自于文件系统一个目录一样。例如，你可以通过它来解压整个压缩包，拷贝其中的某个文件，或者把压缩包和另一个压缩包合并。

Example 16.7. Using an archive as a file tree

build.gradle

    // Create a ZIP file tree using path
    FileTree zip = zipTree('someFile.zip')
    
    // Create a TAR file tree using path
    FileTree tar = tarTree('someFile.tar')
    
    //tar tree attempts to guess the compression based on the file extension
    //however if you must specify the compression explicitly you can:
    FileTree someTar = tarTree(resources.gzip('someTar.ext'))

## 16.5. 指定输入文件集
Gradle中很多对象都有属性可以接受一组文件作为输入。例如，[JavaCompile](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.compile.JavaCompile.html)任务有一个source属性，定义了需要编译的源文件。你可以使用任何file()方法支持的类型设置给source属性。也就是说，可以使用File、String、集合、FileCollection甚至闭包。下文列出一些栗子：

Example 16.8. Specifying a set of files

build.gradle

    // Use a File object to specify the source directory
    compile {
        source = file('src/main/java')
    }
    
    // Use a String path to specify the source directory
    compile {
        source = 'src/main/java'
    }
    
    // Use a collection to specify multiple source directories
    compile {
        source = ['src/main/java', '../shared/java']
    }
    
    // Use a FileCollection (or FileTree in this case) to specify the source files
    compile {
        source = fileTree(dir: 'src/main/java').matching { include 'org/gradle/api/**' }
    }
    
    // Using a closure to specify the source files.
    compile {
        source = {
            // Use the contents of each zip file in the src dir
            file('src').listFiles().findAll {it.name.endsWith('.zip')}.collect { zipTree(it) }
        }
    }

通常情况下都有一个与属性相同名称的方法，可以用来追加文件到已有的文件集。再次申明，这个方法接受任何files()方法支持的参数类型。

Example 16.9. Specifying a set of files

build.gradle

    compile {
        // Add some source directories use String paths
        source 'src/main/java', 'src/main/groovy'
    
        // Add a source directory using a File object
        source file('../shared/java')
    
        // Add some source directories using a closure
        source { file('src/test/').listFiles() }
    }

## 16.6. 复制文件
可以用[Copy](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.Copy.html) task来复制文件。复制task非常灵活，并且允许过滤需要拷贝的文件，或者修改文件名称。

要使用拷贝task，你必须提供需要拷贝的文件集，以及拷贝到哪里去的目标目录。还可以指定在拷贝过程中如何转换文件。以上都是通过”拷贝说明“来实现。一份拷贝说明由[CopySpec](https://gradle.org/docs/current/javadoc/org/gradle/api/file/CopySpec.html)接口来描述。Copy task实现了这个接口。可以通过[CopySpec.from()](https://gradle.org/docs/current/javadoc/org/gradle/api/file/CopySpec.html#from(java.lang.Object[]))方法来指定源文件，用[CopySpec.into()](https://gradle.org/docs/current/javadoc/org/gradle/api/file/CopySpec.html#into(java.lang.Object))方法来指定目标目录。

Example 16.10. Copying files using the copy task

build.gradle

    task copyTask(type: Copy) {
        from 'src/main/webapp'
        into 'build/explodedWar'
    }

from()方法接受任何files()方法能接受的参数。如果一个参数被认为是目录，目录下所有子目录和文件（但不包含此目录本身）都将会被递归的拷贝到目标目录。如果一个参数被认为是文件，这个文件本身将被拷贝到目标目录。如果一个参数被认为是个不存在的文件，那就忽略啥也不干。如果参数被认为是一个task，那么它的输出文件（例如task创建的文件）将被拷贝到目标目录，并且task本身也被自动作为Copy task的依赖项添加进来。into()方法也一样接受任何file()能接受的参数。再看个栗子：

Example 16.11. Specifying copy task source files and destination directory

build.gradle

    task anotherCopyTask(type: Copy) {
        // Copy everything under src/main/webapp
        from 'src/main/webapp'
        // Copy a single file
        from 'src/staging/index.html'
        // Copy the output of a task
        from copyTask
        // Copy the output of a task using Task outputs explicitly.
        from copyTaskWithPatterns.outputs
        // Copy the contents of a Zip file
        from zipTree('src/main/assets.zip')
        // Determine the destination directory later
        into { getDestDir() }
    }

可以使用Ant风格的过滤描述模式来选择哪些文件需要拷贝，也可以使用Groovy风格的闭包。

Example 16.12. Selecting the files to copy

build.gradle

    task copyTaskWithPatterns(type: Copy) {
        from 'src/main/webapp'
        into 'build/explodedWar'
        include '**/*.html'
        include '**/*.jsp'
        exclude { details -> details.file.name.endsWith('.html') &&
                             details.file.text.contains('staging') }
    }

也可以使用[Project.copy()](https://gradle.org/docs/current/dsl/org.gradle.api.Project.html#org.gradle.api.Project:copy(groovy.lang.Closure))方法来复制文件。它工作方式和task类似，但有一些限制。

首先，copy()不能支持增量复制。如果使用Copy task，可以通过判断是否UP-TO-DATE来跳过不必要的复制，但copy()方法只能粗暴复制。up-to-date机制可以参见15.9节。

Example 16.13. Copying files using the copy() method without up-to-date check

build.gradle

    task copyMethod << {
        copy {
            from 'src/main/webapp'
            into 'build/explodedWar'
            include '**/*.html'
            include '**/*.jsp'
        }
    }

其次，如果把一个task当做拷贝源（如上文中from()方法介绍），copy()方法也无法建立task依赖关系，毕竟它只是个方法，没有task的那套机制。因此，如果你在一个task的action里调用copy()方法，必须显式的声明所有输入和输出以得到正确的行为。相比之下Copy任务自带输入输出就不用再指定了。

Example 16.14. Copying files using the copy() method with up-to-date check

build.gradle

    task copyMethodWithExplicitDependencies{
        // up-to-date check for inputs, plus add copyTask as dependency
        inputs.file copyTask
        outputs.dir 'some-dir' // up-to-date check for outputs
        doLast{
            copy {
                // Copy the output of copyTask
                from copyTask
                into 'some-dir'
            }
        }
    }

所以，大多数情况下还是推荐尽可能使用拷贝task的，它自带增量拷贝，task依赖判断机制，不需要你额外做什么。copy()方法更适合于作为一个task行为实现的一部分，专门拷贝文件。也就是说，copy()方法适合在自定义task（58章讲自定义Task）里作为功能的一部分在需要拷贝文件时使用。在这种情况下，自定义task得充分声明和拷贝动作有关的输入输出集。

### 16.6.1. 文件重命名

Example 16.15. Renaming files as they are copied

build.gradle

    task rename(type: Copy) {
        from 'src/main/webapp'
        into 'build/explodedWar'
        // Use a closure to map the file name
        rename { String fileName ->
            fileName.replace('-staging-', '')
        }
        // Use a regular expression to map the file name
        rename '(.+)-staging-(.+)', '$1$2'
        rename(/(.+)-staging-(.+)/, '$1$2')
    }

### 16.6.2. 过滤文件

Example 16.16. Filtering files as they are copied

build.gradle

    import org.apache.tools.ant.filters.FixCrLfFilter
    import org.apache.tools.ant.filters.ReplaceTokens
    
    task filter(type: Copy) {
        from 'src/main/webapp'
        into 'build/explodedWar'
        // Substitute property tokens in files
        expand(copyright: '2009', version: '2.3.1')
        expand(project.properties)
        // Use some of the filters provided by Ant
        filter(FixCrLfFilter)
        filter(ReplaceTokens, tokens: [copyright: '2009', version: '2.3.1'])
        // Use a closure to filter each line
        filter { String line ->
            "[$line]"
        }
    }

### 16.6.3. 使用CopySpec类
复制说明可以构造一个层级结构。一份复制说明可以包括目标路径，包含模式，排除模式，拷贝动作，文件名映射，过滤器等。

Example 16.17. Nested copy specs

build.gradle

    task nestedSpecs(type: Copy) {
        into 'build/explodedWar'
        exclude '**/*staging*'
        from('src/dist') {
            include '**/*.html'
        }
        into('libs') {
            from configurations.runtime
        }
    }

## 16.7. 使用同步task
Sync task扩展自Copy task。当它执行时，会拷贝源文件到目标目录，然后移除目标目录里所有不是由它复制的文件，保持源和目标的同步。某些情况下比如安装应用程序、创建压缩包的展开副本，或维护项目依赖关系的副本的时候非常有用。

本例在build/libs目录下维护了一份工程依赖副本：

Example 16.18. Using the Sync task to copy dependencies

build.gradle

    task libs(type: Sync) {
        from configurations.runtime
        into "$buildDir/libs"
    }

## 16.8. 创建归档文件
一个工程可以有很多JAR文件。你也可以根据需要创建WAR、ZIP、TAR等归档文件。归档文件可以被大把相关task创建：[Zip](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.bundling.Zip.html)，[Tar](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.bundling.Tar.html)，[Jar](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.bundling.Jar.html)，[War](https://gradle.org/docs/current/dsl/org.gradle.api.tasks.bundling.War.html)，[Ear](https://gradle.org/docs/current/dsl/org.gradle.plugins.ear.Ear.html)等。它们的工作模式都一样，所以我们只挑Zip看看如何创建一个zip文件。

Example 16.19. Creating a ZIP archive

build.gradle

    apply plugin: 'java'
    
    task zip(type: Zip) {
        from 'src/dist'
        into('libs') {
            from configurations.runtime
        }
    }

归档task工作模式和复制task完全一样，并且也实现了CopySpec接口。像Copy task一样，使用from()方法指定输入文件，并且可以选择是否通过into()方法指定它们在归档文件中的最终位置。可以过滤文档内容，重命名文件，以及其他所有能用拷贝说明做到的事。

### 16.8.1. 归档命名
默认会使用"projectName - version.type"这样的格式来生产归档名称。

Example 16.20. Creation of ZIP archive

build.gradle

    apply plugin: 'java'
    
    version = 1.0
    
    task myZip(type: Zip) {
        from 'somedir'
    }
    
    println myZip.archiveName
    println relativePath(myZip.destinationDir)
    println relativePath(myZip.archivePath)

以上添加了一个名叫myZip的zip归档task，会产生一个ZIP文件zipProject-1.0.zip。注意区分归档task的名称和归档task产生的归档文件的名称是不同的。默认的归档名称可以通过project的`archivesBaseName`属性来设置。文件名也可以在之后随便什么时候修改。

针对归档task有一堆属性可以设置，本章最后的表格会列出来。例如，可以通过修改属性来改变归档文件的名称：

Example 16.21. Configuration of archive task - custom archive name

build.gradle

    apply plugin: 'java'
    version = 1.0
    
    task myZip(type: Zip) {
        from 'somedir'
        baseName = 'customName'
    }
    
    println myZip.archiveName

还可以有更多方法自定义文档名称：

Example 16.22. Configuration of archive task - appendix & classifier

build.gradle

    apply plugin: 'java'
    archivesBaseName = 'gradle'
    version = 1.0
    
    task myZip(type: Zip) {
        appendix = 'wrapper'
        classifier = 'src'
        from 'somedir'
    }
    
    println myZip.archiveName

Table 16.1. Archive tasks - naming properties

|    Property name    |    Type	    |    Default value    |    Description    |
|---------------------|------------|---------------------|-------------------|
|archiveName|String|*baseName-appendix-version-classifier.extension*<br>如果这些属性中任何一个为空，它后面跟着的-号不会添加进来|生成的归档文件名。|
|archivePath|File|*destinationDir*/*archiveName*|生成的归档文件的绝对路径。|
|destinationDir|File|取决于归档文件类型。JAR和WAR会放在*project.buildDir*/libraries。ZIP和TAR会放在*project.buildDir*/distributions。|存放生成的归档文件的目录。|
|baseName|String|*project.name*|归档文件名的基础名称部分。|
|appendix|String|null|归档文件名的附加部分。|
|version|String|*project.version*|归档文件名的版本号部分。|
|classifier|String|null|归档文件名的分类名部分。|
|extension|String|取决于归档文件类型。用于TAR文件。可以是以下压缩类型：zip,jar,war,tar,tgz,tbz2。|归档文件名的扩展名。|

### 16.8.2. 在多个归档文件中共享内容
你可以使用[Project.copySpec()]()方法在归档文件之间共享内容。

你经常会需要发布一个归档文件，以便在另一个工程中使用。这个过程将在52章讲述。说实话这两句话太简单，我也没明白，所以我们52章再见。
