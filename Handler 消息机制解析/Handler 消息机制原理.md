# Handler 消息机制原理

## Thread

主线程==>ActivityThread
子线程==>new Thread
Thread 中持有 ThreadLocal.ThreadLocalMap
在 ThreadLocal 的 set/get 方法中，会通过 Thread.currentThread()获取当前线程；
并将自身 ThreadLocal 与 Value 保存到当前线程的 ThreadLocalMap 中；
从而实现 Thread 与 Value 对象的绑定；

    // ThreadLocal 类中方法
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }

## Looper

ActivityThread 主线程 main()方法中，直接调用 Looper.prepareMainLooper()/Looper.loop();无需手动调用；
子线程中需要手动执行 Looper.prepare()/Looper.loop();没有调用 Looper.prepare()，则 Thread 中没有关联的 Looper；
Looper 的构造函数，持有 MessageQueue 和 Thread，和一个唯一的 ThreadLocal；

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    final MessageQueue mQueue;
    final Thread mThread;
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

Looper 通过 prepare 来初始化；

    /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

sThreadLocal.set(new Looper(quitAllowed))会将自身 ThreadLocal 与 Looper 保存到 Thread 的 ThreadLocalMap 中；
从而保证 ThreadLocal 与 Looper 进行 1 对 1 绑定，也保证了 Thread 与 Looper 的 1 对 1 绑定；
Thread==>ThreadLocal.ThreadLocalMap==>key:ThreadLocal;Value:Looper
Map 中的 Key:ThreadLocal 是唯一的，为了保证 Value:Looper 也唯一，在 sThreadLocal.set(new Looper(quitAllowed))前；
先判断(sThreadLocal.get() != null)，保证 ThreadLocal.set 调用前，ThreadLocal 中没有数据；

## Handler

handleMessage()方法供调用方自己实现，完成具体逻辑；

    public void handleMessage(@NonNull Message msg) {
    }

dispatchMessage()方法中会调用 handleMessage()；
Handler 处理消息的顺序是：Message 的 Callback > Handler 的 Callback > Handler 的 handleMessage 方法

    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

new Handler 首先会获取 Looper.myLooper()来持有一个 Looper；

    public Handler() {
        this(null, false);
    }

    public Handler(@Nullable Callback callback, boolean async) {
        // ignore some code
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

Looper.myLooper()获取 sThreadLocal.get()

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }

在 Looper.prepare()方法执行 sThreadLocal.set(new Looper(quitAllowed))；
通过 ThreadLocal，将 Thread 与 Looper 关联；
如果在子线程中 new Handler，没有调用 Looper.prepare()，则 Thread 中没有关联的 Looper；
会执行(mLooper == null)分支的 RuntimeException

## MessageQueue

MessageQueue.enqueueMessage()给消息队列添加消息；

    boolean enqueueMessage(Message msg, long when) {
        // ignore some code
        synchronized (this) {
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
                nativeWake(mPtr);
            }
        }
        return true;
    }

MessageQueue.next()取出队列中的消息；

    Message next() {
        // ignore some code
        for (;;) {
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
            }
        }
    }

发送消息和添加消息：
Handler.sendxxx()
Handler.postxxx()
//sendMessage/postxxx 等方法最后都会调用 sendMessageAtTime
==>Handler.sendMessageAtTime()
//sendMessageAtTime()中会执行 queueMessage()
==>MessageQueue.queueMessage();
queueMessage()会把消息添加到消息队列中；
MessageQueue 存储消息的方式是优先级队列；
MessageQueue 的本质：链表+先进先出的队列算法+按时间排序==>优先级队列
消息的时间越早，在队列中优先级越高，会先被获取；

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

获取消息和处理消息：
Looper.loop()==>MessageQueue.next()
Looper.loop()中有一个 for 循环的方式，无限的调用 MessageQueue.next()；
MessageQueue.next()返回值为当前获取的消息，消息获取后执行
==>Message.target.dispatchMessage(msg)
Message.getTarget()返回值为发送当前消息的 Handler；
在 Handler.dispatchMessage()中最终调用了 handleMessage
==>Handler.handleMessage()

    public static void loop() {
        // ignore some code
        final Looper me = myLooper();
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            // ignore some code
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
        }
    }

## Message

Message.obtain 中使用 sPool 来共享 Message 对象，而不是每次都创建新的 Message；
如果每次都 new Message；则每次都创建新内存，Message 处理完之后被销毁；重复这个步骤会发生内存抖动
因此使用 Message 时要通过 Message.obtain() 创建；

    Handler target;
    public static final Object sPoolSync = new Object();
    private static Message sPool;

    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
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

在 Looper.loop() 和 MessageQueue.removMessagexxx()时调用

    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    @UnsupportedAppUsage
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

Message.getTarget()返回发送当前消息的 Handler；

    public Handler getTarget() {
        return target;
    }


## 简易流程图

Thread==>通过 ThreadLocal==>关联 Looper==>持有 MessageQueue
Handler==>关联并持有 Looper==>持有 MessageQueue
Handler.sendMessage()==>MessageQueue.queueMessage()==>实现消息队列添加消息
Looper.loop()==>MessageQueue.next()==>Handler.dispatchMessage()==>Handler.handleMessage()==>处理消息
主线程 main()方法中自动调用 Looper.prepareMainLooper()/Looper.loop()；子线程中需要手动执行 Looper.prepare()/Looper.loop()；
Looper.quit()==>MessageQueue.quit()==>MessageQueue.next()返回 null==>Looper.loop()跳出 for 循环，结束 Looper.loop()==>线程结束
