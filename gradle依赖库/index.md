# Gradle：依赖库


# 引言
很多同学写了Android很久，对于依赖库的引入却是傻傻分不清。因为Gradle随着版本的升级，语义在不断的变化，从`compile`到`implementation`，从`apt`到`annotationProcessor`...已经绕晕了不少人，今天就来明确下依赖最最基础的依赖库的用法。

{{< admonition >}}
很少有关注gradle的变化，认为gradle区区配置不重要，但是对于大规模项目而言，gradle是整个项目的骨架，极为重要。
{{< /admonition >}}

## Gradle依赖
Gradle并非孤例运行，他自身也会依赖于一些插件，在project的`build.gradle`中，使用`classpath`引入
```gralde
classpath "lib-gradle"
```

## 开发依赖库
正式运行时使用的依赖库，对应我们的开发目录。
在module的`build.gradle`中，使用`implementation`引入
```gradle
dependencies {
    implementation “lib-source”
}
```


## 测试依赖库
测试程序时使用的依赖库，对应`androidTest`目录。
在module的`build.gradle`中，使用`androidTestImplementation`引入
```gradle
dependencies {
    androidTestImplementation "lib-source"
}
```

## 测试工具依赖库
在module的`build.gradle`中，使用`testImplementation`引入
```gradle
dependencies {
    testImplementation "test-tool-source"
}
```
例如junit的库。

## 本地依赖库
如果是本地依赖库，则需要将依赖库放入module的`libs`文件夹中，在module的`build.gradle`中，用`fileTree`引入
```gradle
dependencies {
    // 从文件夹libs中引入后缀为jar文件作为依赖库
    implementation fileTree(dir: "libs", include: ["*.jar"])
}
```

## 注释处理器
带有注解的代码会通过注释处理器**APT（Annotation Processing Tool）**进行转换，变成不带注解的代码（注解往往都是控制反转用的，我们的虚拟机可运行不了带注解的代码）。对于kotlin和java有不同的配置方式。

### kotlin
在project的`build.gradle`中配置
```gradle
apply plugin: 'kotlin-kapt'

kapt {
    arguments {
        arg("AROUTER_MODULE_NAME", project.getName())
    }
}
```
在module的`build.gradle`中，先引入`kotlin-kapt`插件，再通过`kapt`引入
```gradle
apply plugin: 'kotlin-kapt'

dependencies {
    kapt "apt-source"
}
```


### java
在project的`build.gradle`中配置
```gradle
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
}
```
在module的`build.gradle`中，使用`annotationProcessor`引入
```gradle
dependencies {
    annotationProcessor "apt-source"
}
```

`annotationProcessor`同时支持javac和jack两种编译方式。

{{< admonition >}}
如果你的某个模块是Java和Kotlin混合开发，那么两者都要写上，他们会分别处理.java和.kt文件。
{{< /admonition >}}


