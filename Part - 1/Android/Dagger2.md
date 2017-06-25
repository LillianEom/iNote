# Dagger2

**Dagger2 相较于 Dagger1 的优势是什么？**

更好的性能：相较于 Dagger1，它是通过 apt 动态生成代码来实现的.它使用的预编译期间生成代码来完成依赖注入，而不是用的反射。反射是在程序运行时加载类来进行处理所以会比较耗时，而手机硬件资源有限，所以相对来说会对性能产生一定的影响。

容易跟踪调试：因为 dagger2 是使用生成代码来实现完整依赖注入，所以完全可以在相关代码处下断点进行运行调试。

**使用依赖注入的最大好处是什么？**

模块间解耦！ 就拿当前 Android 非常流行的开发模式MVP来说，使用Dagger2可以将MVP中的V 层与P层进一步解耦，这样便可以提高代码的健壮性和可维护性。

## 使用

### 注解

Dagger2 通过注解来生成代码，定义不同的角色，主要的注解如下：

**@Module:**

 Module 类里面的方法专门**提供依赖**，所以我们定义一个类，用`@Module `注解，这样 Dagger 在构造类的实例的时候，就知道从哪里去找到需要的依赖。

**@Provides: **

在 Module 中，我们定义的**方法**是用这个注解，以此来告诉 Dagger 我们想要构造对象并提供这些依赖。

**@Inject:**

有两个作用，一是用来标记**需要依赖**的变量，以此告诉 Dagger2 为它提供依赖；二是用来**标记构造函数**，Dagger2 通过`@Inject  `注解可以在需要这个类实例的时候来找到这个构造函数并把相关实例构造出来，以此来为被 `@Inject` 标记了的变量提供依赖；

**@Component: **

Component 用于标注接口，从根本上来说就是一个**注入器**，也可以说是 `@Inject `和 `@Module` 的桥梁，它的主要作用就是连接这两个部分。将 Module 中产生的依赖对象自动注入到需要依赖实例的 Container 中。

划分粒度小的 Component，那划分的规则这样的：

- 要有一个全局的 Component (可以叫 ApplicationComponent )，负责管理整个 app 的全局类实例（全局类实例整个 app 都要用到的类的实例，这些类基本都是单例的）
- 每个页面对应一个 Component，比如一个 Activity 页面定义一个 Component，一个 Fragment 定义一个 Component。当然这不是必须的，有些页面之间的依赖的类是一样的，可以公用一个 Component。

**@Scope: **

Dagger2 可以通过自定义注解限定注解作用域，来管理每个对象实例的生命周期。我能可以通过 `@Scope` 自定义的注解来限定注解作用域，实现局部的单例。

**@Qualifier: **

当类的类型不足以鉴别一个依赖的时候，我们就可以使用这个注解标示。例如：在 Android 中，我们会需要不同类型的 context，所以我们就可以定义 qualifier 注解`@perApp`和`@perActivity`，这样当注入一个 context 的时候，我们就可以告诉 Dagger 我们想要哪种类型的 context。

**@Singleton：**`@Singleton` 其实就是一个通过 `@Scope` 定义的注解，我们一般通过它来实现全局单例。但实际上它并不能提前全局单例，是否能提供全局单例还要取决于对应的 Component 是否为一个全局对象。



我们提到 `@Inject` 和 `@Module` 都可以提供依赖，那如果我们即在构造函数上通过标记 `@Inject` 提供依赖，有通过 `@Module` 提供依赖 Dagger2会如何选择呢？具体规则如下：

- 步骤1：首先查找`@Module`标注的类中是否存在提供依赖的方法。
- 步骤2：若存在提供依赖的方法，查看该方法是否存在参数。
  - a：若存在参数，则按从步骤1开始依次初始化每个参数；
  - b：若不存在，则直接初始化该类实例，完成一次依赖注入。
- 步骤3：若不存在提供依赖的方法，则查找@Inject标注的构造函数，看构造函数是否存在参数。
  - a：若存在参数，则从步骤1开始依次初始化每一个参数
  - b：若不存在，则直接初始化该类实例，完成一次依赖注入。

依赖配置查看官方教程  https://github.com/google/dagger

### 结构

Dagger2 要实现一个完整的依赖注入，必不可少的元素有三种：Module，Component，Container。

![](http://upload-images.jianshu.io/upload_images/1825062-28a0c53895b72f12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Dagger2 主要分为三个模块:**

> 1. 依赖提供方 Module，负责提供依赖中所需要的对象，实际编码中类似于工厂类
> 2. 依赖需求方实例,它声明依赖对象，它在实际编码中对应业务类,例如 Activity，当你在 Activity 中需要某个对象时，你只要在其中声明就行，声明的方法在下面会讲到。
> 3. 依赖注入组件 Component，负责将对象注入到依赖需求方,它在实际编码中是一个接口,编译时 Dagger2 会自动为它生成一个实现类。

**Dagger2 的主要工作流程分为以下几步:**

> 1. 将依赖需求方实例传入给 Component 实现类
> 2. Component 实现类根据依赖需求方实例中依赖声明,来确定该实例需要依赖哪些对象
> 3. 确定依赖对象后,Component 会在与自己关联的 Module 类中查找有没有提供这些依赖对象的方法,有的话就将 Module 类中提供的对象设置到依赖需求方实例中

### 简单的例子

#### 实现Module

```java
@Module // 注明本类是Module
public class MyModule{
    @Provides  // 注明该方法是用来提供依赖对象的方法
    public B provideB(){
        return new B();
    }
}
```

#### 实现Component

```java
@Component(modules={ MyModule.class}) // 指明 Component 查找 Module 的位置
public interface MyComponent{    
  	// 必须定义为接口，Dagger2 框架将自动生成 Component 的实现类，对应的类名是 Dagger×××××，这里对应的实现类是 DaggerMyComponent 
    void inject(A a);   // 注入到A(Container)的方法，方法名一般使用inject
 }
```

#### 实现Container

A 就是可以被注入依赖关系的容器。

```java
public A{
     @Inject   //标记b将被注入
     B b;   // 成员变量要求是包级可见，也就是说@Inject不可以标记为private类型。 
     public void init(){
         DaggerMyComponent.create().inject(this); // 将实现类注入
     }
 }
```

当调用 A 的 init() 方法时， b 将会自动被赋予实现类的对象。

### 更多用法

#### 方法参数

上面的例子 `@Provides` 标注的方法是没有输入参数的，Module 中 `@Provides` 标注的方法是可以带输入参数的，其参数值可以由 Module 中的其他被 `@Provides` 标注的方法提供。

```java
@Module
public class MyModule{
    @Provides
    public B provideB(C c){         
        return new B(c);
    }
    @Provides
    pulic C provideC(){
        return new C();
    }
}
```

如果找不到被 `@Provides` 注释的方法提供对应参数对象的话，将会**自动调用被 `@Inject` 注释的构造方法**生成相应对象。

```java
@Module
public class MyModule{
    @Provides
    public B provideB(C c){
        return new B(c);
    }
}
public class C{
    @Inject
    Public C(){
    }
}
```

#### 添加多个 Module

一个 Component 可以添加多个 Module，这样 Component 获取依赖时候会自动从多个 Module 中查找获取。

添加多个 Module 有两种方法:

* 一种是在 Component 的注解 `@Component(modules={××××，×××})` 中添加多个 modules

```java
@Component(modules={ModuleA.class,ModuleB.class,ModuleC.class}) 
public interface MyComponent{
    ...
}
```

* 另外一种添加多个 Module 的方法可以使用`@Module` 的 includes 的方法`(includes={××××，×××})`

```java
@Module(includes={ModuleA.class,ModuleB.class,ModuleC.class})
public class MyModule{
    ...
}
@Component(modules={MyModule.class}) 
public interface MyComponent{
    ...
}
```

#### 创建 Module 实例

上面简单例子中，当调用 **DaggerMyComponent.create()** 实际上等价于调用了 **DaggerMyComponent.builder().build()**。可以看出，DaggerMyComponent 使用了 Builder 构造者模式。在构建的过程中，默认使用 Module 无参构造器产生实例。如果需要传入特定的 Module 实例，可以使用

```java
DaggerMyComponent.builder()
.moduleA(new ModuleA()) 
.moduleB(new ModuleB())
.build()
```

#### 区分 @Provides 方法

这里以 Android Context 来举例。当有 Context 需要注入时，Dagger2 就会在 Module 中查找返回类型为 Context 的方法。但是，当 Container 需要依赖两种不同的 Context 时，你就需要写两个 `@Provides` 方法，而且这两个 `@Provides` 方法都是返回 Context 类型，靠判别返回值的做法就行不通了。

##### 使用@Named 注解来区分

```java
//定义Module
@Module
public class ActivityModule{
private Context mContext    ;
private Context mAppContext = App.getAppContext();
    public ActivityModule(Context context) {
        mContext = context;
    }
    @Named("Activity")
    @Provides
    public Context provideContext(){  
        return mContext;
    }
    @Named("Application")
    @Provides
    public Context provideApplicationContext (){
        return mAppContext;
    }
}

//定义Component
@Component(modules={ActivityModule.class}) 
interface ActivityComponent{   
    void inject(Container container);   
}

//定义Container
class Container extends Fragment{
    @Named("Activity") 
    @Inject
    Context mContext; 

    @Named("Application") 
    @Inject
    Context mAppContext; 
    ...
    public void init(){
         DaggerActivityComponent.
     .activityModule(new ActivityModule(getActivity()))
     .inject(this);
     }

}
```

这样，只有相同的 `@Named` 的 `@Inject` 成员变量与 `@Provides` 方法才可以被对应起来。

##### 使用注解 @Qualifier 来自定义注解

自定义一个 annotation，其实 `@Named` 是 `@Qualifier` 的一种实现

```java
@Qualifier   
@Documented   //起到文档提示作用
@Retention(RetentionPolicy.RUNTIME)  //注意注解范围是Runtime级别
public @interface ContextLife {
    String value() default "Application";  // 默认值是"Application"
}
```

接下来使用我们定义的 `@ContextLife` 来修改上面的例子

```java
//定义Module
@Module
public class ActivityModule{
private Context mContext    ;
private Context mAppContext = App.getAppContext();
    public ActivityModule(Context context) {
        mContext = context;
    }
    @ContextLife("Activity")
    @Provides
    public Context provideContext(){  
        return mContext;
    }
    @ContextLife ("Application")
    @Provides
    public Context provideApplicationContext (){
        return mAppContext;
    }
}

//定义Component
@Component(modules={ActivityModule.class}) 
interface ActivityComponent{   
    void inject(Container container);   
}

//定义Container
class Container extends Fragment{
    @ContextLife ("Activity") 
    @Inject
    Context mContext; 

    @ContextLife ("Application") 
    @Inject
    Context mAppContext; 
    ...
    public void init(){
         DaggerActivityComponent.
     .activityModule(new ActivityModule(getActivity()))
     .inject(this);

     }
}
```

#### 组件间依赖

假设 ActivityComponent 依赖 ApplicationComponent。当使用 ActivityComponent 注入 Container 时，如果找不到对应的依赖，就会到 ApplicationComponent 中查找。但是，ApplicationComponent 必须显式把 ActivityComponent 找不到的依赖提供给 ActivityComponent。

```java
//定义ApplicationModule
@Module
public class ApplicationModule {
    private App mApplication;

    public ApplicationModule(App application) {
        mApplication = application;
    }

    @Provides
    @ContextLife("Application")
    public Context provideApplicationContext() {
        return mApplication.getApplicationContext();
    }

}

//定义ApplicationComponent
@Component(modules={ApplicationModule.class})
interface ApplicationComponent{
    @ContextLife("Application")
    Context getApplication();  // 对外提供ContextLife类型为"Application"的Context
}

//定义ActivityComponent
@Component(dependencies=ApplicationComponent.class, modules=ActivityModule.class)
 interface ActivityComponent{
    ...
}
```

### 进阶用法

#### @Singleton 单例的使用

创建某些对象有时候是耗时、浪费资源的或者需要确保其唯一性，这时就需要使用 `@Singleton` 注解标注为单例了。

```java
@Module
class MyModule{
    @Singleton   // 标明该方法只产生一个实例
    @Provides
    B provideB(){
        return new B();
    }
}

@Singleton  // 标明该 Component 中有 Module 使用了 @Singleton
@Component(modules=MyModule.class)
class MyComponent{
    void inject(Container container)
}
```

**注意：**Java中，单例通常保存在一个静态域中，这样的单例往往要等到虚拟机关闭时候，该单例所占用的资源才释放。但是，Dagger通过注解创建出来的单例并不保持在静态域上，而是保留在 Component 实例中。所以说不同的 Component 实例提供的对象是不同的。



#### 自定义 Scope

`@Singleton` 就是一种 Scope 注解，也是 Dagger2 唯一自带的 Scope 注解,下面是 `@Singleton` 的源码

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton{}
```

可以看到定义一个 Scope 注解，必须添加以下三部分：
`@Scope` ：注明是 Scope
`@Documented` ：标记文档提示
`@Retention(RUNTIME)` ：运行时级别

对于Android，我们通常会定义一个针对整个 APP 全生命周期的 `@PerApp` 的 Scope 注解和针对一个 Activity 生命周期的 `@PerActivity` 注解，如下

```java
@Scope
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface PerApp {
}

@Scope
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface PerActivity {
}

@PerApp 的使用例：
@Module
public class ApplicationModule {
    private App mApplication;

    public ApplicationModule(App application) {
        mApplication = application;
    }

    @Provides
    @PerApp
    @ContextLife("Application")
    public Context provideApplicationContext() {
        return mApplication.getApplicationContext();
    }

}

@PerApp
@Component(modules = ApplicationModule.class)
public interface ApplicationComponent {
    @ContextLife("Application")
    Context getApplication();

}

// 单例的有效范围是整个Application
public class App extends Application {
private static  ApplicationComponent mApplicationComponent; // 注意是静态
    public void onCreate() {
        mApplicationComponent = DaggerApplicationComponent.builder()
                .applicationModule(new ApplicationModule(this))
                .build();
    }

    // 对外提供ApplicationComponent
    public static ApplicationComponent getApplicationComponent() {
        return mApplicationComponent;
    }
}
```

`@PerActivity`  的使用例：

```java
// 单例的有效范围是Activity的生命周期
public abstract class BaseActivity extends AppCompatActivity {
    protected ActivityComponent mActivityComponent; //非静态，除了针对整个App的Component可以静态，其他一般都不能是静态的。
    // 对外提供ActivityComponent
    public ActivityComponent getActivityComponent() {
        return mActivityComponent;
    }

    public void onCreate() {
        mActivityComponent = DaggerActivityComponent.builder()
                .applicationComponent(App.getApplicationComponent())
                .activityModule(new ActivityModule(this))
                .build();
    }
}
```

通过上面的例子可以发现，使用自定义 Scope 可以很容易区分单例的有效范围。

#### @Subcomponent 子组件

可以使用 `@Subcomponent` 注解拓展原有 component。Subcomponent 其功能效果优点类似 component 的 dependencies。但是使用`@Subcomponent` 不需要在父 component 中显式添加子 component 需要用到的对象，只需要添加返回子 Component 的方法即可，子 Component 能自动在父 Component 中查找缺失的依赖。

```java
//父Component：
@Component(modules=××××)
public AppComponent{    
    SubComponent subComponent ();  // 这里返回子Component
}
//子Component：
@Subcomponent(modules=××××)   
public SubComponent{
    void inject(SomeActivity activity); 
}

// 使用子Component
public class SomeActivity extends Activity{
    public void onCreate(Bundle savedInstanceState){
        App.getComponent().subCpmponent().inject(this); // 这里调用子Component
    }    
}
```

Subcomponent 同时具备两种不同生命周期的 scope，SubComponent 具备了父 Component 拥有的 Scope，也具备了自己的 Scope。

SubComponent 的 Scope 范围小于父 Component。

#### Lazy 与 Provider 懒加载模式和强制加载

可以使用 Lazy 来包装 Container 中需要被注入的类型为延迟加载，另外可以使用 Provider 实现强制加载，每次调用 get 都会调用 Module 的Provides 方法一次，和懒加载模式正好相反。

```java
public class Container{
    @Inject Lazy<User> lazyUser; //注入Lazy元素
    @Inject Provider<User> providerUser; //注入Provider元素
    public void init(){
        DaggerComponent.create().inject(this);
        User user1=lazyUser.get();  //在这时才创建user1,以后每次调用get会得到同一个user1对象

        User user2=providerUser.get(); 
      //在这时创建user2，以后每次调用get会再强制调用Module的Provides方法一次，根据Provides方法具体实现的不同，可能返回跟user2是同一		个对象，也可能不是。
    }
}
```

### 结合 MVP 模式使用例

接下来看一下我们是如何在 Activity 中注入一个 Presenter 变量，来实现 MVP 的 V 层与 P 层之间的解耦。

```java
// Module
@Module
public class ActivityModule {
    private Activity mActivity;

    public ActivityModule(Activity activity) {
        mActivity = activity;
    }

    @Provides
    @PerActivity
    @ContextLife("Activity")
    public Context ProvideActivityContext() {
        return mActivity;
    }
}

// Component
@PerActivity
@Component(dependencies = ApplicationComponent.class, modules = ActivityModule.class)
public interface ActivityComponent {

    @ContextLife("Activity")
    Context getActivityContext(); 

    @ContextLife("Application")
    Context getApplicationContext();

    void inject(NewsActivity newsActivity);
}

// Presenter
public class NewsPresenter {
    ...
    @Inject
    public NewPresenter () {
    }
     private void loadNews () {
        ...
    }
    ....
}

// Activity
public class NewsActivity extends BaseActivity
        implements NewsView {
    @Inject
    NewPresenter mNewsPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
    ActivityComponent activityComponent;
        activityComponent = DaggerActivityComponent.builder()
         .applicationComponent(App.getApplicationComponent())
                .activityModule(new ActivityModule(this))
                .build();
        activityComponent.inject(this);

    mNewsPresenter.loadNews();

    }
}
```

在这个例子中，注入了一个名叫 NewsPresenter 的类，假设它负责在后台处理新闻数据。但是我们并没有在 Module 中提供生产 NewsPresenter 实例的 Provides 方法。这时根据 Dagger2 的注入规则，用 `@Inject` 注释的成员变量的依赖会首先从 Module 的 `@Provides` 方法集合中查找。如果查找不到的话，则会查找成员变量类型是否有 `@Inject` 构造方法，并会调用该构造方法注入该类型的成员变量。这时如果被 `@Inject` 注释的构造方法有参数的话，则将会继续使用注入规则进行递归查找。

## 总结

![](http://om4rextnc.bkt.clouddn.com/17-6-13/88312840.jpg)



1. componet 的 inject 方法接收父类型参数，而调用时传入的是子类型对象则无法注入
2. component 关联的 modules 中不能有重复的 provide
3. module 的 provide 方法使用了 scope ，那么 component 就必须使用同一个注解
4. module 的 provide 方法没有使用 scope ，那么 component 和 module 是否加注解都无关紧要，可以通过编译
5. component 的 dependencies 与 component 自身的 scope 不能相同，即组件之间的 scope 不同
6. Singleton 的组件不能依赖其他 scope 的组件，只能其他 scope 的组件依赖 Singleton 的组件
7. 没有 scope 的 component 不能依赖有 scope 的 component
8. 一个 component 不能同时有多个 scope(Subcomponent 除外)
9. @Singleton 的生命周期依附于 component，同一个 module  provide singleton ,不同 component 也是不一样



## References

[Dagger2系列2 实例解析](http://www.jianshu.com/p/7abc7938818b)

[神兵利器Dagger2 ](https://juejin.im/post/5857f70361ff4b006cb0d9fd?utm_source=gold_browser_extension)

[Dagger2 Scope 注解能保证依赖在 component 生命周期内的单例性吗？](http://blog.piasy.com/2016/04/11/Dagger2-Scope-Instance/)



















































