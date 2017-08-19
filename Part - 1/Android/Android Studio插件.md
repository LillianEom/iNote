# 1. [GsonFormat](http://plugins.jetbrains.com/plugin/7654?pr=androidstudio)

快速将json字符串转换成一个Java Bean，免去我们根据json字符串手写对应Java Bean的过程.

![img](http://upload-images.jianshu.io/upload_images/1621204-7c767660aef15858.png?imageMogr2/auto-orient/strip)

使用方法：快捷键Alt+S也可以使用Alt+Insert选择GsonFormat

# 2.[Android ButterKnife Zelezny](http://plugins.jetbrains.com/plugin/7369?pr=androidstudio)

配合ButterKnife实现注解，从此不用写findViewById，想着就爽啊。在Activity，Fragment，Adapter中选中布局xml的资源id自动生成butterknife注解。

![img](http://upload-images.jianshu.io/upload_images/1621204-4847ff43d5a7d095.png?imageMogr2/auto-orient/strip)

使用方法：Ctrl+Shift+B选择图上所示选项

# 3.[Android-ButterKnife-Plugin-Plus](https://github.com/OriginalLove/Android-ButterKnife-Plugin-Plus)

Android Studio 的插件，方便快速实现ButterKnife注解框架，包含了android-butterknife-zelezny 1.6版本的所有功能，并在此基础上新增如下功能：

1.可以自由选择是否在当前类中对ButterKnife进行初始化，避免了原版本只要使用插件初始化控件会自动在onCreate中进行ButterKnife.bind(this)的尴尬。

![img](https://github.com/OriginalLove/Android-ButterKnife-Plugin-Plus/raw/master/img/1.png)

这样就可以在基类中进行ButterKnife的初始化，不必要每个类中都要初始化，对开发框架的搭建更加方便。

# 4.[Android Code Generator](http://plugins.jetbrains.com/plugin/7595?pr=androidstudio)

根据布局文件快速生成对应的Activity，Fragment，Adapter，Menu。

![img](http://upload-images.jianshu.io/upload_images/1621204-da62c01355f5ad21.png?imageMogr2/auto-orient/strip)

![img](http://upload-images.jianshu.io/upload_images/1621204-129cc34063347c1c.png?imageMogr2/auto-orient/strip)

# 5.[GradleDependenciesHelperPlugin](https://github.com/ligi/GradleDependenciesHelperPlugin)

maven gradle 依赖支持自动补全

![img](https://camo.githubusercontent.com/d9b1b39eda21e0e33b656e2821f01897d915f7c5/68747470733a2f2f6c68332e676f6f676c6575736572636f6e74656e742e636f6d2f2d51364e7970315864594c772f556a73325a5175666634492f414141414141414144624d2f624d704c516742664d6b632f773538372d683330392d6e6f2f696465615f677261646c655f706c7567696e2e706e67)

# 6.[dagger-intellij-plugin](https://github.com/square/dagger-intellij-plugin)

Dagger 可视化辅助工具

![img](https://github.com/square/dagger-intellij-plugin/raw/master/images/inject-to-provide.gif)

# 7.[Exynap](http://exynap.com/)

Exynap 一个帮助开发者自动生成样板代码的 AndroidStudio 插件

![img](http://upload-images.jianshu.io/upload_images/1621204-e569b243a54357fe.gif?imageMogr2/auto-orient/strip)

# 8.[Android Parcelable code generator](http://plugins.jetbrains.com/plugin/7332?pr=androidstudio)

JavaBean序列化，快速实现Parcelable接口。

![img](https://segmentfault.com/image?src=http://img.blog.csdn.net/20160416104459926&objectId=1190000005092842&token=ab29ed79d41be9e42b3a3d2ed1ec3bef)

# 9.[LayoutFormatter](https://github.com/drakeet/LayoutFormatter)

drakeet 开发一个一键格式化你的 XML 文件的 Android Studio 插件，至于为什么不用 Android Studio 自带的格式化功能而用这个插件，可以看下作者的一篇 Blog -> [当我们谈 XML 布局文件代码的优雅性](https://drakeet.me/layoutformatter)

![img](http://upload-images.jianshu.io/upload_images/1621204-b6c6598181db0d64.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 10.[eventbus3-intellij-plugin](https://github.com/likfe/eventbus3-intellij-plugin/blob/master/README-zh.md)

引导 EventBus 的 post 和 event(对于最新版的 EventBus 3.0.0 有效)
主要Bug修复工作：
修改包名和方法名以适应 EventBus 3.X
替换一个在新版的 intellij plugin SDK 已经不存在的类
增加若干 try-catch ，防止插件崩溃

![img](https://raw.githubusercontent.com/likfe/eventbus3-intellij-plugin/master/art/cap.gif)

# 11.[eventbus-intellij-plugin](https://github.com/kgmyshin/eventbus-intellij-plugin)

eventbus导航插件(对于最新版的 EventBus 3.0.0 好像无效,请替换为eventbus3-intellij-plugin此插件地址在本文第51个)

![img](https://raw.githubusercontent.com/kgmyshin/eventbus-intellij-plugin/master/art/cap.gif)

# 12.[Android Studio Prettify](https://plugins.jetbrains.com/plugin/7405)

可以将代码中的字符串写在string.xml文件中

选中字符串鼠标右键选择图中所示

![img](https://plugins.jetbrains.com/files/7405/screenshot_14417.png)

这个插件还可以自动书写findViewById

![img](https://plugins.jetbrains.com/files/7405/screenshot_14418.png)

![img](https://plugins.jetbrains.com/files/7405/screenshot_14416.png)

![img](https://plugins.jetbrains.com/files/7405/screenshot_14501.png)

![img](https://plugins.jetbrains.com/files/7405/screenshot_14419.png)

![img](https://plugins.jetbrains.com/files/7405/screenshot_14415.png)

# 13.[folding-plugin](https://github.com/dmytrodanylyk/folding-plugin)

布局文件分组的插件

![img](https://github.com/dmytrodanylyk/folding-plugin/raw/master/screenshots/Preview.png)

# 14.[SelectorChapek for Android](http://plugins.jetbrains.com/plugin/7298?pr=androidstudio)

通过资源文件命名自动生成Selector文件。

![img](http://upload-images.jianshu.io/upload_images/1621204-b9968a79dea329f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-bb001ecfa5e7c992.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-ed901fc313424d25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 15.[Android Methods Count](http://plugins.jetbrains.com/plugin/8076?pr=androidstudio)

显示依赖库中得方法数

![img](http://upload-images.jianshu.io/upload_images/1621204-5d818ba8b01a81e4.png?imageMogr2/auto-orient/strip)

# 16.[Lifecycle Sorter](http://plugins.jetbrains.com/plugin/7742?pr=androidstudio)

可以根据Activity或者fragment的生命周期对其生命周期方法位置进行先后排序，快捷键Ctrl + alt + K

![img](http://upload-images.jianshu.io/upload_images/1621204-fc01434b35ca8970.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-4d1408cd3d6d4549.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 17.[CodeGlance](http://plugins.jetbrains.com/plugin/7275?pr=androidstudio)

在右边可以预览代码，实现快速定位

![img](http://upload-images.jianshu.io/upload_images/1621204-98b3f4288839e97c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-7f9d683a75288176.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 18.[android-strings-search-plugin](https://github.com/konifar/android-strings-search-plugin)

一个可以通过输入文字找到strings.xml资源的插件

![img](https://github.com/konifar/android-strings-search-plugin/raw/master/art/demo.gif)

# 19.[findBugs-IDEA](http://plugins.jetbrains.com/plugin/3847?pr=androidstudio)

查找bug的插件，Android Studio也提供了代码审查的功能（Analyze-Inspect Code...）

![img](http://upload-images.jianshu.io/upload_images/1621204-4b3c06253da0614f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 20.[ADB WIFI](http://plugins.jetbrains.com/plugin/7856?pr=androidstudio)

使用wifi无线调试你的app，无需root权限
也可参考以下文章：
[Android wifi无线调试App新玩法ADB WIFI](http://www.jianshu.com/p/21d1b65d92a4)

![img](http://upload-images.jianshu.io/upload_images/1621204-453fe2150b1edc0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 21.[JsonOnlineViewer](http://plugins.jetbrains.com/plugin/7838?pr=androidstudio)

在Android Studio中请求、调试接口

![img](http://upload-images.jianshu.io/upload_images/1621204-4314c283b0397406.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 22.[Android Styler](http://plugins.jetbrains.com/plugin/7972?pr=androidstudio)

根据 xml 自动生成 style 代码的插件

![img](http://upload-images.jianshu.io/upload_images/1621204-dcfec08540fa7300.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-4752e512ecd4aedb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-23d78bacd7304ea0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Usage:

a. copy lines with future style from your layout.xml file
b. paste it to styles.xml file with Ctrl+Shift+D (or context menu)
c. enter name of new style in the modal window
d. your style is prepared!

# 23.[Android Drawable Importer](http://plugins.jetbrains.com/plugin/7658?pr=androidstudio)

这是一个非常强大的图片导入插件。它导入Android图标与Material图标的Drawable ，批量导入Drawable ，多源导入Drawable（即导入某张图片各种dpi对应的图片）

![img](http://upload-images.jianshu.io/upload_images/1621204-62b2479e26bf02af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-afa40d4ffd6a456a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-b2295c9b862c7511.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-43f77856d01aec37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-2fc177e734171a37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-f1be52555bf16d6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-c2e466eb6f03f8fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](http://upload-images.jianshu.io/upload_images/1621204-f957f7878d6e3123.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# 24.[GenerateSerialVersionUID](http://plugins.jetbrains.com/plugin/185?pr=androidstudio)

实现Serializable序列化bean

Adds a new action 'SerialVersionUID' in the generate menu (alt + ins). The action adds an serialVersionUID field in the current class or updates it if it already exists, and assigns it the same value the standard 'serialver' JDK tool would return. The action is only visible when IDEA is not rebuilding its indexes, the class is serializable and either no serialVersionUID field exists or its value is different from the one the 'serialver' tool would return.

# 25.[SQLScout](https://plugins.jetbrains.com/plugin/8322-sqlscout-sqlite-support-)

在 Android Studio 上调试数据库 ( SQLite )

详细使用参考：[在 Android Studio 上调试数据库 ( SQLite )](https://juejin.im/post/58e0d781a0bb9f0069ec08d3)

![img](https://plugins.jetbrains.com/files/8322/screenshot_15823.png)

# 26.[Android Postfix Completion](http://plugins.jetbrains.com/plugin/7775?pr=androidstudio)

可根据后缀快速完成代码，这个属于拓展吧，系统已经有这些功能，如sout、notnull等，这个插件在原有的基础上增添了一些新的功能，我更想做的是通过原作者的代码自己定制功能，那就更爽了

![img](http://upload-images.jianshu.io/upload_images/1621204-1844b61e27958fa9.png?imageMogr2/auto-orient/strip)

# 27.[Android Holo Colors Generator](https://plugins.jetbrains.com/plugin/7366?pr=)

通过自定义Holo主题颜色生成对应的Drawable和布局文件

![img](https://plugins.jetbrains.com/files/7366/screenshot_14379.png)





# 28.[RemoveButterKnife](https://github.com/u3shadow/RemoveButterKnife)

ButterKnife这个第三方库每次更新之后，绑定view的注解都会改变，从bind,到inject，再到bindview，搞得很多人都不敢升级，一旦升级，就会有巨量的代码需要手动修改，非常痛苦
当我们有一些非常棒的代码需要拿到其他项目使用，但是我们发现，那个项目对第三方库的使用是有限制的，我们不能使用butterknife，这时候，我们又得从注解改回findviewbyid
针对上面的两种情况，如果view比较少还好说，如果有几十个view，那么我们一个个的手动删除注解，写findviewbyid语句，简直是一场噩梦（别问我为什么知道这是噩梦）
所以，这种有规律又重复简单的工作为什么不能用一个插件来实现呢？于是RemoveButterKnife的想法就出现了。

[具体介绍](http://www.u3coding.com/2016/06/24/androidstudio-plugin-removebutterknife-di/)

![img](https://camo.githubusercontent.com/0327cda5b531ab6f2b803abe295c42225668d28d/687474703a2f2f7777772e7533636f64696e672e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031362f30362f312e676966)

# 23.[AndroidProguardPlugin](https://github.com/zhonghanwen/AndroidProguardPlugin)

一键生成项目混淆代码插件，值得你安装~(不过目前可能有些第三方项目的混淆还未添加完全)

![img](http://upload-images.jianshu.io/upload_images/1621204-9d51a88fb02c1512.gif?imageMogr2/auto-orient/strip)

# 24.[otto-intellij-plugin](https://github.com/square/otto-intellij-plugin)

otto事件导航工具。

![img](https://github.com/square/otto-intellij-plugin/raw/master/images/produce-to-subscribe.gif)

![img](https://github.com/square/otto-intellij-plugin/raw/master/images/event-to-subscribe.gif)



# 26.[idea-markdown](https://github.com/nicoulaj/idea-markdown)

markdown插件

![img](https://github.com/nicoulaj/idea-markdown/raw/assets/screenshots/preview.png)





# 27.[Android-DPI-Calculator](https://github.com/JerzyPuchalski/Android-DPI-Calculator)

DPI计算插件

![img](https://camo.githubusercontent.com/ce3be2aaa3b1f70b90f5b825c529694509d70313/68747470733a2f2f7261772e6769746875622e636f6d2f4a65727a7950756368616c736b692f416e64726f69642d4450492d43616c63756c61746f722f6d61737465722f696d672f6469616c6f672e706e67)

使用：

![img](https://camo.githubusercontent.com/598d3b5c9efc5f0b57b58c25a79a323d06307fad/68747470733a2f2f7261772e6769746875622e636f6d2f4a65727a7950756368616c736b692f416e64726f69642d4450492d43616c63756c61746f722f6d61737465722f696d672f616374696f6e2e706e67)

或者

![img](https://camo.githubusercontent.com/7a8f977de7a1ba6cd23fb64cbd37566690c27cdc/68747470733a2f2f7261772e6769746875622e636f6d2f4a65727a7950756368616c736b692f416e64726f69642d4450492d43616c63756c61746f722f6d61737465722f696d672f6d656e752e706e67)



# 28.[PermissionsDispatcher plugin](https://plugins.jetbrains.com/plugin/8349)

github:[PermissionsDispatcher plugin](https://github.com/shiraji/permissions-dispatcher-plugin)
自动生成6.0权限的代码

![img](https://github.com/shiraji/permissions-dispatcher-plugin/raw/master/website/images/pd.gif)





# 29.[gradle-cleaner-intellij-plugin](https://github.com/Softwee/gradle-cleaner-intellij-plugin)

Force clear delaying & no longer needed Gradle tasks.

![img](https://camo.githubusercontent.com/cb48bca7f8bd0b513f350f7320c74054d1b9fbce/687474703a2f2f6936352e74696e797069632e636f6d2f726a687863382e706e67)

# 30.[MVPHelper](http://androidwing.net/index.php/27)

一款Intellj IDEA 和Android Studio的插件，可以为MVP生成接口以及实现类，解放双手。
具体请查看[Android Studio插件之MVPHelper，一键生成MVP代码](http://androidwing.net/index.php/27)一文

![img](https://github.com/githubwing/MVPHelper/raw/master/img/mvp_presenter.gif)

# lint-cleaner-plugin

删除未使用的资源,包括String字符串,颜色和尺寸。 这是一个Gradle插件，所以如何配置可以去github的源码上看。
插件源码地址：[github.com/marcoRS/lin…](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FmarcoRS%2Flint-cleaner-plugin)

# Android strings.xml tools

可以用来管理Android项目中的字符串资源。它提供了排序Android本地文件和添加缺少的字符串的基本操作。虽然这个插件是有限制的，但如果应用程序有大量的字符串资源，那这个插件就非常有用了。
插件下载地址：[plugins.jetbrains.com/plugin/7498…](https://link.juejin.im/?target=https%3A%2F%2Fplugins.jetbrains.com%2Fplugin%2F7498%3Fpr%3D)
插件源码地址：[github.com/constantine…](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fconstantine-ivanov%2Fstrings-xml-tools)

# JavaDoc

添加注释，可自定义模板。
插件下载地址：[plugins.jetbrains.com/plugin/?ide…](https://link.juejin.im/?target=https%3A%2F%2Fplugins.jetbrains.com%2Fplugin%2F%3Fidea_ce%26amp%3BpluginId%3D7157)
插件源码地址：[github.com/setial/inte…](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fsetial%2Fintellij-javadocs)
推荐指数： 五星

# AndroidAccessors

快速实现get和set方法的插件。
插件下载地址：[plugins.jetbrains.com/plugin/7496…](https://link.juejin.im/?target=https%3A%2F%2Fplugins.jetbrains.com%2Fplugin%2F7496%3Fpr%3D)
插件文档地址：[github.com/jonstaff/An…](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fjonstaff%2FAndroidAccessors)



























