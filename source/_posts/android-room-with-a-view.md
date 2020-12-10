---
title: Android Jetpack 实战之使用 Room 操作数据库
date: 2020-12-10 19:22:00
tags: Jetpack
categories: Android
---

# Android Jetpack 实战之使用 Room 操作数据库

Room 是 Android Jetpack 中用来处理数据库的框架，它可以用来替代原有的 SQLiteOpenHelper，简化数据库操作。

Android Room with a View 是一个 Google Codelab 用来展示 Jetpack Room 用法的 app。它可以输入一个单词并自动刷新显示数据库中的所有单词。通过 Android Room with a View 这个项目可以熟悉 Android Jetpack 的 LiveData、ViewModel、Room 的用法。

## LiveData

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

### asLiveData

```kotlin
class WordViewModel(private val repository: WordRepository) : ViewModel() {

    val allWords: LiveData<List<Word>> = repository.allWords.asLiveData()
    ...

}
```

## ViewModel

## Room

## 参考资料

[Codelab : Android Room with A View - Kotlin](https://developer.android.com/codelabs/android-room-with-a-view-kotlin)

[googlecodelabs/android-room-with-a-view](https://github.com/googlecodelabs/android-room-with-a-view)