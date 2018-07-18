---
layout: post
title: "如何实现一个Android Gradle Plugin"
categories: android
author: "Jeremy Liao"
---

本文会介绍如何实现一个Android Gradle Plugin

# 一、创建Plugin项目
1、 新建new module，类型选择为java library

2、 把main下面的java目录重命名成groovy

3、 在main下新建资源目录：resources->META-INF->gradle-plugins

4、 在gradle-plugins下新建文件**.properties，如com.jeremy.demoplugin.properties，其中com.jeremy.demoplugin为后面要用到的plugin的名字，内容为相应的Plugin类

配置插件短名称，新建文件「src/main/resources/META-INF/gradle-plugins/插件短名称.properties」

```
implementation-class=com.jeremy.plugin.DemoPlugin
```

5、 建好Plugin类，如：DemoPlugin.groovy

```java
package com.jeremy.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class DemoPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {

        project.task('test_task') << {
            println 'hello, world!'
        }
    }
}
```
6、 修改build.gradle
```
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}
```
最后工程的目录结构如下：
![image](https://123.sankuai.com/km/api/file/32499433/32534898)

# 二、将 plugin module 传到本地 maven 仓库
可以验证两个流程：把plugin上传到仓库；在其他module中使用仓库中的plugin

1、 添加 gradle.properties

```
PROJ_NAME=demoplugin
PROJ_ARTIFACTID=demoplugin
PROJ_POM_NAME=Local Repository

LOCAL_REPO_URL=file:///Users/hailiangliao/Develop/Android/repo/

PROJ_GROUP=com.jeremy

PROJ_VERSION=0.0.1
PROJ_VERSION_CODE=1

PROJ_WEBSITEURL=http://kvh.io
PROJ_ISSUETRACKERURL=https://github.com/kevinho/Embrace-Android-Studio-Demo/issues
PROJ_VCSURL=https://github.com/kevinho/Embrace-Android-Studio-Demo.git
PROJ_DESCRIPTION=demo apps for embracing android studio

PROJ_LICENCE_NAME=The Apache Software License, Version 2.0
PROJ_LICENCE_URL=http://www.apache.org/licenses/LICENSE-2.0.txt
PROJ_LICENCE_DEST=repo

DEVELOPER_ID=jeremy
DEVELOPER_NAME=jeremy
DEVELOPER_EMAIL=190545925@qq.com
```
2、 在 build.gradle 添加上传功能

关键参数：
- repository
- groupId
- artifactId
- version

```
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"

apply plugin: 'maven'

uploadArchives {
    repositories.mavenDeployer {
        repository(url: LOCAL_REPO_URL)
        pom.groupId = PROJ_GROUP
        pom.artifactId = PROJ_ARTIFACTID
        pom.version = PROJ_VERSION
    }
}
```
3、 上传通过运行命令：

```
./gradlew -p plugin/ clean build uploadArchives
```
4、 如果不想使用maven仓库，而想使用子module的lib，方便调试，可以如下修改：

引入repositories：

```
flatDir {
    dirs './probe-compiler/build/libs'
}
```
引入dependencies 的时候不要加版本号：

```
classpath 'com.jeremy.probe:probe-compiler'
```
# 三、在 app module 中使用插件
1、 在项目的 buildscript 添加插件作为 classpath

```
buildscript {

    repositories {
        maven {
            url 'file:///Users/hailiangliao/Develop/Android/repo/'
        }
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.2'
        classpath 'com.jeremy:demoplugin:0.0.1'

    }
}
```
2、 在 app module 中使用插件

```
apply plugin: 'com.jeremy.demoplugin'

```
# 四、实例：com.jeremy.probe
## 1、module划分
- probe-annotations：定义anotation和aspect
- probe-compiler：定义plugin
- 新建这两个module，probe-annotations类型为Android library；probe-compiler类型为java library

## 2、probe-annotations
probe-annotations主要实现相关的anotation和切面定义

```java
/**
 * Aspect representing the cross cutting-concern: Method and Constructor Tracing.
 */
@Aspect
public class TraceExecuteTimeAspect {

    private static final String TAG_TRACE_EXE_TIME = "TraceExecuteTime";


    @Pointcut("execution(@com.jeremy.probe_annotations.annotation.TraceExecuteTime * *(..))")
    public void methodAnnotatedTraceExecuteTime() {
    }

    @Pointcut("execution(@com.jeremy.probe_annotations.annotation.TraceExecuteTime *.new(..))")
    public void constructorAnnotatedTraceExecuteTime() {
    }

    @Around("methodAnnotatedTraceExecuteTime() || constructorAnnotatedTraceExecuteTime()")
    public Object weaveJoinPointTraceExecuteTime(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        SpendTimeWatcher spendTimeWatcher = new SpendTimeWatcher();
        spendTimeWatcher.start();
        Object result = joinPoint.proceed();
        spendTimeWatcher.stop();
        Log.d(TAG_TRACE_EXE_TIME, buildMessageTraceExeTime(methodSignature.getDeclaringType().getName(),
                methodSignature.getName(),
                spendTimeWatcher.getTotalTimeMillis()));
        return result;
    }

    private static String buildMessageTraceExeTime(String className, String methodName, long methodDuration) {
        return new StringBuilder()
                .append("Spend time: ")
                .append(methodDuration)
                .append(" ms")
                .append("\n --> ")
                .append("during execute method: ")
                .append(methodName)
                .append("()")
                .append("\n --> ")
                .append("in class: ")
                .append(className)
                .toString();
    }
}
```

```java
@Aspect
public class TraceReturnAspect {

    private static final String TAG_TRACE_RETURN = "TraceReturn";

    @Pointcut("execution(@com.jeremy.probe_annotations.annotation.TraceReturn * *(..))")
    public void methodAnnotatedTraceReturn() {
    }

    @Around("methodAnnotatedTraceReturn()")
    public Object weaveJoinPointTraceReturn(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Object result = joinPoint.proceed();
        Log.d(TAG_TRACE_RETURN, buildMessageTraceReturn(methodSignature.getDeclaringType().getName(),
                methodSignature.getName(),
                result));
        return result;
    }

    private static String buildMessageTraceReturn(String className, String methodName, Object returnValue) {
        return new StringBuilder()
                .append("Return value: ")
                .append(returnValue)
                .append("\n --> ")
                .append("by method: ")
                .append(methodName)
                .append("()")
                .append("\n --> ")
                .append("in class: ")
                .append(className)
                .toString();
    }
}
```
build.gradle:

```
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.9.1'
    }
}

apply plugin: 'com.android.library'

android {
    compileSdkVersion 27

    defaultConfig {
        minSdkVersion 15
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    implementation 'com.android.support:appcompat-v7:27.1.1'
    implementation 'org.aspectj:aspectjrt:1.9.1'
}

apply plugin: 'maven'

uploadArchives {
    repositories.mavenDeployer {
        repository(url: LOCAL_REPO_URL)
        pom.groupId = PROJ_GROUP
        pom.artifactId = PROJ_ARTIFACTID
        pom.version = PROJ_VERSION
    }
}

import com.android.build.gradle.LibraryPlugin
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

android.libraryVariants.all { variant ->
    LibraryPlugin plugin = project.plugins.getPlugin(LibraryPlugin)
    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.7",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", plugin.project.android.bootClasspath.join(
                File.pathSeparator)]

        println("execute aspectj args")

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler)

        def log = project.logger
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```
需要注意的是，由于定义的Aspect也需要被增强，所以需要在build.gradle中通过相应的代码来实现，当时花了很多时间来查这个问题
## 3、probe-compiler
定义ProbePlugin：

```java
package com.jeremy.probe_compiler

import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.compile.JavaCompile

class ProbePlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.android.applicationVariants.all { variant ->

            println("hello ProbePlugin")

            if (!variant.buildType.isDebuggable()) {
                project.logger.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
                return;
            }

            JavaCompile javaCompile = variant.javaCompile
            javaCompile.doLast {
                def log = project.logger
                String[] args = ["-showWeaveInfo",
                                 "-1.7",
                                 "-inpath", javaCompile.destinationDir.toString(),
                                 "-aspectpath", javaCompile.classpath.asPath,
                                 "-d", javaCompile.destinationDir.toString(),
                                 "-classpath", javaCompile.classpath.asPath,
                                 "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
                def argsStr = Arrays.toString(args)
                println("begin apply ProbePlugin")
                println("execute aspectj args: " + argsStr)
                log.debug "ajc args: " + argsStr

                MessageHandler handler = new MessageHandler(true);
                new Main().run(args, handler);
                for (IMessage message : handler.getMessages(null, true)) {
                    switch (message.getKind()) {
                        case IMessage.ABORT:
                        case IMessage.ERROR:
                        case IMessage.FAIL:
                            log.error message.message, message.thrown
                            break;
                        case IMessage.WARNING:
                            log.warn message.message, message.thrown
                            break;
                        case IMessage.INFO:
                            log.info message.message, message.thrown
                            break;
                        case IMessage.DEBUG:
                            log.debug message.message, message.thrown
                            break;
                    }
                }
            }
        }
    }
}
```
build.gradle:

```
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'org.aspectj:aspectjtools:1.9.1'
    implementation 'org.aspectj:aspectjrt:1.9.1'

}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"

apply plugin: 'maven'

uploadArchives {
    repositories.mavenDeployer {
        repository(url: LOCAL_REPO_URL)
        pom.groupId = PROJ_GROUP
        pom.artifactId = PROJ_ARTIFACTID
        pom.version = PROJ_VERSION
    }
}
```
# 五、其他
Q：编写groovy插件的时候没有代码提示，怎么办？

A：在Android module的gradle文件中编写是有代码提示的，可以现在这个文件中编写，写好之后拷到groovy文件中，不知道有没有更好的方法
- 后来发现可以通过在dependency中添加compileOnly 'com.android.tools.build:gradle:3.1.2'解决