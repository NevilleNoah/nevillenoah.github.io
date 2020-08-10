# Jetpack：Paging分页库之使用基础

# 引言
也许你还在`RecyclerView`的下拉加载而烦恼，也许你还在为做分页缓存而头疼，所以，他来了——`Paging`分页库。
我将代码存到了gitub上，点击[PagingDemo](https://github.com/NevilleNoah/PagingDemo)跳转。强烈建议下载代码后，再对照博文阅读。

{{< admonition tip >}}
别盯着油管上谷歌官方2018年IO大会的教程了，Paging库中的代码早已发生了翻天覆地的变化。时代变了，大人～
{{< /admonition >}}

## 配置
配置我使用versions的写法，请结合Demo查看，我提一下几个要点。

### 注解使用kapt
请确保你使用kapt来解析注解
```
apply plugin: 'kotlin-kapt'

dependencies {
    kapt deps.room.compiler
}
```
请确保注解解释器的版本大于2.3.0
```
versions.room = "2.3.0-alpha01"
```
否则你的注解将无法正确解析。

### Kotlin/Java编译器版本
请确保在你的Kotlin/Java版本皆指向1.8。
在module的`build.gradle`中
```
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
        freeCompilerArgs += ["-Xopt-in=kotlin.RequiresOptIn"]
    }
}
```

{{< admonition >}}
如果你注意以上两点，那么你的程序基本不会因为配置的原因原地爆炸。
{{< /admonition >}}

## 数据库
数据库包含实体层（Table）、Dao层（SQL）、数据库层（Scheme）。
### Entity
为类打上`@Entity`注解，为主键id打上`@PrimaryKey`注解并标明自增。
```kotlin
@Entity
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Int,
    val name: String
)
```

### Dao
定义了三个方法
* `allUsers`查询，我们的终极目的就是查出来
* `insert`插入，后面用于填入模拟数据
* `deleteAll`删除，会清空数据库中所有的数据

重点：
* 我们的`allUsers()`返回`PagingSource<K, V>`

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM User")
    fun allUsers(): PagingSource<Int, User>

    @Insert
    fun insert(users: List<User>)

    @Query("DELETE FROM User")
    fun deleteAll()
}
```

### Database
为类打上`Database`注解并指明包含的类和数据库版本，引入了dao层，并且通过静态方法创建静态实例。

{{< admonition tip >}}
建议大家学会这种方法，不要再在Repositories层或者viewModel层build数据库了，太杂了。
{{< /admonition>}}

```kotlin
@Database(entities = [User::class], version = 1)
abstract class UserDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao

    companion object {
        private var instance: UserDatabase? = null

        /**
         * 提供互斥的静态方法用于获取数据库实例
         * 如果实例没创建，则创建；已经创建了，直接返回实例
         * 减少了创建实例所花费的开销
         */
        @Synchronized
        fun get(context: Context): UserDatabase {

            if (instance == null) {

                instance = Room.databaseBuilder(
                    context.applicationContext,
                    UserDatabase::class.java,
                    "UserDatabase"
                ).build()
            }
            return instance!!
        }
    }
}
```

## 适配器
适配器完成三件事
* Layout
* ViewHolder
* Adapter

### layout
为Activity和RecyclerView的item分别建立布局。

#### Activity
注意RecyclerView要指定layoutManager
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".activity.MainActivity">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerview"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layoutManager="LinearLayoutManager"/>

    <Button
        android:id="@+id/delete"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Button"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent" />


</androidx.constraintlayout.widget.ConstraintLayout>
```

#### RecycleView的item
两个文本控件，分别用于显示User的id和name。
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/item_id"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="TextView"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/item_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="TextView"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toEndOf="@+id/item_id"
        app:layout_constraintTop_toTopOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

### ViewHolder
根据我们创建的item布局和User实体类，我们写一个ViewHolder
```kotlin
class UserViewHolder(parent: ViewGroup): RecyclerView.ViewHolder(
    LayoutInflater.from(parent.context).inflate(R.layout.item, parent, false)
) {

    // 控件
    private val idView = itemView.item_id
    private val nameView = itemView.item_name

    // 数据
    private var user: User? = null

    // 用于数据与控件的绑定
    fun bindTo(user : User?) {
        this.user = user
        idView.text = user?.id.toString()
        nameView.text = user?.name
    }
}
```

### Adapter
重点：
* 继承了PagingDataAdapter<Object, ViewHolder>
* diffCallback的判断，建议采用这种变量写法会比较清晰（Kotlin的特性不用白不用）

```kotlin
/**
 * 继承PagingDataAdapter
 */
class UserAdapter : PagingDataAdapter<User, UserViewHolder>(diffCallback) { // diffCallback用内部定义的变量
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        // 数据绑定：传入条目的位置信息，将数据绑定到这个位置的条目上
        holder.bindTo(getItem(position))
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        // 布局创建：传入父布局，在父布局中创建布局
        return UserViewHolder(parent)
    }

    companion object {
        // 同异判断：分别用于判断条目是否更新和内容是否更新
        private val diffCallback = object : DiffUtil.ItemCallback<User>() {
            override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
               return oldItem.id == newItem.id
            }

            override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
               return oldItem == newItem
            }

        }
    }
}
```

## 活动
活动需要完成
* ViewModel
* Activity

### ViewModel
重点：
* 我们的数据由`Pager`对象构建，`PagingConfig`作为参数传入，方法体中执行数据库查询操作，最后flow来获得`Flow<PagingData<Value>>`对象，最终将其赋值给`allUsers`。
* 我定义了两个挂起方法和一组模拟数据，我将在Activity中的Kotlin协程中使用它们。
* 你发现我的模拟数据由`id`，但是上面的数据库中又指明`id`是自增的，这很矛盾。其实模拟数据的`id`只是我用来方便创建对象用的，真正的id由数据库生成，我们最终数据呈现所使用的id也是数据库中的`id`。但我这样做希望提醒你一点：我在`insertAll`的`map`中设置`id=0`，那么就会启用自增，数据库中的id，与你创建对象的id属性无关。

```kotlin
class MainViewModel(app: Application): AndroidViewModel(app) {

    private val dao = UserDatabase.get(app).userDao()

    // 模拟数据，用于填充数据库
    private val USER_DATA = arrayListOf<User>(
        User(1, "Name1"),
        User(2, "Name2"),
        User(3, "Name3"),
        User(4, "Name4"),
        User(5, "Name5")
    )

    /**
     * 通过Pager的构造方法中的config参数配置分页的参数
     * 通过Pager的方法体传入数据
     */
    val allUsers = Pager(
        PagingConfig(
            pageSize = 5,
            enablePlaceholders = true
        )
    ) {
        dao.allUsers()
    }.flow

    // 删除所有数据
    suspend fun deleteAll() {
        withContext(Dispatchers.IO) {
            dao.deleteAll()
        }
    }

    // 填充模拟数据
    suspend fun insertAll(context: Context) {
        withContext(Dispatchers.IO) {
            UserDatabase.get(context.applicationContext).userDao()
                .insert(
                    USER_DATA.map { User(id = 0, name = it.name) }
                )
        }
    }
}
```

### Activity
重点：
* 在协程中，你需要`adapter.submitData(it)`来将数据传递给适配器。

```kotlin
lass MainActivity : AppCompatActivity() {

    private val viewModel by viewModels<MainViewModel>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val adapter = UserAdapter()
        recyclerview.adapter = adapter

        lifecycleScope.launch {
            // 插入所有模拟数据
            viewModel.insertAll(application)
            // 更新适配器数据
            @OptIn(ExperimentalCoroutinesApi::class)
            viewModel.allUsers.collectLatest {
                adapter.submitData(it)
            }
        }

        // 绑定删除数据事件
        delete.setOnClickListener {
            lifecycleScope.launch(Dispatchers.IO) {
                viewModel.deleteAll()
            }
        }
    }
}
```

# 总结
这只是一个很基础的Demo。我们平时的开发场景，更多是结合通过网络来进行分页处理。在下一篇博文中，我将陈述如何操作。
我建议你更多的参考Google在Github上留下的Demo。虽然有些写法并不好，但核心的东西都会表达出来，不要再依赖过时的视频来进行学习。







