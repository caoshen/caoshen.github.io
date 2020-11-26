# Android 源码的迭代器模式

## 迭代器模式的介绍

迭代器模式（Iterator Pattern） 又称为游标（Cursor）模式，是行为型设计模式之一。迭代器模式是一个比较古老的设计模式，其源于对容器的访问，比如 Java 中的 List、Map、数组等。

## 迭代器模式的定义

提供一个方法顺序访问一个容器对象中的各个元素，而又不需要暴露该对象的内部表示。

## Android 源码中的迭代器模式

Android 中典型的迭代器模式例子是数据库查询使用 Cursor，当我们使用 SQLiteDatabase 的 query 方法查询数据时，会返回一个 Cursor 游标对象，该游标对象实质就是一个具体的迭代器，我们可以使用它遍历数据库查询所得到的结果集。

首先定义一个 SQLiteOpenHelper。

```java
public class DBHelper extends SQLiteOpenHelper {

    public DBHelper(Context context) {
        super(context, "DB_AIGE", null, 1);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE table_aige (_id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, sex TEXT)");
        db.execSQL("INSERT INTO table_aige (name, sex) values ('Aige', 'man')");
        db.execSQL("INSERT INTO table_aige (name, sex) values ('SMBrother', 'man')");
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }
}
```

构造 ContentProvider：

```java
public class DataProvider extends ContentProvider {

    private DBHelper dbHelper;

    @Override
    public boolean onCreate() {
        dbHelper = new DBHelper(getContext());
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        return db.query("table_aige", projection, null, null, null, null, null);
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        return null;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
        return 0;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
        return 0;
    }
}
```

在 Activity 使用 ContentProvider:

```java
public class IteratorActivity extends ListActivity {

    private static final Uri URI = Uri.parse("content://com.android.androidsamples.dataprovider/table_aige");

    private static final String[] PROJECTION = new String[]{"name", "sex"};

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Cursor cursor = getContentResolver().query(URI, PROJECTION, null, null, null);

        List<Map<String, String>> list = new ArrayList<>();
        cursor.moveToFirst();
        do {
            Map<String, String> item = new HashMap<>();
            item.put("name", cursor.getString(0));
            item.put("sex", cursor.getString(1));
            list.add(item);
        } while (cursor.moveToNext());
        cursor.close();
        setListAdapter(new SimpleAdapter(this, list, android.R.layout.simple_list_item_2,
                new String[]{"name", "sex"}, new int[]{android.R.id.text1, android.R.id.text2}));
    }
}
```

注册组件：

```xml
        <activity android:name=".iterator.IteratorActivity" />

        <provider
            android:authorities="com.android.androidsamples.dataprovider"
            android:name=".iterator.DataProvider"/>
```