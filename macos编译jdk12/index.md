# MacOS编译JDK12


# 

## 获取源码

直接从OpenJDK的官网获取，我选用的是JDK12

http://hg.openjdk.java.net/jdk/jdk12/

> 点击左侧栏的zip就会开始下载

## 主要环境

* MacOS 10.15.5
* Xcode 12.1
* Clang 12.0.0
* Java 11

我是如何知道编译需要这些的？前三者通过两条途径得知：

* 官网的OpenJDKWiki https://wiki.openjdk.java.net/display/Build/Supported+Build+Platforms
* 下载的JDK源码的doc目录下building.html

安装方式多种多样，大家各显神通吧。

Java则是从后来检查**依赖环境**时的日志中得知：

```
configure: (Your Boot JDK version must be one of: 11 12)
```

你可以使用`Homebrew`直接安装Java

```
> brew search java
==> Formulae
app-engine-java            java11                     jslint4java
google-java-format         javacc                     libreadline-java
java                       javarepl                   pdftk-java
==> Casks
eclipse-java                             netbeans-java-se
eclipse-javascript                       oracle-jdk-javadoc
font-noto-sans-javanese                  homebrew/cask-versions/java-beta
netbeans-java-ee

> brew install java11
// 这里会提示如果想要被找到，则需要将其链接到java虚拟机的目录下
For the system Java wrappers to find this JDK, symlink it with
  sudo ln -sfn /usr/local/opt/openjdk@11/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-11.jdk
...

> sudo ln -sfn /usr/local/opt/openjdk@11/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-11.jdk
```



## 命令行环境

Xcode安装完毕后，要在Preference->Locations选择Command Line Tools（我选的是Xcode 12.1）从而为命令行添加Xcode的编译环境，否则编译的时候检测不到Xcode的相关工具，编译无法正常进行。

另外，要写一个脚本来设置clang及其他各种命令行的参数，网上有各种版本，仅供参考：

```
# 设定语言选项，必须设置
export LANG=C
# Mac平台，C编译器不再是GCC，是clang
export CC=gcc
# 跳过clang的一些严格的语法检查，不然会将N多的警告作为Error
export COMPILER_WARNINGS_FATAL=false
# 链接时使用的参数
export LFLAGS='-Xlinker -lstdc++'
# 是否使用clang
export USE_CLANG=true
# 使用64位数据模型
export LP64=1
# 告诉编译平台是64位，不然会按32位来编译
export ARCH_DATA_MODEL=64
# 允许自动下载依赖
export ALLOW_DOWNLOADS=true
# 并行编译的线程数，编译时间长，为了不影响其他工作，我选择为2
export HOTSPOT_BUILD_JOBS=2
# 是否跳过与先前版本的比较
export SKIP_COMPARE_IMAGES=true
# 是否使用预编译头文件，加快编译速度
export USE_PRECOMPILED_HEADER=true
# 是否使用增量编译
export INCREMENTAL_BUILD=true
# 编译内容
export BUILD_LANGTOOLS=true
export BUILD_JAXP=false
export BUILD_JAXWS=false
export BUILD_CORBA=false
export BUILD_HOTSPOT=true
export BUILD_JDK=true
# 编译版本
export SKIP_DEBUG_BUILD=true
export SKIP_FASTDEBUG_BUILD=false
export DEBUG_NAME=debug
# 避开javaws和浏览器Java插件之类的部分的build
export BUILD_DEPLOY=false
export BUILD_INSTALL=false
# 加上产生调试信息时需要的 objcopy
export OBJCOPY=gobjcopy
```



## 依赖环境

编译自然不会这么简单，还需要一些依赖环境，我们如何知道需要什么？

依赖环境根据你想要的编译版本（例如FastDebug版或者其他各种类型的版本）来决定，进入jdk源码根目录，可以通过

```
bash configure -help
```

来查看可以添加的参数，通过这些参数来决定编译的目标版本。

例如变异FastDebug版、仅含Server模式的HotSpot虚拟机，命令行就是

```
bash configure --enable-debug --with-jvm-variants=server
```

执行后目录下会新增build文件夹（还没编译所以是空的）和日志文件config.log、configure.log，我们根据命令行给出的各种提示逐步安装所需要的依赖。

如果一切配置准备就绪，执行完毕会出现：

```
====================================================
A new configuration has been successfully created in
/Users/neville/lab/jdk/openjdk/jdk12/build/macosx-x86_64-server-fastdebug
using configure arguments '--enable-debug --with-jvm-variants=server'.

Configuration summary:
* Debug level:    fastdebug
* HS debug level: fastdebug
* JVM variants:   server
* JVM features:   server: 'aot cds cmsgc compiler1 compiler2 dtrace epsilongc g1gc graal jfr jni-check jvmci jvmti management nmt parallelgc serialgc services shenandoahgc vm-structs'
* OpenJDK target: OS: macosx, CPU architecture: x86, address length: 64
* Version string: 12-internal+0-adhoc.neville.jdk12 (12-internal)

Tools summary:
* Boot JDK:       openjdk version "11.0.9" 2020-10-20 OpenJDK Runtime Environment (build 11.0.9+11) OpenJDK 64-Bit Server VM (build 11.0.9+11, mixed mode)  (at /Library/Java/JavaVirtualMachines/openjdk-11.jdk/Contents/Home)
* Toolchain:      clang (clang/LLVM from Xcode 12.1)
* C Compiler:     Version 12.0.0 (at /usr/bin/gcc)
* C++ Compiler:   Version 12.0.0 (at /usr/bin/clang++)

Build performance summary:
* Cores to use:   16
* Memory limit:   32768 MB
```



## 编译源码

```
make all
```

如果有多个版本可以用参数`CONF`指定版本，版本名称是`build`目录下的文件夹名：

```
make CONF=[版本名称]
```

例如

```
make CONF=macosx-x86_64-server-fastdebug
```



### 错误1

```
/src/hotspot/share/runtime/arguments.cpp:1452:35:
if (old_java_vendor_url_bug != DEFAULT_VENDOR_URL_BUG) {
```

修改

```
if (old_java_vendor_url_bug != DEFAULT_VENDOR_URL_BUG) {
```

为

```
if (strcmp(old_java_vendor_url_bug, DEFAULT_VENDOR_URL_BUG) != 0) {
```

### 错误2

```
/test/hotspot/gtest/classfile/test_symbolTable.cpp:62:6:
s1 = s1; // self assignment
```

这行代码应该是没用的，可以直接注释，毕竟自己给自己赋值是没有意义...

也可以参考这个回答修改代码http://hg.openjdk.java.net/jdk/jdk/rev/297ddf282627

### 错误3

```
/src/hotspot/share/runtime/sharedRuntime.cpp:2873:85
error: expression does not compute the number of elements in this array; element type is 'double', not 'relocInfo'
buffer.insts()->initialize_shared_locs((relocInfo*)locs_buf, sizeof(locs_buf) / sizeof(relocInfo));
```

参考了这个文件中别处使用`initialize_shared_locs()`的方式，将修改

```
double locs_buf[20];
buffer.insts()->initialize_shared_locs((relocInfo*)locs_buf, sizeof(locs_buf) / sizeof(relocInfo));
```

为

```
short locs_buf[20];
buffer.insts()->initialize_shared_locs((relocInfo*)locs_buf, sizeof(locs_buf) / sizeof(relocInfo));
```

### 错误4

```
/test/hotspot/jtreg/vmTestbase/nsk/jvmti/unit/FollowReferences/followref003/followref003.cpp:813:14: error: comparison of different enumeration types in switch statement ('jvmtiHeapReferenceKind' and 'jvmtiObjectReferenceKind') [-Werror,-Wenum-compare-switch]
case JVMTI_REFERENCE_ARRAY_ELEMENT:
```

一个case中套了多种类型，一种是Heap的枚举类，一种是Object的枚举类。因此根据周边的改为相同的类型即可。周边代码：

```
case JVMTI_REFERENCE_ARRAY_ELEMENT:
case JVMTI_HEAP_REFERENCE_JNI_GLOBAL:
case JVMTI_HEAP_REFERENCE_SYSTEM_CLASS:
case JVMTI_HEAP_REFERENCE_MONITOR:
case JVMTI_HEAP_REFERENCE_OTHER:
```

所以应当使用应该使用`jvmtiHeapReferenceKind`，即Heap的枚举类。

修改

```
case JVMTI_REFERENCE_ARRAY_ELEMENT:
```

为

```
case JVMTI_HEAP_REFERENCE_ARRAY_ELEMENT:
```

### 错误5

```
/Users/neville/lab/jdk/openjdk/jdk12/src/java.desktop/macosx/native/libawt_lwawt/awt/CSystemColors.m:134:9: 
error: converting the result of '?:' with integer constants to a boolean always evaluates to 'true' [-Werror,-Wtautological-constant-compare]
    if (colorIndex < (useAppleColor) ? sun_lwawt_macosx_LWCToolkit_NUM_APPLE_COLORS : java_awt_SystemColor_NUM_COLORS) {
```

修改方式https://github.com/openjdk/jdk/commit/4622a18a



## 编译完成

我个人修改完上述五个问题后，就编译成功了。每个人所使用的工具、依赖和编译版本不同，可能遇到的bug也不同。源码也并不是完美的，还是要手动改一改。如果你遇到了和我同样的问题，很高兴能帮到你。

最后再验证一下，运行下述命令查看编译后的jdk的版本：

```
> build/macosx-x86_64-normal-server-slowdebug/jdk/bin/java -version
openjdk version "12-internal" 2019-03-19
OpenJDK Runtime Environment (fastdebug build 12-internal+0-adhoc.neville.jdk12)
OpenJDK 64-Bit Server VM (fastdebug build 12-internal+0-adhoc.neville.jdk12, mixed mode)
```




