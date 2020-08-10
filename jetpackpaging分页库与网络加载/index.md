# Jetpack：Paging分页库与网络加载

# 引言
Paging3的模型改了，主要特点是新增了中转器以实现网络数据的持久化存储。

## Paging3模型
![5b07d09d821d80b313bfc285be11b929](Jetpack：Paging分页库与网络加载.resources/6EC12CA9-F881-4747-A61D-C1ADC7B1FDB4.png)
Paing数据加载的途径一：如果你希望数据实现持久化存储，你可以使用上述模型，网络与数据库的中转器RemoteMediator，将从NetworkService获取的数据写入数据库中，数据库返回PagingSource作为ViewModel的数据源。
Paing数据加载的途径二：如果你只是想从网络中加载数据而不打算持久化存储他们，你可以直接将创建PagingSource并从NetworkService将数据载入PagingSource。
下面，让我们一起来尝试一下，我将Google官方的Demo进一步简化，以方便你能快速抓住重点。
Demo的代码请点击[PagingWithNetworkDemo](https://github.com/NevilleNoah/PagingWithNetworkDemo)

## 前置准备
非重点，一笔带过。
### Gradle
关注Paging、Room的版本号和用kapt解析注解，可参考上一篇博文。

### NetworkService
用于获取数据，建议添加一个`isFinal`字段来判断是否是最后一个。

### Database
数据库的三层，entity、dao、databse



## Mediator
Mediator是网络与数据库之间的中介，用于将网络中的数据存储到数据库中。我们要完成两项基本工作：
* 准备路径参数：与api请求路径相关的参数。
* 确定加载行为：判断是操作是**刷新（REFRESH）**、**向前追加（PREPEND）**还是**向后追加（APPEND）**，并据此确定接下来的数据加载行为。
* 从网络中获取数据：使用NetworkService，获取response，并从中拿到data。
* 将数据写入数据库：使用Database，获取dao，并通过dao操作将数据写入数据库。
* 判断是否还有后续数据


### 准备路径参数
我们根据上述提到的工作，需要传入
* `Databse` 数据库。
* `NetworkService` 网络服务，包含Api接口。
* `Query` 请求路径`@Path`的相关参数（如果不需要也可以不传）。

我们的实体类是`User`，因此继承`RemoteMediator<Int, User>`。

```kotlin
class UserMediator(
    private val database: AppDatabase,
    private val networkService: NetworkService,
    private val query: String
) : RemoteMediator<Int, User>() {
    ...
}
```

### 确定加载行为
继承后，我们要实现`load`方法，该方法包含两个参数
* `LoadType` 加载类型
* `PagingState<Int, User>` 分页状态

并返回`MediatorResult`。
```kotlin
override suspend fun load(loadType: LoadType, state: PagingState<Int, User>): MediatorResult {
   ...
}
```

在方法中，我们根据参数`loadType`来确定加载行为。
`LoadType`是一个枚举类
```kotlin
enum class LoadType {
    REFRESH,
    PREPEND,
    APPEND
}
```

包含刷新REFRESH、向前追加PREPEND、向后追加APPEND三种状态。
假设我们这是一个下拉加载的列表，我们对这三种状态分别进行处理：
* 刷新REFRESH，即加载的方式与上一次一样，返回null即可
* 向前追加PREPEND，不需要上拉加载，所以返回`MediatorResult.Success(endOfPaginationReached = true)`来说明向前的时候已经没有更多数据了，我们上拉的时候就不会加载数据。
* 向后追加APPEND，一般是要加载数据的，我们根据参数`state`来判断一下是不是已经没有后续数据了，如果是就返回`MediatorResult.Success(endOfPaginationReached = true)`，不是就返回最后一条数据的id，用于后续的网络请求。


```kotlin
// 根据加载类型，设置加载行为的关键字
val loadKey = when (loadType) {
    // 刷新，
    LoadType.REFRESH -> null
    // 向上加载
    LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true) // endOfPaginationReached判读分页是否到底了，即还有没有后续数据
    // 向下加载
    LoadType.APPEND -> {

        val lastItem = state.lastItemOrNull()
            ?: return MediatorResult.Success(endOfPaginationReached = true)

        lastItem.id
    }
}
```

### 从网络中获取数据
我们分别来看我们先前三种状态处理后的关键字，在网络请求中会发生什么：
```kotlin
// 从网络获取应答
val response = networkService.getDataByItem(
    query = query,
    after = loadKey,
    limit = when (loadType) {
        LoadType.REFRESH -> state.config.initialLoadSize
        else -> state.config.pageSize
    }
)
```
假设我们上一次请求了id为1～10号的数据
* 刷新REFRESH，`after==loadKey==null`，`NetworkService`中会据此判断我们需要发送原来的请求，原来请求了id为1～10的数据，现在还是请求id为1～10的数据。此外，limit会根据state中配置的条目`initialLoadSize`来确定加载数据的数量。
* 向前追加PREPEND，因为早已返回了`RemoteMediator.MediatorResult.Success(endOfPaginationReached = true)`，不会执行此段代码。
* 向后追加APPEND，`after==loadKey==lastItem.id==10`，`NetworkService`需要请求`11~20`号的数据。


### 将数据写入数据库
操作数据库需要用到dao层，因此引入`UserDao`，并通过数据库的事务`withTranscation`来存入数据。如果是刷新，就先清空原数据，再存入新数据。
```kotlin
class UserMediator(
    private val database: AppDatabase,
    private val networkService: NetworkService,
    private val query: String
) : RemoteMediator<Int, User>() {

    private val userDao = database.getUserDao()

    override suspend fun load(loadType: LoadType, state: PagingState<Int, User>): MediatorResult {
    ...
    
            // 将数据写入数据库
            database.withTransaction {

                if (loadType == LoadType.REFRESH) {
                    userDao.clearAll()
                }

                userDao.insertAll(response.data)
            }
    ...
    }
    
```

### 判断是否还有后续数据
在`load`方法中，千万不要忘记判断还有没有后续数据，毕竟load方法最后要返回`MediatorResult`
```kotin
// 根据后台给出的数据判断是否还有后续数据
MediatorResult.Success(
    endOfPaginationReached = response.isFinal
)
```

### 完整代码
```kotlin
// 网络与数据库之间的中转站
@OptIn(ExperimentalPagingApi::class)
class UserMediator(
    private val database: AppDatabase,
    private val networkService: NetworkService,
    private val query: String
) : RemoteMediator<Int, User>() {

    private val userDao = database.getUserDao()

    override suspend fun load(loadType: LoadType, state: PagingState<Int, User>): MediatorResult {

        return try {

            // 根据加载类型进行不同的操作
            val loadKey = when (loadType) {
                // 刷新
                LoadType.REFRESH -> null
                // 向上加载
                LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true) // endOfPaginationReached判读分页是否到底了，即还有没有后续数据
                // 向下加载
                LoadType.APPEND -> {

                    val lastItem = state.lastItemOrNull()
                        ?: return MediatorResult.Success(endOfPaginationReached = true)

                    lastItem.id
                }
            }

            // 从网络获取应答
            val response = networkService.getDataByItem(
                query = query,
                after = loadKey,
                limit = when (loadType) {
                    LoadType.REFRESH -> state.config.initialLoadSize
                    else -> state.config.pageSize
                }
            )

            // 将数据写入数据库
            database.withTransaction {

                if (loadType == LoadType.REFRESH) {
                    userDao.clearAll()
                }

                userDao.insertAll(response.data)
            }

            // 根据后台给出的数据判断是否还有后续数据
            MediatorResult.Success(
                endOfPaginationReached = response.isFinal
            )

        } catch (e: IOException) {
            return MediatorResult.Error(e)
        } catch (e: HttpException) {
            return MediatorResult.Error(e)
        }
    }
}
```

## PaingSource
曾经的“三人组”已被丢弃：
* PositionalDataSource
* ItemKeyedDataSource
* PageKeyedDataSource

已统一为：
* PagingSource

你可以使用PagingSource来实现“三人组”的功能。
关键在于设置`prevKey`和`nextKey`。

### ItemKeyDataSource
`ItemKeyDataSource`继承`PagingSource`，通过网络请求来直接获得数据。适用于前后加载有关联的数据，例如有序的id等等。此例中就是运用`data.firstOrNull()?.id`和`data.lastOrNull()?.id`作为上下加载的关键字`prevKey`和`nextKey`。上拉加载就启用`prevKey`，下拉就在启用`nextKey`。
```kotlin
// ItemKey，使用条目中的关键信息进行关联查询。
// 适合一些上下有关联的，例如有序的id。
class ItemKeyUserDataSource(
    private val networkService: NetworkService,
    private val query: String
) : PagingSource<String, User>() {

    override suspend fun load(params: LoadParams<String>): LoadResult<String, User> {
        return try {

            // 从网络中获取应答
            val response = networkService.getDataByItem(
                // 后端接口查询数据库所需的信息
                query = query,
                // 如果是Append向后追加，就启用after，after是上一次请求的nextKey
                after = if (params is LoadParams.Append) params.key else null,
                // 如果是Prepend向前追加，就启用before，before是上一次请求的prevKey
                before = if (params is LoadParams.Prepend) params.key else null,
                // 尺寸
                limit = params.loadSize
            )

            // 从应答中获取数据
            val data = response.data

            // 构建PageSource作为数据源
            // prevKey和nextKey用于下一次加载，根据Append和Prepend来决定加载使用哪一个
            LoadResult.Page(
                data = data,
                prevKey = data.firstOrNull()?.id,
                nextKey = data.lastOrNull()?.id
            )

        } catch (e: IOException) {
            return LoadResult.Error(e)
        } catch (e: HttpException) {
            return LoadResult.Error(e)
        }
    }
}
```

### PageKeyDataSource
`PageKeyDataSource`继承`PagingSource`，使用页码`page`作为上下加载的关键字`prevKey`和`nextKey`。适合已经做好分页的数据，例如翻页书籍。
```kotlin
// PageKey，按页码加载。
// 适用于分页加载的场景。
class PageKeyUserDataSource(
    private val networkService: NetworkService,
    private val query: String
) : PagingSource<Int, User>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        return try {

            // 页码
            val page = params.key ?: 0

            // 从网络中获取应答
            val response = networkService.getDataByPage(
                // 后端接口查询数据库所需的信息
                query = query,
                // 如果是Append向后追加，就启用after，after是上一次请求的nextKey
                page = params.key,
                // 尺寸
                limit = params.loadSize
            )

            // 从应答中获取数据
            val data = response.data

            // 构建PageSource作为数据源
            // prevKey和nextKey用于下一次加载，根据Append和Prepend来决定加载使用哪一个
            LoadResult.Page(
                data = data,
                // 如果当前页码不为0，则赋值为前一页的页码
                prevKey = if (page == 0) null else page - 1,
                // 如果是最后一页，则下一页为空，否则赋值为下一页的页码
                nextKey = if (response.isFinal) null else page + 1
            )

        } catch (e: IOException) {
            return LoadResult.Error(e)
        } catch (e: HttpException) {
            return LoadResult.Error(e)
        }
    }
}
```


### PositionalKeyDataSource
你自己写吧，反正都一个套路...
核心就是设置`prevKey`和`nextKey`。



## Repository
针对通过Mediator和直接对接PagingSource的数据，最终都要在`Repository`中转化为`PageData`才能供UI使用，我们在`ViewModel`中调用`Repository`中的方法来拿到`PagingData`。

### Network+Mediator+Database
对于通过`Mediator`的数据，我们创建`Pager`对象时传入中转器和页面配置，并在lambda表达式中进行数据库查询操作，查询的结果作为数据源，最后`flow`成`PagingData`。
```kotlin
// 持久化存储，运用remoteMediator将数据库和网络桥接起来，从网络将数据写入数据库
fun getUsersInDb(query: String, pageSize: Int) = Pager(
    config = PagingConfig(pageSize),
    remoteMediator = UserMediator(
        database,
        networkService,
        query
    )
) {
    database.getUserDao().selectAll()
}.flow
```


### Network+PagingSource
对于直接对接PagingSource的数据，我们创建`Pager`时传入页面配置，并在lambda表达式中直接创建我们的数据源，最后`flow`成`PagingData`。
```kotlin
// 非持久化存储，直接从网络写入内存
fun getUsersInMemory(query: String, pageSize: Int) = Pager(
    config = PagingConfig(pageSize)
) {
    ItemKeyUserDataSource(networkService, query)
}.flow
```

### 完整代码
```kotlin
class UserRepository(private val database: AppDatabase, private val networkService: NetworkService) {

    // 持久化存储，运用remoteMediator将数据库和网络桥接起来，从网络将数据写入数据库
    fun getUsersInDb(query: String, pageSize: Int) = Pager(
        config = PagingConfig(pageSize),
        remoteMediator = UserMediator(
            database,
            networkService,
            query
        )
    ) {
        database.getUserDao().selectAll()
    }.flow

    // 非持久化存储，直接从网络写入内存
    fun getUsersInMemory(query: String, pageSize: Int) = Pager(
        config = PagingConfig(pageSize)
    ) {
        ItemKeyUserDataSource(networkService, query)
    }.flow

}
```
再往上ViewModel、Adapter...反正大家都会了就不写了，不会的参考我上一篇博文。










