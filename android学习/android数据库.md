# sqlite
## sqlite数据库操作流程
sqlite一般情况下的处理逻辑为，创建SQLiteOpenHelper的子类，在此子类中定义数据的文件名，当前版本号，数据库版本升级处理，需要进行数据库处理的地方获取SQLiteOpenHelper子类对象，获取readable或writeable的SqliteDatabase对象，使用SqliteDatabase的方法执行一系列的增删改查操作。
## 创建SQLiteOpenHelper的子类
一个Helper实例会产生一个database连接也就是通过获取，即SqliteDatabase对象，一个SqliteDatabase对应着一个数据库文件。

```javascript
public class DBHelper extends SQLiteOpenHelper {
	 //数据库的文件名
    private static final String DB_NAME = "userdb";
    //数据库最新版本号
    private static final int DB_VERSION = 1;
    public DBHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }
    @Override
    public void onCreate(SQLiteDatabase db) {
    //创建一个新的数据库表，一般情况下在此处创建表
        StringBuffer sql = new StringBuffer();
        sql.append("create table users");
        sql.append("(_id int PRIMARY KEY,name varchar,gender int,age int,phoneNumber varchar,address varchar)");
        db.execSQL(sql.toString());
    }
    //数据升级操作,在这里执行升级操作，比如新添表，对表结构进行修改，修改数据库数据等
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
    }
}
```
## 调用SQLiteOpenHelper执行操作

```javascript
DBHelper dbHelper = new DBHelper(context);
SQLiteDatabase db = dbHelper.getWritableDatabase();
db执行各种数据库操作
```
也就是说需要通过SQLiteOpenHelper获取SQLiteDatabase对象db，使用db进行数据库的各种操作。
## 何时调用SQLiteOpenHelper的onCreate和onUpgrade方法
SQLiteOpenHelper的getWritableDatabase和getReadableDatabase都是调用getDatabaseLocked获取到SQLiteDatabase对象的，此方法的简化代码如下:<br>

```javascript
 if (version == 0) {
     onCreate(db);
 } else {
        if (version > mNewVersion) {
             onDowngrade(db, version, mNewVersion);
          } else {
             onUpgrade(db, version, mNewVersion);
          }
        }
        db.setVersion(mNewVersion);
        db.setTransactionSuccessful();
        } finally {
          db.endTransaction();
    }
 }               
```
通过上面的代码可知onCreate和onUpgrade方法只执行一个，而且onCreate方法在整个系统中只执行一次。而且在onCraete和onUpgrade方法中数据库已经处于一个事务中，因此不必再写事务了。

## 事务
android的sqlite同样支持事务。具体使用方法如下:<br>

```javascript
//开始事务
db.beginTransaction();
执行操作
//标识事务执行成功，如果setTransactionSuccessful不被调用则会回滚，撤销所做的操作
db.setTransactionSuccessful();
//结束事务
db.endTransaction();
```
## 线程安全
sqlite本身不支持线程安全。如果想要实现线程安全，必须保证对sqlite的操作是线程安全的。<br>
正如上文所述，一个SqliteOpenHepler对应一个数据库连接，只要保证SqliteOpenHepler是整个系统中是单例的则可保证数据数据库的操作是单例的。<br>
通常情况下使用sqlite会遇到如下的几个问题:<br>

* android.database.sqlite.SQLiteDatabaseLockedException: database is locked (code 5)
* java.lang.IllegalStateException: attempt to re-open an already-closed object: SQLiteDatabase
*  java.lang.IllegalStateException: Cannot perform this operation because the connection pool has been closed.

第一个问题出现在线程1正在写数据，而线程2也试图想数据库写数据，由于线程1已经将数据库锁死，则线程2无法执行操作。第二个和第三个问题是由于线程1已经将数据库关闭，但是线程2在执行的时候仍然使用原来的SqliteOpenHeler返回的SqliteDatabase对象。<br>
只有保证同一时间只能执行一个数据库操作，而且在关闭数据时需要考虑是否有其他线程也要使用数据库，如果有则不关闭数据库。具体代码如下:

```javascript
public class DatabaseManager {
    //返回当前的要执行的操作个数
    private AtomicInteger mOpenCounter = new AtomicInteger();
    private static DatabaseManager instance;  
    private static SQLiteOpenHelper mDatabaseHelper;  
    private SQLiteDatabase mDatabase; 
   
    //保证是单例的SQLiteOpenHelper
    public static synchronized void initializeInstance(SQLiteOpenHelper helper) {  
        if (instance == null) {  
            instance = new DatabaseManager();  
            mDatabaseHelper = helper;  
        }  
    }  
    
    public static synchronized DatabaseManager getInstance(SQLiteOpenHelper helper) {  
        if (instance == null) {  
            initializeInstance(helper);
        }  
        return instance;  
    }  
    //获取SQLiteDatabase对象
    public synchronized SQLiteDatabase getWritableDatabase() { 
    		//如果是第一个获取数据库对象，则返回一个新的Database对象 
        if(mOpenCounter.incrementAndGet() == 1) {  
            // Opening new database  
            mDatabase = mDatabaseHelper.getWritableDatabase();  
        }  
        return mDatabase;  
    }  
    
    public synchronized SQLiteDatabase getReadableDatabase() {  
        if(mOpenCounter.incrementAndGet() == 1) {  
            // Opening new database  
            mDatabase = mDatabaseHelper.getReadableDatabase();  
        }  
        return mDatabase;  
    }  
    
    public synchronized void closeDatabase() {  
        //如果没有其他的操作需要数据库则关闭
        if(mOpenCounter.decrementAndGet() == 0) {  
            // Closing database  
            mDatabase.close();  
        }  
    }
 }
```
使用方法为:mDatabaseManager = DatabaseManager.getInstance(helper);可以考虑在Application中执行该方法,加入使用上文的DBHelper，简化代码如下:

```javascript
public class Application extends android.app.Application {
	 private DatabaseManager databaseManager;
	 @Override
	 public void onCreate() {
	   super.onCreate();
		....其他代码
		//初始化DatabaseManager
		DBHelper dbHelper = new DbHelper(this);
		DatabaseManager databaseManager = DatabaseManager.getInstance(dbHelper);
	}
	
	public DatabaseManager getDatabaseManager(){
		return databaseManager;
	}
}
```
这样就可以通过Application获取DatabaseManager对象了，而且是唯一的对象。保证线程安全。

## 优化
* 插入数据使用SQLiteStatement，进行sql语句的变异，具体可以参看[Android应用性能优化之使用SQLiteStatement优化SQLite操作](https://liuzhichao.com/p/1664.html)
* 使用事务，减少数据库打开,关闭操作
* 数据库建立索引
* 及时关闭cursor
* 耗时操作异步化

# realm
realm是一个比sqlite更容易使用的数据库，而不是sqlite的ORM版本或sqlite的封装，关于其使用方法可以参考[java教程](https://realm.io/docs/java/latest/)。
## 使用方法
在项目的build.gradle文件中增加如下代码:<br>

```javascript
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "io.realm:realm-gradle-plugin:2.0.2"
    }
}
```
在使用realm的module的build.gradle文件中添加如下代码:<br>

```javascript
apply plugin: 'realm-android'
```

## 模型
所有的Realm对象都必须继承自RealmObject。

# 参考
[Android应用性能优化之使用SQLiteStatement优化SQLite操作](https://liuzhichao.com/p/1664.html)<br>
[SQLite在多线程并发访问的应用](http://blog.csdn.net/lang791534167/article/details/38984887)<br>
[Android开发——多线程环境下SQLite数据库并发访问的解决方案](http://www.mobile-open.com/2015/38534.html)<br>
[android Sqlite多线程访问异常解决方案](http://www.cnblogs.com/wangmars/p/4530670.html)<br>
[Android 中 SQLite 性能优化](http://droidyue.com/blog/2015/12/13/android-sqlite-tuning/)<br>
[android数据库优化](http://www.jianshu.com/p/3b4452fc1bbd)<br>
[Android 编程下 SQLite 大数据量操作优化](http://www.cnblogs.com/sunzn/archive/2013/01/27/2878377.html)<br><br>

[realm官方网址](https://realm.io)



# 示例代码

[示例代码](https://github.com/wsy1983wsy/AndroidDatabase.git)
