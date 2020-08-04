---
title: ContentProvider必备基础
date: 2020-07-29 20:44:07
categories: [Android基础]
tags: [Android基础]
---

### 前言

本文是介绍Android的四大组件之一的ContentProvider。

### 目录

#### 一、什么是ContentProvider

ContentProvider 是 Android 中提供的专门用于不同应用间数据交互和共享的组件。它实际上是对SQLiteOpenHelper 的进一步封装，以一个或多个表的形式将数据呈现给外部应用，通过 Uri 映射来选择需要操作数据库中的哪个表，并对表中的数据进行增删改查处理。ContentProvider 其底层使用了 Binder 来完成APP 进程之间的通信，同时使用匿名共享内存来作为共享数据的载体。ContentProvider 支持访问权限管理机制，以控制数据的访问者及访问方式，保证数据访问的安全性。

<!--more-->

#### 二、相关知识

##### URI

**Uniform Resource Identifier** 即统一资源标识符，外界进程通过 `URI` 找到对应的 ContentProvider 和其中的数据，再进行数据操作。以联系人Contacts 的 Uri 为例，其结构如下所示：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghdopxmxsmj30ig04kdfp.jpg)

**schema:**  Android 中固定为 `content://`。

**authority:**  用于唯一标识一个 ContentProvider。

**path:**  ContentProvider 中数据表的表名。

**id:**  数据表中数据的标识，可选字段。

##### MIME类型

指定某种扩展名的文件用什么应用程序来打开的方式类型。ContentProvider 会根据 URI 来返回一个包含两部分 MIME 类型的字符串，每种 `MIME` 类型一般由2部分组成 = 类型 + 子类型，如：

```java
text/html // 类型 = text、子类型 = html
text/css 
text/xml 
application/pdf
```

##### UriMatcher类

是一个工具类，帮助匹配 ContentProvider 中的 Uri。提供了两个方法 **addURI()** 和 **match()** 方法。

- **addURI(String authority,String path, int code)：**是在 ContentProvider 添加一个用于匹配的 Uri，当匹配成功时返回 code 。在 ContentProvider 中注册 URI ，把 Uri 和 code 相关联，Uri可以是精确的字符串，Uri 中带有*表示可匹配任意text，#表示只能匹配数字
- **match(Uri uri) ：**根据 URI 匹配 ContentProvider 中对应的数据表，对 Uri 进行验证。

```java
// 步骤1：初始化UriMatcher对象
    UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH); 
    //常量UriMatcher.NO_MATCH  = 不匹配任何路径的返回码
    // 即初始化时不匹配任何东西

// 步骤2：在ContentProvider 中注册URI（addURI（））
    int URI_CODE_a = 1；
    int URI_CODE_b = 2；
    matcher.addURI("com.prsuit.myprovider", "user", URI_CODE_a); 
    matcher.addURI("com.prsuit.myprovider", "book", URI_CODE_b); 
    // 若URI资源路径 = content://com.prsuit.myprovider/user ，则返回注册码URI_CODE_a
    // 若URI资源路径 = content://com.prsuit.myprovider/book ，则返回注册码URI_CODE_b

// 步骤3：根据URI 匹配 URI_CODE，从而匹配ContentProvider中相应的资源（match（））

    @Override   
    public String getMatchTableName(Uri uri) {   
      Uri uri = Uri.parse(" content://com.prsuit.myprovider/user");   

      switch(matcher.match(uri)){   
     // 根据URI匹配的返回码是URI_CODE_a
     // 即matcher.match(uri) == URI_CODE_a
      case URI_CODE_a:   
        return tableNameUser;   
        // 如果根据URI匹配的返回码是URI_CODE_a，则返回ContentProvider中的名为tableNameUser的表
      case URI_CODE_b:   
        return tableNameBook;
        // 如果根据URI匹配的返回码是URI_CODE_b，则返回ContentProvider中的名为tableNameBook的表
    }   
}
```

##### ContentUris类

用来操作 `URI`，核心方法有两个：**withAppendedId()** 和 **parseId()** 。

```java
// withAppendedId（）作用：向URI追加一个id
Uri uri = Uri.parse("content://com.prsuit.myprovider/user") 
Uri resultUri = ContentUris.withAppendedId(uri, 7);  
// 最终生成后的Uri为：content://com.prsuit.myprovider/user/7

// parseId（）作用：从URL中获取ID
Uri uri = Uri.parse("content://com.prsuit.myprovider/user/7") 
long personid = ContentUris.parseId(uri); 
//获取的结果为:7
```

##### ContentProvider类

组织数据方式，ContentProvider 主要以 **表格的形式** 组织数据。进程间共享数据的本质是：添加、删除、获取和修改（更新）数据，所以其核心方法也主要是上述4个作用。

```java
<-- 4个核心方法 -->
  public Uri insert(Uri uri, ContentValues values) 
  // 外部进程向 ContentProvider 中添加数据

  public int delete(Uri uri, String selection, String[] selectionArgs) 
  // 外部进程 删除 ContentProvider 中的数据

  public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)
  // 外部进程更新 ContentProvider 中的数据

  public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs,  String sortOrder)　 
  // 外部应用 获取 ContentProvider 中的数据

// 注：
  // 1. 上述4个方法由外部进程回调，并运行在ContentProvider进程的Binder线程池中（不是主线程）
 // 2. 存在多线程并发访问，需要实现线程同步
   // a. 若ContentProvider的数据存储方式是使用SQLite & 一个，则不需要，因为SQLite内部实现好了线程同步，若是多个SQLite则需要，因为SQL对象之间无法进行线程同步
  // b. 若ContentProvider的数据存储方式是内存，则需要自己实现线程同步

<-- 2个其他方法 -->
public boolean onCreate() 
// ContentProvider创建后 或 打开系统后其它进程第一次访问该ContentProvider时 由系统进行调用
// 注：运行在ContentProvider进程的主线程，故不能做耗时操作

public String getType(Uri uri)
// 得到数据类型，即返回当前 Url 所代表数据的MIME类型
```

##### ContentResolver类

统一管理不同 ContentProvider 间的操作，ContentProvider 类并不会直接与外部进程交互，而是通过ContentResolver 类。它提供了与 ContentProvider 类相同名字和作用的4个方法。

```java
// 外部进程向 ContentProvider 中添加数据
public Uri insert(Uri uri, ContentValues values)　 
// 外部进程 删除 ContentProvider 中的数据
public int delete(Uri uri, String selection, String[] selectionArgs)
// 外部进程更新 ContentProvider 中的数据
public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)　 
// 外部应用 获取 ContentProvider 中的数据
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)
```

##### ContentObserver类

内容观察者，观察 Uri 引起 ContentProvider 中的数据变化并通知数据访问者。当 ContentProvider 中的数据发生变化（增、删 、改）时，就会触发该 ContentObserver 类，可以通过 ContentResolver 的registerContentObserver 和 unregisterContentObserver 方法来注册和注销 ContentObserver 监听器。当被监听的ContentProvider发生变化时，就会回调对应的 ContentObserver 的 onChange 方法。

```java
// 步骤1：注册内容观察者ContentObserver
    getContentResolver().registerContentObserver（uri）；
    // 通过ContentResolver类进行注册，并指定需要观察的URI

// 步骤2：当该URI的ContentProvider数据发生变化时，通知外界（即访问该ContentProvider数据的访问者）
    public class UserContentProvider extends ContentProvider { 
      public Uri insert(Uri uri, ContentValues values) { 
      db.insert("user", "userid", values);
      // 通知访问者
      getContext().getContentResolver().notifyChange(uri, null); 
   } 
}

// 步骤3：解除观察者
 getContentResolver().unregisterContentObserver（uri）；
    // 同样需要通过ContentResolver类进行解除

```

#### 三、具体使用

##### 创建数据库类

创建类 DBHelper 继承 SQLiteOpenHelper 并实现构造方法以及重载 onCreate 和 onUpgrade 方法。

```java
public class DBHelper extends SQLiteOpenHelper {
    private static final String DATABASE_NAME = "com_sample_provider.db";
    public static final String USER_TABLE_NAME = "user";
    public static final String BOOK_TABLE_NAME = "book";
    private static final int DATABASE_VERSION = 1;

    private static final String CREATE_USER_TABLE = "CREATE TABLE IF NOT EXISTS "
            + USER_TABLE_NAME
            + "(id INTEGER PRIMARY KEY AUTOINCREMENT,name VARCHAR(64))";
    private static final String CREATE_BOOK_TABLE = "CREATE TABLE IF NOT EXISTS "
            + BOOK_TABLE_NAME
            + "(id INTEGER PRIMARY KEY AUTOINCREMENT,name VARCHAR(64))";

    public DBHelper(Context context){
        super(context,DATABASE_NAME,null,DATABASE_VERSION);
    }

    public DBHelper(@Nullable Context context, @Nullable String name, @Nullable SQLiteDatabase.CursorFactory factory, int version) {
        super(context, name, factory, version);
    }

    public DBHelper(@Nullable Context context, @Nullable String name, @Nullable SQLiteDatabase.CursorFactory factory, int version, @Nullable DatabaseErrorHandler errorHandler) {
        super(context, name, factory, version, errorHandler);
    }

    @RequiresApi(api = Build.VERSION_CODES.P)
    public DBHelper(@Nullable Context context, @Nullable String name, int version, @NonNull SQLiteDatabase.OpenParams openParams) {
        super(context, name, version, openParams);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        //创建数据表格:用户表和图书表
        db.execSQL(CREATE_USER_TABLE);
        db.execSQL(CREATE_BOOK_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }

    //清空user表数据
    public static void clearTable(Context context){
        SQLiteDatabase database = new DBHelper(context).getWritableDatabase();
        database.execSQL("delete from user");
    }
}
```

##### 自定义 ContentProvider 类，**实现相关的抽象方法**

```java
public class MyContentProvider extends ContentProvider {
    private static final String TAG = "MyContentProvider";
    private Context mContext;
    private DBHelper dbHelper = null;
    private SQLiteDatabase mDatabase = null;
    private String matchTableName = null;

    //ContentProvider的唯一标识
    private static final String AUTHORITY = "com.prsuit.myprovider";
    private static final String USER_PATH = "user";
    private static final String BOOK_PATH = "book";
    private static final int USER_CODE = 1;
    private static final int BOOK_CODE = 2;
    private static UriMatcher matcher;

    //在ContentProvider 中注册URI
    static {
        matcher = new UriMatcher(UriMatcher.NO_MATCH);
        // 若URI资源路径 = content://com.prsuit.myprovider/user ，则返回注册码USER_CODE
        // 若URI资源路径 = content://com.prsuit.myprovider/book ，则返回注册码BOOK_CODE
        matcher.addURI(AUTHORITY,USER_PATH,USER_CODE);
        matcher.addURI(AUTHORITY,BOOK_PATH,BOOK_CODE);
    }

    @Override
    public boolean onCreate() {
        mContext = getContext();
        dbHelper = new DBHelper(getContext());
        mDatabase = dbHelper.getWritableDatabase();
        return true;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        matchTableName = getMatchTableName(uri);
        long row = -1;//返回值是插入数据所在的行号
        row = mDatabase.insert(matchTableName,null,values);
        if (row > -1){
            mContext.getContentResolver().notifyChange(uri,null);
            return ContentUris.withAppendedId(uri,row);
        }
        return null;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        matchTableName = getMatchTableName(uri);
        Cursor cursor = mDatabase.query(matchTableName,projection,selection,selectionArgs,null,null,sortOrder);
        return cursor;
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
        matchTableName = getMatchTableName(uri);
        //返回值代表此次操作影响到的行数
        int deleteRow = mDatabase.delete(matchTableName, selection, selectionArgs);
        if (deleteRow > 0){
            mContext.getContentResolver().notifyChange(uri,null);
        }
        return deleteRow;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
        matchTableName = getMatchTableName(uri);
        //返回值代表此次操作影响到的行数
        int updateRow = mDatabase.update(matchTableName, values, selection, selectionArgs);
        if (updateRow > 0){
            mContext.getContentResolver().notifyChange(uri,null);
        }
        return updateRow;
    }

    /**
     * 根据URI匹配 URI_CODE，从而匹配ContentProvider中相应的表名
     * @param uri
     * @return
     */
    public String getMatchTableName(Uri uri){
        String tableName = null;
        int uriCode = matcher.match(uri);
        switch (uriCode){
            case USER_CODE:
                tableName = DBHelper.USER_TABLE_NAME;
                break;
            case BOOK_CODE:
                tableName = DBHelper.BOOK_TABLE_NAME;
                break;
        }
        return tableName;
    }
}
```

##### 在 AndroidManifest 中声明 provider 以及定义相关访问权限

在注册ContentProvider的时候通过 android:process 属性设置 provider 运行在单独的进程里，模拟进程间通信。

```java
<!-- MyProvider 访问权限声明 -->
  // 细分读 & 写权限如下，也可直接采用全权限
<permission android:name="com.prsuit.myprovider.READ" 
  android:protectionLevel="normal" />
<permission android:name="com.prsuit.myprovider.WRITE"
        android:protectionLevel="normal" />
 <!-- <permission android:name="com.prsuit.myprovider.PROVIDER"
        android:protectionLevel="normal" /> -->
          
 <!-- 声明ContentProvider -->
 <application>
    <provider
            android:name=".contentprovider.MyContentProvider"
            android:authorities="com.prsuit.myprovider"
            android:process=":provider"
            // 声明外界进程可访问该Provider的全权限（读 & 写）
            //android:permission="com.prsuit.myprovider.PROVIDER"
            // 权限可细分为读 & 写的权限
            android:readPermission="com.prsuit.myprovider.READ"
            android:writePermission="com.prsuit.myprovider.WRITE"
            android:exported="true"
            />
 </application
```

在其他应用要访问 MyContentProvider，需要在AndroidManifest中声明相应权限才可进行相应操作，否则会报错。

```java
<!-- 其他应用声明ContentProvider所需权限 --> 
<uses-permission android:name="com.prsuit.myprovider.READ" />
<uses-permission android:name="com.prsuit.myprovider.WRITE" />
  //采用全权限
<!--<uses-permission android:name="com.prsuit.myprovider.PROVIDER" /> -->
```

##### 通过ContentResolver根据URI进行增删改查

```java
public class ContentProviderActivity extends AppCompatActivity {
    private static final String AUTHORITY = "com.prsuit.myprovider";
    private static final Uri USER_URI = Uri.parse("content://" + AUTHORITY + "/user");

    public static void startAct(Context context) {
        context.startActivity(new Intent(context, ContentProviderActivity.class));
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_content_provider);
    }

    public void insertValue(View view){
        ContentValues contentValues1 = new ContentValues();
        contentValues1.put("id",0);
        contentValues1.put("name","kobe");
        Uri insert1 = getContentResolver().insert(USER_URI, contentValues1);
        System.out.println("--insertValue1-->"+insert1.toString());

        ContentValues contentValues2 = new ContentValues();
        contentValues2.put("id",1);
        contentValues2.put("name","sh");
        Uri insert2 = getContentResolver().insert(USER_URI,contentValues2);
        System.out.println("--insertValue2-->"+insert2.toString());
    }

    public void updateValue(View view){
        ContentValues contentValues = new ContentValues();
        contentValues.put("id",1);
        contentValues.put("name","sh2");
        int row = getContentResolver().update(USER_URI,contentValues,"id = ?", new String[]{"1"});
        System.out.println("--updateValue-->"+row);
    }

    public void deleteValue(View view){
        int row = getContentResolver().delete(USER_URI,"name = ?",new String[]{"sh2"});
        System.out.println("--deleteValue-->"+row);
    }

    public void queryValue(View view){
        Cursor cursor = getContentResolver().query(USER_URI, new String[]{"id", "name"}, null, null, null);
        while (cursor.moveToNext()){
            System.out.println("query user："+cursor.getInt(0)+" "+cursor.getString(cursor.getColumnIndex("name")));
        }
        cursor.close();
    }
}
```

### 总结

本文基于 ContentProvider 的使用过程涉及到的相关知识点进行简单地介绍和整理，归纳总结了ContentProvider 使用步骤。

[Demo](https://github.com/prsuit/Android-Learn-Sample)

引用文章：

> [Android四大组件——ContentProvider（基础篇）](https://www.jianshu.com/p/94b8582d089a)
>
> [Android：关于ContentProvider的知识都在这里了！](https://www.jianshu.com/p/ea8bc4aaf057)

