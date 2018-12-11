---
Android 数据持久化之 SQLite
---

#### 目录

1. 思维导图
2. SQLite
   - 基本使用
     - SQLiteOpenHelper
     - SQLiteDatabase
   - 常见问题
     - 数据库升级之更改表名、增删改字段
     - 
   - 优化建议
     - 事务
     - 建立索引
3. 参考

#### 思维导图

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

##### 其他优化

1. 语句拼接使用 StringBuilder 替代 String

2. 读写表

   getReadableDatabase、getWritableDatabase

3. 只查询所需的结果集

4. 少用 cursor.getColumnIndex

5. 异步线程

#### 参考

[Android中数据库Sqlite的性能优化](https://www.cnblogs.com/daishuguang/p/4015478.html)