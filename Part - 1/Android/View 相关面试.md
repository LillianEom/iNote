# View 相关

**7. Android触摸分发机制****

### 1. 介绍触摸事件的分发机制

![](http://om4rextnc.bkt.clouddn.com/17-8-4/7692458.jpg)

1. 事件从Activity.dispatchTouchEvent()开始传递，只要没有被停止或拦截，从**最上层的View(ViewGroup)开始一直往下(子View)传递**。子View可以通过onTouchEvent()对事件进行处理。
2. 事件由父View(ViewGroup)传递给子View，ViewGroup可以通过**onInterceptTouchEvent()**对事件做拦截，停止其往下传递。
3. 如果事件从上往下传递过程中一直没有被停止，且最底层子View没有消费事件，事件会反向往上传递，这时父View(ViewGroup)可以进行消费，如果还是没有被消费的话，最后会到Activity的onTouchEvent()函数。
4. 如果View没有对ACTION_DOWN进行消费，之后的其他事件不会传递过来。
5. OnTouchListener优先于onTouchEvent()对事件进行消费。上面的消费即表示相应函数返回值为true。

### 2.View中 setOnTouchListener的onTouch，onTouchEvent，onClick的执行顺序

追溯到View的dispatchTouchEvent源码查看，有这么一段代码

```java
public boolean dispatchTouchEvent(MotionEvent event) {  
  if (!onFilterTouchEventForSecurity(event)) {  
    return false;  
  }  
  if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
      mOnTouchListener.onTouch(this, event)) {  
    return true;  
  }  
  return onTouchEvent(event);  
}
```


当以下三个条件任意一个不成立时，

mOnTouchListener不为null，view是enable的状态，mOnTouchListener.onTouch(this, event)返回true，函数会执行到onTouchEvent。在这里我们可以看到，首先执行的是**mOnTouchListener.onTouch**的方法，然后是onTouchEvent方法继续追溯源码，到onTouchEvent()观察，发现在处理ACTION_UP事件里有这么一段代码 

```java
if (!post(mPerformClick)) { 
  performClick();  
}
```

此时可知，onClick方法也在最后得到了执行所以三者的顺序是：setOnTouchListener() 的onTouchonTouchEvent() onClick()

### 3. View的绘制流程

![](http://om4rextnc.bkt.clouddn.com/17-8-4/49155959.jpg)

measure()方法，layout()，draw()三个方法主要存放了一些标识符，来判断每个View是否需要再重新测量，布局或者绘制，主要的绘制过程还是在onMeasure，onLayout，onDraw 这个三个方法中

**1.onMesarue() **为整个View树计算实际的大小，即设置实际的高(对应属性:mMeasuredHeight)和宽(对应属性: mMeasureWidth)，每个View的控件的实际宽高都是由父视图和本身视图决定的。

**2.onLayout()** 为将整个根据子视图的大小以及布局参数将View树放到合适的位置上。

**3. onDraw() **开始绘制图像，绘制的流程如下首先绘制该View的背景调用onDraw()方法绘制视图本身 (每个View都需要重载该方法，ViewGroup不需要实现该方法)如果该View是ViewGroup，调用dispatchDraw ()方法绘制子视图绘制滚动条

**自定义View如何考虑机型适配**
**自定义View的事件分发机制**
**View和ViewGroup分别有哪些事件分发相关的回调方法**
**自定义View如何提供获取View属性的接口**

**如何自定义ViewGroup？**

### 4. Android中的动画有哪些，区别是什么

**逐帧动画(Drawable Animation)**： 加载一系列Drawable资源来创建动画，简单来说就是播放一系列的图片来实现动画效果，可以自定义每张图片的持续时间

**补间动画(Tween Animation)**： Tween可以对View对象实现一系列简单的动画效果，比如位移，缩放，旋转，透明度等等。但是它并不会改变View属性的值，只是改变了View的绘制的位置，比如，一个按钮在动画过后，不在原来的位置，但是触发点击事件的仍然是原来的坐标。**属性动画(Property Animation)**： 动画的对象除了传统的View对象，还可以是Object对象，动画结束后，Object对象的属性值被实实在在的改变了

### 5.dp, dip, dpi, px, sp是什么意思以及他们的换算公式？layout-sw400dp, layout-h400dp分别代表什么意思；

dip: device independent pixels(设备独立像素). 不同设备有不同的显示效果,这个和设备硬件有关，一般我们为了支持WVGA、HVGA和QVGA 推荐使用这个，不依赖像素。
dp: dip是一样的
px: pixels(像素). 不同设备显示效果相同，一般我们HVGA代表320x480像素，这个用的比较多。
pt: point，是一个标准的长度单位，1pt＝1/72英寸，用于印刷业，非常简单易用；
sp: scaled pixels(放大像素). 主要用于字体显示best for textsize。

据px = dip * density / 160，则当屏幕密度为160时，px = dip根据 google 的建议，TextView 的字号最好使用 sp 做单位，而且查看TextView的源码可知Android默认使用sp作为字号单位。将dp作为其他元素的单位。

### 6.ViewPager

1. viewpager里面只能嵌套view吗 可不可以嵌套Activity

   ViewPager继承自ViewGroup，自然只能嵌套View。如果实在要嵌套Activity，去看过时的TabActivity的用法，然后获取Activity展示后的窗体View（getWindow().getDecorView().getRootView()），没必要这么搞。


1. 假如viewpager里面的每一页都有有很大数据量的内容，那么在快速的左右滑动时，如果出现了内存被回收的情况，如何处理 假如出现了OOM，怎么处理

   大量（海量）数据避免OOM从来的处理方法就是分而治之，一次显示不完，那就分页显示，分步加载；复用View（列表ContentView），使用缓存。使用缓存又带来及时回收的问题，推荐可以去看看ImageDownloader（最新的是LRUCache）中多级缓存（WeakReference，SoftReference等）机制，使用固定大小的Cache，移除时加入到WeakReference中，并定时清理（或者引入LRUCache等算法）；


1. 同上情况，使用Fragment，又当如何？与viewpager有什么区别

   使用Fragement就更简单了，基本思路跟2类似，处理好单个Fragment中的效率问题就好了，要说不同的地方，Fragment可以不必把所有的逻辑都写在一个Activity类中，模块更加清晰，维护起来也更方便，并且有自己的生命周期，如果你实现的逻辑比较简单（类似新版本介绍那样滑动几张图片，用ViewPager就可以），每个页面比较复杂的话，力荐Fragment！

### 7.不使用动画，怎么实现一个动态的 View？

可以通过检测touch事件，动态改变view的位置，这样就没有使用动画，是纯粹使用代码控制。自定义view，定时重绘。

### 8.Postvalidata与Validata有什么区别？

Android中实现view的更新有两组方法，一组是invalidate，另一组是postInvalidate，其中前者是在UI线程自身中使用，而后者在非UI线程中使用。 Android提供了Invalidate方法实现界面刷新，但是Invalidate不能直接在工作线程中调用，因为他是违背了单线程模型：Android UI在工作线程中操作并不是线程安全的，并且这些操作必须在UI线程中调用。



### 9. Scrollview怎么判断是否滑倒底部

* 滚动到顶部判断 getScrollY() == 0
* 滚动到底部判断

首先，判断到底部，我们应该先对相关函数有一个了解
    getChildAt：得到ScrollView的child View
    childView.getMeasuredHeight()表示得到子View的高度
    getScrollY()表示得到y轴的滚动距离
    getHeight()为scrollView的高度
其次，我们来看一下核心代码（getScrollY()达到最大时加上scrollView的高度就的就等于它内容的高度了）
```java
View childView = getChildAt(0);
childView.getMeasuredHeight() <= getScrollY() + getHeight();
```
判断滑动位置的地方，可以有两种方式：
第一种：实现OnTouchListener来监听是否滑动到最底部

```java
OnTouchListener onTouchListener=new OnTouchListener(){ 
    @Override 
    public boolean onTouch(View v, MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                if (childView  != null && childView .getMeasuredHeight() <= getScrollY() + getHeight()) {
                } else if (getScrollY() == 0) {
                }
            break;
        }
        return false;
    }
 }
```

**第二种：**重写ScrollView的onScrollChanged的方法，在onScrollChanged函数中判断

```java
public class myScrollView extends ScrollView{
    public myScrollView(Context context) {
        super(context);
    }
    public myScrollView(Context context, AttributeSet attributeSet){
        super(context,attributeSet);
    }
    @Override
    protected void onScrollChanged(int l, int t, int oldl, int oldt){
        View view = (View)getChildAt(getChildCount()-1);
        int d = view.getBottom();
        d -= (getHeight()+getScrollY());
        if(d==0){
            //you are at the end of the list in scrollview
            //do what you wanna do here
        }
        else
            super.onScrollChanged(l,t,oldl,oldt);
    }
}
```



### 10.ExpandableListView的Adapter怎么写

http://www.codexiu.cn/android/blog/32105/

http://blog.csdn.net/wangkeke1860/article/details/46477361

MyExpandableListViewAdapter适配器，适配器中有两个关键方法，分别是getGroupView（类似于ListView的getView方法）,每次加载组列表时都会执行该方法重新绘制页面；另一个是getChildView,当展开分组时会调用此方法来绘制当前分组的子项，值得注意的是，当用户点击某个分组时，ExpandableListView页面的其他分组也会重新绘制（即每次都会重新绘制所有的分组）；下面贴出MainActivity.java的代码，关键部分已经做了注释，很容易理解。



获取组的个数 getGroupCount( )

获取指定组中的子元素个数 getChildrenCount(intgroupPosition)

获取指定组中的数据 getGroup（int groupPosition）

获取指定组中的指定子元素数据。getChild(intgroupPosition,intchildPosition)

获取指定组的ID，这个组ID必须是唯一的 getGroupId(intgroupPosition)

获取指定组中的指定子元素ID getChildId(intgroupPosition,intchildPosition)

获取显示指定组的视图对象getGroupView(intgroupPosition,booleanisExpanded, View convertView, ViewGroup parent)

获取一个视图对象，显示指定组中的指定子元素数据。getChildView(intgroupPosition,intchildPosition,booleanisLastChild, View convertView, ViewGroup parent)



http://www.cnblogs.com/sczjhh/archive/2012/11/26/szh.html

android **expandablelistview**的实现最为关键是重写BaseExpandableListAdapter，而最为关键的两个方法是getChildView（），getGroupView（），注意在写这两个方法的时候最重要的是或者最容易出错的地方是注意返回的内容不要是 null 还有就是用红线所画的两个图片的切换就要在getGroupView（）方法里面进行判断



http://blog.csdn.net/to_be_designer/article/details/48028649
**ExpandableListView**是一种双层显示的View，为什么这么说呢，以我们的QQ为例，打开我们的QQ，我们首先会有一个联系人的分组，点击分组我们会看到分组内联系人的列表。这就是一种双层显示的View。ExpandableListView相当于两层的ListView嵌套。

ExpandableListView的使用与ListView类似，只不过ExpandableListView在创建Adapter的时候需要重写两倍的ListView重写的方法（原因就是：ExpandableListView相当于两层的ListView嵌套）。ExpandableListView在创建Adapter需要继承BaseExpandableListAdapter。



### 11. 如何让两个TextView在一个RelativeLayout水平居中显示

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent">
    <TextView
        android:id="@+id/name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerVertical="true"
        android:layout_toLeftOf="@id/center"
        android:layout_marginRight="10dp"
        android:textSize="30sp"
        android:text="name"
        android:background="@android:color/holo_green_light"
        />
    <TextView
        android:id="@+id/center"
        android:layout_width="1dp"
        android:layout_height="20dp"
        android:layout_centerInParent="true"
        android:background="@android:color/holo_red_light"
        />
    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerVertical="true"
        android:layout_toRightOf="@id/center"
        android:layout_marginLeft="10dp"
        android:text="title"
        android:textSize="30sp"
        android:background="@android:color/holo_green_light"
        />
</RelativeLayout>
```











