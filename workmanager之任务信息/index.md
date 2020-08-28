# WorkManager之任务信息


# 前言
我们在《WorkManager之任务请求》结尾提到了标签Tag可以用来观察任务，它其实是任务信息`WorkInfo`的一部分，本章将详细陈述任务信息。

## 任务信息
想要管理任务，首先我们要考虑拿到任务的相关信息，`WorkInfo`就是用于存储任务的信息的类：
```java
public final class WorkInfo {

    private @NonNull UUID mId; // 唯一id
    private @NonNull State mState; // 状态
    private @NonNull Data mOutputData; // 输出数据
    private @NonNull Set<String> mTags; // 标签
    private @NonNull Data mProgress; // 进度
    private int mRunAttemptCount; // 重试运行的次数
}
```
围绕它持有的属性信息，我们可以很快猜到它会有以下操作：
* 通过Id获取一个WorkInfo...
* 通过Tag获取一批WorkInfo...
* 通过获取状态，根据任务当前所处不同的状态，进项不同的操作...
* 获取输出（之前的章节提到过）
* 进度管理，可能是通过获取进度将进度通过UI界面展示给用户...
* 重试管理，可能是重试多少次后执行其他操作...

没错，其实主要的也就是这些操作。

### 获取任务信息
#### 通过id获取特定任务的信息
```kotlin
// androidx.work.WorkManager
// 根据id获取Listenable<WorkInfo>
public @NonNull ListenableFuture<WorkInfo> getWorkInfoById(@NonNull UUID id) { ... }

// 根据id获取LiveData<WorkInfo>
public @NonNull LiveData<WorkInfo> getWorkInfoByIdLiveData(@NonNull UUID id) { ... }
```

#### 通过tag获取一批相关的信息
```kotlin
// androidx.work.WorkManager
// 获取持有特定Tag的ListenableFuture<List<WorkInfo>> 
public @NonNull ListenableFuture<List<WorkInfo>> getWorkInfosByTag(@NonNull String tag) { ... }

// 获取持有特定Tag的LiveData<List<WorkInfo>> 
public @NonNull LiveData<List<WorkInfo>> getWorkInfosByTagLiveData(@NonNull String tag) { ... }
```

### 添加观察者
看到LiveData相信你会自然而然地想到观察者Observer，事实也确实如此，我们为获取到的`WorkInfo`添加观察者，运用观察者的触发机制实现对`Work`的管理。
```kotlin
WorkManager
   // 获取WorkManager的实现类的实例
   .getInstance(myContext)
   // 根据id获取到任务的LiveData<WorkInfo>
   .getWorkInfoByIdLiveData(uploadWorkRequest.id)
   .observe(lifecycleOwner, Observer { workInfo ->
        // 根据WorkInfo的状态做出反应
        if (workInfo != null && workInfo.state == WorkInfo.State.SUCCEEDED) {
        displayMessage("Work finished!")
    }
})
```

## 任务状态
`mState`是`WorkInfo.State`枚举类，用于描述任务状态：
* `ENQUEUED` 延迟时间结束，已进入队列
* `RUNNING` 正在运行
* `SUCCEEDED` 运行结束且成功
* `FAILED` 运行结束且失败
* `BLOCKED` 阻塞
* `CANCELLED` 已被取消

对于`OneTimeWorkRequest`有以上完整的状态；而对于`PeriodicWorkRequest`没有`SUCCEEDED`和`FAILED` 运行结束且失败，毕竟这两个是终止状态，而周期性任务（除非手动取消）不会自动终止。

## 任务进度
一些下载任务，我们希望让用户看到进度，那就用到了任务进度。
为了能够使用进度信息，我们需要注意以下要点：
* 接入协程任务接口`CoroutineWork`（不了解协程的小伙伴可以看我在Kotlin分类下关于协程的文章）。
* doWork要写成挂起函数（添加suspend关键字）

```kotlin
import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.Data
import androidx.work.WorkerParameters
import kotlinx.coroutines.delay

class ProgressWorker(context: Context, parameters: WorkerParameters) :
    CoroutineWorker(context, parameters) {

    companion object {
        const val Progress = "Progress"
        private const val delayDuration = 1L
    }

    override suspend fun doWork(): Result {
        val firstUpdate = workDataOf(Progress to 0)
        val lastUpdate = workDataOf(Progress to 100)
        setProgress(firstUpdate)
        delay(delayDuration)
        setProgress(lastUpdate)
        return Result.success()
    }
}
```
使用示例：
```java
WorkManager
    .getInstance(applicationContext)
    // requestId是WorkRequest的id
    .getWorkInfoByIdLiveData(requestId)
    .observe(observer, Observer { workInfo: WorkInfo? ->
        if (workInfo != null) {
            val progress = workInfo.progress
            val value = progress.getInt(Progress, 0)
            // TODO：运用进度信息做一些操作
        }
    })
```




