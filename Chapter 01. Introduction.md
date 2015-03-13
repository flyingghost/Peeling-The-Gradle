Chapter 1. 简介
===
简介就没啥看头了，某些形容词大家都有，某些特性也只能深入学习之后才能深入理解。

有一点值得关注，也是很多人都会问到的一个问题：

**Gradle和Groovy到底是什么关系？**

简单的说，Gradle是用Groovy写的一个框架，一个DSL。放眼所见所有.gradle文件，其实都是标标准准的Groovy脚本。只是使用Groovy灵活的语法营造了一个看起来很像配置文件的DSL而已。所以Gradle的学习有三层境界：

- 熟悉DSL提供的所有blocks和properties，成为配置专家。
- 熟悉Gradle框架，并熟悉Groovy语法，在DSL不够用的复杂逻辑需求时动手写灵活的脚本。
- 使用Groovy和Java打造自己的Groovy扩展，形成自己的DSL。

