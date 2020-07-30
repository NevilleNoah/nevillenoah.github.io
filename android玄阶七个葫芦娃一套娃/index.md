# Android玄阶：七个葫芦娃（一）——套娃

在我们写Activity的时候，很多方法需要传入参数Context，官方文档告诉我们传入this。
为什么可以传入this？this不是Activity？为什么Context参数可以是Activity？你猜的没错，Activity就是Context的子类，其中还蕴涵着一套“套娃”。
后续几个篇章，Neville就带大家一起学习这组“套娃”，彻底透析源码。

# 套娃层次
先呈上一组继承关系
```
class AppCompatActivity : FragmentActivity {...}
class FragmentActivity : ComponentActivity {...}
class ComponentActivity : Activity {...}
class Activity : ContextThemeWrapper {...}
class ContextThemeWrapper : ContextWrapper {...}
class ContextWrapper : Context {...}
```
发现套娃链路：
AppCompatActivity
=>FragmentActivity
=>ComponentActivity
=>Activity
=>ContextThemeWrapper
=>ContextWrapper
=>Context

我们的七位葫芦娃闪亮登场，让我们向他们打个招呼吧！
* Context：上下文抽象类，实现了逻辑实现。
* ContextWrapper：对Context进行打包，并实现Context中的抽象方法。
* ContextThemeWrapper ：在ContextWrapper基础上，引入了对主题资源的打包。
* Activity：最基础的活动组件类。
* ComponentActivity：
* FragmentActivity：
* AppComatActivity：实现了对低版本API对兼容，是Android Studio中创建组件Activity时的默认类型。

在后续的篇章中，让我们与他们深入交流～












