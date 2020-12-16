---
title: Android Jetpack 之使用 Room 操作数据库
date: 2020-12-10 19:22:00
tags: Jetpack
categories: Android
---

# Android Jetpack 之使用 Room 操作数据库

Room 是 Android Jetpack 中用来处理数据库的框架，它可以用来替代原有的 SQLiteOpenHelper，简化数据库操作。

Android Room with a View 是一个 Google Codelab 用来展示 Jetpack Room 用法的 app。它可以输入一个单词并自动刷新显示数据库中的所有单词。通过 Android Room with a View 这个项目可以熟悉 Android Jetpack 的 LiveData、ViewModel、Room 的用法。

## Room

Room 需要先新建一个 abstract 的 RoomDatabase 类，它继承自 RoomDatabase。

```kotlin
abstract class WordRoomDatabase : RoomDatabase() {
}
```

使用 @Database 注解表明使用 RoomDatabase 构造数据库。

@Database 注解有 2 个 方法：

- entities 表示数据库中有哪些表。通常用一个数据类表示表结构，比如 Word::class。
- version 表示当前数据库的版本号。数据库升级时会用到版本号。

```kotlin
@Database(entities = [Word::class], version = 1)
```

WordRoomDatabase 类有一个抽象方法，用来返回 Dao 类，比如 WordDao。WordDao 定义了数据的增删改查接口。

```kotlin
abstract class WordRoomDatabase : RoomDatabase() {

    abstract fun wordDao(): WordDao
}
```

getDatabase 方法使用单例模式构造 WordRoomDatabase 的实例。

```kotlin
abstract class WordRoomDatabase : RoomDatabase() {

        fun getDatabase(context: Context, scope: CoroutineScope): WordRoomDatabase {
            return INSTANCE ?: synchronized(this) {
                var instance = Room.databaseBuilder(
                        context.applicationContext,
                        WordRoomDatabase::class.java,
                        "word_database"
                )
                        // when migrate database, wipes and rebuilds database
                        .fallbackToDestructiveMigration()
                        .addCallback(WordDatabaseCallback(scope))
                        .build()
                INSTANCE = instance
                instance
            }
        }
```

getDatabase 使用 Room.databaseBuilder 构造 database。databaseBuilder 可以传入一个 Callback，它可以在数据库的生命周期执行回调，比如 database 的 onCreate 执行数据库的初始化操作，插入几条数据。

```kotlin
        class WordDatabaseCallback(private val scope: CoroutineScope) : RoomDatabase.Callback() {
            override fun onCreate(db: SupportSQLiteDatabase) {
                super.onCreate(db)
                // init database content
                INSTANCE?.let { database ->
                    scope.launch(Dispatchers.IO) {
                        populateDatabase(database.wordDao())
                    }
                }
            }
        }
```

完整的 WordRoomDatabase 代码如下：

```kotlin
@Database(entities = [Word::class], version = 1)
abstract class WordRoomDatabase : RoomDatabase() {

    abstract fun wordDao(): WordDao

    companion object {
        @Volatile
        private var INSTANCE: WordRoomDatabase? = null

        fun getDatabase(context: Context, scope: CoroutineScope): WordRoomDatabase {
            return INSTANCE ?: synchronized(this) {
                var instance = Room.databaseBuilder(
                        context.applicationContext,
                        WordRoomDatabase::class.java,
                        "word_database"
                )
                        // when migrate database, wipes and rebuilds database
                        .fallbackToDestructiveMigration()
                        .addCallback(WordDatabaseCallback(scope))
                        .build()
                INSTANCE = instance
                instance
            }
        }

        class WordDatabaseCallback(private val scope: CoroutineScope) : RoomDatabase.Callback() {
            override fun onCreate(db: SupportSQLiteDatabase) {
                super.onCreate(db)
                // init database content
                INSTANCE?.let { database ->
                    scope.launch(Dispatchers.IO) {
                        populateDatabase(database.wordDao())
                    }
                }
            }
        }

        suspend fun populateDatabase(wordDao: WordDao) {
            wordDao.deleteAll()
            var word = Word("Hello")
            wordDao.insert(word)
            word = Word("World!")
            wordDao.insert(word)
        }
    }
}
```

## LiveData 与 Flow

LiveData 将数据转换为可以被观察的数据。当数据发生变化时，LiveData 会将变化传递给 LiveData 定义的观察者。

wordViewModel.allWords 是一个 LiveData，当 words 发生变化时通知 adapter 刷新 recyclerview。

```kotlin
class MainActivity : AppCompatActivity() {
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        wordViewModel.allWords.observe(this) { words ->
            words.let {
                adapter.submitList(it)
            }
        }
    }
}
```

repository.allWords 是 wordDao 定义的 Flow 流，它以 Flow [异步流](https://www.kotlincn.net/docs/reference/coroutines/flow.html)的形式返回 Word 列表。

asLiveData 方法将 Flow 转换为 LiveData。

```kotlin
class WordViewModel(private val repository: WordRepository) : ViewModel() {

    val allWords: LiveData<List<Word>> = repository.allWords.asLiveData()

}
```

```kotlin
class WordRepository(private val wordDao: WordDao) {

    val allWords: Flow<List<Word>> = wordDao.getAlphabetizeWords()
}
```

lifecycle-livedata-ktx 依赖库的 FlowLiveData.kt 文件定义了 asLiveData 扩展函数。它可以将 Flow 转换为 LiveData。当数据库发生变化时，Flow 可以实时更新，从而通知 LiveData 刷新界面。

```kotlin
fun <T> Flow<T>.asLiveData(
    context: CoroutineContext = EmptyCoroutineContext,
    timeoutInMs: Long = DEFAULT_TIMEOUT
): LiveData<T> = liveData(context, timeoutInMs) {
    collect {
        emit(it)
    }
}
```

实际上 Flow 的构造过程定义在 room-ktx 依赖库的 CoroutinesRoom 类的 createFlow，它给数据库加上了观察者（observer）。

```kotlin
class CoroutinesRoom private constructor() {

        @JvmStatic
        fun <R> createFlow(
            db: RoomDatabase,
            inTransaction: Boolean,
            tableNames: Array<String>,
            callable: Callable<R>
        ): Flow<@JvmSuppressWildcards R> = flow {
        }
    }
}
```

WordDao 数据访问接口在查询时返回 Flow。

```kotlin
@Dao
interface WordDao {
    // Flow notify
    @Query("SELECT * FROM word_table ORDER BY word ASC")
    fun getAlphabetizeWords(): Flow<List<Word>>

    /**
     * ignore 如果有冲突就不插入
     */
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insert(word: Word)

    @Query("DELETE FROM word_table")
    suspend fun deleteAll()
}
```

最后在 WordViewModel 保存 LiveData。外部 Activity 通过 WordViewModel 操作数据。

```kotlin
class WordViewModel(private val repository: WordRepository) : ViewModel() {

    val allWords: LiveData<List<Word>> = repository.allWords.asLiveData()
    ...

}
```

## 参考资料

[Codelab : Android Room with A View - Kotlin](https://developer.android.com/codelabs/android-room-with-a-view-kotlin)

[googlecodelabs/android-room-with-a-view](https://github.com/googlecodelabs/android-room-with-a-view)

[Kotlin 数据流](https://developer.android.com/kotlin/flow?hl=zh-cn)