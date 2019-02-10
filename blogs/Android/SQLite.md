---
Android 数据持久化之 SQLite
---

#### 目录

1. 思维导图
2. SQLite
   - 基本使用
     - SQLiteOpenHelper
     - SQLiteDatabase
   - ORM
     - greenDAO
     - Room
   - 进程与线程并发
   - 常见问题
     - 数据库升级之更改表名、增删改字段
     - SQLiteDatabaseLockedException
   - 优化建议
     - 事务
     - 建立索引
     - 页大小和缓存大小
3. 参考

#### 思维导图

![](https://i.loli.net/2019/01/15/5c3dbf22234b0.png)

#### 基本使用

##### SQLiteOpenHelper

```java
public class MySQLiteHelper extends SQLiteOpenHelper {

    public MySQLiteHelper(Context context, String name, SQLiteDatabase.CursorFactory factory, int version) {
        super(context, name, factory, version);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        String sql = "create table userinfo(id integer primary key autoincrement,username varchar(64),pwd varchar(64))";
        db.execSQL(sql);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }
}
```

##### SQLiteDatabase

```java
        MySQLiteHelper sqLiteHelper = new MySQLiteHelper(this, "userinfo", null, 1);
        SQLiteDatabase sqLiteDatabase = sqLiteHelper.getWritableDatabase();
        sqLiteDatabase.beginTransaction();
        ContentValues values = new ContentValues();
        values.put("username", "Omooo");
        values.put("pwd", "Test");
        sqLiteDatabase.insert("userinfo", null, values);
        sqLiteDatabase.endTransaction();
        sqLiteDatabase.close();
```

#### ORM

大部分为了提高开发效率，都会引入 ORM 框架，ORM 即对象关系映射，用面向对象的概念把数据库中表和对象关联起来。

Android 中最常用的 ORM 框架有 greenDAO 和 Room，使用 ORM 框架很简单，但是易用性是需要牺牲部分执行效率为代价的。

#### 进程与线程并发

SQLiteDatabaseLockedException 是会经常遇到的一个问题，归根结底是因为并发导致，而 SQLite 的并发有两个维度，一个是多进程并发，一个是多线程并发。

##### 多进程并发

SQLite 默认是支持多进程并发操作的，它通过文件锁来控制多进程的并发。多进程可以同时获取 SHARED 锁来读取数据，但是只有一个进程可以获取 EXCLUSIVE 锁来写数据库。

在 EXCLUSIVE 模式下，数据库连接在断开前都不会释放 SQLite 文件的锁，从而避免不必要的冲突，提高数据库访问的速度。

##### 多线程并发

相比多进程，多线程的数据库访问可能会更加常见。SQLite 支持多线程并发模式，系统 SQLite 会默认开启多线程 Multi-thread 模式。

跟多进程的锁机制一样，为了实现简单，SQLite 锁的粒度都是数据库文件级别，并没有实现表级甚至行级的锁。还有需要说明的是，同一个句柄同一时间只有一个线程在操作，这个时候我们需要打开连接池 Connection Pool。

跟多进程类似，多线程可以同时读取数据库数据，但是写数据依然是互斥的。SQLite 提供了 Busy Retry 的方案，即发生阻塞时会触发 Busy Handler，此时可以让线程休眠一段时间后，重新尝试操作。

为了进一步提高并发性能，我们还可以打开 WAL（Write-Ahead Logging）模式。WAL 模式会将修改的数据单独写到一个 WAL 文件中，同时也会引入 WAL 日志文件锁。通过 WAL 模式读和写可以完全的并发执行，不会相互阻塞。

```java
db.enableWriteAheadLogging();	//返回 false / true
```

如果开启了 WAL 模式，开启事务要使用 benginTransactionNonExclusive()，注意捕获异常，源码如下：

```java
    /**
     * <pre>
     *   db.beginTransactionNonExclusive();
     *   try {
     *     ...
     *     db.setTransactionSuccessful();
     *   } finally {
     *     db.endTransaction();
     *   }
     * </pre>
     */
    public void beginTransactionNonExclusive() {
        beginTransaction(null /* transactionStatusCallback */, false);
    }
```

但是需要注意的是，**写之间仍然不能并发。**如果出现多个写并发的情况，依然有可能出现 SQLiteDatabaseLockedException，这个时候可以让应用捕获这个异常，然后等待一段时间后重试。

总的来说，通过连接池与 WAL 模式，可以很大程度上增加 SQLite 的读写并发，大大减少由于并发导致的等待耗时，建议在应用中尝试开启。

#### 常见问题

##### 数据库升级之更改表名、增删改字段

SQLite 仅仅支持 ALTER TABLE 语句的一部分功能，可以用 ALTER TABLE 语句来更改一个表的名字，也可以向表中增加一个字段，但是我们不能删除一个已经存在的字段，或更改一个已经存在的字段的名称、数据类型、限定符等。

修改表名以及增加字段都是相同的操作：

SQLiteOpenHelper#onUpgrade：

```java
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        switch (newVersion) {
            case VERSION_2:
                db.execSQL("alter table userinfo rename to user");
                db.execSQL("alter table userinfo add column sex varchar");
                break;
            default:
                break;
        }
    }
```

修改字段：

修改字段的思路是先由原表创建一个临时表，然后创建一个新表把需要的信息从临时表取出即可。

```java
//1. 创建临时表
ALTER TABLE "Student" RENAME TO "_Student_old_20140409";
//2. 创建新表，把需要修改的字段或者删除的字段更新上去
CREATE TABLE "Student" (
"Id"  INTEGER PRIMARY KEY AUTOINCREMENT,
"Name"  Text);
//3. 从临时表拿所需数据
INSERT INTO "Student" ("Id", "Name") SELECT "Id", "Title" FROM "_Student_old_20140409";
//4. 删除临时表（可选）
```

#### 优化建议

##### 建立索引

简单来说，索引就像书本的目录，目录可以快速找到所在页数，数据库中索引可以帮助快速找到数据，而不用全表扫描，合适的索引可以大大提高数据库查询的效率。

优点：

大大加快了数据库检索的速度，经常是一到两个数量级的性能提升。

缺点：

索引的创建和维护存在消耗，索引会占用物理空间，在对数据库进行增删改查时需要维护索引，所以会对增删改的性能存在影响。

分类：

1. 直接创建索引和间接创建索引

   直接创建：使用 SQL 语句创建，Android 中可以在 SQLiteOpenHelper 的 onCreate 或者 onUpgrade 中直接 excuSql 创建语句，语句例如：

   Create index column_index on mytable

   间接创建：

   定义主键约束，可以间接创建索引，主键默认为唯一索引。

2. 单个索引和复合索引

3. 聚簇索引和非聚簇索引

使用场景：

1. 当字段数据更新频率较低，查询效率较高，经常有范围查询或 order by、group by 发生时建议使用索引，并且选择度越大，建索引越有优势。
2. 经常同时存取多列，且每列都含有重复值可考虑建立复合索引

##### 使用事务

使用事务的两大好处是原子提交和更优性能。

原子提交：

原子提交意味着同一事务内的所有修改要么都完成要么都不做，如果某个修改失败，会自动回滚使得所有修改不生效。

更优性能：

SQLite 默认会为每个插入、更新操作创建一个事务，并且在每次插入、更新后立即提交。

这样，如果连续插入一百次数据实际是创建事务 -> 执行语句 -> 提交 这个过程被重复执行了一百次。如果我们显示的创建事务 -> 执行一百条数据 -> 提交 会使得这个创建事务和提交过程只做了一次，通过这种一次性事务可以使得性能大幅提升。尤其当数据库位于 SD 卡时，时间上能节省两个数量级左右。

##### 页大小与缓存大小

数据库就像一个小文件系统，事实上它内部也有页和缓存的概念。

对于 SQLite 的 DB 文件来说，页是最小的存储单位。跟文件系统的页缓存一样，SQLite 会将读过的页缓存起来，用来加快下一次读取速度，页大小默认是 1024 Byte，缓存大小默认是 1000 页。如果使用 4KB 的 pageSize 性能能提升 5%～10% 左右，所以在建数据库的时候，就提前选择 4KB 作为默认的 page size 以获得更好的性能。

```java
db.setPageSize(1024 * 4);
```



##### 其他优化

1. 语句拼接使用 StringBuilder 替代 String

2. 读写表

   getReadableDatabase、getWritableDatabase

3. 只查询所需的结果集

4. 少用 cursor.getColumnIndex

5. 异步线程

##### 总结

通过引入 ORM，可以大大的提升开发效率；通过 WAL 模式和连接池，可以提高 SQLite 的并发性能；通过正确的建立索引，可以提升 SQLite 的查询速度；通过调整默认的页大小和缓存大小，可以提升 SQLite 的整体性能。

#### 参考

[Android中数据库Sqlite的性能优化](https://www.cnblogs.com/daishuguang/p/4015478.html)

[http://huili.github.io/sqlite/sqliteintro.html]()