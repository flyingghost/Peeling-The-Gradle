#Chapter 10. Web应用快速入门
Gradle为web应用带来2个插件：war和jetty。war扩展了java插件，可以把工程打成war包。jetty插件扩展了war插件，方便你把打好的war包部署到jetty容器。

##10.1. 创建一个WAR包

    apply plugin: 'war'

引入war插件，同时也引入了java插件。运行`gradle build`会对工程进行编译、测试并且war打包。gradle将会到`src/main/webapp`下的文件打进war包。你的class文件以及运行时依赖也会打进去，包括WEB-INF/classes目录和WEB-INF/lib目录。

##10.2. 运行web应用

    apply plugin: 'jetty'

使用jetty插件可以运行web应用，同时也相当于使用了war插件。命令行运行`gradle jettyRun`将在一个嵌入式jetty容器里运行web应用。使用命令行`gradle jettyRunWar`将会先生成war包然后运行web应用。

##10.3. 综上
综上，我们仅仅知道了两个插件的存在。26章和28章将会对这两个插件做进一步详解。

至于现在，下课。