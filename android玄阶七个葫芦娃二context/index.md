# Android玄阶：七个葫芦娃（二）——Context

Context直译为“上下文”。Android四大组件中最核心的便是Activity，Activity最核心的便是Context，Activity是Context的子类。那么他究竟做了什么使得他如此重要呢？

>阅读建议：您不需要记住那些方法的参数，我列举参数仅为方便您浏览，但您应当注意概念、方法名称以及标签，我所提到的概念也仅为协助您理解而不是公认的至理。您的阅读目标应当是了解Context的基本轮廓。


>内容简述：我将对存储管理和Context组件做详细介绍，其他内容则一笔带过。这并不是偷懒，而是Context的内容太多，人们无法短时间内一下子接受太多知识，而且其他内容结合其他系列文章讲解更加合适。

# 存储管理
Android中主要的存储管理方式包括：
* 文件File
* 首选项Preferences
* 数据库Database

Context分别为三者定义了**模式常量**、**模式接口**、**入口抽象方法**。（Tip：我之所以称之为**入口抽象方法**，是因为这些方法仅包含get、set、open、close的逻辑，并不提供具体的内部操作）。

## 文件File
### File模式常量
```
@IntDef(flag = true, prefix = { "MODE_" }, value = {
        MODE_PRIVATE,
        MODE_WORLD_READABLE,
        MODE_WORLD_WRITEABLE,
        MODE_APPEND,
})
```
常量的具体含义
```
// 默认的模式，创建的文件只可被调用它的应用程序或者其他拥有相同用户id的应用程序调用
public static final int MODE_PRIVATE = 0x0000;
// 允许其他应用程序读取文件
public static final int MODE_WORLD_READABLE = 0x0001;
// 允许其他应用程序写文件
public static final int MODE_WORLD_WRITEABLE = 0x0002;
// 在文件末尾追加写入
public static final int MODE_APPEND = 0x8000;
```   

### File模式接口
```
@Retention(RetentionPolicy.SOURCE)
public @interface FileMode {}
```

### File入口抽象方法
#### 私有文件
首先约定一个概念：**私有**，即当前应用程序拥有。例如，**私有文件**，即仅属于当前应用程序的文件。后续所写的**私有**都是这个涵义。

```
// 打开私有文件，用于文件读取
public abstract FileInputStream openFileInput(String name)
        throws FileNotFoundException;

// 打开或创建与私有文件，用于文件写入
public abstract FileOutputStream openFileOutput(
    String name, 
    FileMode int mode
)
    throws FileNotFoundException;

// 获取私有文件
public abstract File getDataDir();

// 获取文件创建
public abstract File getFileStreamPath(String name);

// 获取文件
public abstract File getFilesDir();

// 删除私有文件
public abstract boolean deleteFile(String name);

// 获取私有文件的列表
public abstract String[] fileList();

// 检索或创建一个路径用于存放私有自定义文件
public abstract File getDir(
    String name, 
    @FileMode int mode
);
```

* `openFileOutput`和`getDir`方法的参数```@FileMode int mode```使用了模式变量，根据传入的模式变量决定访问权限与访问方式。模式常量已在上文陈述。
* 注意`getDataDir`和`getFilesDir`，`getDataDir`应用范围为私有文件，而`getFilesDir`应用范围为文件系统中`openFileOutput`存储文件的目录。

#### 外部存储
对于文件管理还包括外部存储。外部存储的定义较为广泛，它包括一些外置SD卡、共享存储空间，甚至是一些外部连接设备上的存储设备。不过好在，只要将这些挂载到Android系统中，Android系统就会为其设定路径，开发者可以很轻松地访问到他们。
```
// 获取外部文件
public abstract File getExternalFilesDir(@Nullable String type);

// 获取外部文件的列表
public abstract File[] getExternalFilesDirs(String type);

// 获取Obb类型文件
public abstract File getObbDir();

// 获取Obb类型文件的列表
public abstract File[] getObbDirs();

// 获取外部多媒体文件的列表
public abstract File[] getExternalMediaDirs();
```

* `getObbDir`和`getObbDirsObb`，获取Obb类型文件。Obb类型文件在游戏App中经常用到，经常作为数据包。

#### 缓存
```
// 获取缓存
public abstract File getCacheDir();

// 获取二进制缓存
public abstract File getCodeCacheDir();

// 获取应用预加载缓存
@SystemApi
public abstract File getPreloadsFileCache();

// 获取外部存储的缓存
public abstract File[] getExternalCacheDirs();

// 获取外部存储的缓存的列表
public abstract File[] getExternalCacheDirs();
```

* `getPreloadsFileCache`，获取应用预加载缓存。拥有`@SystemApi`,仅系统可用。由此可见，应用启动时并不是直接页面加载我们指定的一些数据，而是在加载前先通过`getPreloadsFileCache`加载了一些特定的缓存数据。
>@SystemApi标记表示方法面向系统而非开发者，带有@SystemApi的方法将会对开发者隐藏，即开发者无法调用。

## 首选项Preferences
### Preferences模式常量
```
@IntDef(flag = true, prefix = { "MODE_" }, value = {
        MODE_PRIVATE,
        MODE_WORLD_READABLE,
        MODE_WORLD_WRITEABLE,
        MODE_MULTI_PROCESS,
})
```
与File模式常量有相同之处，都拥有`MODE_PRIVATE`、`MODE_WORLD_READABLE`、`MODE_WORLD_WRITEABLE`，涵义也相同。不同是多了`MODE_MULTI_PROCESS`。
```
// 允许应用程序的多个进程写入相同的Preferences文件
@Deprecated
public static final int MODE_MULTI_PROCESS = 0x0004;
```
注意，`MODE_MULTI_PROCESS`被打上了`@Deprecated`（即将弃用），这将意味着不久的将来它会被删除。因此，我们应当避免使用它。

### Preferences模式接口
```
@Retention(RetentionPolicy.SOURCE)
public @interface PreferencesMode {}
```

### Preferences入口抽象方法
```
// 通过文件名，检索并保存首选项文件的内容
public abstract SharedPreferences getSharedPreferences(String name, @PreferencesMode int mode);

// 通过文件对象，检索并保存首选项文件的内容
public abstract SharedPreferences getSharedPreferences(File file, @PreferencesMode int mode);

// 获取首选项文件
public abstract File getSharedPreferencesPath(String name);

// 从给定的源Context中将已存在的首选项移到当前Context中
public abstract boolean moveSharedPreferencesFrom(Context sourceContext, String name);

// 删除首选项文件
public abstract boolean deleteSharedPreferences(String name);

// 重载首选项文件
public abstract void reloadSharedPreferences();

// 获取不备份的文件路径
public abstract File getNoBackupFilesDir();
```

* `getSharedPreferences`的参数`@PreferencesMode int mode`使用了模式常量，根据传入的模式常量决定首选项的访问权限与访问方式。模式常量已在上文陈述。
* 我们注意到一个很特别的方法`moveSharedPreferencesFrom(Context sourceContext, String name)`，它能将首选项从一个Context移动到另一个Context，这是一种极为重要的操作。我们开篇说过，
>Android四大组件中最核心的便是Activity，Activity最核心的便是Context，Activity是Context的子类。因此，`moveSharedPreferencesFrom(Context sourceContext, String name)`意味着首选项可以在Activity之间进行传递，这无疑对Activity之间的数据共享尤为重要。
* 我们还注意到一个即将弃用的方法`getSharedPrefsFile`，我们发现它返回了`getSharedPreferencesPath`的执行结果。
```
@Deprecated
public File getSharedPrefsFile(String name) {
        return getSharedPreferencesPath(name);
}
```

>这种**兼容**方式这是Android进行版本过渡的常用方法，为避免突如其来的弃用导致开发者不知道选择什么方法，便在过渡的1～2个版本中保留了即将弃用的方法，并在开发过程中提示开发者，使之快速找到新版本的方法。若以后我们要开发自己的移动操作系统的SDK，这种方式值得借鉴。

* `getNoBackupFilesDir`，获取不备份的文件路径。Android能够通过BackupAgent对文件进行自动备份，而一些不需要备份的文件就可以放到无需备份文件的目录下，例如一些临时文件，我们可以通过`getNoBackupFilesDir`获取到存放无需备份文件的绝对路径，并将无需备份的文件存入其中。


## 数据库Database
### Database模式常量
```
@IntDef(flag = true, prefix = { "MODE_" }, value = {
        MODE_PRIVATE,
        MODE_WORLD_READABLE,
        MODE_WORLD_WRITEABLE,
        MODE_ENABLE_WRITE_AHEAD_LOGGING,
        MODE_NO_LOCALIZED_COLLATORS,
})
```
与File、Preferences的模式常量有相同之处，都拥有`MODE_PRIVATE`、`MODE_WORLD_READABLE`、`MODE_WORLD_WRITEABLE`，涵义也相同。不同是`MODE_MULTI_PROCESS`与`MODE_NO_LOCALIZED_COLLATORS`。
涵义如下
```
// 数据库启用预写式日志
public static final int MODE_ENABLE_WRITE_AHEAD_LOGGING = 0x0008;

// 数据库支持本地排序器
public static final int MODE_NO_LOCALIZED_COLLATORS = 0x0010;
```
**预写式日志**（WAL，Write-Ahead Logging）是在数据库写入数据之前，现将数据写入到日志中，这对于数据库通过日志记录恢复有着举足轻重的作用。
**本地排序**即排序过程中采取本地的排序规则，仍然建议各端数据库尽量使用相同的排序规则，否则数据库的开销很大。
这些都是数据库的知识，挖个坑，以后有机会再填。

### Database模式接口
```
@Retention(RetentionPolicy.SOURCE)
    public @interface DatabaseMode {}
```

### Database入口抽象方法
```
// 打开或创建数据库对象
public abstract SQLiteDatabase openOrCreateDatabase(
    String name,
    @DatabaseMode int mode, 
    CursorFactory factory
);

// 从给定的源Context中将已存在的数据库对象移到当前Context中
public abstract boolean moveDatabaseFrom(
    Context sourceContext, 
    String name
);

// 获取数据库文件
public abstract File getDatabasePath(String name);
// 删除数据库对象
public abstract boolean deleteDatabase(String name);

// 获取私有数据库对象列表
public abstract String[] databaseList();
```

#### openOrCreateDatabase
`openOrCreateDatabase`的参数`@DatabaseMode int mode`使用了模式常量，根据传入的模式常量决定数据库的访问权限和访问特性。模式常量已在上文陈述。

#### moveDatabaseFrom
相信你已经联想到了首选项中的`moveSharedPreferencesFrom(Context sourceContext, String name)`，不错，`moveDatabaseFrom(Context sourceContext, String name)`在这里也是同样的涵义，它允许Database对象在Context对象之间，也意味着在Activity之间传递。

# Context与四大组件
## 与Activity
Activity是Context的子类，Context定义了对Activity的启动操作。

### 面向开发者的启动
#### 单启动
提到启动Activity，最熟悉的莫过于`startActivity`这位老熟人。
```
// 启动Activity
public abstract void startActivity(
    @RequiresPermission Intent intent
);
```
#### 多启动
大多数场景下，我们往往一次只启动一个Activity，但是有些时候我需要同时启动多个Activity，就需要用到`startActivities`。
```
// 启动多个Activity
public abstract void startActivities(
    @RequiresPermission Intent[] intents
);

// 
public abstract void startActivities(
    @RequiresPermission Intent[] intents, 
    Bundle options
);
```
#### 获取回调数据
我们还可以通过`startActivityForResult`来获取回调数据，注意，这个方法仅支持`Fragment`和`View`。
```
// 获取启动Activity的回调
public void startActivityForResult(
    @NonNull String who,
    Intent intent, 
    int requestCode, 
    @Nullable Bundle options
) {
    throw new RuntimeException("This method is only implemented for Activity-based Contexts. "+ "Check canStartActivityForResult() before calling.");
}
```

### 面向系统的启动
Activity的启动不仅面向开发者，还面向系统。

#### startActivityAsUser
`startActivityAsUser`指定特定的用户来启动Activity
```
@SystemApi
public void startActivityAsUser(
    @RequiresPermission @NonNull Intent intent,
    @NonNull UserHandle user
) {
    throw new RuntimeException("Not implemented. Must override in a subclass.");
}
```

## 与Service
Context定义了Service的启动、绑定、解绑、停止的基本操作。

### 启动

#### 启动服务
```
// 启动服务
public abstract ComponentName startService(Intent service);

// 指定用户启动服务
public abstract ComponentName startServiceAsUser(
    Intent service, 
    UserHandle user
);
```

#### 启动前台服务

```
// 启动前台服务
public abstract ComponentName startForegroundService(Intent service);

// 指定用户启动前台服务
public abstract ComponentName startForegroundServiceAsUser(
    Intent service, 
    UserHandle user
);
```

#### 停止服务
```
// 停止服务
public abstract boolean stopServiceAsUser(Intent service, UserHandle user);

// 指定用户停止服务
public abstract boolean stopServiceAsUser(Intent service, UserHandle user);
```

### 绑定
#### 绑定服务常量
```
@IntDef(flag = true, prefix = { "BIND_" }, value = {
    BIND_AUTO_CREATE,
    BIND_DEBUG_UNBIND,
    BIND_NOT_FOREGROUND,
    BIND_ABOVE_CLIENT,
    BIND_ALLOW_OOM_MANAGEMENT,
    BIND_WAIVE_PRIORITY,
    BIND_IMPORTANT,
    BIND_ADJUST_WITH_ACTIVITY,
    BIND_NOT_PERCEPTIBLE,
    BIND_INCLUDE_CAPABILITIES
})
@Retention(RetentionPolicy.SOURCE)
public @interface BindServiceFlags {}
```
#### 绑定服务入口方法
```
// 绑定服务
public abstract boolean bindService(
    @RequiresPermission Intent service,
    @NonNull ServiceConnection conn, 
    @BindServiceFlags int flags
);

public boolean bindService(
    @RequiresPermission @NonNull Intent service,
    @BindServiceFlags int flags, 
    @NonNull @CallbackExecutor Executor executor,
    @NonNull ServiceConnection conn
) {
        throw new RuntimeException("Not implemented. Must override in a subclass.");
}

// 指定用户绑定服务
@SystemApi
public boolean bindServiceAsUser(
    @RequiresPermission Intent service, 
    ServiceConnection conn,
    int flags, UserHandle user) {
    throw new RuntimeException("Not implemented. Must override in a subclass.");
}

public boolean bindServiceAsUser(
    Intent service, 
    ServiceConnection conn, 
    int flags,
    Handler handler, 
    UserHandle user) {
        throw new RuntimeException("Not implemented. Must override in a subclass.");
}

// 多个服务实例绑定单个组件
public boolean bindIsolatedService(
    @RequiresPermission @NonNull Intent service,
    @BindServiceFlags int flags, 
    @NonNull String instanceName,
    @NonNull @CallbackExecutor Executor executor, 
    @NonNull ServiceConnection conn
) {
        throw new RuntimeException("Not implemented. Must override in a subclass.");
}

// 更新服务组（包含bindIsolatedService相关的多个服务实例）
public void updateServiceGroup(
    @NonNull ServiceConnection conn, 
    int group,
    int importance) {
        throw new RuntimeException("Not implemented. Must override in a subclass.");
}
```

#### 解绑
```
public abstract void unbindService(@NonNull ServiceConnection conn);
```
绑定用到的参数`@BindServiceFlags int flags`需要使用到绑定常量，这将决定服务与Context的绑定的方式。


## 与Broadcast Receiver
Conext定义了Broadcast Receiver的接收、注册、接收的基本操作。

### 接收
允许接收广播需要执行IntentSensor。
#### 允许接收
通过`startIntentSender`执行IntentSensor：
```
public abstract void startIntentSender(
    IntentSender intent, 
    @Nullable Intent fillInIntent,
    @Intent.MutableFlags int flagsMask, 
    @Intent.MutableFlags int flagsValues,int extraFlags
) 
    throws IntentSender.SendIntentException;
            
public abstract void startIntentSender(
    IntentSender intent,
    @Nullable Intent fillInIntent,
    @Intent.MutableFlags int flagsMask, 
    @Intent.MutableFlags int flagsValues,
    int extraFlags, 
    @Nullable Bundle options
) 
    throws IntentSender.SendIntentException;
```

### 注册
#### 注册广播
```
public abstract Intent registerReceiver(
    @Nullable BroadcastReceiver receiver,
    IntentFilter filter
);

public abstract Intent registerReceiver(
    @Nullable BroadcastReceiver receiver,
    IntentFilter filter,
    @RegisterReceiverFlags int flags
);

public abstract Intent registerReceiver(
    BroadcastReceiver receiver,
    IntentFilter filter, 
    @Nullable String broadcastPermission,
    @Nullable Handler scheduler
);

public abstract Intent registerReceiver(
    BroadcastReceiver receiver,
    IntentFilter filter, 
    @Nullable String broadcastPermission,
    @Nullable Handler scheduler, 
    @RegisterReceiverFlags int flags
);

// 指定用户注册广播
public abstract Intent registerReceiverAsUser(
    BroadcastReceiver receiver,
    UserHandle user, 
    IntentFilter filter, 
    @Nullable String broadcastPermission,
    @Nullable Handler scheduler
);
```
* `registerReceiver` 注册广播。
* `registerReceiverAsUser` 指定用户注册广播

### 注销广播
```
public abstract void unregisterReceiver(BroadcastReceiver receiver);
```
* `unregisterReceiver` 用于注销广播

### 发送
#### 无序广播
```
// 无序广播
@SystemApi
public abstract void sendBroadcast(
    Intent intent, 
    @Nullable String receiverPermission, 
    @Nullable Bundle options
);

public abstract void sendBroadcast(
    Intent intent, 
    String receiverPermission, 
    int appOp
);

// 指定用户无序广播
public abstract void sendBroadcastAsUser(
    @RequiresPermission Intent intent,
    UserHandle user
);

public abstract void sendBroadcastAsUser(
    @RequiresPermission Intent intent,
    UserHandle user, 
    @Nullable String receiverPermission
);

@SystemApi
public abstract void sendBroadcastAsUser(
    @RequiresPermission Intent intent,
    UserHandle user, 
    @Nullable String receiverPermission, 
    @Nullable Bundle options
);

public abstract void sendBroadcastAsUser(
    @RequiresPermission Intent intent,
    UserHandle user, 
    @Nullable String receiverPermission, 
    int appOp
);

```
* `sendBroadcast` 无序广播
* `sendBroadcastAsUser` 指定用户无序广播

#### 有序广播
```
// 有序广播
public abstract void sendOrderedBroadcast(
    @RequiresPermission Intent intent,
    @Nullable String receiverPermission
);

public abstract void sendOrderedBroadcast(
    @RequiresPermission @NonNull Intent intent,
    @Nullable String receiverPermission, 
    @Nullable BroadcastReceiver resultReceiver,
    @Nullable Handler scheduler, int initialCode,
    @Nullable String initialData,
    @Nullable Bundle initialExtras
);

// 指定用户有序广播
public abstract void sendOrderedBroadcastAsUser(
    @RequiresPermission Intent intent,
    UserHandle user, 
    @Nullable String receiverPermission, 
    BroadcastReceiver resultReceiver,
    @Nullable Handler scheduler, 
    int initialCode, 
    @Nullable String initialData,
    @Nullable  Bundle initialExtras
);

public abstract void sendOrderedBroadcastAsUser(
    Intent intent, UserHandle user,
    @Nullable String receiverPermission, 
    int appOp, BroadcastReceiver resultReceiver,
    @Nullable Handler scheduler, 
    int initialCode, 
    @Nullable String initialData,
    @Nullable  Bundle initialExtras
);

public abstract void sendOrderedBroadcastAsUser(
    Intent intent, UserHandle user,
    @Nullable String receiverPermission, 
    int appOp, 
    @Nullable Bundle options,
    BroadcastReceiver resultReceiver,
    @Nullable Handler scheduler, 
    int initialCode,
    @Nullable String initialData, 
    @Nullable  Bundle initialExtras
);
```
* `sendOrderedBroadcast` 有序广播
* `sendOrderedBroadcastAsUser` 指定用户有序广播


#### 强制广播
```
// 强制权限数组中权限被准许，并向所有相关的广播接收器发送广播
public abstract void sendBroadcastMultiplePermissions(
    Intent intent, 
    String[] receiverPermissions
);

// 对于指定用户，强制权限数组中权限被准许，并向所有相关的广播接收器发送广播
public abstract void sendBroadcastAsUserMultiplePermissions(
    Intent intent, 
    UserHandle user, 
    String[] receiverPermissions
);           
```
`sendBroadcastMultiplePermissions`与`sendBroadcastAsUserMultiplePermissions`全被隐藏（注释中带有@hide），开发者无法调用。
>@hide意味着该方法对开发者不可见，即开发者无法调用。

## 与Notification
Context与Notification无直接关联。

# 其他
Context还包括其他一些格外重要的内容，但本章仅作简述，不铺陈开。这些内容将在另一系列的章节中详细陈述，你现在仅需要知道他们的存在。

## 权限管理
```
public abstract PackageManager getPackageManager();
```

## 消息管理
```
public abstract Looper getMainLooper();
```

## 资源管理
```
public abstract AssetManager getAssets();
public abstract Resources getResources();
```

# 总结
Context是个大熔炉，一切都在这里汇聚，组件关联、存储管理、权限管理、消息管理、资源管理等等。
>也许您会有以下疑问，Context为什么涉及这么多？PackageManager为什么跟权限有关？Looper为什么跟消息管理有关？资源管理为什么有两块？您当然可以自行检索答案，在未来不久的日子我也会更新相关的知识模块供您参考。










