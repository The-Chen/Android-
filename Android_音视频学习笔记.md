# 一、发展方向

1、计算机相关专业本科及以上学历，具有扎实的计算机理论基础，熟悉常⽤数据结构算法。

2、熟悉 iOS/Android ⾳视频采集录制编辑相关框架。

3、熟悉 OpenGL/OpenGLES/DirectX/Metal/Vulkan 其中⼀种图形库开发，较好地掌握图形渲染基础算法和核⼼渲染技术。

4、熟悉 MP4、FLV 等视频格式；熟悉 RTMP、RTSP、HTTP-FLV、HLS、Mpeg-Dash 等协议；熟悉 H.264、HEVC、AV1、 AAC 等⾳视频编码格式。熟悉端侧的软硬解⽅案和实现

5、有短视频编辑模块或直播推流模块的开发经验，有视频剪辑和特效的开发经验。有视频质量客观和主观质量量化和评估优化经验

6、熟悉多线程编程，熟悉多线程的调试分析⽅法。



# 二、Lifecycle与协程、Flow



# 三、targetSdkVersion、compileSdkVersion、minSdkVersion作用与区别

## 1.minSdkVersion

指定app运行的最低设备sdk版本，如minSdkVersion=19 表示该app最低支持Android 4.4（API 19）设备，低于此版本的设备将不能使用该app。



## 2.compileSdkVersion

和编译时有关。比如我们当前compileSdkVersion=28（Andorid 9.0），Android 10 新增了有关5G的api。我们的app想尽早使用5G相关的api，这时只需要将compileSdkVersion=29（Android 10）,就能在编译阶段编译通过。



## 3.targetSdkVersion

直观翻译是“目标版本”，它的作用是兼容旧的api。

举个例子，Android 8.0/8.1 系统规定了透明的activity 不能固定方向，否则抛出异常。假设用了透明的activity，且固定了方向，在Android 8.0版本以下运行正常，当我们运行在Android 8.0/8.1系统上，结果如何会如何呢？很不幸，直接crash。对于我们来说，这结果当然不能接受了。

![img](https://upload-images.jianshu.io/upload_images/19073098-d3593349aad40400.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

从上图可以看出：getApplicationInfo().targetSdkVersion取的值即是在build.gradle里设置的targetSdkVersion值，这里判断targetSdkVersion是否大于Android 8.0（O），如果成立才执行后续逻辑（透明且固定方向），进而抛出异常。在此我们知道，想让这段逻辑不成立，那么我们将targetSdkVersion设置为26（Android 8.0 对应26）以下，即使我们app运行在Andorid 8.0以上的设备，都不会出现问题。这样，我们的app不更新，但是新的Android SDK 里Activity onCreate方法变更了部分逻辑行为，通过targetSdkVersion限制，我们的app依然能够以旧的逻辑运行新的设备上，达到了兼容老版本api的目的。



总结：

**对于同一个API(方法），新版本的系统更改了它的内部实现逻辑，新逻辑可能会影响之前调用此API的App，为了兼容此问题，引入targetSdkVersion。当targetSdkVersion>=xx(某个系统版本）时再生效其新修改的逻辑，否则还是沿用之前的逻辑。换句话说：如果你更改了targetSdkVersion值，说明你已经测试过兼容性问题了。**



``

```java
    /**
     * The minimum SDK version this application targets.  It may run on earlier
     * versions, but it knows how to work with any new behavior added at this
     * version.  Will be {@link android.os.Build.VERSION_CODES#CUR_DEVELOPMENT}
     * if this is a development build and the app is targeting that.  You should
     * compare that this number is >= the SDK version number at which your
     * behavior was introduced.
     */
    public int targetSdkVersion;
```

更新targetSdkVersion时，比如从26（Android 8.0）变更到29（Android 9.0），意味着我们对26～29之间的系统兼容性进行了充分的测试，因此每当我们变更targerSdkVersion时，要充分测试其系统兼容性。

也许你会说，那我可以不更新targetSdkVersion值嘛，一劳永逸，理论上没啥问题。但是，如果你想在新的系统上使用api新的行为，那么就需要更新targetSdkVersion。再者，google会对targetSdkVersion进行一些强关联，比如app上传到google play，必须要求targetSdkVersion>=26（具体的值随着时间的推移，要求不一样）。

![img](https://upload-images.jianshu.io/upload_images/19073098-63025a6b56456d4c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1142/format/webp)

image.png

到这，也许你还会说，为啥Andorid 6.0（包含）以上动态权限不根据targetSdkVersion来适配呢？我们说的targetSdkVersion是向前兼容，兼容之前的app使用的同一api旧的行为逻辑，而动态权限是google在Android 6.0（包含）以上强制要求的，人家就不想给你转圜的余地，有啥办法呢？还是乖乖判断当前系统是否是Android 6.0及其以上的，再决定是否动态申请权限了。。



# 四、Android事件驱动Handler-Message-Looper解析

``

```java
                Thread t1 = new Thread(new Runnable() {
                    @Override
                    public void run() {
                        doTask1();
                    }
                });
                t1.start();

                Thread t2 = new Thread(new Runnable() {
                    @Override
                    public void run() {
                        doTask2();
                    }
                });
                t2.start();
```

如上所示，t2需要等待t1执行完毕再执行，那么t2怎么知道t1完毕呢？有两种方式

**1、t1主动告诉t2自己已执行完毕，属于等待通知方式（wait&notify、condition await&signal）**
**2、t2不断去检测t1是否完毕，属于轮询方式**



## Handler发送Message

```java
    Button button;
    TextView tv;

    private Handler handler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
        @Override
        public boolean handleMessage(@NonNull Message msg) {
            tv.setText(msg.arg1 + "");
            return false;
        }
    });

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        button = findViewById(R.id.start_thread_one);
        tv = findViewById(R.id.tv);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread(()->{
                    Message message = Message.obtain();
                    message.arg1 = 2;
                    handler.sendMessage(message);
                }).start();
            }
        });
    }
```

主线程里构造handler，子线程里构造message，并使用handler将message发送出去，在主线程里执行handleMessage方法就可以更新UI了。很明显，使用handler来切换线程我们只需要两步：

**1、主线程里构造handler，重写回调方法**
**2、子线程里发送message告知主线程执行回调方法**

[Android Studio关联Android SDK源码（Windows&amp;Mac）]: https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fwekajava%2Farticle%2Fdetails%2F120564987%3Fspm%3D1001.2014.3001.5501

## Message分析

既然message充当信使，那么其应当有必要的字段来存储携带的数据、区分收发双方等，实际上message不只有这些字段。上个例子里，我们通过静态方法获取message实例。

```java
    private static Message sPool;
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

sPool是静态变量，从名字可以猜测出是message存储池。obtain()作用是：先从存储池里找到可用的message，如果没有则创建新的message并返回，那什么时候将message放到存储池里呢？我们只需要关注sPoolSize增长的地方即可，搜索找到：

```csharp
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

message里有个message类型的next字段，熟悉链表的同学应该知道这是链表的基本操作，该字段用于指向链表里下一个节点的。现在要将一个message对象放入message存储池里：

**将待存储的message next字段指向存储池的头部，此时该头部指向的是第一个元素，再将存储池的头指向待插入的message，最后将存储池里个数+1。这时候message已经放入存储池里，并且当作存储池的头。**

我们再回过头来看看怎么从存储池里取出message：

**声明message引用，该引用指向存储池头部，将存储池头部指向链表下一个元素，并将message next置空，并将存储池里个数-1，这时候message已经从存储池里取出。也就是说，message对象存储池是通过栈来实现的。**

### MessageQueue

```cpp
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

sendMessageAtTime()方法输入两个参数，一个是待发送的message对象，另一个是该message延迟执行的时间。这里引入了一个新的类型：MessageQueue，其声明了一个字段：mQueue。接着调用enqueueMessage，该方法里msg.target = this，target实际上指向了handler。最后调用queue.enqueueMessage(msg, uptimeMillis)，我们来看看queue.enqueueMessage方法：

```kotlin
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
                //第一点
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //第二点
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
            //第三点
            if (needWake) {
                nativeWake(mPtr);
            }
        }
```

### 第一点

**队列为空、或者延迟时间为0、或者延迟时间小于队列头节点，那么将该message next字段指向队列头，并将队列头指向该message。此时该message已经插入enqueueMessage的队列头。**



### 第二点

> **如果第一点不符合，那么message不能插入队列头，于是在队列里继续寻找，直至找到延迟时间大于message的节点或者到达队列尾，再将message插入队列。**

从第一点和第二点可以看出，enqueueMessage实现了队列，并且是根据message延迟时间从小到大排序的，也就是说队列头的延迟时间是最小的，最先需要被执行的。

### 第三点

唤醒MessageQueue队列。

### 什么时候取出message

定位到Handler构造函数

```tsx
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

mQueue指向了mLooper里的mQueue，而mLooper = Looper.myLooper()

## Looper分析

```kotlin
##Looper.java
    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
   /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

##ThreadLocal.java
   public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

可以看出，Looper对象存储在当前线程里，我们暂时不管ThreadLocal原理，看到get方法，我们有理由相信有set方法，果不其然。

```csharp
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

既然ThreadLocal get方法在Looper.java里调用，那么ThreadLocal set方法是否也在Looper.java里调用?

```java
Looper.java
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

看到了我们熟悉的ActivityThread，也就是我们app启动main方法所在的类。

![img](https://upload-images.jianshu.io/upload_images/19073098-8c69faa580907ba6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

已经追溯到源头了，我们现在从源头再回溯。

```csharp
    private static Looper sMainLooper;  // guarded by Looper.class
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

在prepare方法里，构造了Looper对象。此刻我们看到了主角mQueue被构造出来了，也就是说主线程启动的时候就已经构造了Looper对象，并存储在静态变量sThreadLocal里。同时静态变量sMainLooper指向了新构建的Looper对象。
 接着查找mQueue的引用，发现

![img](https://upload-images.jianshu.io/upload_images/19073098-404edb651fe048b3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

final MessageQueue queue = me.mQueue，此处引用了mQueue，接着调用了MessageQueue里的next()方法，该方法返回message对象。

```csharp
MessageQueue.java
next()方法
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
```

Message msg = mMessages，我们之前分析了handler发送message后最终是将message挂在mMessages上，也就是我们的message队列。而next方法后续操作是：从队列头取出message，并将该message从队列里移除。此时我们已经找到message队列入队时机与方式、message队列出队方式。那么message队列出队时机呢？也就是啥时候调用的。继续回溯分析上面提到的Looper loop()方法。

```java
for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
            try {
                msg.target.dispatchMessage(msg);
                if (observer != null) {
                    observer.messageDispatched(token, msg);
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } catch (Exception exception) {
                if (observer != null) {
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            msg.recycleUnchecked();
        }
```

msg.target.dispatchMessage(msg);msg.target就是我们之前提到的handler，也就是handler.dispatchMessage(msg)

```java
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);(1)
        } else {
            if (mCallback != null) {
                (2)
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            (3)
            handleMessage(msg);
        }
    }
```

****1、handleCallback()调用的是message里Runnable类型的callback字段，我们经常用的方法view.post(runnable)来获取view的宽高，此处runnable会赋值给callback字段。**
 **2、handler mCallback字段，Callback是接口，接口里有handleMessage()方法，该方法的返回值决定是否继续分发此message。mCallback可选择是否实现，目的是如果不想重写handler handleMessage()方法，那么可以重写callback方法。**
 3、handleMessage，构造handler子类的时候需要重写该方法。**



1、handleCallback()

```java
                    //延时执行某任务
                    handler.postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            
                        }
                    }, 100);
                    
                    //view 获取宽高信息
                    tv.post(new Runnable() {
                        @Override
                        public void run() {
                            
                        }
                    });
```

此种方式，只是为了在某个时刻执行回调方法，message本身无需携带附加信息

2、mCallback.handleMessage(msg)
构造Handler匿名内部类实例

```java
    private Handler handler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
        @Override
        public boolean handleMessage(@NonNull Message msg) {
            tv.setText(msg.arg1 + "");
            return false;
        }
    });
```

3、handleMessage(msg)
内部类继承Handler，重写handleMessage方法

```java
    class MyHandler extends Handler {
        @Override
        public void handleMessage(@NonNull Message msg) {
            tv.setText(msg.arg1 + "");
        }
    }
    
    MyHandler handler = new MyHandler();
```

****1、message通过obtain()从存储池里获取实例，在loop()里message被使用后，调用msg.recycleUnchecked()放入存储池里（栈）。**
 2、message对象不仅可以通过obtain()获取，也可以直接构造。最终都会放入存储池里，推荐用obtain()，更好地复用message。**



## MessageQueue next()分析

该方法是从MessageQueue里取出message对象。loop()通过next调用获取message，如果message为空则loop()退出，假设这种情况出现，那么主线程就退出了，app就没法玩了。因此从设计上来说，next返回的message不能为空，那么它需要一直检测message直到获取到有效的message，所以next就有可能阻塞。实际上，next使用的一个for循环，确实可能会阻塞，我们来看看细节之处：

```kotlin
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            //第一点
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    //第二点
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        //第三点
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        //第四点
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
                    //第五点
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
                    //第六点
                    mBlocked = true;
                    continue;
                }
            }
        }
```

1、nativePollOnce(ptr, nextPollTimeoutMillis)，查找native层MessageQueue是否有数据。nextPollTimeoutMillis=0 该方法立即返回；nextPollTimeoutMillis=-1；该方法一直阻塞直到被主动唤醒；nextPollTimeoutMillis>0；表示阻塞到特定时间后返回。现在我们来解答之前的问题2，当有message入队列的时候，调用如下代码：

```undefined
            if (needWake) {
                nativeWake(mPtr);
            }
```

而needWake=mBlocked，当阻塞的时候mBlocked=true，执行nativeWake(mPtr)后，nativePollOnce()被唤醒返回。这就是最初我们为什么说handler-looper 采用等待通知方式。

2、如果队头节点是屏障消息，那么遍历队列直到找到一条异步消息，否则继续循环直到nativePollOnce阻塞。此处的应用场景之一是：在对view进行invalidate时候，为了保证UI绘制能够及时，先插入一条屏障消息，屏障消息delayTime=0，因此插入队列头部（注意和普通消息插入不同的是：如果队列某个节点和待插入的延迟时间一致，那么待插入的节点排在已有节点的后面，而屏障消息始终插在前面）。插入屏障消息后，后续的刷新消息设置为异步，再插入队列，那么该条消息会被优先执行。
 3、如果还未到执行时间，则设置阻塞超时时间。
 4、取出message节点，并调整队列头。
 5、队列里没有message，那么设置阻塞
 6、设置阻塞标记位mBlocked=true，并继续循环

## Handler线程切换本质

### message对象存储池

入栈

![img](https://upload-images.jianshu.io/upload_images/19073098-7781996b8ca838cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

出栈

![img](https://upload-images.jianshu.io/upload_images/19073098-498044016285ce86.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

### MessageQueue队列

出入队

![img](https://upload-images.jianshu.io/upload_images/19073098-a25f054082d75469.png?imageMogr2/auto-orient/strip|imageView2/2/w/914/format/webp)



### 调用栈

```java
handler.sendMessage(message)
 sendMessage->sendMessageDelayed(msg, 0)->sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis)->enqueueMessage(queue, msg, uptimeMillis)->queue.enqueueMessage(msg, uptimeMillis)
```

### 取消息调用栈

msg = loop()->queue.next()
msg->dispatchMessage(msg)->handleCallback(msg)/handleMessage(msg)

将Handler、MessageQueue、Looper用图表示如下：

![img](https://upload-images.jianshu.io/upload_images/19073098-310768e7660e73cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

由于MessageQueue是线程间共享对象，因此对队列的操作（入队、出队）需要加锁。

### Handler和Looper联系

1、从取消息调用栈可知，执行loop()的线程就是执行handleMessage(msg)的线程，而loop()在main()方法里执行，因此这时候取出的消息就在主线程执行
 2、Handler发送的message存储在Handler里的messagequeue，而此队列指向的是Looper的messagequeue，loop()方法从Looper的messagequeue里取消息。因此Handler和Looper共享同一个queue，handler负责发送（入队），Looper负责收取（出队），通过中间桥梁messagequeue实现了线程切换。
 3、接着第二点，Handler和Looper是如何共享queue呢？在构建Handler的时候，会根据当前线程获取存储的looper并获取其messagequeue，而looper对象需要在同一线程里构造，并将该looper引用设置到当前线程。在app启动的时候，系统已经在主线程的looper并赋值给静态变量sMainLooper，我们可以通过getMainLooper()获取主线程的looper对象。
 4、handler构造时，可以选择是否传入looper对象，若不传则默认从当前线程获取，若取不到则会抛出异常。这就是为什么在子线程里不能简单用消息循环的原因，因为handler没有和looper建立起关联。



### 子线程消息循环

以上，我们分析的是子线程发送消息，主线程在轮询执行。那我们能够在主线程（其它线程）发送消息，子线程轮询执行吗？这种场景是有现实需求的：

> A线程不断从网络拉取数据，而数据分为好几种类型，拿到数据后需要分类处理，处理可能比较耗时。为了不影响A线程效率，需要另起B线程处理，B线程一直在循环处理A拉取回来的数据。

是不是似曾相识，我们想要B线程达到类似主线程的looper功能，接下来看看该怎么做



```java
            Handler handlerB = null;
            Thread b = new Thread(new Runnable() {
                @Override
                public void run() {
                    //子线程里创建looper对象，并保存引用到当前线程里
                    Looper.prepare();
                    
                    //Handler构造方法默认获取当前线程的looper引用
                    handlerB = new Handler(new Callback() {
                        @Override
                        public boolean handleMessage(@NonNull Message msg) {
                            return false;
                        }
                    });
                    //子线程开启loop()循环
                    Looper.loop();
                }
            });
            b.start();
            
            Thread a = new Thread(()->{
                //线程发送消息
                handlerB.sendMessage(new Message())；
            });
            a.start();
```