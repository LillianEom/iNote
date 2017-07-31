## View的生命周期

![](http://om4rextnc.bkt.clouddn.com/17-7-30/74871651.jpg)

## 自定义View绘制流程函数调用链(简化版)

![](http://om4rextnc.bkt.clouddn.com/17-7-30/21474915.jpg)

注意：

- invalidate/postInvalidate 只会触发 draw；
- requestLayout，会触发 measure、layout 和 draw 的过程；
- 它们都是走的 `scheduleTraversals -> performTraversals`，用不同的标记位来进行区分；
- resume 会触发 invalidate；
- dispatchDraw 是用来绘制 child 的，发生在自己的 onDraw 之后，child 的 draw 之前

## 自定义View分类

1. 自定义View

   继承自 View、TextureView 或 SurfaceView，**不包含子View** 然后重写核心的回调方法，以 View 为例，按需复写其构造、onMeasure、onLayout、onTouchEvent、onDraw、onAttachedToWindow、onDetachedFromWindow 等方法。

2. 自定义ViewGroup

   通过Android的基础控件(TextView、CheckBox、Button、ProgressBar等)组合而成，多数继承自 ViewGroup 或各种Layout，**包含有子View**。比如试题控件（TextView+VideoGroup）、下拉刷新、瀑布流控件、带左/右滑功能的控件、视频控件等，这种自定义View的难点在于程序的逻辑处理；

## 核心知识点

### View、SufaceView、TextureView的区别

**View：**

普通的 View，与宿主窗口共享同一个绘图表面，UI 在主线程中绘制，在有无硬件加速的情况下都能工作（没有硬件加速的情况下，canvas的有些方法会失效）；

**SurfaceView：**

​继承自 View，绘制和显示效率高，因为拥有独立的绘图表面，UI在一个独立的线程中进行绘制，不会占用主线程的资源。SurfaceView 的使用和普通的View不一样，需要结合 SurfaceHodler 一起使用。因为和宿主窗口不是共享同一个绘图表面的原因，笔者在实际使用 SurfaceView 的过程中发现对其做动画操作会达不到想要的效果（一坨黑色）；

**TextureView：**

​继承自 View，与 SurfaceView 相比，TextureView不会创建一个单独的绘图表面，这使得它可以像一般的View一样执行一些变换操作，比如移动、动画等等，但 TextureView 必须在硬件加速开启的窗口中才能正常工作；

### 几个重要函数

- **1、Measure测量一个View的大小**
- **2、Layout摆放一个View的位置**
- **3、Draw画出View的显示内容**
  其中 measure 和 layout 方法都是 final 的，无法重写，虽然 draw 不是 final 的，但是也不建议重写该方法。
  这三个方法都已经写好了 View 的逻辑，如果我们想实现自身的逻辑，而又不破坏 View 的工作流程，可以重写 onMeasure、onLayout、onDraw 方法。

1. **构造函数**

   构造函数是 View 的入口，可以用于**初始化一些的内容，和获取自定义属性**。

   View 的构造函数有四种重载分别如下:

   ```Java
   public void SloopView(Context context) {}
   public void SloopView(Context context, AttributeSet attrs) {}
   public void SloopView(Context context, AttributeSet attrs, int defStyleAttr) {}
   public void SloopView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {}
   ```

2. **onMeasure(测量View的大小)**

   测量 View 大小使用的是onMeasure函数，我们可以从 onMeasure 的两个参数中取出宽高的相关数据：

   ```Java
   @Override
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       int widthsize  MeasureSpec.getSize(widthMeasureSpec);      //取出宽度的确切数值
       int widthmode  MeasureSpec.getMode(widthMeasureSpec);      //取出宽度的测量模式
       
       int heightsize  MeasureSpec.getSize(heightMeasureSpec);    //取出高度的确切数值
       int heightmode  MeasureSpec.getMode(heightMeasureSpec);    //取出高度的测量模式
   }
   ```

   从上面可以看出 onMeasure 函数中有 widthMeasureSpec 和 heightMeasureSpec 这两个 int 类型的参数， 毫无疑问他们是和宽高相关的， **但它们其实不是宽和高， 而是由宽、高和各自方向上对应的测量模式来合成的一个值：**

   **测量模式一共有三种， 被定义在 Android 中的 View 类的一个内部类View.MeasureSpec中：**

   | UNSPECIFIED | 00   | 默认值，父控件没有给子view任何限制，子View可以设置为任意大小。    |
   | ----------- | ---- | -------------------------------------- |
   | EXACTLY     | 01   | 表示父控件已经确切的指定了子View的大小。                 |
   | AT_MOST     | 10   | 表示子View具体大小没有尺寸限制，但是存在上限，上限一般为父View大小。 |

   #### 注意：

   **如果对 View 的宽高进行修改了，不要调用  super.onMeasure( widthMeasureSpec, heightMeasureSpec); 要调用 setMeasuredDimension( widthsize, heightsize); 这个函数。**

3. **onSizeChanged(确定View大小)**

   这个函数在视图大小发生改变时调用。

   onSizeChanged 如下：

   ```Java
   @Override
   protected void onSizeChanged(int w, int h, int oldw, int oldh) {
       super.onSizeChanged(w, h, oldw, oldh);
   }
   ```

4. **onLayout**

   **确定布局的函数是 onLayout，它用于确定子  View 的位置，在自定义V iewGroup 中会用到，他调用的是子 View 的 layout 函数。**

   在自定义 ViewGroup 中，onLayout一般是循环取出子 View，然后经过计算得出各个子View位置的坐标值，然后用以下函数设置子 View 位置。

   ```Java
   child.layout(l, t, r, b);
   ```

   四个参数分别为：

   | 名称   | 说明                | 对应的函数        |
   | ---- | ----------------- | ------------ |
   | l    | View左侧距父View左侧的距离 | getLeft();   |
   | t    | View顶部距父View顶部的距离 | getTop();    |
   | r    | View右侧距父View左侧的距离 | getRight();  |
   | b    | View底部距父View顶部的距离 | getBottom(); |

5. **onDraw**

   onDraw是实际绘制的部分，也就是我们真正关心的部分，使用的是Canvas绘图。

   ```Java
   @Override
   protected void onDraw(Canvas canvas) {
       super.onDraw(canvas);
   }
   ```
   在onDraw()中就有一个参数，该参数就是Canvas canvas对象，使用这个对象即可进行绘图操作；而如果在其他地方，通常需要使用代码创建一个Canvas对象：
   ```Java
   Canvas canvas = new Canvas(bitmap);
   ```
   之所以要传入一个bitmap,是因为传进来的bitmap与通过这个bitmap创建的Canvas画布是紧紧联系在一起的，这个过程称之为装载画布。在View类的onDraw()方法中，我们通过下面的代码，让canvas与bitmap发生直接的联系：

   ```Java
      canvas.drawBitmap(bitmap, 0, 0, null);
   ```
   然后将bitmap装载到另外一个Canvas对象中：

   ```Java
       Canvas mCanvas = new Canvas(bitmap);
   ```
   通过mCanvas将绘制效果作用在了bitmap上，再通过invalidate()刷新的时候，我们就会发现通过onDraw()方法画出来的bitmap已经发生了改变。













































