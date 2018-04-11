## 【干货】快速理解Android 中的 Handler机制

> 本文目标，采用简单易懂的方式理解Android中的Handler机制，以及自己动手去模拟实现Android系统的Handler机制；

> Handler 是为了解决Android中子线程与主线程之间通信的相关问题而存在的，主要涉及到：Handler、Message、MessageQueue、Looper这几个系统类，它们存放在``` android.os```包下；
> 简单介绍下它们的作用：
>- **Handler** ：子线程向主线程发送消息、主线程处理接收到的消息； 
>- **Message**：消息载体，如果传输数据简单可以直接使用arg1、arg2这两个整型数据，如果需要传复杂的消息，使用obj传输对象数据； 
>- **MessageQueue**：消息队列，用来存储管理当前的线程中的所有Message消息； 
>-  **Looper** ：消息轮询器，不断的从消息队列MessageQueue中取出Message进行分发； 
### 一、基本使用方式
讲解之前我们先看下使用案例，如下：
功能描述：点击按钮开始倒计时，将当前剩余的时间更新到UI控件TextView中；
![Handler机制讲解效果图.gif](https://upload-images.jianshu.io/upload_images/1786025-31620448548b37c0.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)

```java
public class HandlerActivity extends AppCompatActivity {
    private static final int UPDATE = 0x1;
    public Button btnStartCountDown;
    public TextView tvCountDown;

    //默认是在主线程中执行
    private final MyHandler mHandler = new MyHandler(this);

    //继承实现自己的Handler，处理子线程与主线程的通讯交互
    static class MyHandler extends Handler {
        //使用弱引用，防止内存泄露
        private final WeakReference<HandlerActivity> mActivity;

        private MyHandler(HandlerActivity mActivity) {
            this.mActivity = new WeakReference<>(mActivity);
        }

        //重写handleMessage进行处理接收到的Message
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            //获取引用的UI主线程Activity，用来获取UI线程的控件等
            HandlerActivity activity = mActivity.get();
            if (activity != null) {
                //分发处理消息
                switch (msg.what) {
                    case UPDATE:
                        activity.tvCountDown.setText("还有" + String.valueOf(msg.arg1) + "秒");
                        break;
                }
            }
        }
    }

    //模拟子线程进行耗时任务
    private final Runnable mRunnable = new Runnable() {
        @Override
        public void run() {
            //这里做的是一个倒计时定时一秒发送一次数据
            for (int i = 60; i > 0; i--) {
                //构建属于子线程的Message
                Message msg = new Message();
                msg.what = UPDATE;
                msg.arg1 = i;

                //通过主线程中的Handler实例进行消息发送，将子线程中的消息发送到主线程中
                mHandler.sendMessage(msg);
                try {
                    //休眠1秒
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                //打印Log
                Log.i("TAG", "还有 " + i + " 秒");
            }
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        btnStartCountDown = findViewById(R.id.btn_start_count_down);
        tvCountDown = findViewById(R.id.tv_count_down);
        btnStartCountDown.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //开启子线程运行耗时操作
                new Thread(mRunnable).start();
            }
        });
    }
}
```
可以看出Handler的使用很简单，但是他却可以帮助我们有效的解决子线程与主线程间通信的问题，接下来讲解下原理；

### 二、源码分析
![快速理解Handler机制.png](https://upload-images.jianshu.io/upload_images/1786025-31f37907c27e60f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

对照上面的流程图，我们进行源码片段分析，关键的几个地方提取出来，如下：
```java
public final class Looper {
    // 线程本地存储,通过它就可以在指定的线程中存储数据，然后只有在这个指定的线程中才能够访问得到之前存储的数据。
    // 但是对于其他线程来说，是获取不到保存在另外线程中的数据的。
    // 一般来说，当某些数据是以线程为作用域并且不同的线程对应着不同的数据副本的时候，就可以考虑使用ThreadLocal了
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    //消息队列
    final MessageQueue mQueue;
        private Looper() {
        //初始化消息队列
        mQueue = new MessageQueue();
    }

    /**
     * 在当前的线程中准备一个消息轮询器
     */
    public static void prepare() {
        //一个线程只能对应一个轮询器
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //向ThreadLocal添加一个轮询器
        sThreadLocal.set(new Looper());
    }

    /**
     * 返回当前线程对应的轮询器
     *
     * @return Looper
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
    	//取出当前的线程的Looper对象
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //取出Looper对应的MessageUeue
        final MessageQueue queue = me.mQueue;
        //……
        for (;;) {
            //取出MessageQueue中的栈顶的消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            //……
            try {
                // 调用Message对应的Handler中的dispatchMessage(Message msg)进行消息的分发处理
                msg.target.dispatchMessage(msg);
               //……
            } finally {
             //……
            }               
            //释放资源
            msg.recycleUnchecked();
        }
    }
}
```

```java
public class Handler {

    //持有当前线程的Looper
    final Looper mLooper;
    //持有Looper中的MessageQueue消息队列，sendMessage需要向队列插入消息
    final MessageQueue mQueue;

    public Handler() {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                    "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
    }

    /**
     * 子类需要重写这个方法写入自己的处理逻辑
     *
     * @param msg Message消息
     */
    public void handleMessage(Message msg) {
    }

    /**
     * 调度消息
     *
     * @param msg Message消息
     */
    public void dispatchMessage(Message msg) {
        //这里为了简单说明原理与原码有些不一样，删减了一些判断逻辑
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            handleMessage(msg);
        }
    }

    /**
     * 处理回调
     *
     * @param message Message消息
     */
    private static void handleCallback(Message message) {
        //取出Message中 Runnable 进行运行
        message.callback.run();
    }


    /**
     * 发送一条Message消息
     *
     * @param msg 要发送的Message消息
     * @return 是否发送成功
     */
    public final boolean sendMessage(Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    /**
     * 发送延迟消息
     *
     * @param msg         要发送的Message消息
     * @param delayMillis 要延迟的时间
     * @return 是否发送成功
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        //这里很巧妙的将消息的时间点进行延长，从而达到了延迟发送的效果
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    /**
     * 发送消息所对应的时间
     *
     * @param msg          要发送的Message消息
     * @param uptimeMillis 对应的消息所在的时间点
     * @return
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        //获取当前线程对应的Looper中的MessageQueue
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //消息压入队列操作
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    /**
     * 消息压入消息队列操作
     *
     * @param queue        当前Handler对应的消息队列
     * @param msg          要发送的Message消息
     * @param uptimeMillis 压入队列的时间，系统的实习会对这个时间进行排序，从而保证消息的有序出栈 {@link MessageQueue#next()}
     * @return 是否压入队列成功
     */
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}
```

```java
public class Message implements Parcelable {
    //用户定义的消息代码，以便收件人可以识别此消息的内容
    public int what;
    //如果只是传递整形数据可以使用 arg1、arg2
    public int arg1;
    public int arg2;

    //如果传递复杂数据可以使用这个
    public Object obj;

    //标记当前的Message要作用在那个Handler中
    Handler target;

    //当前的Message要执行的子线程
    Runnable callback;

    // sometimes we store linked lists of these things
    Message next;

    private static final Object sPoolSync = new Object();
    private static Message sPool;

    public Message() {
    }


    /**
     * 这里是性能优化，从线程池获取一个对象，避免重新创建对象，本案例中没有使用到这个特性
     *
     * @return Message
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                return m;
            }
        }
        return new Message();
    }

    //////////////////////////////////////下面是实现Parcelable固定的写法与逻辑关系不大/////////////////////////////////////////////
    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };

    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                    "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable) obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                        "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
    }

    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
    }
}
```
```java
public final class MessageQueue {

    //获取消息队列的下一个数据
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
    
    //向队列中添加一个消息
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                //这里调用的jni进行了Message存储
                nativeWake(mPtr);
            }
        }
        return true;
    }
}
```
### 三、自定义实现Handler机制
> 通过上面的源码分析我们不难看出Handler、Message、MessageQueue、Looper之间的关系，画一个类图解释一下，可以看到下图中所描述的，这四个类相互持有，并且是一一对应的关系；

![快速理解Handler机制类图.png](https://upload-images.jianshu.io/upload_images/1786025-ccc69b75c35e8d36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

经过上面的一系列分析我们来自己手动实现Handler机制。

```java
public class Looper {
    //静态常量，整个APP共享这一个
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();

    //跟当前的线程持有的Looper绑定的消息队列
    final MessageQueue mQueue;

    private Looper() {
        //初始化消息队列
        mQueue = new MessageQueue();
    }

    /**
     * 在当前的线程中准备一个消息轮询器
     */
    public static void prepare() {
        //一个线程只能对应一个轮询器
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        //向ThreadLocal添加一个轮询器
        sThreadLocal.set(new Looper());
    }

    /**
     * 返回当前线程对应的轮询器
     *
     * @return Looper
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

    /**
     * 开启轮询
     */
    public static void loop() {
        //获取当前线程的轮询器
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //获取轮询器对应的消息队列
        final MessageQueue queue = me.mQueue;

        //永真循环不断地去取消息队列中的消息
        for (; ; ) {
            //取出消息队列中Message消息
            Message msg = queue.next(); // might block

            // 由于我们这里采用的是java帮我们实现的BlockingQueue，
            // 这里跟系统的实现判断有些不一样
            if (msg != null) {
                //将取出消息给发出当前消息的Handler的dispatchMessage进行消息的调度
                msg.target.dispatchMessage(msg);
            }
        }
    }
}
```

```java
public class Handler {

    //持有当前线程的Looper
    final Looper mLooper;
    //持有Looper中的MessageQueue消息队列，sendMessage需要向队列插入消息
    final MessageQueue mQueue;

    public Handler() {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                    "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
    }

    /**
     * 子类需要重写这个方法写入自己的处理逻辑
     *
     * @param msg Message消息
     */
    public void handleMessage(Message msg) {
    }

    /**
     * 调度消息
     *
     * @param msg Message消息
     */
    public void dispatchMessage(Message msg) {
        //这里为了简单说明原理与原码有些不一样，删减了一些判断逻辑
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            handleMessage(msg);
        }
    }

    /**
     * 处理回调
     *
     * @param message Message消息
     */
    private static void handleCallback(Message message) {
        //取出Message中 Runnable 进行运行
        message.callback.run();
    }


    /**
     * 发送一条Message消息
     *
     * @param msg 要发送的Message消息
     * @return 是否发送成功
     */
    public final boolean sendMessage(Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    /**
     * 发送延迟消息
     *
     * @param msg         要发送的Message消息
     * @param delayMillis 要延迟的时间
     * @return 是否发送成功
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        //这里很巧妙的将消息的时间点进行延长，从而达到了延迟发送的效果
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    /**
     * 发送消息所对应的时间
     *
     * @param msg          要发送的Message消息
     * @param uptimeMillis 对应的消息所在的时间点
     * @return
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        //获取当前线程对应的Looper中的MessageQueue
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        //消息压入队列操作
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    /**
     * 消息压入消息队列操作
     *
     * @param queue        当前Handler对应的消息队列
     * @param msg          要发送的Message消息
     * @param uptimeMillis 压入队列的时间，系统的实习会对这个时间进行排序，从而保证消息的有序出栈 {@link MessageQueue#next()}
     * @return 是否压入队列成功
     */
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}
```

```java
public class Message implements Parcelable {
    //用户定义的消息代码，以便收件人可以识别此消息的内容
    public int what;
    //如果只是传递整形数据可以使用 arg1、arg2
    public int arg1;
    public int arg2;

    //如果传递复杂数据可以使用这个
    public Object obj;

    //标记当前的Message要作用在那个Handler中
    Handler target;

    //当前的Message要执行的子线程
    Runnable callback;

    // sometimes we store linked lists of these things
    Message next;

    private static final Object sPoolSync = new Object();
    private static Message sPool;

    public Message() {
    }


    /**
     * 这里是性能优化，从线程池获取一个对象，避免重新创建对象，本案例中没有使用到这个特性
     *
     * @return Message
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                return m;
            }
        }
        return new Message();
    }

    //////////////////////////////////////下面是实现Parcelable固定的写法与逻辑关系不大/////////////////////////////////////////////
    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };

    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                    "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable) obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                        "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
    }

    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
    }
}
```

```java
public class MessageQueue {
    private static final int MAX_QUEUE_SIZE = 50;

    // 这里使用BlockingQueue集合进行模拟Native层的队列
    // 后续在进行讲解Android中的C++如何实现的Message队列操作
    final BlockingQueue<Message> mMessages;

    MessageQueue() {
        //创建固定大小的消息队列
        mMessages = new ArrayBlockingQueue<>(MAX_QUEUE_SIZE);
    }

    /**
     * 取出队列中的下一个消息
     *
     * @return Message消息
     */
    Message next() {
        Message msg = null;
        try {
            //从队列中取出头部的消息，并从队列中移除，通知 {@link BlockingQueue#put()} 可以入栈
            msg = mMessages.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        //将取出的消息返回
        return msg;
    }

    /**
     * 消息压入队列操作
     *
     * @param msg  要操作的消息
     * @param when 压入的时间
     * @return 是否要入成功
     */
    boolean enqueueMessage(Message msg, long when) {
        try {
            //压入消息队列，如果消息队列处于饱和状态，这里则会出现 block,直到 有调用 {@link BlockingQueue#take()}
            mMessages.put(msg);

            return true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }
}
```
改写一下测试代码，测试我们自己定义的Handler机制是否能够实现子线程与主线程的通信
```java
public class CustomerHandlerActivity extends AppCompatActivity {
    private static final int UPDATE = 0x1;

    //默认是在主线程中执行
    private MyHandler mHandler = null;

    //继承实现自己的Handler，处理子线程与主线程的通讯交互
    static class MyHandler extends Handler {
        //使用弱引用，防止内存泄露
        private final WeakReference<CustomerHandlerActivity> mActivity;

        private MyHandler(CustomerHandlerActivity mActivity) {
            this.mActivity = new WeakReference<>(mActivity);
        }

        //重写handleMessage进行处理接收到的Message
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            //获取引用的UI主线程Activity，用来获取UI线程的控件等
            CustomerHandlerActivity activity = mActivity.get();
            if (activity != null) {
                //分发处理消息
                switch (msg.what) {
                    case UPDATE:
                        //打印Log
                        Log.i("TAG", "还有 " + String.valueOf(msg.arg1) + " 秒");
                        break;
                }
            }
        }
    }

    //模拟子线程进行耗时任务
    private final Runnable mRunnable = new Runnable() {
        @Override
        public void run() {
            //这里做的是一个倒计时定时一秒发送一次数据
            for (int i = 60; i > 0; i--) {
                //构建属于子线程的Message
                Message msg = new Message();
                msg.what = UPDATE;
                msg.arg1 = i;

                //通过主线程中的Handler实例进行消息发送，将子线程中的消息发送到主线程中
                mHandler.sendMessage(msg);
                try {
                    //休眠1秒
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);

        //在当前的线程中准备一个消息轮询器
        Looper.prepare();

        //创建主线程的Handler
        mHandler = new MyHandler(this);

        //开启子线程运行耗时操作
        new Thread(mRunnable).start();

        //这里开启消息轮询器会造成UI的绘制阻塞
        Looper.loop();
    }
}
```
![image.png](https://upload-images.jianshu.io/upload_images/1786025-e927579caeb0bee8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/340)

可以通过日志查看我们自己定义的Handler机制是可以实现线程之间通讯的。

至于我们自己编写的Looper.loop();造成的阻塞问题我们后面在研究；