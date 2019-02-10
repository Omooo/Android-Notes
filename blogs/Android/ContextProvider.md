---
ContentProvider
---

#### 目录

1. 思维导图
2. 概述
3. 统一资源标识符（URI）
4. MIME 数据类型
5. ContentProvider
   - 组织数据方式
   - 主要方法
6. 辅助工具类
   - ContentResolver
   - ContentUris
   - UriMatcher
   - ContentObserver
7. 实例
   - 获取手机通讯录信息
   - 结合 SQLite
8. 参考

#### 思维导图

![](https://i.loli.net/2019/02/05/5c597c4f25157.png)

#### 概述

ContextProvider 为存储和获取数据提供了统一的接口，可以在不同的应用程序之间安全的共享数据。它允许把自己的应用数据根据需求开放给其他应用进行增删改查。数据的存储方式还是之前的方式，它只是提供了一个统一的接口去访问数据。

#### 统一资源标识符

统一资源标识符即 URI，用来唯一标识 ContentProvider 其中的数据，外界进程通过 URI 找到对应的 ContentProvider 其中的数据，在进行数据操作。

URI 分为系统预置和自定义，分别对应系统内置的数据（如通讯录等）和自定义数据库。

##### 系统内置 URI

比如获取通讯录信息所需要的 URI：ContactsContract.CommonDataKinds.Phone.CONTENT_URI。

##### 自定义 URI

```java
格式:content://authority/path/id
authority:授权信息，用以区分不同的 ContentProvider
path:表名，用以区分 ContentProvider 中不同的数据表
id: ID号，用以区别表中的不同数据
示例:content://com.example.omooo.demoproject/User/1
上述 URI 指向的资源是：名为 com.example.omooo.demoproject 的 ContentProvider 中表名为 User 中 id 为 1 的数据。
```

注意，URI 也存在匹配通配符：* & #

#### MIME 数据类型

它是用来指定某个扩展名的文件用某种应用程序来打开。

可以通过 ContentProvider.getType(uri) 来获得。

每种 MIME 类型由两部分组成：类型 + 子类型。

示例：text/html、application/pdf

#### ContentProvider

##### 组织数据方式

ContentProvider 主要以表格的形式组织数据，同时也支持文件数据，只是表格形式用的比较多，每个表格中包含多张表，每张表包含行和列，分别对应数据。

##### 主要方法

```java
public class MyProvider extends ContentProvider {

    @Override
    public boolean onCreate() {
        return false;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        return null;
    }

    @Override
    public String getType(Uri uri) {
        return null;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        return null;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        return 0;
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        return 0;
    }
}
```

#### 辅助工具类

##### ContentResolver 

统一管理不同的 ContentProvider 间的操作。

1. 即通过 URI 即可操作不同的 ContentProvider 中的数据
2. 外部进程通过 ContentResolver 类从而与 ContentProvider 类进行交互

一般来说，一款应用要使用多个 ContentProvider，若需要了解每个 ContentProvider 的不同实现从而在完成数据交互，操作成本高且难度大，所以在 ContentProvider 类上多加一个 ContentResolver 类对所有的 ContentProvider 进行统一管理。

ContentResolver 类提供了与 ContentProvider 类相同名字和作用的四个方法：

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

```java
// 使用ContentResolver前，需要先获取ContentResolver
// 可通过在所有继承Context的类中 通过调用getContentResolver()来获得ContentResolver
ContentResolver resolver =  getContentResolver(); 

// 设置ContentProvider的URI
Uri uri = Uri.parse("content://cn.scu.myprovider/user"); 
 
// 根据URI 操作 ContentProvider中的数据
// 此处是获取ContentProvider中 user表的所有记录 
Cursor cursor = resolver.query(uri, null, null, null, "userid desc"); 
```

##### ContentUris

用来操作 URI 的，常用有两个方法：

```java
// withAppendedId（）作用：向URI追加一个id
Uri uri = Uri.parse("content://cn.scu.myprovider/user") 
Uri resultUri = ContentUris.withAppendedId(uri, 7);  
// 最终生成后的Uri为：content://cn.scu.myprovider/user/7

// parseId（）作用：从URL中获取ID
Uri uri = Uri.parse("content://cn.scu.myprovider/user/7") 
long personid = ContentUris.parseId(uri); 
//获取的结果为:7
```

##### UriMatcher

在 ContentProvider 中注册 URI，根据 URI 匹配 ContentProvider 中对应的数据表。

```java
	// 步骤1：初始化UriMatcher对象
    UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH); 
    //常量UriMatcher.NO_MATCH  = 不匹配任何路径的返回码
    // 即初始化时不匹配任何东西

	// 步骤2：在ContentProvider 中注册URI（addURI（））
    int URI_CODE_a = 1；
    int URI_CODE_b = 2；
    matcher.addURI("cn.scu.myprovider", "user1", URI_CODE_a); 
    matcher.addURI("cn.scu.myprovider", "user2", URI_CODE_b); 
    // 若URI资源路径 = content://cn.scu.myprovider/user1 ，则返回注册码URI_CODE_a
    // 若URI资源路径 = content://cn.scu.myprovider/user2 ，则返回注册码URI_CODE_b

	// 步骤3：根据URI 匹配 URI_CODE，从而匹配ContentProvider中相应的资源（match（））

@Override   
    public String getType(Uri uri) {   
      Uri uri = Uri.parse(" content://cn.scu.myprovider/user1");   

      switch(matcher.match(uri)){   
     // 根据URI匹配的返回码是URI_CODE_a
     // 即matcher.match(uri) == URI_CODE_a
      case URI_CODE_a:   
        return tableNameUser1;   
        // 如果根据URI匹配的返回码是URI_CODE_a，则返回ContentProvider中的名为tableNameUser1的表
      case URI_CODE_b:   
        return tableNameUser2;
        // 如果根据URI匹配的返回码是URI_CODE_b，则返回ContentProvider中的名为tableNameUser2的表
    }   
}
```

##### ContentObserver

内容观察者，当 ContentProvider 中的数据发生变化时，就会触发 ContentObserver 类。

```java
	// 步骤1：注册内容观察者ContentObserver
    getContentResolver().registerContentObserver（uri）；
    // 通过ContentResolver类进行注册，并指定需要观察的URI

	// 步骤2：当该URI的ContentProvider数据发生变化时，通知外界（即访问该ContentProvider数据的访问者）
    public class UserContentProvider extends ContentProvider { 
      public Uri insert(Uri uri, ContentValues values) { 
      db.insert("user", "userid", values); 
      getContext().getContentResolver().notifyChange(uri, null); 
      // 通知访问者
   } 
}

 // 步骤3：解除观察者
 getContentResolver().unregisterContentObserver（uri）；
 // 同样需要通过ContentResolver类进行解除
```

#### 实例

##### 获取通讯录信息

这里就不需要自己写 ContentProvider 的实现了，用系统已经给的 URI。

```java
    /**
     * 获取通讯录信息
     */
    private void getContactsInfo() {
        Cursor cursor = getContentResolver().query(
                ContactsContract.CommonDataKinds.Phone.CONTENT_URI, null, null, null, null
        );
        if (cursor != null) {
            while (cursor.moveToNext()) {
                //联系人姓名
                String name = cursor.getString(cursor.getColumnIndex(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME));
                //联系人手机号
                String phoneNumber = cursor.getString(cursor.getColumnIndex(ContactsContract.CommonDataKinds.Phone.NUMBER));
                Log.i(TAG, "getContactsInfo: name: " + name + "  phone: " + phoneNumber);
            }
            cursor.close();
        }
    }
```

##### 结合 SQLite

1. 创建数据库
2. 自定义 ContentProvider 并注册
3. 进程内访问数据

创建数据库：

```java
public class MySQLiteOpenHelper extends SQLiteOpenHelper {

    public MySQLiteOpenHelper(Context context) {
        super(context, "user.info", null, 1);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.setPageSize(1024 * 4);
//        db.enableWriteAheadLogging();
        db.execSQL("CREATE TABLE if not exists user (name text, age string)");
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }
}
```

自定义 ContentProvider 并注册：

```java
public class MyProvider extends ContentProvider {

    private static UriMatcher mUriMatcher;

    static {
        mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        mUriMatcher.addURI("com.example.omooo.demoproject.provider", "user", 1);
    }

    private MySQLiteOpenHelper mMySQLiteOpenHelper;
    private SQLiteDatabase mSQLiteDatabase;
    private Context mContext;

    @Override
    public boolean onCreate() {
        mContext = getContext();
        mMySQLiteOpenHelper = new MySQLiteOpenHelper(mContext);
        mSQLiteDatabase = mMySQLiteOpenHelper.getWritableDatabase();
        mSQLiteDatabase.execSQL("insert into user values('Omooo','18');");
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        return mSQLiteDatabase.query("user", projection, selection, selectionArgs, null, null, sortOrder, null);
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        mSQLiteDatabase.insert("user", null, values);
        mContext.getContentResolver().notifyChange(uri, null);
        return uri;
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

```xml
	<provider
            android:exported="false"
            android:authorities="com.example.omooo.demoproject.provider"
            android:name=".provider.MyProvider"/>
```

进程内访问数据：

```java
    private void insertTable() {
        Uri uri = Uri.parse("content://com.example.omooo.demoproject.provider/user");
        ContentValues values = new ContentValues();
        values.put("name", "Test");
        values.put("age", "21");
        ContentResolver resolver = getContentResolver();
        resolver.insert(uri, values);
        Cursor cursor = resolver.query(uri, new String[]{"name", "age"}, null, null, null);
        if (cursor != null) {
            while (cursor.moveToNext()) {
                String name = cursor.getString(cursor.getColumnIndex("name"));
                String age = cursor.getString(cursor.getColumnIndex("age"));
                Log.i(TAG, "insertTable: name: " + name + " age: " + age);
            }
            cursor.close();
        }
    }
```



#### 参考

[Android：关于ContentProvider的知识都在这里了！](https://www.jianshu.com/p/ea8bc4aaf057)

[ContentProvider 从入门到精通](https://juejin.im/entry/5726d30e49830c0053562f1a)

[系统自带ContentProvider的常用Uri地址](https://www.cnblogs.com/tangs/articles/5756320.html)