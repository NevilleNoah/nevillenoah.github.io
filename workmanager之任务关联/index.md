# WorkManager之任务关联


## 任务排序
我们使用`WorkManager`的函数：
* `beginWith()`
* `then()`

来完成任务排序，例如：
```kotlin
WorkManager
    .getInstance(myContext)
    // 开始时对数据进行过滤
    .beginWith(filter)
    // 再对数据进行压缩
    .then(compress)
    // 再上传数据
    .then(upload)
    // 千万别忘记enqueue()，进入队列后才能执行
    .enqueue()
```

### 链接原理
在《WorkManager之添加任务》中我们提到过`WorkContinuation`，他是用于任务排序的。
`WorkManager`之所能像这样执行，是因为`beginWith()`会返回`WorkContinuation`，每个`WorkContinuation`又能执行`then()`，执行完`then`之后又返回`WorkContinuation`...由此不断循环，构成最终的一个`WorkContinuation`，将其传入执行队列中。
这就是一个单向单链表的构造的过程，每次添加新结点，都返回新的链条，`WorkContinuation`就相当于每次返回的链条。
此外，不仅仅是能加入新结点实现纵向连接，链表和链表之间还可以横向连接：
```java
/*
 * <pre>
 *     A       C
 *     |       |
 *     B       D
 *     |       |
 *     +-------+
 *         |
 *         E    </pre>
 */

WorkContinuation left = workManager.beginWith(A).then(B);
WorkContinuation right = workManager.beginWith(C).then(D);
WorkContinuation final = WorkContinuation.combine(Arrays.asList(left, right)).then(E);
final.enqueue();
```
上述代码就是先构造了左链表和右链表，最后在左右链表结合后再添加E结点，形成最终的**工作链**final。当final开始执行时，左右链表会先按各自规定的顺序执行AB、CD，等左右链表的最后一个任务B和D执行完后，再执行E。

## 输出合并
我们在《WorkManager之添加任务》中学习了“任务输入/输出”。在工作链中，父级任务的输出将作为输入传递给直接相连的子任务，例如A的输出会传给B作为输入，C的输出会传给D作为输入，B和D的输出会作为输入传给E。
等等，有点不对劲。B和D两个输入源，怎么输出给E呢？既然E要等B和D，说明B和D肯定是同时用到的，两者必须同时传入，不可能先传一个再传另一个。没错，因此我们必须将两者的结果进行合并。
我们通过给`WorkRequest`设置`InputMerger`来实现，例如对于`OneTimeWorkRequest`：
```kotlin
val compress: OneTimeWorkRequest = OneTimeWorkRequestBuilder<CompressWorker>()
        // 设置合并方式为ArrayCreatingInputMerger
        .setInputMerger(ArrayCreatingInputMerger::class)
        .build()
```
有两种合并的方式：
* `OverwritingInputMerger`:覆盖式，将所有输入中的所有键添加到输出中。如果发生冲突，后面的键会覆盖前面的键。
* `ArrayCreatingInputMerger`:保留式，会创建数组用于存储所有输入。

## 链接与状态
我们在《WorkManager之任务信息》中提到了`WorkInfo.State`，我们要注意三点：
* 仅当所有父级任务成功完成后，当前任务会变为`ENQUEUED`。
* 任意父级任务失败后，当前任务会变成`FAILED`。
* 任意父级任务取消后，当前任务会变成`CANCELLED`。

这里详细陈述取消任务。

### 取消任务
`CANCELLED` ，表示已经被取消的任务。
我们取消任务时，可能是取消**一个**，通过id取消
```java
// 根据id取消特定的一个任务
cancelWorkById(@NonNull UUID id)
```

也可能是取消**一批**，那就要考虑关联的方式
任务之间的关联方式：
* 标签Tag
* 链接WorkContinuation（之前有提过，是任务排序用的，下一章详细讲解）

```java
// 根据tag取消所有带有这个tag的任务
cancelAllWorkByTag(@NonNull final String tag)

// 如果任务之间是有链接关系的，那么取消了当前这个任务，后续的任务都会直接。
cancelWorkById(@NonNull UUID id)
```

也可能是取消**全部**
```java
cancelAllWork()
```
被取消后的任务全部进去`CANCELLED`状态。


