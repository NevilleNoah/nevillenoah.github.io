# WorkManager之任务请求


# 前言
继上一章《WorkMananger之添加任务》之后，我们再来详细学习任务请求`WorkRequest`。

## 构造器
`WorkRequest`有两种构造器：
* `OneTimeWorkRequestBuilder<WorkRequest>()` 一次性任务构造器
* `PeriodicWorkRequestBuilder<WorkRequest>()` 周期性任务构造器

分别构造两种`WorkRequest`：
* `OneTimeWorkRequest` 一次性任务
* `PeriodicWorkRequest` 周期性任务

使用示例：
```kotlin
// 一次性任务
var oneTimeWorkRequest = OneTimeWorkRequestBuilder<MyWorkRequest>()
        .builder()

// 每10秒执行一次
var periodicWorkRequest = PeriodicWorkRequestBuilder<MyWorkRequest>(10, TimeUnit.SECONDS)
        .builder()
```
周期性任务的时间设置常用方式：
* Duration
* Long+TimeUnit

其他方式可自行查阅源码，此处不展开。

## 约束
### 设置约束
任务满足约束条件，才会执行
```kotlin
// 创建约束
val constraints = Constraints.Builder()
        .setRequiresDeviceIdle(true)
        .setRequiresCharging(true)
        .build()

// 设置约束
val compressionWork = OneTimeWorkRequestBuilder<CompressWorker>()
        .setConstraints(constraints)
        .build()
```

### Constraints
包
```kotlin
androidx.work.Constraints
```

#### 一般约束
包含以下约束方法：
* `setRequiresDeviceIdle()` 设备空闲时运行(Api23及以上）
* `setRequiresCharging()` 正在充电时运行
* `setRequiredNetworkType()` 在特定网络类型下运行
* `setRequiresBatteryNotLow()` 在电量充足时运行
* `setRequiresStorageNotLow()` 在存储空间充足时运行

其他的好理解，网络类型补充说明一下，枚举类`NetworkType`声明了以下五种网络类型：
* `NOT_REQUIRED` 任务不需要网络
* `CONNECTED` 任务要求连接网络
* `UNMETERED` 任务要求非计量网络（不限量使用的网络，例如包月包年的宽带）
* `NOT_ROAMING` 要求非漫游网络（不是4G、5G之类的，而是连接wifi。别告诉我你手机可能连网线，如果可以那应该也算是非漫游）
* `METERED` 任务要求计量网络（按量收费的网络）

#### 触发器式约束
除了上述约束，`Constraints`还包含以下一些触发器式的约束：
* `addContentUriTrigger()` 当本地特定的Uri更新时运行(Api24及以上）
* `setTriggerContentUpdateDelay()` 特定内容更新后延迟一段时再运行
* `setTriggerContentMaxDelay` 特定内容更新后最大延迟多少时间再运行
    
其中`setTriggerContentUpdateDelay()`和`setTriggerContentMaxDelay()`：
* Api24及以上使用参数`duration`（延迟时间）和`timeunit`（时间单位）
* Api26以上使用参数`duration`(延迟时间）

前者规定了延迟时间的下限，后者规定了延迟时间的上限。

## 延迟时间
### 设置延迟时间
调用`setInitialDelay()`设置延迟时间：
```java
// androidx.work.WorkRequest
public @NonNull B setInitialDelay(long duration, @NonNull TimeUnit timeUnit) { ... }
```
使用示例：
```kotlin
val uploadWorkRequest = OneTimeWorkRequestBuilder<UploadWorker>()
        .setInitialDelay(10, TimeUnit.MINUTES)
        .build()
```

### 实际延迟时间
结合我们在执行约束中提到的约束，我们可以知道任务实际执行的时间为：**初始化延迟+约束延迟**

## 重试与退避政策
### 设置退避政策
我们在上一章《WorkMananger之添加任务》提到过`Worker`执行完成后会返回三种状态：
* `Result.success()` 执行成功
* `Result.failure()` 执行失败 
* `Result.retry()` 稍后重试

对于`Result.retry()`，我们可以设置退避政策`BackoffPolicy`，使得任务暂时退避不执行，等过一段时间后再执行，从而达到“稍后重试”的效果。
调用`setBackoffCriteria()`来设置`BackoffPolicy`：
```java
// androidx.work.WorkRequest
public final @NonNull B setBackoffCriteria(
                @NonNull BackoffPolicy backoffPolicy,
                long backoffDelay,
                @NonNull TimeUnit timeUnit)
```
使用示例：
```kotlin
 val uploadWorkRequest = OneTimeWorkRequestBuilder<UploadWorker>()
            .setBackoffCriteria(
                BackoffPolicy.LINEAR,
                OneTimeWorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS)
            .build()
```

### 退避政策的时间
退避政策`BackoffPolicy`是枚举类，包含两种时间延迟方式：
* `EXPONENTIAL` 指数增长型
* `LINEAR` 线性增长型

每次重试失败后，后一次的时间延迟将按`BackoffPlicy`指定的方式增长。

## 任务的输入/输出
### 数据包
任务的输入和输出，使用`Data`作为数据包，打包的过程由内联函数`workOfData`。
```kotlin
// androidx.work.Data
inline fun workDataOf(vararg pairs: Pair<String, Any?>): Data {
    val dataBuilder = Data.Builder()
    for (pair in pairs) {
        dataBuilder.put(pair.first, pair.second)
    }
    return dataBuilder.build()
}
```

### 设置输入包
我们调用`setInputData()`来设置数据包：
```java
// androidx.work.WorkRequest
public final @NonNull B setInputData(@NonNull Data inputData)  { ... }
```
使用示例：
```kotlin
val imageData = workDataOf(Constants.KEY_IMAGE_URI to imageUriString)

val uploadWorkRequest = OneTimeWorkRequestBuilder<UploadWorker>()
    .setInputData(imageData)
    .build()
```

### 设置输出包
同样使用`workOfData`来购建`Data`来作为输出，使用示例：
```kotlin
class UploadWorker(appContext: Context, workerParams: WorkerParameters)
    : Worker(appContext, workerParams) {

    override fun doWork(): Result {

        // 获得输入
        val imageUriInput = getInputData().getString(Constants.KEY_IMAGE_URI)
        // TODO：健壮性判断
        // 使用输入完成工作
        val response = uploadFile(imageUriInput)

        // 创建输出
        val outputData = workDataOf(Constants.KEY_IMAGE_URL to response.imageUrl)

        // 返回输出
        return Result.success(outputData)

    }
}
```

## 标记
### 设置标记
生活中为事物做标记是为了快速找到并了解他们，在这里也是一样，为`WorkRequest`添加标记是为了方便后续观察他们。
使用`addTag()`为`WorkRequest`添加标记：
```java
// androidx.work.WorkRequest
public final @NonNull B addTag(@NonNull String tag) { 
    mTags.add(tag);
    return getThis();
}
```
在`WorkRequest`中的集合`Set<String> mTags`用于存储添加的标记。
使用示例：
```kotlin
val cacheCleanupTask = OneTimeWorkRequestBuilder<CacheCleanupWorker>()
        .setConstraints(constraints)
        .addTag("cleanup")
        .build()
```

### 取消标记
调用`cancelAllWorkByTag()`来取消所有带有特定`Tag`的任务：
```java
// androidx.work.WorkManagerImpl
public @NonNull Operation cancelAllWorkByTag(@NonNull final String tag) { ... }
```
使用示例：
```kotlin
WorkManager.getInstance(this).cancelAllWorkByTag()
```

> 关于如何通过标签来观察他们，我们将在后续的章节中陈述。


