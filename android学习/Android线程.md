# Android线程概述
线程分为主线程和子线程，主线程主要处理和界面相关的事情，子线程则往往用于处理耗时操作。<br>
线程是操作系统最小的调度单元，同时线程又是一种受限的系统资源，即线程不可能无限制的产生，并且线程的创建和销毁都会有响应的开销。线程池会缓存一定数量的线程，通过线程池可以避免因为频繁创建和销毁线程所带来的系统开销。android中的线程池来源于Java,主要是通过Executor来派生特定类型的线程池。
# 主线程和子线程
主线程主要处理界面交互相关的逻辑，因为用户随时会和界面发生交互，因此主线程在任何时候都必须要有较高的响应速度，否则就会产生一种界面卡顿的感觉。<br>
子线程，也叫工作线程，除了主线程以外的线程都叫子线程。<br>
Android中的主线程也叫UI线程，作用是运行四大组件，以及处理它们和用户的交互。子线程执行耗时任务，比如网络请求，I/O操作等。
# 四种线程形态
## Thread
最基本的线程使用方法，从Java继承而来。使用方法可以直接从Thread生成变量，或者使用Runnable初始化Thread。不再赘述。
## AsyncTask
封装了线程池，和Handler，主要为了方便开发者在子线程中更新UI。<br>
可以在线程池中执行后台任务，然后把执行的进度和最终的结果传递给主线程中更新UI。从实现上来说，AsyncTask封装了Thread和Handler，可以很方便的执行后台任务并更新UI。但是AsyncTask不适合进行特别耗时的任务，耗时任务建议使用线程池做。<br>
### 四个核心方法
* onPreExecute，在主线程中执行，异步任务执行之前，此方法被调用，一般可以用于做一些准备工作
* doInBackground，在线程池中执行，用于执行异步任务，可以调用publishProgres更新任务的进度，publishProgress会调用onProgressUpdate方法。
* onProgressUpdate，主线程中执行，进度发生变化。
* onPostExecute，主线程中执行，异步任务执行完成之后，此方法会被调用。

### 一些限制
* AsyncTask的必须在主线程中加载。
* AsyncTask的对象必须在主线程中创建。
* execute方法必须在UI线程中调用。
* 不能直接调用onPreExecute，onPostExecute，doInBackground，onProgressUpdate。
* 一个AsyncTask对象只能执行一次，即只能调用一次execute方法。
* android 1.6之前AsyncTask串行执行，android 1.6到android 3.0之前为并行执行，android 3.0包括（3.0）串行执行。

### 工作原理
execute直接调用了executeOnExecutor，两个方法源代码如下:

```javascript
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
        //直接调用了executeOnExecutor方法。
}
 public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
            //此处保证一个AsyncTask只执行一次
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException(".....");
                case FINISHED:
                    throw new IllegalStateException("......");
            }
        }
        mStatus = Status.RUNNING;
        //调用onPreExecute，此时仍然在主线程上
        onPreExecute();
        mWorker.mParams = params;
        //在线程池上执行
        exec.execute(mFuture);
        return this;
    }
```
sDefaultExecutor是一个串行的线程池，sDefaultExecutor代码如下：

```javascript
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
private static class SerialExecutor implements Executor {
			//保存需要执行的线程的队列
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        //当前正在执行的线程
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
        		//offer，想队列的尾部插入一个数据。
        		//不能直接将参数的Runnable加入到队列中，因为在参数的run方法执行完之后执行scheduleNext方法，所以新建了一个Runnable对象，在新建的Runnable对象的run方法中直接执行了参数的run方法。
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
        		//获取头部的对象，并赋值给mActivie，如果不是null，则在线程池THREAD_POOL_EXECUTOR上执行。
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
   }
```
从上面的代码中可以看到AsyncTask是现行执行的。<br>
AsyncTask有两个线程池，分别为sDefaultExecutor和THREAD_POOL_EXECUTOR，其中THREAD_POOL_EXECUTOR用来真正的执行AsyncTask，sDefaultExecutor用来做排队，保证AsyncTask顺序执行。
### 如何保证能够更新UI
AsyncTask有一个InternalHandler，其代码如下：

```javascript
 private static class InternalHandler extends Handler {
		 //构造函数，使用的Looper时主线程Looper，保证其处理消息是在主线程上。
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
### 使用方法
1. 生成AsncTask的对象，需要重写doInBackground，和onPostExecute方法。其中doInBackground方法是子线程需要执行的操作，onPostExecute表示子线程处理之后主线程需要执行的操作。如果需要可以重写onProgressUpdate方法，在主线程显示进度的变化。
2. 执行AsyncTask的execute方法。

## HandlerThread
是一种具有了消息循环的线程，在它的内部可以使用Handler。实现非常简单，就是在run方法中通过Looper.prepare来创建消息队列，通过loop方法开启消息循环，这样在实际的使用中就可以在HandlerThread中创建Handler了。其run方法如下：<br>

```javascript
@Override
    public void run() {
        mTid = Process.myTid();
        //创建消息队列
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        //开启消息循环
        Looper.loop();
        mTid = -1;
    }
```
HandlerThread有getLooper方法获取该线程对应的Looper，将次Looper传递给Handler就可以子线程中执行一些操作，比如主线程需要子线程执行网络请求，则可以通过Handler post一个Runnable即可，此Runnable会请求网络数据，数据请求完成之后将数据保存起来，或者将数据更新到UI（需要进行线程切换，比如使用主线程的handler来更新界面）。使用方法示例代码：

```javascript
public class Activity1 extends Activity implement OnClickListener{
	//HandlerThread使用的Handler
	private Handler mHandler;
	private HandlerThread mHandlerThread;
	private Handler uiHandler = new Handler(){
		@override
		 public void handleMessage(Message msg) {
		 //处理消息
		 //更新界面
		 }
	};
	
	private Button btn;
	@Override
	protected void onCreate(Bundle savedInstanceState) {
	    super.onCreate(savedInstanceState);
	    setContentView(R.layout.activity_main);
	    btn = (Button) findViewById(R.id.btn);
	    btn.setOnClickListener(this);
	    mHandlerThread = new HandlerThread("Test");
	    mHandlerThread.start();
	    mHandler = new Handler(mHandlerThread.getLooper());
	}
	
	private Runnable mRunnable = new Runnable() {
	    @Override
	    public void run() {
	      Log.d("MainActivity", "test HandlerThread...");
	      try {
	       //从网络请求数据
	       getDataFromNetWork();
	       //使用uiHandler执行一个post请求
	       uiHandler.obtainMessage(1,"finished").sendToTarget();
	       } catch (Exception e) {
	           e.printStackTrace();
	       }
	    }
	};
	
	@Override
	public void onClick(View v) {
	    switch(v.getId()) {
	    case R.id.btn :
	        mHandler.post(mRunnable);
	        break;
	    default :
	        break;
	    }
	}
	
	@Override
	protected void onDestroy() {
	    mRunning = false;
	    //移除回调
	    mHandler.removeCallbacks(mRunnable);
	    super.onDestroy();
	}
}
```
使用步骤:

1. 声明Handler，HandlerThread对象。
2. 生命Runnable对象
3. 初始化HandlerThread对象， mHandlerThread = new HandlerThread("Test");
4. HandlerThread启动，mHandlerThread.start();
5. 初始化Handler对象，mHandler = new Handler(mHandlerThread.getLooper());
6. 执行Runnable，mHandler.post(mRunnable);
7. 在Runnable对象的run方法执行完毕后可以调用主线程的Handler，进行界面更新。

## IntentService
是一个服务（Service），系统对其进行了封装使其可以更方便地执行后台任务，IntentService内部采用了HandlerThread来执行任务，当任务执行完毕之后IntentService会自动退出。作为一个服务，IntentService不容易被系统杀死从而可以尽量保证任务的执行。<br>
IntentService是Service的子类，但它是一个抽象类，因此需要创建它的子类才能使用。其封装了HandlerThread和Handler。从其代码中可以看出其使用了HandlerThread。

```javascript
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```
### Intent的处理
看其onStartCommand方法，onStartCommand掉哦那个onStart方法，而onStart的代码为：

```javascript
    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        //直接调用了mServiceHandler发送消息，保证了消息在子线程中被调用。
        mServiceHandler.sendMessage(msg);
    }
```
ServiceHanlder对消息的处理如下:

```javascript
private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
        		//调用onHandleIntent处理消息
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
}
```
IntentService的onHandleIntent方法为一个抽象方法，需要子类重写。

```javascript
protected abstract void onHandleIntent(Intent intent);
```
由于Looper是顺序执行任务的，因此IntentService也是顺序执行任务的。
# 线程池
## 线程池的优点
* 重用线程池中的线程，避免因为线程的创建和销毁带来的性能开销。
* 能有效的控制线程池的最大并发数，避免大量的线程之间因相互抢占系统资源而导致的阻塞现象。
* 能够对线程惊醒简单的管理，并提供定时执行以及指定间隔循环执行等功能。

Android中得线程池来源于Java中的Executor，Executor是一个接口，真正的线程池的实现为ThreadPoolExecutor. ThreadPoolExecutor提供了一些列参数来配置线程池，根据不同的参数可以创建不同的线程池，从线程池的功能特性来说，Android的线程池主要有四类，通过Executors所提供的方法来得到。
## ThreadPoolExecutor
ThreadPoolExecutor是线程池的真正实现，其构造方法提供了一系列参数来配置线程池。构造方法如下所示：

```javascript
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) 
```
* corePoolSize，核心线程数，默认情况下核心线程会一直存活，即使它们处于闲置状态。如果线程池的allowCoreThreadTimeOut属性为true，则闲置的核心线程在等待新任务到来时会有超时策略，时间间隔有keepAliveTime决定。
* maximumPoolSize，线程池所能容纳的最大线程数，当活动线程数达到这个数值后，后续的新任务将会被阻塞。
* keepAliveTime，非核心线程闲置时的超时时长，超过这个时长，非核心线程就会被回收。allowCoreThreadTimeOut属性为true，keepAliveTime同样作用于核心线程。
* unit，keepAliveTime的单位，使用TimeUnit的值。
* workQueue，任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中。
* threadFactory，线程工厂，为线程池创建一个新的线程。ThreadFactory是一个接口，只有一个Thread newThread(Runnable r)方法。

除了上面的参数外还有一个参数，RejectedExecutionHandler，当线程池无法执行任务时会调用handler的rejectedExecution方法通知调用者。
### 规则
* 如果线程池的线程数量未达到核心线程的数量，就会直接启动一个核心线程。
* 如果线程池中的线程数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
* 如果无法将任务插入到任务队列中，这往往是由于任务队列已经满了，这个时候如果线程数量未达到线程池规定的最大值，那么会立刻启动一个非核心线程执行任务。
* 如果上面的线程数量已经达到线程池规定的最大值，那么就拒绝执行此任务会调用RejectedExecutionHandler 的rejectedExecution方法。

### AsyncTask的线程池

```javascript
  private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
  private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
  private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
  private static final int KEEP_ALIVE = 1;
  
  private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

从上面的代码可以知道AsyncTask对THREAD_POOL_EXECUTOR这个线程池进行了配置，最后的规则为：

* 核心线程数为CPU核心数+1
* 线程池的最大线程数为CPU核心数的2倍 + 1
* 核心线程无超市机制，非核心线程在闲置时的超时时间为1秒
* 任务队列的容量为128

### 线程池分类
* FixedThreadPool，通过Executors的newFiexedThreadPool方法创建，是线程数量固定的线程池。只有核心线程且核心线程不会被回收，能更快的响应外界的请求。
* CachedThreadPool，通过Executors的newCachedThreadPool方法创建，只有非核心线程，且最大线程数为Integer.MAX_VALUE。
* ScheduleedThreadPool,通过Executors的newScheduleedThreadPool方法创建，核心线程个数固定，非核心线程个数无限制，用于执行定时任务和具有固定周期的重复任务。
* SingleThreadPool，通过Executors的newSingleThreadPool方法创建，只有一个核心线程，确保所有的任务都在同一个线程中顺序执行。
