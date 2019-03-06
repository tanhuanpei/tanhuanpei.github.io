---
title: Android进阶之Aspectj in Android入门
date: 2019-01-14 15:56:40
tags: AspectJ 进阶
---


#### 背景

AOP，Aspect-oriented programming，面向切面编程，是一种可以通过预编译方式和运行期动态代理实现在不修改源码的情况下给程序动态统一添加功能的技术。主要用途有**日志记录，行为统计，安全控制，事务处理，异常处理，系统统一的认证、权限管理**等 。

常见实现AOP编程库

- AspectJ：和Java语言无缝衔接的面向切面的编程的扩展工具（可用于Android）。
- Javassit for Android：一个移植到Android平台的非常知名的操纵字节码的java库。
- DexMaker：用于在Dalvik VM编译时或运行时生成代码的基于java语言的一套API。
- ASMDEX：一个字节码操作库（ASM），但它处理Android可执行文件（DEX字节码）。

要学习AspectJ，先理解其中一些概念。

`Join Point `：连接点，程序中可切入的点，例如方法调用时、读取某个变量时。

`Pointcut` ：切入点，代码注入的位置，其实就是有条件限定的 Join Point，例如只在特定方法中注入代码。

`Advice `：在切入点注入的代码，一般有 before、after、around 三种类型。

`Target Object` ：被一个或多个 aspect 横切拦截操作的目标对象。

`Weaving `： 把 Advice 代码织入到目标对象的过程。

`Inter-type declarations` : 用来给一个类型声明额外的方法或属性。



#### 工程引入

集成AspectJ主要有两种方式：

1. 原始Gradle配置的方式：

   项目根目录的build.gradle添加配置

   ```groovy
   buildscript {
       ...
       dependencies {        
           classpath 'org.aspectj:aspectjtools:1.8.6'
           ...
       }
   }
   ```

   在app module的build.gradle文件中添加配置

   ```groovy
   apply plugin: 'com.android.application'
   
   android {
       ...
   }
   
   import org.aspectj.bridge.IMessage
   import org.aspectj.bridge.MessageHandler
   import org.aspectj.tools.ajc.Main
   
   final def log = project.logger
   final def variants = project.android.applicationVariants
   //在构建工程时，执行编织
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
   
   
   dependencies {
       ...
       compile 'org.aspectj:aspectjrt:1.8.6'
   }
   ```

   

   

2. 插件的方式 ：

   Github上的开源插件 -[AspectJX](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/)，一个基于AspectJ并在此基础上扩展出来可应用于Android开发平台的AOP框架，可作用于java源码，class文件及jar包，同时支持kotlin的应用，提供了AspectJ同样的功能。

   

   在项目根目录的build.gradle里依赖AspectJX

   ```groovy
    dependencies {
           classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.4'
    }
   ```

   

   在app项目的build.gradle里应用插件

   ```groovy
   apply plugin: 'android-aspectjx'
   //或者这样也可以
   apply plugin: 'com.hujiang.android-aspectjx'
   ```

   

   下面是使用 android-aspectjx 插件需要注意的点：

   1. android-aspectjx 插件是 使用在 application module 的插件，只能在编译 application module 的过程中织入代码。

   2. AspectJ 的原理是在编译期注入代码，所以**切面只能是项目代码、依赖的 jar 或 aar，不能注入 Android 平台 android.jar**。例如，可以在 support 包的 Fragment 中注入代码，但是无法在 Activity 中注入代码，只能注入项目的继承自 Activity 的 XXActivity。

   3. android-aspectjx 默认会遍历项目编译后所有的 .class 文件和依赖的第三方库去查找符合织入条件的切点，为了提升效率，可以加入过滤条件，具体见[ Android Aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/blob/master/README-zh.md) 的文档。

      

#### 简单Demo

新建`` FragmentAspect ``类，添加注解，实现在Fragment调用onResume和onPause之前打印log

```java
@Aspect
public class FragmentAspect {

    private static final String TAG = "FragmentAspect";

    @Pointcut("execution(void android.support.v4.app.Fragment.onResume()) && target(fragment)")
    public void resume(Fragment fragment) {
    }

    @Pointcut("execution(void android.support.v4.app.Fragment.onPause()) && target(fragment)")
    public void pause(Fragment fragment) {
    }

    @Before("resume(fragment)")
    public void beforeOnResume(Fragment fragment) {
        Log.d(TAG, fragment.getClass().getSimpleName() + " onResume");
    }

    @Before("pause(fragment)")
    public void beforeOnPause(Fragment fragment) {
        Log.d(TAG, fragment.getClass().getSimpleName() + " onPause");
    }
 }
```

编译成功后找到MainFragment.class，已经生成注入代码

```java
public class MainFragment extends Fragment {
    private MainViewModel mViewModel;

    public MainFragment() {
    }

    public static MainFragment newInstance() {
        return new MainFragment();
    }

    @Nullable
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(2131296284, container, false);
    }

    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        this.mViewModel = (MainViewModel)ViewModelProviders.of(this).get(MainViewModel.class);
    }

    public void onResume() {
        FragmentAspect.aspectOf().beforeOnResume(this);

        try {
            Thread.sleep(2000L);
        } catch (InterruptedException var2) {
            var2.printStackTrace();
        }

        super.onResume();
    }

    public void onPause() {
        FragmentAspect.aspectOf().beforeOnPause(this);
        super.onPause();
    }
}
```

#### 语法详解

Join Point表示连接点，即AOP可织入代码的点，下表列出了AspectJ的所有连接点：

| Join Point            | 说明                     |
| --------------------- | ------------------------ |
| Method call           | 方法被调用               |
| Method execution      | 方法执行                 |
| Constructor call      | 构造函数被调用           |
| Constructor execution | 构造函数执行             |
| Field get             | 读取属性                 |
| Field set             | 写入属性                 |
| Pre-initialization    | 与构造函数有关，很少用到 |
| Initialization        | 与构造函数有关，很少用到 |
| Static initialization | static 块初始化          |
| Handler               | 异常处理                 |
| Advice execution      | 所有 Advice 执行         |

具体语法：

- @Aspect

  AOP中的关键单位 - 切面，开发中一般将Pointcut和Advice放一个Aspect类中，在给予Aspect注解开发方式中只需要在类的头部加上@Aspect注解即可，@Aspect不可用于修饰接口。

- @Pointcut

  `语法结构`：@Pointcut("Pointcut syntax")

```java
@Pointcut("execution(void android.support.v4.app.Fragment.onResume()) && target(fragment)")
    public void resume(Fragment fragment) {
}
```

​	Pointcuts 是具体的切入点，可以确定具体织入代码的地方，基本的 Pointcuts 是和 Join Point 相对应的。

| Join Point            | Pointcuts syntax                      | 说明                                                         |
| --------------------- | :------------------------------------ | ------------------------------------------------------------ |
| Method call           | call(MethodPattern)                   | 方法被调用                                                   |
| Method execution      | execution(MethodPattern)              | 方法执行                                                     |
| Constructor call      | call(ConstructorPattern)              | 构造函数被调用                                               |
| Constructor execution | execution(ConstructorPattern)         | 构造函数执行                                                 |
| Field get             | get(FieldPattern)                     | 读取属性                                                     |
| Field set             | set(FieldPattern)                     | 写入属性                                                     |
| Pre-initialization    | initialization(ConstructorPattern)    | 对象预先初始化                                               |
| Initialization        | preinitialization(ConstructorPattern) | 对象初始化                                                   |
| Static initialization | staticinitialization(TypePattern)     | static块初始化                                               |
| Handler               | handler(TypePattern)                  | 异常处理                                                     |
| Advice execution      | adviceexcution()                      | 所有 Advice 执行                                             |
|                       | within(TypePattern)                   | 符合 TypePattern 的代码中的 Join Point                       |
|                       | withincode(MethodPattern)             | 符合 TypePattern 的代码中的 Join Point                       |
|                       | withincode(ConstructorPattern)        | 在某些构造函数中的 Join Point                                |
|                       | cflow(Pointcut)                       | Pointcut 选择出的切入点 P 的控制流中的所有 Join Point，包括 P 本身 |
|                       | cflowbelow(Pointcut)                  | Pointcut 选择出的切入点 P 的控制流中的所有 Join Point，不包括 P 本身 |
|                       | this(Type or Id)                      | Join Point 所属的 this 对象是否 instanceOf Type 或者 Id 的类型 |
|                       | target(Type or Id)                    | Join Point 所在的对象（例如 call 或 execution 操作符应用的对象）是否 instanceOf Type 或者 Id 的类型 |
|                       | args(Type or Id, …)                   | 方法或构造函数参数的类型                                     |
|                       | if(BooleanExpression)                 | 满足表达式的 Join Point，表达式只能使用静态属性、Pointcuts 或 Advice 暴露的参数、thisJoinPoint 对象 |

​	上面 Pointcuts 的语法中涉及到一些 Pattern，下面是这些 Pattern 的规则，`[]`里的内容是可选的：

| Pattern            | 规则                                                         |
| ------------------ | ------------------------------------------------------------ |
| MethodPattern      | [!] [@Annotation] [public,protected,private] [static] [final] [返回值类型] [类名.]方法名(参数类型列表) [throws 异常类型] |
| ConstructorPattern | [!] [@Annotation] [public,protected,private] [final] [类名.]new(参数类型列表) [throws 异常类型] |
| FieldPattern       | [!] [@Annotation] [public,protected,private] [static] [final] 属性类型 [类名.]属性名 |
| TypePattern        | 其他 Pattern 涉及到的类型规则也是一样，可以使用 ‘!’、’*‘、’..’、’+’，’!’ 表示取反，’*‘ 匹配除 . 外的所有字符串，’*’ 单独使用事表示匹配任意类型，’..’ 匹配任意字符串，’..’ 单独使用时表示匹配任意长度任意类型，’+’ 匹配其自身及子类，还有一个 ‘…’表示不定个数 |

​	Pointcut 表达式还可以 ！、&&、|| 来组合，!Pointcut 选取不符合 Pointcut 的 Join Point，Pointcut0 && Pointcut1 选取符合 Pointcut0 和 Pointcut1 的 Join Point，Pointcut0 || Pointcut1 选取符合 Pointcut0 或 Pointcut1 的 Join Point。

- @Advice

  `语法结构`：Advice("Pointcut定义的方法")  或者  Advice("Pointcut表达式")

  ```java
  @Before("resume(fragment)")
      public void beforeOnResume(Fragment fragment) {
          Log.d(TAG, fragment.getClass().getSimpleName() + " onResume");
  }
  ```

  Advice 是在切入点上织入的代码，在 AspectJ 中有五种类型：Before、After、AfterReturning、AfterThrowing、Around。
  

  | Advice          | 说明                                                         |
  | --------------- | ------------------------------------------------------------ |
  | @Before         | 在执行 Join Point 之前                                       |
  | @After          | 在执行 Join Point 之后，包括正常的 return 和 throw 异常      |
  | @AfterReturning | Join Point 为方法调用且正常 return 时，不指定返回类型时匹配所有类型 |
  | @AfterThrowing  | Join Point 为方法调用且抛出异常时，不指定异常类型时匹配所有类型 |
  | @Around         | 替代 Join Point 的代码，如果要执行原来代码的话，要使用 ProceedingJoinPoint.proceed() |

  

  Advice注解修改的方法必须是**``public``**,Before、After、AfterReturning、AfterThrowing 四种类型修饰的方法返回值也必须为 **``void``**，Around的目标因为是替换原来的Joint Point，所以他会有返回值，一般是**``Object``**。Advice 需要使用 `JoinPoint`、`JoinPointStaticPart`、`JoinPoint.EnclosingStaticPart`时，要在方法中声明为额外的参数，@Around 方法可以使用 `ProceedingJoinPoint`，用以调用 proceed() 方法。

  ```java
  @Around("execution(* com.example.tanhuanpei.learndemo..*.*(..))")
      public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
          long start = System.currentTimeMillis();
          Object proceed = joinPoint.proceed();
          long consume = System.currentTimeMillis() - start;
          Log.e(TAG, consume + "ms " + joinPoint.getSignature() + " main thread method");
          return proceed;
  }
  ```



至此，AspectJ在Android的应用入门介绍完毕。