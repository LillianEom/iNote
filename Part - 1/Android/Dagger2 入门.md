# Dagger2 入门

Dagger2 是通过 apt 动态生成代码来实现的 IOC 框架。依赖注入的最大好处：就是模块间解耦！ 就拿当前Android非常流行的开发模式 MVP 来说，使用 Dagger2 可以将 MVP 中的 V  层与 P 层进一步解耦，这样便可以提高代码的健壮性和可维护性。

## 三个模块

1. 依赖提供方 **Module**,负责提供依赖中所需要的对象,实际编码中类似于工厂类
2. 依赖需求方实例,它声明依赖对象,它在实际编码中对应业务类,例如Activity,当你在Activity中需要某个对象时,你只要在其中声明就行,声明的方法在下面会讲到.
3. 依赖注入组件 **Component**,负责将对象注入到依赖需求方,它在实际编码中是一个接口,编译时Dagger2会自动为它生成一个实现类.



## 工作流程

1. 将依赖需求方实例传入给Component实现类
2. Component实现类根据依赖需求方实例中依赖声明,来确定该实例需要依赖哪些对象
3. 确定依赖对象后,Component会在与自己关联的Module类中查找有没有提供这些依赖对象的方法,有的话就将Module类中提供的对象设置到依赖需求方实例中

## 引入Dagger2

在项目中 build.gradle 文件中添加 apt 插件

```
buildscript {
    ...
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'       
 // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
        //添加apt插件
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}
...
```

在app目录的build.gradle文件中添加:

```
//应用apt插件
apply plugin: 'com.neenbedankt.android-apt'

...

dependencies {
    ...    //引入dagger2
    compile 'com.google.dagger:dagger:2.4'
    apt 'com.google.dagger:dagger-compiler:2.4'    //java注解
    provided 'org.glassfish:javax.annotation:10.0-b28'
}
```
## 注解

Dagger2 通过注解来生成代码，定义不同的角色，主要的注解如下：
**`@Module:`** Module类里面的方法专门提供依赖，所以我们定义一个类，用 @Module 注解，这样 Dagger 在构造类的实例的时候，就知道从哪里去找到需要的依赖。
**`@Provides: `**在Module中，我们定义的方法是用这个注解，以此来告诉 Dagger 我们想要构造对象并提供这些依赖。
**`@Inject:` **通常在需要依赖的地方使用这个注解。换句话说，你用它告诉 Dagger 这个类或者字段需要依赖注入。这样，Dagger就会构造一个这个类的实例并满足他们的依赖。
**`@Component: `**Component 从根本上来说就是一个注入器，也可以说是 @Inject 和 @Module 的桥梁，它的主要作用就是连接这两个部分。将 Module 中产生的依赖对象自动注入到需要依赖实例的 Container 中。
**`@Scope:`** Dagger2可以通过自定义注解限定注解作用域，来管理每个对象实例的生命周期。
**`@Qualifier: `**当类的类型不足以鉴别一个依赖的时候，我们就可以使用这个注解标示。例如：在Android中，我们会需要不同类型的 context，所以我们就可以定义 qualifier注解“@perApp”和“@perActivity”，这样当注入一个context的时候，我们就可以告诉 Dagger我们想要哪种类型的context。