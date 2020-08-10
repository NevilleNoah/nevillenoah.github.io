# Jetpack：Paging分页库之进度状态

# 引言
我们在数据加载的过程中需要用到进度组件（例如加载圈、进度条）来显示加载状态。`Paging`为此提供了一个非常优秀的适配器`LoadStateAdapter`来支持这些进度组件。
代码用的是谷歌官方的Demo。

## 挂载到PagingDataAdapater
我们在之前的文章中提到到`PagingDataAdapater`，它是用来给`RecyclerView`加载`PaingData`的。我们的`LoadStateAdapter`需要通过`withLoadStateHeaderAndFooter`挂载到`PagingDataAdapater`下。

```kotlin
mPagingDataAdapater.withLoadStateHeaderAndFooter {
    // 头部加载的适配器
    header = mLoadStateAdapter(mPagingDataAdapater),
    // 底部加载的适配器
    footer = mLoadStateAdapter(mPagingDataAdapater)
}
```
我们将`PagingDataAdapter`传入到`LoadStateAdapater`中，这样`LoadStateAdapter`就能拿到`PagingDataAdapter`的状态了。

## LoadStateAdapter
然后就是适配器的常规操作，传入`ViewHolder`，绑定数据与视图。不过这里要注意的是：
* 绑定的是`LoadState`
* `ViewHolder`是`NetworkStateItemViewHolder`。这是一个自定义的`ViewHolder`，用于存放进度组件。

```kotlin
class MyLoadStateAdapter(
        private val adapter: PostsAdapter
) : LoadStateAdapter<NetworkStateItemViewHolder>() {
    override fun onBindViewHolder(holder: NetworkStateItemViewHolder, loadState: LoadState) {
        holder.bindTo(loadState)
    }

    override fun onCreateViewHolder(
            parent: ViewGroup,
            loadState: LoadState
    ): NetworkStateItemViewHolder {
        return NetworkStateItemViewHolder(parent) { adapter.retry() }
    }
}
```

## ViewHolder
布局文件`network_state_item.xml`
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:orientation="vertical"
              android:padding="8dp">
    <TextView
        android:id="@+id/error_msg"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_horizontal"/>
    <ProgressBar
        android:id="@+id/progress_bar"
        style="?android:attr/progressBarStyle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="center"/>
    <Button
        android:id="@+id/retry_button"
        style="@style/Widget.AppCompat.Button.Colored"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:text="@string/retry"/>
</LinearLayout>
```

ViewHolder实现
```kotlin
class NetworkStateItemViewHolder(
    parent: ViewGroup,
    private val retryCallback: () -> Unit
) : RecyclerView.ViewHolder(
    LayoutInflater.from(parent.context).inflate(R.layout.network_state_item, parent, false)
) {
    private val binding = NetworkStateItemBinding.bind(itemView)
    private val progressBar = binding.progressBar
    private val errorMsg = binding.errorMsg
    private val retry = binding.retryButton
        .also {
            it.setOnClickListener { retryCallback() }
        }

    fun bindTo(loadState: LoadState) {
        progressBar.isVisible = loadState is Loading
        retry.isVisible = loadState is Error
        errorMsg.isVisible = !(loadState as? Error)?.error?.message.isNullOrBlank()
        errorMsg.text = (loadState as? Error)?.error?.message
    }
}
```
我们在布局文件中放入了三个组件，分别是错误信息显示、加载圈、重试按钮。
我们在`ViewHolder`中的`bindTo`方法中，根据`LoadState`加载状态来改变组件的可见性从而实现加载效果。
这已经可以满足一般性的加载需求，你也可以根据自己的需要适当添加动画，来增强视觉观感。
