# ButterKnife解析

## ButterKnife 基本使用

```text
    implementation 'com.jakewharton:butterknife:8.8.0'
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.8.0'
```

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.tv_main_show)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        textView.setText("Butter-Knife-Test");
    }
}
```

## ButterKnife 实现--反射

### BindView

```java
/**
 * 查找控件ID的注解
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface BindView {
    int value() default -1;
}
```

### InjectLayout

```java
/**
 * 给Activity注入布局文件的注解
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface InjectLayout {
    int value() default -1;
}
```

### OnClick

```java
/**
 * 给View设置监听事件的注解
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OnClick {
    int[] value();
}
```

### BindProcessor

```java
/**
 * 使用注解注入布局文件省去setContentView
 * 使用注解省去findViewById
 * 使用注解省去setOnClickListener
 * 参考资料：
 * https://blog.csdn.net/qq_20521573/article/details/82054088
 */
public class BindProcessor {

    /**
     * 供Activity调用，用于绑定注解逻辑
     *
     * @param activity
     */
    public static void bind(Activity activity) {
        injectLayout(activity);
        bindView(activity);
        onClick(activity);
    }

    private static void injectLayout(Activity activity) {
        Class<?> activityCls = activity.getClass();
        if (activityCls.isAnnotationPresent(InjectLayout.class)) {
            InjectLayout layout = activityCls.getAnnotation(InjectLayout.class);
            int layoutId = layout.value();
            try {
                Method setContentView = activityCls.getMethod("setContentView", int.class);
                setContentView.setAccessible(true);
                setContentView.invoke(activity, layoutId);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private static void bindView(Activity activity) {
        Class<?> activityCls = activity.getClass();
        Field[] fields = activityCls.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(BindView.class)) {
                BindView bindView = field.getAnnotation(BindView.class);
                int viewId = bindView.value();
                try {
                    Method findViewById = activityCls.getMethod("findViewById", int.class);
                    findViewById.setAccessible(true);
                    Object view = findViewById.invoke(activity, viewId);
                    field.setAccessible(true);
                    field.set(activity, view);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private static void onClick(final Activity activity) {
        Class<?> activityCls = activity.getClass();
        Method[] methods = activityCls.getMethods();
        for (final Method method : methods) {
            if (method.isAnnotationPresent(OnClick.class)) {
                OnClick onclick = method.getAnnotation(OnClick.class);
                int[] viewIdArray = onclick.value();
                for (int viewId : viewIdArray) {
                    final View view = activity.findViewById(viewId);
                    if (view == null) {
                        continue;
                    }
                    view.setOnClickListener(new View.OnClickListener() {
                        @Override
                        public void onClick(View v) {
                            try {
                                method.setAccessible(true);
                                method.invoke(activity, view);
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                        }
                    });
                }
            }
        }
    }
}
```

### 使用 MyButterKnife

```java
/**
 * 自定义butterknife
 * 通过注解和反射，来绑定view
 */
@InjectLayout(R.layout.activity_main)
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    @BindView(R.id.tv_main)
    TextView tv_main;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //setContentView(R.layout.activity_main);
        BindProcessor.bind(this);
        tv_main.setText("注解绑定View");
        Log.i(TAG, "MainActivity onCreate");
    }

    @OnClick({R.id.btn_main_click1, R.id.btn_main_click2})
    public void onClickView(View view) {
        int viewId = view.getId();
        if (viewId == R.id.btn_main_click1) {
            Toast.makeText(this, "Btn01", Toast.LENGTH_SHORT).show();
        } else if (viewId == R.id.btn_main_click2) {
            Toast.makeText(this, "Btn02", Toast.LENGTH_SHORT).show();
        } else {
            Log.i(TAG, "onClickView unknown");
        }
    }
}
```

## ButterKnife 实现--APT

使用 APT，需要使用 annotationProcessor 依赖 APT 编译自动生成Java代码
需要分别创建两个 module；annotation_compile 存放 APT 源码，annotation_lib 存放要使用的注解

```text
    annotationProcessor project(path: ':annotation_compile')
    implementation project(path: ':annotation_lib')
```

### annotation_lib

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface BindView {
    int value();
}
```

### annotation_compile

```java
/**
 * 注解处理器，用于生成Java代码
 */
@AutoService(Processor.class)
public class AnnotationCompiler extends AbstractProcessor {

    /**
     * 用来生成文件的对象
     */
    private Filer mFiler;

    /**
     * 支持的 Java 版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    /**
     * 当前APT处理的注解类型
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new HashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }

    /**
     * 获取用来生成文件的对象：Filer
     * 通过 Filer 生成源文件
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnvironment.getFiler();
    }

    /**
     * 注解处理逻辑
     * 生成的代码在 app/build/generated/ap_generated_sources
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

        // 获取所有的 BindView 注解元素；将所有的 BindView 注解都获取到 Set 集合
        Set<? extends Element> elementsAnnotatedWith = roundEnvironment.getElementsAnnotatedWith(BindView.class);

        // 对所有的 BindView 注解元素分类；相同的Activity中的 BindView 元素放在一起
        Map<String, List<VariableElement>> map = new HashMap<>();
        for (Element element : elementsAnnotatedWith) {
            VariableElement variableElement = (VariableElement) element;
            // 得到 VariableElement 所在的 activity 名称
            String activityName = variableElement.getEnclosingElement().getSimpleName().toString();
            // 用 List 保存所在 activity 的 VariableElement
            List<VariableElement> variableElementList = map.get(activityName);
            if (variableElementList == null) {
                variableElementList = new ArrayList<>();
                map.put(activityName, variableElementList);
            }
            variableElementList.add(variableElement);
        }

        // 对 Map 中的元素进行解析，拼接成Java文件
        if (map.size() > 0) {
            Writer writer = null;
            Iterator<String> iterator = map.keySet().iterator();
            while (iterator.hasNext()) {
                String activityName = iterator.next();
                List<VariableElement> variableElementList = map.get(activityName);
                // 获取包名
                TypeElement typeElement = (TypeElement) variableElementList.get(0).getEnclosingElement();
                String packageName = processingEnv.getElementUtils().getPackageOf(typeElement).toString();
                // 写入文件
                try {
                    // 创建文件
                    JavaFileObject sourceFile = mFiler.createSourceFile(
                            packageName + "." + activityName + "_ViewBinding");
                    // 创建输入流
                    writer = sourceFile.openWriter();
                    // 写入具体内容
                    writer.write("package " + packageName + ";\n");
                    writer.write("import " + packageName + ".IBinder;\n");
                    writer.write("public class " + activityName +
                            "_ViewBinding implements IBinder<"
                            + packageName + "." + activityName + ">{\n");
                    writer.write("@Override\n");
                    writer.write("public void bind("
                            + packageName + "." + activityName + " target){\n");
                    for (VariableElement element : variableElementList) {
                        // 获取被 BindView 注解的变量名称
                        String variableName = element.getSimpleName().toString();
                        // 获取 BindView 注解的 id
                        int id = element.getAnnotation(BindView.class).value();
                        // 获取被 BindView 注解的变量类型
                        TypeMirror typeMirror = element.asType();
                        writer.write("target." + variableName
                                + " = (" + typeMirror + ") target.findViewById(" + id + ");\n");
                    }
                    writer.write("\n}\n}");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    if (writer != null) {
                        try {
                            writer.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }

        return false;
    }
}
```

### app module

#### IBinder

annotation_compile 中生成的 xxx_ViewBinding 类实现了 IBinder

```java
/**
 * 用户绑定activity
 */
public interface IBinder<T> {
    void bind(T target);
}
```

#### MyButterKnife

```java
/**
 * 用于给用户绑定
 */
public class MyButterKnife {
    public static void bind(Activity activity) {
        String viewBindingName = activity.getClass().getName() + "_ViewBinding";
        try {
            Class<?> viewBindingClass = Class.forName(viewBindingName);
            // 执行 xxx_ViewBinding 类中的 bind 方法
            IBinder iBinder = (IBinder) viewBindingClass.newInstance();
            iBinder.bind(activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 使用 MyButterKnife

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.btn_main)
    Button btn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        MyButterKnife.bind(this);
        btn.setText("MyButterKnife");
    }
}
```
