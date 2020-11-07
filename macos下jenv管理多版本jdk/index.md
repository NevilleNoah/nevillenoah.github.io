# MacOS下jenv管理多版本JDK




## 操作指南

官方操作指南：https://www.jenv.be/



## 安装jenv

使用brew安装jenv

```
$ brew install jenv
```



## 配置变量

配置命令行变量，我使用的是zsh

```
$ echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(jenv init -)"' >> ~/.zshrc
```

> 配置完后关闭命令行重新打开，确保变量重新被加载。



## 链接目录

将jdk链接到jenv目录下。这会将JDK从MacOS的默认地址`/System/Library/Java/JavaVirtualMachines`链接到jenv的目录下`/Users/{username}/.jenv/versions`。但是jenv的目录没有被创建，所以我们要手动创建`.jenv/versions`，否则链接的时候会找不到文件夹。

```
// 创建目录
$ cd ~
$ mkdir .jenv
$ cd .jenv
$ mkdir versions
```

链接的命令格式如下

```
$ jenv add {jdk's path} {jdk's name} added
```

我们还需要知道jdk的路径和jdk的名称，如何操作？通过下述命令可以知道当前所有Java虚拟机的信息：

```
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (2):
    14.0.1, x86_64:	"OpenJDK 14.0.1"	/Users/neville/Library/Java/JavaVirtualMachines/openjdk-14.0.1/Contents/Home
    11.0.9, x86_64:	"OpenJDK 11.0.9"	/Library/Java/JavaVirtualMachines/openjdk-11.jdk/Contents/Home

/Users/neville/Library/Java/JavaVirtualMachines/openjdk-14.0.1/Contents/Home
```

然后粘贴复制一下

```
$ jenv add /Users/neville/Library/Java/JavaVirtualMachines/openjdk-14.0.1/Contents/Home openjdk-14.0.1 added
14 added
14.0 added
14.0.1 added
openjdk-14.0.1 added
$ jenv add /Library/Java/JavaVirtualMachines/openjdk-11.jdk/Contents/Home openjdk-11.0.9 added
11 added
11.0 added
11.0.9 added
openjdk-11.0.9 added
```

我们会发现jenv为我们创建了4个链接，这些链接都是等价的，因为他们连接的虚拟机地址都是`/Users/neville/Library/Java/JavaVirtualMachines/openjdk-14.0.1/Contents/Home`。后面你要切换版本14，那你输入的版本号可以是：

* 14
* 14.0
* 14.0.1
* openjdk-14.0.1

进入`/Users/{username}/.jenv/versions`，你也会看到相应的软链接目录。

输入`jenv versions`，你会看到jenv管理的所有链接（当前使用的是14）。

```
$ jenv versions
  system
  11
  11.0
  11.0.9
* 14 (set by /Users/neville/.jenv/version)
  14.0
  14.0.1
  openjdk64-11.0.9
  openjdk64-14.0.1
```



## 切换版本

链接完成，切换到11试试。正如前文所说，这里你输入11、11.0、11.0.9、openjdk64-11.0.9都是等价的，因为这四个链接连接的都是同一个虚拟机。

```
$java -version
openjdk version "14.0.1" 2020-04-14
OpenJDK Runtime Environment (build 14.0.1+7)
OpenJDK 64-Bit Server VM (build 14.0.1+7, mixed mode, sharing)
$ jenv local 11
$ java -version
openjdk version "11.0.9" 2020-10-20
OpenJDK Runtime Environment (build 11.0.9+11)
OpenJDK 64-Bit Server VM (build 11.0.9+11, mixed mode)
```

JDK的版本从14变为了11。

`java local {version}`仅仅是作用于当前目录。当你输入`jenv local 11`的时候，你会发现在当前目录下多了一个`.java-version`的文件，打开后里面写着链接号`11`，这意味着当前目录下的JDK版本是11

`java local {version}`作用于全局。比如，想要全局切换到11，则需要使用

```
$ jenv global 11
```

最后，祝您用餐愉快！
