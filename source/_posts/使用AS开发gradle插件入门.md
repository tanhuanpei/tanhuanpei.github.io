---
title: 使用AS开发gradle插件 (一)
date: 2018-07-05 19:45:51
tags: gradle插件
---

# 0X00 前言
Gradle是一个使用Groovy语言实现的用于构建项目的框架。构建项目时真正起作用的是基于gradle框架使用Groovy实现的各种gradle插件。Gradle默认提供了很多插件，如Java-Plugin、Maven-Plugin等。Android Studio使用的是Android-Gradle-Plugin，由Google自主开发。在Android项目中，一个build.gradle文件，其实就是一个Groovy类。
# 0X01 在项目中使用Android gradle插件
配置插件路径，在Project目录中的build.gradle添加
```
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.0'
    }
```
使用具体插件，在主Module目录中的build.gradle添加
```
apply plugin: 'com.android.application'
```
# 0X02 自定义插件
我们可以利用Android Studio进行gradle插件开发，使用Groovy语言。简单步骤如下，
1.新建一个Module，选择Android Library。
2.删除src文件夹下的mian文件夹、删除build.gradle账文件中的所有内容。
3.在src目录下创建 groovy和resources目录，resouces目录下创建META-INF/gradle-plugins目录。创建完后的文件目录结构如下图：
![QQ截图20170123114005](/使用AS开发gradle插件入门/1.png)
4.修改Module中的 ``build.gradle``文件，引入groovy和maven相关依赖
```
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
```
5.创建插件类``TimeImpl.groovy``，继承 ``Plugin<Project>``，实现``apply``方法。

```
public class TimeImpl implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.gradle.addListener(new TimeListener())
    }
}

```

在gradle-plugins文件夹下创建properties文件，文件名就是以后插件的名字。配置插件类
```
implementation-class = com.thp.plugin.TimeImpl
```
![QQ截图20170123114005](/使用AS开发gradle插件入门/2.png)

6.在``build.gradle``配置插件版本和发布到本地仓库

```
group = 'com.thp.plugin'
version = '1.0.0'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('../repo'))
        }
    }
}
```
这时候，右侧的gradle Toolbar就会在module下多出一个task

![QQ截图20170123114005](/使用AS开发gradle插件入门/3.png)

点击uploadArchives，项目目录多出repo文件夹，打开可以看到生成的gradle插件

![QQ截图20170123114005](/使用AS开发gradle插件入门/4.png)



# 0X03 项目中引用插件
在app module的``build.gradle``上添加
```
buildscript {
    repositories {
        jcenter()
        maven {
            url uri('../repo')
        }
    }

    dependencies {
        classpath 'com.thp.plugin:gradletime:1.0.0'
    }
}

apply plugin: 'gradle.time'

```
以上，就是一个自定义插件的开发和引用流程。在这里，我们是引用本地生成的插件文件，下一篇将介绍如何上传到jcenter上，方便引用。

# 中文文档
- [深入理解Android之Gradle-邓凡平](http://www.infoq.com/cn/articles/android-in-depth-gradle) 
- [Gradle User Guide 中文版](https://legacy.gitbook.com/book/dongchuan/gradle-user-guide-/details)
- [拥抱Android Studio系列](http://kvh.io/cn/tags/EmbraceAndroidStudio/)

# 外籍文档

- [Groovy Documentation](http://www.groovy-lang.org/documentation.html) ：Groovy 的详细介绍文档
- [Groovy API Reference](http://www.groovy-lang.org/api.html) ：Groovy 的 API 文档
- [Gradle User Guid](https://docs.gradle.org/current/userguide/userguide.html)：Gradle 的详细介绍文档
- [Gradle Build Language Reference](https://docs.gradle.org/current/dsl/) : Gradle DSL 参考，重点的几个 DSL 过一下，其他的用到再查
- [Android Plugin DSL Reference](http://google.github.io/android-gradle-dsl/current/index.html) : 使用 Android 插件必备