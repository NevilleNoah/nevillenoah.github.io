# Gradle：多模块下的版本管理

## 引言
如果我们的project中有许多module，那么每个module下就会有一个`build.gradle`用于管理该module的依赖库。多模块对于开发人员的意义就是可以分块开发进而提升效率，但这也意味着沟通成本的提高和冗余工作的增加。

* 版本无法统一管理
甲引入了moduleA的1.0.0版本，乙在moduleB引入了1.0.1版本，如果需要甲乙统一版本，那就需要确保他们有效沟通。那如果再有丙在moduleC引入了1.0.2版本，丁在moduleD引入了1.0.3版本...每次改个依赖库的版本都是一次群体大会，一个人没统一，代码可能就会由于版本不统一出现问题。

* 人人研究配置
作为一位Android工程师，研究配置没有错，但是如果每个人每天都在折腾这些版本号，其他活就不用干了。

* 工作重心不当
你说不怕，我们有全局的`build.gradle`可以解决这个问题，但是想想，我们传统的一行`implementation 'xxx'`虽然引入很方便，但是可读性极差。我们对于配置的主要工作不在引入工作，而是版本管理工作，因此我们需要使用一种新的写法来解决版本管理。

{{< admonition >}}
针对以上问题，我们引入了version版本控制的写法。
{{< /admonition >}}

## 创建版本管理文件
在project根目录下创建`versions.grale`文件，并在project的`build.grale`中引入
```gradle
buildscript {
    apply from: 'versions.gradle'
}
```
这样,`versions.gradle`就在整个工程中生效了，接下来我们来编写`versions.gradle`。

## 声明额外属性
我们在Gradle中声明额外属性需要用到ext
```gradle
ext {
    deps
    versions
}
```
或
```gradle
ext.deps = [:] // [:]相当于null
ext.version = [:]
```
上述代码就声明两个额外属性`deps`和`verisons`。如果我们定义了这些属性，那么就可以在后续的代码中使用这些属性。

## 依赖版本
在`version.gradle`中，
>阅读过程中注意**首尾呼应**，区分**变量**和**额外属性**。
```gradle
// 声明额外属性deps，用于存放依赖的信息
ext.deps = [:] 

// 声明版本变量
def versions = [:]
// 版本变量中为各种依赖库创建字段，并赋值版本号
versions.activity = '1.1.0'
versions.android_gradle_plugin = '4.0.0'
versions.annotations = "1.0.0"
// 完成版本变量中全部版本号的定义后，将版本变量存入额外属性versions中
ext.versions = versions

// 声明依赖库变量
def deps = [:]
// 声明activity库变量
def activity = [:]
// 为activity库变量的activity_ktx字段赋值来源信息，其中的$version.activity会引入我们上面定义的version.activity
activity.activity_ktx = "androidx.activity:activity-ktx:$versions.activity"
// 将activity库来源信息存入deps的activity字段中
deps.activity = activity

// 将android_gralde_plugin库来源信息赋值到deps的字段android_gradle_plugin字段中
deps.android_gradle_plugin = "com.android.tools.build:gradle:$versions.android_gradle_plugin"
// 将annotations库来源信息赋值到deps的字段nnotations字段中
deps.annotations = "androidx.annotation:annotation:$versions.annotations"


// 完成所有依赖库中所有依赖的定义后，将依赖变量存入额外属性deps
ext.deps = deps
```
像一些父库（例如`activity`）都会有各种子库（例如`activity.activity_ktx`），我们可以采用上述代码中`activity`的定义方式声明，先定义父库的变量，再在父库中添加子库的字段，最后把父库的变量存入deps。
而一些单一的库（例如`android_gradle_plugin`、`annotations`）则可直接赋值到字段中。
定义完后，我们就可以直接在工程中的全局build.gradle和module的build.gradle中使用存入deps中的字段来决定引入的依赖库而不需要纠结版本问题。
```gradle
implementation deps.activity.activity_ktx
implementation deps.android_gradle_plugin
implementation deps.annotations
```
通过这种方式，团队中只需要指定一个人管理库版本即可，其他人仅负责引入。

诶？既然可以用来搞定依赖的版本，是不是也可以用这种方式控制其他版本呢？一个个文件找多累啊！没错，你的直觉是对的。我们还可以用这种方式控制build版本、Java版本。

## build版本
在`versions.gradle`中
```gradle
def build_versions = [:]
build_versions.min_sdk = 14
build_versions.compile_sdk = 29
build_versions.target_sdk = 29
build_versions.build_tools = "29.0.3"
ext.build_versions = build_versions
```
在module的`build.gralde`中
```gradle
android {
    compileSdkVersion build_versions.compile_sdk
    buildToolsVersion build_versions.build_tools
    defaultConfig {
        minSdkVersion build_versions.min_sdk
        targetSdkVersion build_versions.target_sdk
    }
```

## Java版本
在`versions.gradle`中
```gradle
def java_version = [:]
java_version.source = JavaVersion.VERSION_1_8
java_version.target = JavaVersion.VERSION_1_8
ext.java_version = java_version
```
在module的`build.gralde`中
```gradle
android {
    compileOptions {
        sourceCompatibility java_version.source
        targetCompatibility java_version.target
    }
}
```




