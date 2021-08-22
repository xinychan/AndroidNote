# Handler 消息机制

典型问题：

## 一个线程有几个 Handler？

Handler 无限制，可以任意 new Handler；

## 一个线程有几个 Looper？如何保证？

一个线程，对应一个 Looper；
通过 ThreadLocal 保证 Thread 与 Looper 的 1 对 1 绑定；

Looper 的构造函数，持有 MessageQueue 和 Thread，和一个唯一的 ThreadLocal；

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    final MessageQueue mQueue;
    final Thread mThread;
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

Looper 通过 prepare 来初始化，初始化时通过 ThreadLocal 来绑定 Looper 和 MessageQueue

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

Thread 中，持有 ThreadLocal.ThreadLocalMap；
sThreadLocal.set(new Looper(quitAllowed))会将自身 ThreadLocal 与 Looper 保存到 Thread 的 ThreadLocalMap 中；
从而保证 ThreadLocal 与 Looper 进行 1 对 1 绑定，也保证了 Thread 与 Looper 的 1 对 1 绑定；
Thread==>ThreadLocal.ThreadLocalMap==>key:ThreadLocal;Value:Looper
ThreadLocal 是唯一的，为了保证 Looper 唯一，在 sThreadLocal.set(new Looper(quitAllowed))前；
先判断(sThreadLocal.get() != null)，保证 ThreadLocal.set 调用前，ThreadLocal 中没有数据；
主线程创建 Looper(boolean false)，子线程创建 Looper(boolean ture)

## Handler 内存泄漏的原因？为什么其他内部类没有这个问题？

可能造成内存泄漏的方式：

    // 方式1：匿名内部类
    Handler handler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };

    // 方式2：非静态内部类
    protected class AppHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                // TODO: 2019/4/30
            }
        }
    }

原因：非静态内部类，或者匿名内部类。默认持有外部类引用
Handler 默认持有 this==>Activity

为什么其他内部类没有内存泄漏这个问题？
Handler 处理原理：
Message 持有 Handler target==>持有 Activity
当 Message 有延迟 delay 调用时，在未处理这个 Message 之前，一直存在消息队列中；
MessageQueue 持有==>Message 持有==>Handler 持有==>Activity
直到消息被处理，才会释放这条引用链，而这期间 Activity.onDestory()虽然调用了，但是也还是存在内存中，没有被释放，导致内存泄漏；

处理内存泄漏：

1-Activity 销毁时，清空 Handler 中，未执行或正在执行的 Callback 以及 Message

    // 清空消息队列，移除对外部类的引用
    @Override
    protected void onDestroy() {
        super.onDestroy();
        mHandler.removeCallbacksAndMessages(null);

    }

2-静态内部类+弱引用

    private static class AppHandler extends Handler {
        //弱引用，在垃圾回收时，被回收
        WeakReference<Activity> activity;

        AppHandler(Activity activity){
            this.activity=new WeakReference<Activity>(activity);
        }

        public void handleMessage(Message message){
            switch (message.what){
                //todo
            }
        }
    }

## 为何主线程可以直接创建 Handler？子线程创建 Handler 要注意什么？

ActivityThread 主线程 main()方法中，直接调用 Looper.prepareMainLooper()/Looper.loop();无需手动调用；
子线程中需要手动执行 Looper.prepare()/Looper.loop();没有调用 Looper.prepare()，则 Thread 中没有关联的 Looper；

## 子线程中维护的 Looper，消息队列无消息时的处理方案是什么？有什么用？

Looper.loop()通过 MessageQueue.next()获取消息；
MessageQueue.next()是可以阻塞的方法；
当消息队列中没有消息时，会调用 MessageQueue.nativePollOnce(long ptr, int timeoutMillis)方法；且参数 timeoutMillis=-1
当 nativePollOnce 中的 timeoutMillis 为-1 时，则表示无限等待，直到有事件发生为止；若值为 0，则无需等待立即返回
nativePollOnce==>Linux==>epoll==>Linux 中 MessageQueue
要结束等待，需要退出机制；需要退出线程，首先要唤醒线程；
Looper.quit()==>MessageQueue.quit()==>MessageQueue.nativeWake()
所以 MessageQueue.next() 中的调用的 nativePollOnce() 方法会阻塞；
调用 Looper.quit() ==>MessageQueue.quit()
MessageQueue.quit()会使得 MessageQueue.next()返回 null 给 Looper.loop()方法，并调用 nativeWake()
Looper.loop()中收到 MessageQueue.next()返回值为 null，则跳出 for 循环，结束 Looper.loop()方法；
因此可以在 Activity.onDestroy 中执行 Handler.getLooper().quit() 主动释放掉 Looper；

主线程为什么不可以主动调用 Handler.getLooper().quit()？
ActivityThread 中的 Handler：H；
处理了 ActivityThread 中包括 BIND_APPLICATION/EXIT_APPLICATION/RECEIVER/CREATE_SERVICE/RELAUNCH_ACTIVITY/DUMP_ACTIVITY 等事件；
其中 EXIT_APPLICATION 中执行了 Looper.myLooper().quit()；
所以主线程只有结束应用时，才执行 Looper().quit()；不能主动调用 Looper().quit()；

## 既然可以多个 Handler 往消息队列添加数据（发消息的 Handler 可能处于不同线程），那么内部如何确保线程安全？

一个线程==>Looper==>MessageQueue
多个 Handler 都同时往 MessageQueue 中添加消息时，如何保证消息是准确的？
在添加消息和取出消息的 enqueueMessage/next()方法中，都使用了 synchronized 关键字加锁

## 使用 Message 时应该如何创建它？

享元设计模式：
享元模式的主要目的是实现对象的共享，即共享池，当系统中对象多的时候可以减少内存的开销，通常与工厂模式一起使用

Message.obtain 中使用 sPool 来共享 Message 对象，而不是每次都创建新的 Message；
如果每次都 new Message；则每次都创建新内存，Message 处理完之后被销毁；重复这个步骤会发生内存抖动
因此使用 Message 时要通过 Message.obtain() 创建；

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

## Looper 死循环为什么不会导致应用卡死？

唤醒线程的方法：
1-Looper 中添加 Message，通过 nativeWake()==>loop 运作
2-输入事件

产生 ANR 问题原因不是主线程睡眠，而是输入事件没有响应；
输入事件没有响应它就无法唤醒这个 Looper，所以加入时间限制；

ANR==>消息没有及时处理；与 Looper 是否死循环，是否阻塞无关；

## 消息处理的流程

Handler.sendMessage ==> Handler.handleMessage 的执行流程是怎样的？

发送消息：
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

发送消息之后如何获取消息？
Looper.loop()==>MessageQueue.next()
Looper.loop()中有一个 for 循环的方式，无限的调用 MessageQueue.next()；
MessageQueue.next()返回值为当前获取的消息，消息获取后执行
==>Message.target.dispatchMessage(msg)
Message.getTarget()返回值为发送当前消息的 Handler；
在 Handler.dispatchMessage()中最终调用了 handleMessage
==>Handler.handleMessage()

Looper.loop()什么时候调用？
子线程中需要手动执行 Looper.prepare()/Looper.loop()
ActivityThread 主线程 main()方法中，直接调用 Looper.prepareMainLooper()/Looper.loop();无需手动调用；

如何停下消息队列的获取？
Looper.loop()中 for 循环，会判断 MessageQueue.next()的返回值；
返回值为 null，for 循环停止；

子线程中需要手动执行 Looper.prepare()/Looper.loop()
ActivityThread 主线程 main()方法中，直接调用 Looper.prepareMainLooper()/Looper.loop()

        new Thread(new Runnable() {
            @Override
            public void run() {
                /**
                 *  1、创建了Looper对象，然后Looper对象中创建了MessageQueue
                 *  2、并将当前的Looper对象跟当前的线程（子线程）绑定ThreadLocal
                 */
                Looper.prepare();

                /**
                 * 1、创建Handler对象，然后从当前线程中获取Looper对象，然后获取到MessageQueue对象
                 */
                subHandler = new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                    }
                };

                /**
                 * 1、从当前线程中找到之前创建的Looper对象，然后找到MessageQueue
                 * 2、开启死循环，遍历消息池中的消息
                 * 3、当获取到msg的时候，调用这个msg的handler的dispatchMsg方法，让msg执行起来
                 */
                Looper.loop();
            }
        }).start();

new Handler 如何和 Looper 关联？

new Handler 首先会获取 Looper.myLooper()来持有一个 Looper

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

Looper.prepare()方法通过 ThreadLocal 中的 ThreadLocalMap，将 ThreadLocal 和 Looper 关联；
ThreadLocal 又与 Thread 关联，从而 Thread 与 Looper 关联；

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

Thread 中持有 ThreadLocal.ThreadLocalMap
在 ThreadLocal 的 set/get 方法中，会通过 Thread.currentThread()获取当前线程；
并将自身 ThreadLocal 与 Value 保存到当前线程的 ThreadLocalMap 中；
从而实现 Thread 与 Value 对象的绑定；

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
