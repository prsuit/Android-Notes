---
title: Gradle常用命令与配置
categories: [工具系列]
tags: [工具系列]
date: 2021-10-28 16:13:05
---

### （一）常用命令

> ##### 注：执行“./gradlew xxx”等同于执行“gradle xxx”，但执行“gradle xxx”需配置环境变量

清除build文件夹

```
./gradlew clean
```

检查依赖并编译打包

```
./gradlew build
```

<!--more-->

编译并打Debug包

```
./gradlew assembleDebug
```

编译并打Release包

```
./gradlew assembleRelease
```

获取gradle版本号

```
./gradlew -v
```

查看所有任务

```
./gradlew tasks
```

查看所有工程

```
./gradlew projects
```

查看所有属性

```
./gradlew properties
```

### （二）配置

设置全局参数

```
//Project的build.gradle添加ext{}
ext {
    compileSdkVersion = 29
    buildToolsVersion = "30.0.0"
}
//Module中引用
android {
		compileSdkVersion rootProject.ext.compileSdkVersion
    buildToolsVersion rootProject.ext.buildToolsVersion
}
```

去掉重复依赖

```
implementation 'xxx'{
		exclude module: 'xxx',group: 'xxx'
}
```

本地aar包依赖

```
implementation (name:'aar的名字', ext:'aar')
```

