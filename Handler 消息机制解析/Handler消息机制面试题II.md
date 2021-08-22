# 消息机制面试题

## 1、Handler 有哪些作用?

答：
1、Handler 能够进行线程之间的切换
2、Handler 能够按照顺序处理消息，避免并发
3、Handler 能够阻塞线程
4、Handler 能够发送并处理延迟消息
解析:
1、Handler 能够进行线程之间的切换，是因为使用了不同线程的 Looper 处理消息
2、Handler 能够按照顺序处理消息，避免并发，是因为消息在入队的时候会按照时间升序对当前链表进行排序，Looper 读取的时候，MessageQueue 的 next 方法会循环加锁，同时配合阻塞唤醒机制
3、Handler 能够阻塞线程主要是基于 Linux 的 epoll 机制实现的
4、Handler 能够处理延迟消息，是因为 MessageQueue 的 next 方法中会拿当前消息时间和当前时间做比较，如果是延迟消息，那么就会阻塞当前线程，等阻塞时间到，在执行该消息

## 2、为什么我们能在主线程直接使用 Handler，而不需要创建 Looper？

答：主线程已经创建了 Looper，并开启了消息循环

## 3、如果想要在子线程创建 Handler，需要做什么准备？

答：需要先创建 Looper，并开启消息循环

## 4、一个线程有几个 Handler？

答：可以有任意多个

## 5、一个线程有几个 Looper？如何保证？

答：一个线程只有一个 Looper，通过 ThreadLocal 来保证

## 6、Handler 发送消息的时候，时间为啥要取 SystemClock.uptimeMillis() + delayMillis，可以把 SystemClock.uptimeMillis() 换成 System.currentTimeMillis()吗？

答：不可以
SystemClock.uptimeMillis() 这个方法获取的时间，是自系统开机到现在的一个毫秒数，这个时间是个相对的
System.currentTimeMillis() 这个方法获取的是自 1970-01-01 00:00:00 到现在的一个毫秒数，这是一个和系统强关联的时间，而且这个值可以做修改
1、使用 System.currentTimeMillis()可能会导致延迟消息失效
2、最终这个时间会被设置到 Message 的 when 属性，而 Message 的 when 属性只是需要一个时间差来表示消息的先后顺序，使用一个相对时间就行了，没必要使用一个绝对时间

## 7、为什么 Looper 死循环，却不会导致应用卡死？

答：因为当 Looper 处理完所有消息的时候，会调用 Linux 的 epoll 机制进入到阻塞状态，当有新的 Message 进来的时候会打破阻塞继续执行。
应用卡死即 ANR: 全称 Applicationn Not Responding，中文意思是应用无响应，当我发送一个消息到主线程，Handler 经过一定时间没有执行完这条消息，那么这个时候就会抛出 ANR 异常
Looper 死循环: 循环执行各种事务，Looper 死循环说明线程还活着，如果没有 Looper 死循环，线程结束，应用就退出了，当 Looper 处理完所有消息的时候会调用 Linux 的 epoll 机制进入到阻塞状态，当有新的 Message 进来的时候会打破阻塞继续执行

## 8、Handler 内存泄露原因? 如何解决？

内存泄漏的本质是长生命周期的对象持有短生命周期对象的引用，导致短生命周期的对象无法被回收，从而导致了内存泄漏
下面我们就看个导致内存泄漏的例子

    public class MainActivity extends AppCompatActivity {

        private final Handler mHandler = new Handler() {
            @Override
            public void handleMessage(@NonNull Message msg) {
            //do something
            }
        };

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            //发送一个延迟消息，10分钟后在执行
            mHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
        }
    }

上述代码:
1、我们通过匿名内部类的方式创建了一个 Handler 的实例
2、在 onCreate 方法里面通过 Handler 实例发送了一个延迟 10 分钟执行的消息
我们发送的这个延迟 10 分钟执行的消息它是持有 Handler 的引用的，根据 Java 特性我们又知道，非静态内部类会持有外部类的引用，因此当前 Handler 又持有 Activity 的引用，而 Message 又存在 MessageQueue 中，MessageQueue 又在当前线程中，因此会存在一个引用链关系:
当前线程->MessageQueue->Message->Handler->Activity
因此当我们退出 Activity 的时候，由于消息需要在 10 分钟后在执行，因此会一直持有 Activity，从而导致了 Activity 的内存泄漏
通过上面分析我们知道了内存泄漏的原因就是持有了 Activity 的引用，那我们是不是会想，切断这条引用，那么如果我们需要用到 Activity 相关的属性和方法采用弱引用的方式不就可以了么？我们实际操作一下，把 Handler 写成一个静态内部类

    public class MainActivity extends AppCompatActivity {

        private final SafeHandler mSafeHandler = new SafeHandler(this);

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            //发送一个延迟消息，10分钟后在执行
            mSafeHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
        }

        //静态内部类并持有Activity的弱引用
        private static class SafeHandler extends Handler{

            private final WeakReference<MainActivity> mWeakReference;

            public SafeHandler(MainActivity activity){
                mWeakReference = new WeakReference<>(activity);
            }

            @Override
            public void handleMessage(@NonNull Message msg) {
                MainActivity mMainActivity = mWeakReference.get();
                if(mMainActivity != null){
                    //do something
                }
            }
        }
    }

上述代码
1、把 Handler 定义成了一个静态内部类，并持有当前 Activity 的弱引用，弱引用会在 Java 虚拟机发生 gc 的时候把对象给回收掉
经过上述改造，我们解决了 Activity 的内存泄漏，此时的引用链关系为:
当前线程->MessageQueue->Message->Handler
我们会发现 Message 还是会持有 Handler 的引用，从而导致 Handler 也会内存泄漏，所以我们应该在 Activity 销毁的时候，在他的生命周期方法里，把 MessageQueue 中的 Message 都给移除掉，因此最终就变成了这样：

    public class MainActivity extends AppCompatActivity {

        private final SafeHandler mSafeHandler = new SafeHandler(this);

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            //发送一个延迟消息，10分钟后在执行
            mSafeHandler.sendEmptyMessageDelayed(0x001,10*60*1000);
        }

        @Override
        protected void onDestroy() {
            super.onDestroy();
            mSafeHandler.removeCallbacksAndMessages(null);
        }

        //静态内部类并持有Activity的弱引用
        private static class SafeHandler extends Handler{

            private final WeakReference<MainActivity> mWeakReference;

            public SafeHandler(MainActivity activity){
                mWeakReference = new WeakReference<>(activity);
            }

            @Override
            public void handleMessage(@NonNull Message msg) {
                MainActivity mMainActivity = mWeakReference.get();
                if(mMainActivity != null){
                    //do something
                }
            }
        }
    }

因此当 Activity 销毁后，引用链关系为:
当前线程->MessageQueue
而当前线程和 MessageQueue 的生命周期和应用生命周期是一样长的，因此也就不存在内存泄漏了，完美。
所以解决 Handler 内存泄漏最好的方式就是：将 Handler 定义成静态内部类，内部持有 Activity 的弱引用，并在 Activity 销毁的时候移除所有消息

## 9、线程维护的 Looper，在消息队列无消息时的处理方案是什么？有什么用？

答：当消息队列无消息时，Looper 会阻塞当前线程，释放 cpu 资源，提高 App 性能
我们知道 Looper 的 loop 方法中有个死循环一直在读取 MessageQueue 中的消息，其实是调用了 MessageQueue 中的 next 方法，这个方法会在无消息时，调用 Linux 的 epoll 机制，使得线程进入阻塞状态，当有新消息到来时，就会将它唤醒，next 方法里会判断当前消息是否是延迟消息，如果是则阻塞线程，如果不是，则会返回这条消息并将其从优先级队列中给移除

## 10、MessageQueue 什么情况下会被唤醒？

答：需要分情况
1、发送消息过来，此时 MessageQueue 中无消息或者当前发送过来的消息携带的 when 为 0 或者有延迟执行的消息，那么需要唤醒
2、当遇到同步屏障且当前发送过来的消息为异步消息，判断该异步消息是否插入在所有异步消息的队首，如果是则需要唤醒，如果不是，则不唤醒

## 11、线程什么情况下会被阻塞？

答：分情况
1、当 MessageQueue 中没有消息的时候，这个时候会无限阻塞，
2、当前 MessageQueue 中全部是延迟消息，阻塞时间为(当前延迟消息时间 - 当前时间)，如果这个阻塞时间超过来 Integer 类型的最大值，则取 Integer 类型的最大值

## 12、我们可以使用多个 Handler 往消息队列中添加数据，那么可能存在发消息的 Handler 存在不同的线程，那么 Handler 是如何保证 MessageQueue 并发访问安全的呢？

答：循环加锁，配合阻塞唤醒机制
我们可以发现 MessageQueue 其实是“生产者-消费者”模型，Handler 不断地放入消息，Looper 不断地取出，这就涉及到死锁问题。如果 Looper 拿到锁，但是队列中没有消息，就会一直等待，而 Handler 需要把消息放进去，锁却被 Looper 拿着无法入队，这就造成了死锁。Handler 机制的解决方法是循环加锁。在 MessageQueue 的 next 方法中：

    Message next() {
    ...
        for (;;) {
    ...
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                ...
            }
        }
    }

我们可以看到他的等待是在锁外的，当队列中没有消息的时候，他会先释放锁，再进行等待，直到被唤醒。这样就不会造成死锁问题了。
那在入队的时候会不会因为队列已经满了然后一边在等待消息处理一边拿着锁呢？这一点不同的是 MessageQueue 的消息没有上限，或者说他的上限就是 JVM 给程序分配的内存，如果超出内存会抛出异常，但一般情况下是不会的。

## 13、Handler 是如何进行线程切换的呢？

答：使用不同线程的 Looper 处理消息
我们通常处理消息是在 Handler 的 handleMessage 方法中，那么这个方法是在哪里回调的呢？看下面这段代码

    public static void loop() {
        //开启死循环读取消息
        for (;;) {
            // 调用Message对应的Handler处理消息
            msg.target.dispatchMessage(msg);
        }
    }

上述代码中 msg.target 其实就是我们发送消息的 Handler，因此他会回调 Handler 的 dispatchMessage 方法，而 dispatchMessage 这个方法我们在上一篇中重点分析过，其中有一部分逻辑就是会回调到 Handler 的 handleMessage 方法，我们还可以发现，Handler 的 handleMessage 方法所在的线程是由 Looper 的 loop 方法决定的。平时我们使用的时候，是从异步线程发送消息到 Handler，而这个 Handler 的 handleMessage() 方法是在主线程调用的，因为 Looper 是在主线程创建的，所以消息就从异步线程切换到了主线程。

## 14、我们在使用 Message 的时候，应该如何去创建它？

答：Android 给 Message 设计了回收机制，官方建议是通过 Message.obtain 方法来获取，而不是直接 new 一个新的对象，所以我们在使用的时候应尽量复用 Message ，减少内存消耗，方式有二：
1、调用 Message 的一系列静态重载方法 Message.obtain 获取
2、通过 Handler 的公有方法 handler.obtainMessage，实际上 handler.obtainMessage 内部调用的也是 Message.obtain 的重载方法

## 15、Handler 里面藏着的 CallBack 能做什么？

答: 利用此 CallBack 拦截 Handler 的消息处理
在上一篇中我们分析到，dispatchMessage 方法的处理步骤:
1、首先，检查 Message 的 callback 是否为 null，不为 null 就通过 handleCallBack 来处理消息，Message 的 callback 是一个 Runnable 对象，实际上就是 Handler 的 post 系列方法所传递的 Runnable 参数
2、其次，检查 Handler 里面藏着的 CallBack 是否为 null，不为 null 就调用 mCallback 的 handleMessage 方法来处理消息，并判断其返回值：为 true，那么 Handler 的 handleMessage(msg) 方法就不会被调用了；为 false，那么就意味着一个消息可以同时被 Callback 以及 Handler 处理。
3、最后，调用 Handler 的 handleMessage 方法来处理消息
通过上面分析我们知道 Handler 处理消息的顺序是：Message 的 Callback > Handler 的 Callback > Handler 的 handleMessage 方法
使用场景: Hook ActivityThread.mH ， 在 ActivityThread 中有个成员变量 mH ，它是个 Handler，又是个极其重要的类，几乎所有的插件化框架都使用了这个方法。

## 16、Handler 阻塞唤醒机制是怎么一回事？

答： Handler 的阻塞唤醒机制是基于 Linux 的阻塞唤醒机制。
这个机制也是类似于 handler 机制的模式。在本地创建一个文件描述符，然后需要等待的一方则监听这个文件描述符，唤醒的一方只需要修改这个文件，那么等待的一方就会收到文件从而打破唤醒。和 Looper 监听 MessageQueue，Handler 添加 message 是比较类似的。具体的 Linux 层知识读者可通过这篇文章详细了解（传送门）

## 17、什么是 Handler 的同步屏障？

答: 同步屏障是一种使得异步消息可以被更快处理的机制

## 18、能不能让一个 Message 被加急处理？

答：可以，添加加同步屏障，并发送异步消息

## 19、什么是 IdleHandler？

答: IdleHandler 是 MessageQueue 中一个静态函数型接口，它在主线程执行完所有的 View 事务后，回调一些额外的操作，且不会阻塞主线程

> 参考资料：https://juejin.cn/post/6932608660354891790/
