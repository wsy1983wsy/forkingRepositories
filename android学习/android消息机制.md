# 消息机制的五元素
|类名|作用|
|----|----|
|Message|传递的消息内容|
|Handler|发出消息，处理消息，线程切换|
|MessageQueue| 消息的单向链表，用于存储消息|
|Looper|以无限循环的形式去查找（去MessageQueue中获取）是否有新消息，如果有则进行处理，否则会一直等待|
|ThreadLocal|不是线程，用于在每个线程中存储数据，可以认为是每个线程内的数据存贮器|

## 如何使线程具有消息机制
想要使线程具有消息机制，必须定义一个Looper，典型代码如下所示：
```
class Worker extends Thread {
    public volatile Handler handler; // actually private, of course
    public void run() {
        //设置Looper所在的线程，保存当前Looper到ThreadLocal中，初始化消息队列，ThreadLocal保存的当前Looper和所在线程有一个映射关系。
        Looper.prepare();
        //创建一个Handler，Handler保存了当前线程的Looper，Looper中的消息队列，handler可以不在这里定义。但是保存的Looper，和消息队列肯定是和当前线程相关的。
        mHandler = new Handler() {
            public boolean handleMessage(Message msg) {
                // ...
            }
        };
        //执行loop操作，loop是一个死循环，但是在没有消息的时候会进休眠，释放CPU给其他的线程进行，loop的执行是在当前线程中。
        Looper.loop();
    }
} 

```

```
UI线程的消息循环是在main方法中进行的。
public static void main(String[] args) {
        //一些配置操作
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        AsyncTask.init();
        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

## Message 消息
消息机制中传递的数据，或者叫做任务（需要执行的操作）。
消息包含以下几个参数。
* what 类型，表明消息要做什么事情
* 参数（arg1，arg2），执行任务需要的参数
* target执行消息的目标，任务的执行者
* 回调（callback），需要执行的操作，如果没有target
* next，在消息队列中的下一个消息
* 其中target不能为空，callback可以为空。如果target为空则会报出异常。在MessageQueue的enqueueMessage方法中有target为空的异常抛出：
```
if (msg.target == null) {
   throw new AndroidRuntimeException("Message must have a target.");
}
```
* target和callback的设置
 
|参数|设置位置|
|----|--------|
|callback|两个getPostMessage方法|
|target|enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)方法|
```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        //设置消息的target为Handler自身，即执行post和send的Handler
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

* 其他参数的设置
 what,arg1,arg2均可不进行设置。

## Handler
主要进行消息的发送和处理，线程切换。
* 发送使用post和send方法将消息发送出去，post最终调用send方法，send方法最终调用MessageQueue的enqueueMessage方法，这个时候会唤醒Looper，将消息插入到MessageQueue中。
* 处理消息，通过dispatchMessage方法接收消息，在Looper的loop方法中调用了Message的target的dispatchMessage方法。dispatchMessage方法代码如下:

```
Looper的loop方法
public static void loop() {
        //获取当前的looper
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //根据looper获取queue
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            //获取下一条消息
            //可能会休眠
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            //调用消息的target进行消息处理，target即Handler。Looper所在的线程保证了Handler处理消息的线程，可以认为这里执行了线程切换。
            msg.target.dispatchMessage(msg);
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            //回收消息
            msg.recycle();
        }
    }
```
```
/**
 * Handler的消息处理方法
 */
public void dispatchMessage(Message msg) {
    
    if (msg.callback != null) {
    //如果是通过post方法发送的消息，则msg.callback不为null，则执行post方法的Runnable类型的参数的run方法
        handleCallback(msg);
    } else {
        if (mCallback != null) {
        //如果Handler的mCallback不是null，则执行mCallback的handleMessage方法，有Handler有一个构造函数为 Handler(Callback callback, boolean async)，在这个构造函数中进行了mCallback的设置，通过这个构造函数可以不通过派生Handler的匿名实例来初始化Handler，可以通过Activity实现Callback接口使用次构造函数。
            if (mCallback.handleMessage(msg)) {
            //调用mCallback的handleMessage方法
                return;
            }
        }
        //调用自身的handleMessage方法来处理消息，如果子类重写该方法则调用子类的实现。
        handleMessage(msg);
    }
}
```
* Handler进行线程切换
我们通常情况下使用Handler的场景是，工作线程执行复杂的操作完成后，将结果组织成Message，使用Handler的sendMessage方法将Message传递初期，然后消息进入MessageQueue，Looper无限循环获取Message
,调用Handler的dispatchMessage方法，进行消息处理。典型使用代码为：

```
public class MainActivity extends Activity implements OnClickListener {
		 private static final int FINISH = 0;
	    private TextView stateText;  
	    private Button btn;  
	    //主线程中的Handler，因此Handler的方法也在主线程中执行。 默认的初始化Handler方法 
	    //默认的构造方法设置Handler的Looper为当前线程的Looper
	    private Handler handler = new Handler() {  
	        @Override
	        public void handleMessage(Message msg) {  
	            if (msg.what == FINISH) {  
	                stateText.setText("结束");  
	            }  
	        }  
	    };
	      
	    @Override  
	    public void onCreate(Bundle savedInstanceState) {  
	        super.onCreate(savedInstanceState);  
	        setContentView(R.layout.activity_main);  
	        stateText = (TextView) findViewById(R.id.textView);  
	        btn = (Button) findViewById(R.id.button);  
	          
	        btn.setOnClickListener(this);  
	    }  
	  
	    @Override  
	    public void onClick(View v) {  
	        new WorkThread().start();  
	    }  
	      
	    //工作线程
	    private class WorkThread extends Thread {  
	        @Override  
	        public void run() {  
	            //......处理比较耗时的操作  
	              
	            //处理完成后给handler发送消息
	            Message msg = new Message();  
	            msg.what = FINISH;
	            handler.sendMessage(msg);
	        }  
	    }  
}
```
为什么Handler可以进行线程的切换，dispatchMessage方法的执行不是在工作线程中？这是由于handler对应的Looper和UI线程匹配，因此其loop操作也是在ui线程上执行的，所以handleMessage也是在ui线程上执行的，这样就可以进行线程的切换。

## Looper
消息循环。不停的从消息队列中查看时候有新消息，如果有消息就会立即处理，否则就睡眠，让其他线程执行。<br>
Looper.prepare为当前线程创建一个Looper，loop开启消息循环，不停的检查消息队列。<br>
prepareMainLooper给主线程创建一个Looper。

## ThreadLocal
用于在每个线程中存储数据，在这里主要用于保存当前线程的Looper。

## 为何在UI线程Looper.loop看起来没有阻塞其他操作
消息机制在c语言层使用了epoll和pipe，保证了在没有消息的时候Looper.loop方法UI线程设置成睡眠等待状态，等待有新的消息，Looper.loop方法被设置为睡眠等待状态时，会释放掉CPU，让其他线程执行。当其他线程发送事件时会唤醒UI线程，其他线程的事件包括重绘，触摸事件等；或者通过使用UI线程的Handler发送消息来唤醒UI线程

# 消息处理流程
1. 工作线程执行耗时操作
2. 使用handler发送消息，Handler在创建的时候根据不同的参数确定Handler所在的线程，如果没有指定Looper，则在创建时的那个线程内，如果指定了Looper，则在Looper所在的县城。
3. handler调用enqueue方法将消息插入MessageQueue中，在C++代码层会通过Looper的C++的代码唤醒Looper。
4. Looper通过loop获取消息，会调用MessageQueue的next方法，在next方法中会调用nativePollOnce，如果没有数据则会休眠，将CPU让给其他的操作使用。
5. Looper调用Message的target的dispatchMessage方法处理消息，因此Looper在哪个线程里，dispatchMessage就在哪个线程里，handler的dispatchMessage操作也在哪个线程里。
6. dispatchMessage进行消息的处理，根据条件使用不同的方法的方法进行处理
 6.1 如果消息有callback则执行消息的callback操作，消息的callback是一个Runnable对象，通过Handler的Post方法传递的Runnable对象构建消息。
 6.2 如果Handler有mCallback则使用mCallback执行处理，mCallback是在构建Handler时指定的。
 6.3 如果以上均不成立，则执行Handler的handleMessage方法

# Handler，MessageQueue，Looper，线程关系
* 一个Handler有一个Looper，且只有一个。
* 一个线程有一个Looper，且只有一个。
* 一个Looper可以有多个Handler，保证每个消息的target不同，处理方法不同。
* Looper运行在所在的线程内，导致handler也运行在Looper所在的线程内。
* 一个Looper有一个MessageQueue，Handler操作的MessageQueue时Looper的MessageQueue。