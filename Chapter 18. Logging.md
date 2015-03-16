# Chapter 18. 日志记录
日志是构建工具的最主要“UI”。如果它太啰嗦，真正的警告和问题很容易被掩埋。另一方面，如果出了错，你需要相关信息才能知道错在哪里。Gradle定义了6个日志级别，如下表所示。除了平时经常看到的那些级别之外，有两个在Gradle里才有的级别：`QUIET`和`LIFECYCLE`。默认级别是`LIFECYCLE`，用来报告构建进度。

Table 18.1. Log levels

|级别|用途|
|-----|--------|
|ERROR|错误信息|
|QUIET|重要信息|
|WARNING|警告信息|
|LIFECYCLE|构建进度信息|
|INFO|普通信息|
|DEBUG|调试信息|

## 18.1. 选择日志级别
你可以使用18.2表中列出的命令行参数来选择不同的日志级别。18.3表中的命令行参数会影响调用栈跟踪日志。

Table 18.2. Log level command-line options

|命令行参数|输出日志级别|
|--------|----------|
|无日志参数|LIFECYCLE及以上级别|
|-q 或 --quiet|QUIET及以上级别|
|-i 或 --info|INFO及以上级别|
|-d 或 --debug|DEBUG及以上级别（也就是所有日志消息）|

Table 18.3. Stacktrace command-line options

|命令行参数|含义|
|--------|------|
|无栈跟踪参数|构建出现错误的时候不会有调用栈跟踪输出到控制台。 只有在内部抛异常的情况下才打印调用栈信息。如果选择DEBUG日志级别，则总是显示截短后的调用栈信息|
|-s 或 --stacktrace|输出截短的调用栈信息。相比全调用栈信息，我们更推荐这个选项。Groovy的全栈跟踪信息非常的冗长（主要因为其底层的动态调用机制。导致经常输出一大堆信息但根本看不出来**你的代码**哪里错了。）
|-S 或 --full-stacktrace|会输出全栈跟踪信息。|

## 18.2. 写你自己的日志
一个简单的选择是直接把日志消息写到标准输出。Gradle会把写到标准输出的所有东西都以QUIET级别重定向到它的日志系统。

Example 18.1. Using stdout to write log messages

build.gradle

    println 'A message which is logged at QUIET level'

Gradle还在构建脚本中提供一个`logger`属性，指向[Logger](https://gradle.org/docs/current/javadoc/org/gradle/api/logging/Logger.html)类的实例。这个接口扩展自SLF4J的Logger接口，添加了一些Gradle特有的方法。下面是如何在构建脚本中使用的栗子：

Example 18.2. Writing your own log messages

build.gradle

    logger.quiet('An info log message which is always logged.')
    logger.error('An error log message.')
    logger.warn('A warning log message.')
    logger.lifecycle('A lifecycle info log message.')
    logger.info('An info log message.')
    logger.debug('A debug log message.')
    logger.trace('A trace log message.')

你也可以在构建脚本中hook到Gradle的日志系统中，只需使用SF4J的logger。你可以把这个logger当做构建脚本内建的logger一样使用。

Example 18.3. Using SLF4J to write log messages

build.gradle

    import org.slf4j.Logger
    import org.slf4j.LoggerFactory
    
    Logger slf4jLogger = LoggerFactory.getLogger('some-logger')
    slf4jLogger.info('An info log message logged using SLF4j')

## 18.3. 来自于外部工具和库的日志
Gradle在其内部使用Ant和Ivy。它们都有自己的日志系统。Gradle把它们的日志系统重定向到了Gradle的日志系统。Ant/Ivy的日志级别一对一地映射到Gradle的日志级别，除了一个例外，Ant/Ivy的TRACE级别被映射到Gradle的DEBUG级别。也就是说默认Gradle日志不会显示任何Ant/Ivy的输出，除非发生了错误或者警告。

很多外部工具依然使用标准输出做日志。默认情况下，Gradle把标准输出重定向到QUIET级别，把标准错误重定向到ERROR级别。这个行为也是可配置的。project对象提供一个[LoggingManager](https://gradle.org/docs/current/javadoc/org/gradle/api/logging/LoggingManager.html)，可以让你在构建脚本配置阶段就更改标准输出和标准错误的映射日志级别。

Example 18.4. Configuring standard output capture

build.gradle

    logging.captureStandardOutput LogLevel.INFO
    println 'A message which is logged at INFO level'

任务内也提供一个[LoggingManager](https://gradle.org/docs/current/javadoc/org/gradle/api/logging/LoggingManager.html)，以供你在任务执行阶段更改标准输出和标准错误的映射日志级别。

Example 18.5. Configuring standard output capture for a task

build.gradle

    task logInfo {
        logging.captureStandardOutput LogLevel.INFO
        doFirst {
            println 'A task message which is logged at INFO level'
        }
    }

Gradle也提供对Java Util Logging、Jakarta Commons Logging以及Log4j日志工具的集成。构建脚本使用任何以上日志工具输出的日志都会被重定向到Gradle的日志系统。

## 18.4. Changing what Gradle logs
你可以使用你自己的日志系统对Gradle日志系统做很多替换。你可以用某种方式定制日志API接口 - 记录更多或者更少信息，或者更改日志格式。使用[Gradle.useLogger()](https://gradle.org/docs/current/dsl/org.gradle.api.invocation.Gradle.html#org.gradle.api.invocation.Gradle:useLogger(java.lang.Object))方法来替换日志系统。它可以在构建脚本，或者init脚本，或者通过内嵌API来操作。注意这会完全禁止掉Gradle的默认输出。下面来一个init脚本的栗子，改变了任务执行和构建完成等动作的日志记录方式。

Example 18.6. Customizing what Gradle logs

init.gradle

useLogger(new CustomEventLogger())

    class CustomEventLogger extends BuildAdapter implements TaskExecutionListener {
    
        public void beforeExecute(Task task) {
            println "[$task.name]"
        }
    
        public void afterExecute(Task task, TaskState state) {
            println()
        }
        
        public void buildFinished(BuildResult result) {
            println 'build completed'
            if (result.failure != null) {
                result.failure.printStackTrace()
            }
        }
    }

你的logger可以继承任一个下方列出的接口，每一个接口都会负责一定范围的日志。当你注册一个logger的时候，只有这个logger所实现的接口负责的日志才会替换，其他接口负责的日志不会被改变。可以在56.6节找到更多接口相关的信息。

- [BuildListener](https://gradle.org/docs/current/javadoc/org/gradle/BuildListener.html)
- [ProjectEvaluationListener](https://gradle.org/docs/current/javadoc/org/gradle/api/ProjectEvaluationListener.html)
- [TaskExecutionGraphListener](https://gradle.org/docs/current/javadoc/org/gradle/api/execution/TaskExecutionGraphListener.html)
- [TaskExecutionListener](https://gradle.org/docs/current/javadoc/org/gradle/api/execution/TaskExecutionListener.html)
- [TaskActionListener](https://gradle.org/docs/current/javadoc/org/gradle/api/execution/TaskActionListener.html)

