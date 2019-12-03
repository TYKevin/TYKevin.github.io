---
title: 重构登录管理 - AOP 思想在 Android 中的应用 
date: 2019-12-03 22:39:39
tags:
    - android 进阶
    - android 架构
---
## 什么是AOP？
首先我们需要来了解下，什么是 AOP？AOP 全名是 Aspect Oriented Programming, 即面向切面编程。我认为 AOP 是在 OOP 思想上的延续。我们可以通过 利用 AOP，对很多重复的业务在横向切掉并抽离出来，例如日志打印，埋点统计，以及我们今天的例子登录管理等等。AOP 在 Java 后端中非常普及了，今天我们从一个例子由浅入深，来看看 AOP 在 Android 中可以如何应用。

首先我们从例子入手：我们现在有个需求，需要在数据插入数据库之前都对数据进行一次保存操作，如果在没有接触 AOP 之前，我们是如何操作呢？这里我么来简单的举个例子:

## 运行时织入：通过动态代理实现 AOP
假设我们有个数据库操作接口 `DBOperation`:
```java
public interface DBOperation {
    void insert();
    void delete();
    void update();
    void save();
}
```

然后有个 Activity, 有个按钮模拟点击后，操作数据库，如果要实现需求，我们必须在每次操作前都调用一遍 `db.save()`  方法： 
```java
public class MainActivity extends AppCompatActivity implements DBOperation, View.OnClickListener {

    private static final String TAG = "MainActivity";
    private DBOperation db;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        db = this;

        findViewById(R.id.btn_operationDb).setOnClickListener(this);
    }

    public void databaseOperating(View view) {
        db.save();
        db.insert();
    }

    @Override
    public void insert() {
        Log.d(TAG, "数据库操作：insert()");
    }

    @Override
    public void delete() {
        Log.d(TAG, "数据库操作：delete()");
    }

    @Override
    public void update() {
        Log.d(TAG, "数据库操作：update()");
    }

    @Override
    public void save() {
        Log.d(TAG, "数据库操作：save()");
    }

    @Override
    public void onClick(View v) {
        int viewId = v.getId();
        switch (viewId) {
            case R.id.btn_operationDb:
                databaseOperating(v);
                break;
        }
    }
}
```

如果像例子中只有一次还好，如果每次要多次调用数据库的情况，则会造成代码的冗余以及有可能造成的遗忘。所以我们有没有办法在我们调用 insert/delete/update 的方法前就自动的调用一下 `save()` 方法呢？ 当然是有的，请看下面：
```java

public class MainActivity extends AppCompatActivity implements DBOperation, View.OnClickListener {

    private static final String TAG = "MainActivity";
    private DBOperation db;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.btn_operationDb).setOnClickListener(this);

        db = (DBOperation) Proxy.newProxyInstance(DBOperation.class.getClassLoader(), new Class[]{DBOperation.class}, new DBHandler(this));
    }

    /**
     * 操作数据库
     * @param view
     */
    public void databaseOperating(View view) {
        db.insert();
    }

    @Override
    public void insert() {
        Log.d(TAG, "数据库操作：insert()");
    }

    @Override
    public void delete() {
        Log.d(TAG, "数据库操作：delete()");
    }

    @Override
    public void update() {
        Log.d(TAG, "数据库操作：update()");
    }

    @Override
    public void save() {
        Log.d(TAG, "数据库操作：save()");
    }

    @Override
    public void onClick(View v) {
        int viewId = v.getId();
        switch (viewId) {
            case R.id.btn_operationDb:
                databaseOperating(v);
                break;
        }
    }

    /**
     * 动态代理
     */
    private class DBHandler implements InvocationHandler {
        private DBOperation db;

        public DBHandler(DBOperation dbOperation) {
            this.db = dbOperation;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if(db != null) {
                if (!"save".equals(method.getName())) {
                    Log.d(TAG, "操作数据库之前，开始备份。。。");
                    save();
                    Log.d(TAG, "数据库备份完成。。。");
                }
                return method.invoke(db, args);
            }

            return null;
        }
    }
}
```

上面的代码中使用到了 Java 的动态代理，在每次调用 `DBOperation` 接口中的方法时，都会去自动调用一遍 `save()` 方法，如此一来，我们在代码中只要调用到了 insert/update/delete 方法，则自动就会调用 `save()` 方法，这就是一个简单的 AOP 的实现，将 save 的操作 "横切" 出来，做成一个整体， 在 **运行时织入** 代码。

## 编译时织入：通过 AspectJ 实现集中式登录框架
那么除了上面运行时织入代码的方法，我们还能将"切出来的代码" 在编译时织入，在类加载时织入。下面我们着重介绍下今天的主角，AspectJ 框架。这是一个功能非常强大且好用的 AOP 框架，AspectJ 可以替代 javac 完成编译工作，并支持在编译期生成 class 文件时将代码织入对应的切入点。下面我们还是从实际的例子出发，来看看 AspectJ 在 Android 中如何应用的。

我们都非常清楚 APP 登录判断是个最常见不过的操作了，没有使用 AOP 之前，我们每次需要用户登录权限的操作前都要先去判断一下是否有登录，然后如果没有登录则进行跳转，我们有没有办法在一个地方对这个判断进行集中处理呢？

首先，我们需要搞清楚几个概念：
1. 连接点(Joint point)：所有的目标方法都是连接点
2. 切入点(PointCut)：通过使用特定的表达式 过滤出来的 需要切入 Advice 的切入点，即所有连接点的集合。
3. 通知(Advice)：Advice 向代码中植入 的实现方法（Before（前置）、After（后置） 、 Around（环绕））。
4. 切面(Aspect): 由 PointCut  和 Advice 组成一个切面。

光看这些概念性的东西肯定是不知所以然，下面我们来一步步的通过代码使用 AspectJ 来实现：
#### 添加插件 
在 Application 的 `build.gradle` 中添加插件 
```groovy
buildscript {
    dependencies {
            classpath 'com.android.tools.build:gradle:3.2.1'
    
            // 添加 aspectj 对应的插件
            classpath 'org.aspectj:aspectjtools:1.8.9'
            classpath 'org.aspectj:aspectjweaver:1.8.9'
    }
}
```

#### APP 编译设置
在 app 的 `build.gradle` 中添加引用包，并设置编译时使用的插件，以及添加编译支持代码。
```groovy
apply plugin: 'com.android.application'

buildscript { // 编译时用Aspect专门的编译器，不再使用传统的javac
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.8.9'
        classpath 'org.aspectj:aspectjweaver:1.8.9'
    }
}

android {
    compileSdkVersion 29
    buildToolsVersion "29.0.2"
    defaultConfig {
        applicationId "com.example.myapplication"
        minSdkVersion 23
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.1.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    implementation 'com.google.android.material:material:1.0.0'
    implementation 'androidx.annotation:annotation:1.0.2'
    implementation 'androidx.lifecycle:lifecycle-extensions:2.0.0'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test.ext:junit:1.1.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'

    implementation 'org.aspectj:aspectjrt:1.8.13'
}

// 编译支持
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

final def log = project.logger
final def variants = project.android.applicationVariants

variants.all { variant ->
    if (!variant.buildType.isDebuggable()) {
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.8",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler);
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break;
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```
以上准备工作做好了，下面进入正餐。

#### 新增连接点标示
此处的连接点标示我们使用自定义注解来进行标注
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginCheck {
}
```

#### 添加连接点
我们将用上一步的注解，在我们需要进行登录确认的方法前进行标示，例如如下方法会跳转我的专区页面，但是需要用户登录后才能跳转，我们则在这个方法上加上我们之前的注解：
```java
    /**
     * 需要登录后操作，未登陆需要跳转登录界面
     * @param view
     */
    @LoginCheck
    public void area(View view) {
        Log.d(TAG, "跳转到我的专区界面");
        startActivity(new Intent(this, MainActivity.class));
    }
```
那么如何让这个注解起作用呢？重点就在下一步，添加切面。

#### 添加切面
我们新建一个类作为切面，其中包含了 切入点（PointCut）和 环绕通知（Around）：
```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

/**
 * 通过 @Aspect 注解表明此类为一个切面
 */
@Aspect
public class LoginCheckAspect {
    /**
     * 定义切入点，通过 execution 语法定义哪些方法为切入点
     */
    @Pointcut("execution(@com.example.myapplication.example_login.aop.LoginCheck * *(..))")
    public void methondPointCut() {
    }


    @Around("methondPointCut()")
    public Object joinPoint(ProceedingJoinPoint joinPoint) throws Throwable {
        Context context = (Context) joinPoint.getThis();

        String loginedUser = SharedPreferencesUtil.getLoginedUser();
        // 如果记录用户名不为空，则说明已登录
        boolean isLogin = !TextUtils.isEmpty(loginedUser);
        // 判断是否登录
        if (!isLogin) {
            // 未登录直接跳转到登录页面
            Toast.makeText(context, "请先登录！", Toast.LENGTH_SHORT).show();
            Intent intent = new Intent(context, LoginActivity.class);
            context.startActivity(intent);
            return null;
        }

        // 如果登录了，则执行原方法
        return joinPoint.proceed();
    }
}
```
以上代码通过 @Pointcut（切入点） 和 @Around（通知） 形成了 一个 @Aspect（切面）。 这里需要注意，定义切入点时使用 execution 需要注意语法正确，否则会导致无法正确切入。

这样就使用 AspectJ 完成了一整套 集中式登录架构，接下来只需要在需要登录验证的方法前加上注解即可，相当于将登录验证从一个个纵向流程中横向给切了出来进行单独处理，可以减少了我们需要重复进行登录判断的冗余代码。  

我们还可以想一想，如果我们要用户的点击行为统计如何我们应当如何写呢？思路也是一样哈，定义连接点标示（自定义注解），新增切面类并在其中定义切入点和通知，在通知中做抽取的业务逻辑即可，在下面提供的源码地址中我也有实现哈，以供参考。

这就是 AOP 在 Android 中的应用，对于这个方法论你学会了吗？除此之外，你还有什么妙用吗？不妨实践一下。

源码链接：https://github.com/TYKevin/DemoAopInAndroid






 