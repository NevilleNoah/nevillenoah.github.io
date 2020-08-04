# Gradle：仓库

# 引言
你有没有很奇怪，我们的远程引入的依赖库为什么只需要写入源即可？难道还有不需要网络协议就能获取到的资源？怎么可能。那么协议写到了哪里？写到仓库中。位置在project的`build.gradle`中
```gradle
buildscript {
    repositories {
        // 编译工具用到的仓库
    }
}

allprojects {
    // 所有项目都用到的仓库
}
```

## google()与jcenter()
当我们创建新项目时，就会看到这样的代码
```gradle
buildscript {
    repositories {
        google()
        jcenter()
    }
}

allprojects {
    repositories {
        google()
        jcenter()
    }
}
```
通过上述代码引入和google和jcenter的仓库，然后我们填写依赖的源的时候，会根据源自动在这些仓库中搜索合适的库进行引入。
我们打开上述两个方法会发现
```java
public interface RepositoryHandler extends ArtifactRepositoryContainer {
    MavenArtifactRepository google();
    MavenArtifactRepository jcenter();
}
```
而`MavenArtifactRepository`中包含对`URI`的操作
```
URI getUrl();
void setUrl(URI url);
Set<URI> getArtifactUrls();
...
```
正是通过这些方法从互联网上获取到远程仓库。

## 自定义仓库
你还可以通过`maven`添加新的仓库
```gradle
buildscript {
    repositories {
        maven {
            url "http://repo.mycompany.com/maven2"
        }
    }
}
```

## versions.gralde管理仓库
我们之前提到过使用versions来管理版本，也可以用它来管理仓库。
在project下新建`version.gradle`
在`versions.gradle`中（注意调用方法时，使用符号`&`）
```gradle
def addRepos(RepositoryHandler handler) {
    handler.google()
    handler.jcenter()
    handler.maven { url "https://oss.sonatype.org/content/repositories/snapshots" }
}
ext.addRepos = this.&addRepos
```

在project的`build.gradle`中
```gradle
buildscript {
    // 引入versions.gradle
    apply from: "versions.gradle"
    // 添加仓库
    addRepos(repositories)
}

allprojects {
    // 添加仓库
    addRepos(repositories)
}
```


