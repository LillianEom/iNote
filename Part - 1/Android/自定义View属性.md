# 自定义View属性

### obtainStyledAttributes 四个参数的详细的作用

1. **基本用法**

   在 res 资源目录的 values 目录下创建一个 attrs.xml 的属性定义文件，通过 declare-styleable 标签声明使用自定义属性，并通过 name 属性来确定引用的名称，最后通过 attr 标签来声明具体的自定义属性，并通过 format 属性指定属性类型。

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <resources>
       <declare-styleable name="AttrView">
           <attr name="title" format="string"></attr>
           <attr name="titleColor" format="color"></attr>
           <attr name="leftBackground" format="reference|color"></attr>
       </declare-styleable>
   </resources>
   ```

   确定好属性后，创建一个自定义控件 AttrView，继承 ViewGroup，在构造方法中通过以下代码获取 XML 布局文件中的自定义属性。

   ```java
   public AttrView(Context context, AttributeSet attributeSet) {
       //通过这个方法将 attrs.xml 中定义的 declare-styleable 的所有属性值存储到TypedArray
       TypedArray typedArray = context.obtainStyledAttributes(attributeSet, R.styleable.AttrView);
   	
       //从 TypedArray 中取出相应的值来给要设置的属性赋值
       mTitle = typedArray.getString(R.styleable.AttrView_title);
       mTitleColor = typedArray.getColor(R.styleable.AttrView_color,0);
       .....
       
       //获取完TypedArray值后，一般要调用recycle方法来避免重新创建的时候的错误
       typedArray.recycle();
   }
   ```

   obtainStyledAttributes这个方法实际上调用的是

   ```xml
   public final TypedArray obtainStyledAttributes(
           AttributeSet set, @StyleableRes int[] attrs, @AttrRes int defStyleAttr,
           @StyleRes int defStyleRes) {
       return getTheme().obtainStyledAttributes(
           set, attrs, defStyleAttr, defStyleRes);
   }
   ```

2. **defStyleRes**

   用于指定一个style，如果我们不在布局文件中设置任何属性，那么会从在这个style中读取相关属性。

   在 styles.xml 里面编写

   ```xml
   <style name="AppTheme">
     <item name="colorPrimary">@color/colorPrimary</item>
     <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
     <item name="colorAccent">@color/colorAccent</item>
   </style>
   ```

   修改获取属性的代码：

   ```java
   TypedArray typedArray = context.obtainStyledAttributes(attrs, 
                           R.styleable.AttrView,
                           0,
                           R.style.AppTheme);
   ```

3. **defStyleAttr**

   一个引用类型的属性，指向一个style，并且在当前的 theme 中进行设置

   在attrs.xml 中添加一个新的引用格式的属性 "attrViewStyleRef"

   ```
   <attr name = "attrViewStyleRef" format = "reference"></attr>
   ```

   然后在 styles.xml 中找到所使用的 theme 添加一条 item

   ```xml
   <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
   	<item name = "attrViewStyleRef">@style/AttrViewStyle</item>
   </style>
   <style name = "AttrViewStyle">
     <!-- Customize your theme here. -->
     <item name="colorPrimary">@color/colorPrimary</item>
     <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
     <item name="colorAccent">@color/colorAccent</item>
   </style>
   ```

   修改获取属性的代码：

   ```java
   TypedArray typedArray = context.obtainStyledAttributes(attrs, 
                           R.styleable.AttrView,
                           R.attr.attrViewStyleRef,
                           0);
   ```

   对于第三个参数呢，实际上用的还是比较多的，比如看系统的Button，EditText，它们都会在构造里面指定第三个参数.提供一些参数的样式，比如background，textAppearance，textColor等，所以当切换不同的主题时，你会发现控件的样式会发生一些变化，就是因为不同的主题，设置了一些不同的style。那么推演到自定义的View，如果你的属性非常多，你也可以去提供默认的style，然后让用户去设置到theme里面即可。

   如果同时设置 defStyleAttr与defStyleRes，defStyleAttr优先级更高。

### 自定义View中构造方法中调用初始化代码，两种写法的区别

1. 第一种：

```java
 public GestureLockIndicator(Context context) {
 	this(context, null);
 }

public GestureLockIndicator(Context context, AttributeSet attrs) {
	this(context, attrs, 0);
}

public GestureLockIndicator(Context context, AttributeSet attrs, int defStyleAttr) {
  super(context, attrs, defStyleAttr);
  this.init();
}
```

  2.第二种：

```java
public GestureLockIndicator(Context context) {
	super(context);
	init();
}

public GestureLockIndicator(Context context, AttributeSet attrs) {
	super(context, attrs);
	init();
}

public GestureLockIndicator(Context context, AttributeSet attrs, int defStyleAttr) {
	super(context, attrs, defStyleAttr);
	init();
}
private void init(){
  //初始化
}
```

1.  如果需要设置obtainStyledAttributes的第三个参数，即`defStyleAttr`，一般会使用第一种方式，会在两个参数的构造中，去调用三个参数的构造，同时传入`defStyleAttr `。如果没有此需求，两种写法没有什么区别。

2. 继承系统已有的控件去自定义View，比如你继承Button，去做一些事情

   第一种方式会覆盖掉Button默认在theme里面设置的style，相对来说第二种方式更为合适（这条是一定要注意的，切记，切记）。

### 自定义View中获取自定义属性，两种写法的区别 

第一种：

见上

第二种：

```java
TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CustomTitleView, defStyle, 0);  
    int n = a.getIndexCount();  
    for (int i = 0; i < n; i++)  {  
        int attr = a.getIndex(i);  
        switch (attr)  {  
            case R.styleable.CustomTitleView_titleText:  
                mTitleText = a.getString(attr);  
                break;  
            case R.styleable.CustomTitleView_titleTextColor:  
                mTitleTextColor = a.getColor(attr, Color.BLACK);  
                break;  
            case R.styleable.CustomTitleView_titleTextSize:  
                mTitleTextSize = a.getDimensionPixelSize(attr, 10);  
                break;
        }
    }
```

- 第一种写法，不管你有没有在布局文件中使用该属性，都会去执行getXXX方法赋值，
- 第二种写法，只有在你布局文件中设置了该属性后，才会去调用getXXX方法赋值。

```java
private int attr_mode = 1;//默认为1
//然后你在写getXXX方法的时候，是这么写的：
attr_mode = array.getInt(i, 0);
```

可能你的自定义属性有个默认的值，然后你在写赋值的时候，一看是整形，就默默的第二个参数就给了个0，然而用户根本没有在布局文件里面设置这个属性，你却在运行时将其变为了0（而不是默认值），而第二种就不存在这个问题。当然这个场景可以由规范的书写代码的方式来避免，（把默认值提取出来，都设置对就好了）。

其实还有个场景，假设你是继承自某个View，父类的View已经对该成员变量进行赋值了，然后你这边需要根据用户的设置情况，去更新这个值，第一种写法，如果用户根本没有设置，你可能就将父类的赋值给覆盖了。