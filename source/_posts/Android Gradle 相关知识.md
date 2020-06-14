---
title: Android Gradle 相关知识
date: 2017-11-20 13:23:10
type: 'Android'
---

##  前言

本文主要是记录下 Android 开发过程中，常见的 Gradle 文件配置及命令，便于后续复习。<!-- more -->

## 配置与命令

* 执行 ./gradlew assemble  (可以简写成 ./gradlew a )会分别编译出 debug 和 release 版本的 apķ。

* 执行 ./gradlew assemble release (可以简写成 ./gradlew aR)会编译出 release 版本的 apk。

* 执行 ./gradlew assemble debug (可以简写成./gradlew asD 不能用 aD 因为与 androidDependencies 重复)会编译出 debug 版本的 apk。

* 自定义 buildType，下图中执行 ./gradlew aHfw 可以只打一个 hfw 的 apk，若执行 ./gradlew build 则会打出 debug、release、hfw 三个包。

    ![这里写图片描述](自定义 BuildType.png)

* 自定义 buildType 还可以选择继承一个已有的 buildType 。

    ![这里写图片描述](自定义 BuildType2.png)

* buildType 块内可以使用的子属性，如下图所示。

    ![这里写图片描述](自定义 BuildType3.png)

* 签名，首先要生成一个签名文件使用 build->generate signed apk 生成一个签名文件然后再在 gradle中进行如下配置。

    ![](签名.png)

* 多渠道打包，现在 android 域中加上图一或者图二的配置，再在 manifest 文件中的 appliation 节点下配置图三的节点，最后在图四配置 flavorDimensions 参数(不配置会报错)，然后进行执行 ./gradlew build 就会生成各渠道的 debug、release 包了。

    ![这里写图片描述](多渠道1.png)

    ![这里写图片描述](多渠道2.png)
    
    ![这里写图片描述](多渠道3.png)
    
    ![这里写图片描述](多渠道4.png)

* Check 任务 ./gradlew check 运行 Link 等等。

* Build 任务 ./gradlew build 等价于 assemble 任务和 check 任务，即打包加运行 Link。

* Clean 任务 ./gradlew clean 清除 build 过程中的所有中间数据。

* 通过配置 build.gradle 中 android 域可以把资源文件分成好几个文件夹。

    ![这里写图片描述](多资源文件夹.png)

    ![这里写图片描述](多资源文件夹2.png)

* 可以在项目层的 build.gradle 中的 ext 域配置一些项目中通用的参数类似 java 中的全局静态变量然后 module 层的 build.gradle 中通过 rootProject.ext 进行引用，也可以写在一个 xxx.gradle 文件里面然后在build.gradle 中执行 apply from: 'xxx.gradle' 进行引用。

    ![这里写图片描述](全局常量.png)

    ![这里写图片描述](全局常量2.png)

* build.gradle 中可以对某些配置项进行动态赋值，例如 versionCode。

    ![这里写图片描述](动态配置.png)

* lintOptions(android域下面) 可以配置在编译时 lint 报错后不中断编译。

    ![这里写图片描述](LintOptions.png)

* compileOptions 编译选项用来配置java版本暂时想不到运用场景，书上说是为了使用指定版本的特性。

    ![这里写图片描述](CompileOptions.png)

* minifyEnabled 表示是否开启代码混淆 proguardFiles 用来指定混淆文件，下图中使用的是默认的混淆文件。

    ![这里写图片描述](代码混淆.png)

* 配置文件 gradle.properties 可以配置一些参数然后在 build.gradle 中引用，用以下方式配置的参数可以在命令行中进行赋值。

    ![这里写图片描述](配置文件.png)

    ![这里写图片描述](配置文件2.png)

* gradle 系统参数(直接可以使用)
    1. project module 标识。，
    2. project.name module 名。
    3. project.buildDir module 构建目录。
    4. project.buildFile module 的 build.gradle 路径。
    5. project.version module 版本信息。
    6. name task 的名字。
    7. buildDir 同 project.buildDir。
    8. path task 的全限定路径名。

* 对每个 buildType 执行不同的逻辑，先在下图中创建 buildConfigField 字段，然后系统会在 build 目录下个各个 buildType 下面生成一个 buildConfig 类，其中就包含 build.gradle 中声明的 buildConfigField 字段，在代码中就可以获取到该值从而执行不同的逻辑。

    ![这里写图片描述](BuildConfig.png)

* 对每个 buildType 指定不同的 app_name 只需要在各个 buildType 下加入 resValue (该参数可以直接把资源加到 R 文件中，也可写在 defaultConfig 中，一般可用于获取动态字符串比如构建开始时间)，并且删除 value/string.xml 里面的 app_name，如果不删除会报错说重复定义了 app_name。

    ![这里写图片描述](ResConfig.png)

* 当需要为项目添加 jar 包，先放入libs 目录下然后可以选择右键 add as library 也可以直接 sync project，如果 build.gradle 中存在 implementation fileTree(dir: 'libs', include: ['*.jar']) 则直接加入后就可以使用。

* 使用 Gradle 打 jar 包。

    ![这里写图片描述](Jar.png)

* 执行 ./gradlew build -profile gradle 会生成构建耗时 html 文件。

    ![这里写图片描述](Report.png)

* 从上一个如果发现 Lint 这个 task 所花的时间很长，通过在命令行中加上参数 -x lint 或者在项目的 build.gradle 中加上如下代码即可不执行 lint 这个 task。

    ```
    project.gradle.startParameter.excludedTaskNames.add('lint')
    ```

* 在 debug 版本中可以加上 aaptOptions 域代码加快 aapt 速度从而加快编译速度，release 版本不要使用。

    ![这里写图片描述](AaptOptions.png)

* 加快 gradle 编译速度，通过下图中开启增量编译(已废弃)、提高内存、开启守护进程、并行编译、启用新的孵化模式。

    ![gradle.properties](编译速度.png)

