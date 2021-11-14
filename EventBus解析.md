# EventBus 解析

## EventBus 的基本使用

**1-定义要传递的事件实体**

    public class MsgEvent { }

**2-注册和解注册你的订阅者**

    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }

**3-订阅者：声明并注解订阅方法**

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(MsgEvent event) {
        Log.d("OK");
    }

**4-发送事件**

    EventBus.getDefault().post(new MsgEvent());

## EventBus 3.x 与 2.x 的使用区别

**1-定义要传递的事件实体**

    public class MsgEvent { }

**2-注册和解注册你的订阅者**

    //3.0版本的注册
    EventBus.getDefault().register(this);

    //2.x版本的注册
    EventBus.getDefault().register(this);
    EventBus.getDefault().register(this, 100);
    EventBus.getDefault().registerSticky(this, 100);
    EventBus.getDefault().registerSticky(this);

    // 解除注册，2.x 与 3.x 相同
    EventBus.getDefault().unregister(this);

**3-订阅者：声明并注解订阅方法**

    //3.0版本
    @Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true, priority = 100)
    public void test(String str) {

    }

    //2.x版本
    public void onEvent(String str) {

    }
    public void onEventMainThread(String str) {

    }
    public void onEventBackgroundThread(String str) {

    }

**4-发送事件**

    // 2.x 与 3.x 相同
    EventBus.getDefault().post("str");
    EventBus.getDefault().postSticky("str");

## EventBus 3.x 新特性的使用

**EventBusAnnotationProcessor**

EventBus2 在运行期间使用 Java 反射机制遍历获取方法，反射会有一定的性能损耗；从 EventBus3 以后使用编译时注解，编译时利用 EventBusAnnotationProcessor 注解处理器获取 @Subscribe 所包含的信息，生成索引类来保存订阅者以及订阅的相关性信息，使其能在 EventBus.register() 方法调用之前就能知道相关订阅事件的方法。

**EventBusAnnotationProcessor 的使用**

要使用 EventBus 3 的这个新特性,需要以下几步:

添加依赖：

    compile 'org.greenrobot:eventbus:3.0.0'

由于注解依赖 android-apt-plugin,故需要在项目的 gradle 的 dependencies 中引入 apt

    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'

在 app module 的 build.gradle 中应用 apt 插件,并设置 apt 生成的索引的包名和类名，如果不设置的话在编译时会报错

    apply plugin: 'com.neenbedankt.android-apt'

    apt {
        arguments {
            eventBusIndex "com.demo.eventbusannotationsample.MyEventBusIndex"
        }
    }

最后,需要在 app module 的 dependencies 中引入 EventBusAnnotationProcessor

    apt 'org.greenrobot:eventbus-annotation-processor:3.0.1'

完成以上几步后,重新编译一次,即可在 app/build/generated/source/apt/debug/ 下看到生成的 MyEventBusIndex 类

重新编译之后,在第一次使用 EventBus 之前(如 Application 或 SplashActivity 中),添加如下代码,以使 Index 生效:

    EventBus eventBus=EventBus.builder().addIndex(new MyEventBusIndex()).build();

需要注意的是,如果不利用 EventBusAnnotationProcessor,则 EventBus 3 的解析速度反而会比之前版本更慢。

## EventBus 原理

**定义线程模型**

    public enum ThreadMode {
        /**
        * 子线程
        */
        BACKGROUND,

        /**
        * 主线程
        */
        MAIN
    }

**定义订阅事件方法的注解**

    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD})
    public @interface Subscribe {
        ThreadMode threadMode() default ThreadMode.MAIN;
    }

**定义订阅方法**

    public class SubscriberMethod {

        // 订阅方法名称
        private Method method;
        // 消息事件类型class
        private Class<?> eventType;
        // 订阅方法运行线程
        private ThreadMode threadMode;

        public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode) {
            this.method = method;
            this.eventType = eventType;
            this.threadMode = threadMode;
        }

        public Method getMethod() {
            return method;
        }

        public Class<?> getEventType() {
            return eventType;
        }

        public ThreadMode getThreadMode() {
            return threadMode;
        }
    }

**自定义事件总线**

    public class MyEventBus {

        private static volatile MyEventBus instance;

        // Object 对应 Activity 或者任意类，List<SubscriberMethod> 对应类中使用 Subscriber 注解的方法
        private Map<Object, List<SubscriberMethod>> cacheMap;

        // 主线程的 Handler
        private Handler mHandler;

        private MyEventBus() {
            cacheMap = new HashMap<>();
            mHandler = new Handler(Looper.getMainLooper());
        }

        // 单例
        public static MyEventBus getDefault() {
            if (instance == null) {
                synchronized (MyEventBus.class) {
                    if (instance == null) {
                        instance = new MyEventBus();
                    }
                }
            }
            return instance;
        }

        /**
        * 注册 MyEventBus
        *
        * @param obj 包含订阅事件的类
        */
        public void register(Object obj) {
            // 将包含订阅事件的类，和订阅事件绑定到一起，保存到 Map 中
            List<SubscriberMethod> list = cacheMap.get(obj);
            if (list == null) {
                list = getSubscriberMethodReflection(obj);
                if (list.size() > 0) {
                    cacheMap.put(obj, list);
                }
            }
        }

        /**
        * 反射获取类中的 Subscriber 注解方法
        *
        * @param obj 包含订阅事件的类
        * @return 类中的订阅事件集合
        */
        private List<SubscriberMethod> getSubscriberMethodReflection(Object obj) {
            List<SubscriberMethod> list = new ArrayList<>();
            Class<?> clazz = obj.getClass();

            while (clazz != null) {
                String name = clazz.getName();
                if (name.startsWith("java.")
                        || name.startsWith("javax.")
                        || name.startsWith("android.")
                        || name.startsWith("androidx.")) {
                    // 系统中的类不存在 Subscribe 注解的方法，直接跳出循环
                    break;
                }
                Method[] methods = clazz.getDeclaredMethods();
                for (Method method : methods) {
                    Subscribe subscribe = method.getAnnotation(Subscribe.class);
                    if (subscribe == null) {
                        // 不是 Subscribe 注解的方法
                        continue;
                    }
                    Class<?>[] types = method.getParameterTypes();
                    if (types.length != 1) {
                        // 方法参数必须有且仅为1个
                        throw new RuntimeException("方法参数不唯一");
                    }
                    // 将 Subscribe 注解的方法加入到 List 中
                    ThreadMode threadMode = subscribe.threadMode();
                    SubscriberMethod subscriberMethod = new SubscriberMethod(method, types[0], threadMode);
                    list.add(subscriberMethod);
                }
                // 遍历父类中的 Subscribe 注解方法
                clazz = clazz.getSuperclass();
            }
            return list;
        }

        /**
        * post 方法中的逻辑，是调用某个类中使用 Subscriber 注解的方法；
        * 是运行时才执行，无法使用APT优化，只能使用反射
        */
        public void post(final Object msgEvent) {
            Set<Object> set = cacheMap.keySet();
            Iterator<Object> iterator = set.iterator();
            while (iterator.hasNext()) {
                // 获取订阅事件的类
                final Object obj = iterator.next();
                List<SubscriberMethod> list = cacheMap.get(obj);
                for (final SubscriberMethod subscriberMethod : list) {
                    // 判断消息类型是否与订阅方法中的类型相同
                    if (subscriberMethod.getEventType().isAssignableFrom(msgEvent.getClass())) {
                        // 线程切换；判断Subscriber注解方法要在哪个线程执行
                        switch (subscriberMethod.getThreadMode()) {
                            // Subscriber注解方法在主线程执行
                            case MAIN:
                                // 执行 post 时也在主线程，直接调用 subscriberMethod 方法
                                if (Looper.myLooper() == Looper.getMainLooper()) {
                                    invoke(subscriberMethod, obj, msgEvent);
                                } else {
                                    // 执行 post 时在子线程，用主线程 Handler 执行 subscriberMethod 方法
                                    mHandler.post(new Runnable() {
                                        @Override
                                        public void run() {
                                            invoke(subscriberMethod, obj, msgEvent);
                                        }
                                    });
                                }
                                break;
                            // Subscriber注解方法在子线程执行
                            case BACKGROUND:
                                // 执行 post 时在主线程
                                if (Looper.myLooper() == Looper.getMainLooper()) {
                                    // TODO 原版EventBus中使用 HandlerPoster.enqueue 进行消息发送和线程切换
                                    // TODO 使用了 PendingPostQueue 队列来处理消息
                                } else {
                                    // 执行 post 时也在子线程，直接调用 subscriberMethod 方法
                                    invoke(subscriberMethod, obj, msgEvent);
                                }
                                break;
                            default:
                                break;
                        }
                    }
                }
            }
        }

        /**
        * 反射调用类中的 Subscriber 注解方法
        *
        * @param subscriberMethod Subscriber 注解方法
        * @param obj              订阅事件的类
        * @param msgEvent         订阅事件实例
        */
        private void invoke(SubscriberMethod subscriberMethod, Object obj, Object msgEvent) {
            Method method = subscriberMethod.getMethod();
            method.setAccessible(true);
            try {
                method.invoke(obj, msgEvent);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        }
    }

## EventBus 的 APT 优化原理

**MyEventBusIndex**

    /**
    * 模拟APT优化
    * 不再使用反射获取类中的 Subscriber 注解方法
    * 而是代码直接获取类中的 Subscriber 注解方法
    */
    public class MyEventBusIndex {

        private static final Map<Object, List<SubscriberMethod>> cacheMap = new HashMap<>();

        static {
            cacheMap.put(MainActivity.class.getSimpleName(), getSubscriberMethodMain());
            cacheMap.put(Main2Activity.class.getSimpleName(), getSubscriberMethodMain2());
        }

        /**
        * 返回指定类中的 Subscribe 注解方法集合
        */
        public static List<SubscriberMethod> getSubscriberMethods(Object obj) {
            return cacheMap.get(obj.getClass().getSimpleName());
        }

        /**
        * 获取 MainActivity 中的 Subscribe 注解方法
        */
        private static List<SubscriberMethod> getSubscriberMethodMain() {
            List<SubscriberMethod> methodList = new ArrayList<>();
            try {
                // 将已知的类和方法名添加到 SubscriberMethod 中
                // 类中有多个注解方法，则 methodList 添加多个 SubscriberMethod
                methodList.add(new SubscriberMethod(
                        MainActivity.class.getMethod("getMsgEvent", MsgEvent.class),
                        MsgEvent.class, ThreadMode.MAIN));
            } catch (NoSuchMethodException | SecurityException e) {
                e.printStackTrace();
            }
            return methodList;
        }

        /**
        * 获取 Main2Activity 中的 Subscribe 注解方法
        */
        private static List<SubscriberMethod> getSubscriberMethodMain2() {
            List<SubscriberMethod> methodList = new ArrayList<>();
            try {
                methodList.add(new SubscriberMethod(
                        Main2Activity.class.getMethod("getMsgEvent", MsgEvent.class),
                        MsgEvent.class, ThreadMode.MAIN));
            } catch (NoSuchMethodException | SecurityException e) {
                e.printStackTrace();
            }
            return methodList;
        }
    }

**注册 MyEventBus 时不再使用反射**

    /**
     * 注册 MyEventBus
     *
     * @param obj 包含订阅事件的类
     */
    public void register(Object obj) {
        // 将包含订阅事件的类，和订阅事件绑定到一起，保存到 Map 中
        List<SubscriberMethod> list = cacheMap.get(obj);
        if (list == null) {
            // 通过 MyEventBusIndex 直接获取 Subscriber 注解方法
            list = MyEventBusIndex.getSubscriberMethods(obj);
            if (list.size() > 0) {
                cacheMap.put(obj, list);
            }
        }
    }

参考资料：

[EventBus 3.0 使用详解](https://www.jianshu.com/p/f9ae5691e1bb)

[EventBus 3.0 源码分析](https://www.jianshu.com/p/f057c460c77e)

[EventBus 3.0 高效使用及源码解析](https://zhuanlan.zhihu.com/p/22493328)

[解析 EventBusAnnotationProcessor](https://www.jianshu.com/p/6c6231b094e2)

[深入理解 EventBus 源码](https://juejin.cn/post/6844904082747080717)

[手撸 EventBus](https://juejin.cn/post/6862611500343754766)

[EventBus 从入门到装逼](https://blog.csdn.net/u014702653/article/details/100087264)
