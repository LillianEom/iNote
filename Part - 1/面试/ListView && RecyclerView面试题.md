# ListView &&  RecyclerView

### 1. 从ListView优化谈到RecyclerView，深入分析一下recyclerview特点

RecyclerView 一个特点就是，将 layout 抽象成了一个 LayoutManager，RecylerView 不负责子 View 的布局， 我们可以自定义 LayoutManager 来实现不同的布局效果， 目前只提供了LinearLayoutManager。 LinearLayoutManager 可以指定方向，默认是垂直， 可以指定水平， 这样就轻松实现了水平的 ListView。

1、RecyclerView不关心布局，需要使用setLayoutManager进行设置布局。

2、RecyclerView不关心分割线，因此分割线需要我们自己想办法设置。

3、RecyclerView不关心Item的点击事件与动画效果，需要自己编写接口进行监听。

4、RecyclerView仅关注View的回收与复用。

相关的类：

1、RecyclerView.Adapter：托管数据集合，为每个Item创建视图；

2、RecyclerView.ViewHolder：承载Item视图的子视图；

3、RecyclerView.LayoutManager：负责Item视图的布局；

4、RecyclerView.ItemDecoration：为每个Item视图添加分割线；

5、RecyclerView.ItemAnimator：负责添加、删除数据时的动画效果；

### 2. ListView 优化方法

* **重用converView**： 切忌每次 getView() 都新建，通过复用converview来减少不必要的view的创建，另外Infalte操作会把xml文件实例化成相应的View实例，属于IO操作，是耗时操作。
* **减少findViewById()操作**： 将xml文件中的元素封装成 viewholder静态类，通过converview的setTag和getTag方法将view与相应的holder对象绑定在一起，避免不必要的findviewbyid操作
* **利用好 ViewType**，例如你的 ListView 中有几个类型的 Item，需要给每个类型创建不同的 View，这样有利于 ListView 的回收，当然类型不能太多；尽量让 ItemView 的 Layout 层次结构简单，这是所有 Layout 都必须遵循的；
* **避免在 getView 方法中做耗时的操作**: 例如加载本地 Image 需要载入内存以及解析 Bitmap ，都是比较耗时的操作，如果用户快速滑动listview，会因为getview逻辑过于复杂耗时而造成滑动卡顿现象。用户滑动时候不要加载图片，待滑动完成再加载，可以使用这个第三方库[glide](https://github.com/bumptech/glide)
* 善用自定义 View，自定义 View 可以有效的减小 Layout 的层级，而且对绘制过程可以很好的控制；Item的布局层次结构尽量简单，避免布局太深或者不必要的重绘
* **尽量能保证 Adapter 的 hasStableIds() 返回 true**，这样在 notifyDataSetChanged() 的时候，如果 id 不变，ListView 将不会重新绘制这个 View，达到优化的目的；
* **每个 Item 不能太高**，特别是不要超过屏幕的高度，可以参考 Facebook 的优化方法，把特别复杂的 Item 分解成若干小的 Item，特别推荐看一下这个文章：https://code.facebook.com/posts/879498888759525/fast-rendering-news-feed-on-android/
* **ListView 中元素避免半透明**： 半透明绘制需要大量乘法计算，在滑动时不停重绘会造成大量的计算，在比较差的机子上会比较卡。 在设计上能不半透明就不不半透明。实在要弄就把在滑动的时候把半透明设置成不透明，滑动完再重新设置成半透明。
* **尽量开启硬件加速**： 硬件加速提升巨大，避免使用一些不支持的函数导致含泪关闭某个地方的硬件加速。当然这一条不只是对 ListView。
* **在一些场景中，ScollView内会包含多个ListView，可以把listview的高度写死固定下来。** 由于ScollView在快速滑动过程中需要大量计算每一个listview的高度，阻塞了UI线程导致卡顿现象出现，如果我们每一个item的高度都是均匀的，可以通过计算把listview的高度确定下来，避免卡顿现象出现**使用 RecycleView 代替listview**： 每个item内容的变动，listview都需要去调用**notifyDataSetChanged**来更新全部的item，太浪费性能了。
* **使用 RecycleView 代替**。 ListView 每次更新数据都要 notifyDataSetChanged()，有些太暴力了。RecycleView可以实现当个item的局部刷新，并且引入了增加和删除的动态效果，在性能上和定制上都有很大的改善

有时候，需要从根本上考虑，是否真的要使用 ListView 来实现你的需求，或者是否有其他选择？

### 3. RecycleView与listview的区别

http://www.cnblogs.com/littlepanpc/p/4497290.html

http://www.jianshu.com/p/16712681731e

http://www.jianshu.com/p/f592f3715ae2

**RecyclerView与老前辈ListView的不同点，主要在于以下几个特性：**

Adapter中的ViewHolder模式 - 对于ListView来说，通过创建ViewHolder来提升性能并不是必须的。因为ListView并没有严格的ViewHolder设计模式。但是在使用RecyclerView的时候，Adapter必须实现至少一个ViewHolder，必须遵循ViewHolder设计模式。



定制Item条目 - ListView只能实现垂直线性排列的列表视图，与之不同的是，RecyclerView可以通过设置RecyclerView.LayoutManager来定制不同风格的视图，比如水平滚动列表或者不规则的瀑布流列表。



Item动画 - 在ListView中没有提供任何方法或者接口，方便开发者实现Item的增删动画。相反地，可以通过设置RecyclerView的RecyclerView.ItemAnimator来为条目增加动画效果。



设置数据源 - 在LisView中针对不同数据封装了各种类型的Adapter，比如用来处理数组的ArrayAdapter和用来展示Database结果的CursorAdapter。相反地，在RecyclerView中必须自定义实现RecyclerView.Adapter并为其提供数据集合。



设置条目分割线 - 在ListView中可以通过设置android:divider属性来为两个Item间设置分割线。如果想为RecyclerView添加此效果，则必须使用RecyclerView.ItemDecoration，这种实现方式不仅更灵活，而且样式也更加丰富。



设置点击事件 - 在ListView中存在AdapterView.OnItemClickListener接口，用来绑定条目的点击事件。但是，很遗憾的是在RecyclerView中，并没有提供这样的接口，不过，提供了另外一个接口RcyclerView.OnItemTouchListener，用来响应条目的触摸事件。

### 4. 瀑布流如何实现，不使用recyclerview如何实现瀑布流


瀑布流与滚动方向

前面已经介绍过，RecyclerView实现瀑布流，可以通过一句话设置：recycler.setLayoutManager(new StaggeredGridLayoutManager(2, VERTICAL))就可以了。

其中 StaggeredGridLayoutManager 第一个参数表示列数，就好像 GridView 的列数一样，第二个参数表示方向，可以很方便的实现横向滚动或者纵向滚动。使用 demo 可以查看：Github 【RecyclerView简单使用】
**RecycleView实现瀑布流：**

http://www.jianshu.com/p/6a9f188cfe42
**ListView实现瀑布流：**[http://blog.csdn.net/guolin_blog/article/details/46361889](http://blog.csdn.net/guolin_blog/article/details/46361889)

### 5. 一个listview里面的item怎么优化,如果item的layout不同你要怎么优化

http://www.bkjia.com/Androidjc/853647.html

利用好 View Type，例如你的 ListView 中有几个类型的 Item，需要给每个类型创建不同的 View，这样有利于 ListView 的回收，当然类型不能太多；

在重写ListView的BaseAdapter时，我们常常在getView()方法中复用convertView，优化ListView以提高性能。convertView在Item为单一的同种类型布局时，能够回收并重用，但是多个Item布局类型不同时，convertView的回收和重用会出现问题。比如有些行为纯文本，有些行则是图文混排，这里纯文本行为一类布局，图文混排的行为第二类布局。

1）重写 getViewTypeCount() – 该方法返回多少个不同的布局

2）重写 getItemViewType(int) – 根据position返回相应的Item

3）根据view item的类型，在getView中创建正确的convertView

android的listview怎添加多个格式与布局不一样item,共有4个item？

每个item的data部分里，要有一个type字段，在适配器的getView方法里，根据type的类型，对应的inflate不用的布局layout即可******10.6在ListView的每个item中如果可能出现的view都不一样，如何处理？**http://8840150.blog.51cto.com/8830150/1661140



http://blog.csdn.net/nightyk/article/details/45057777



多个类型的ViewType

当我们在Adapter中调用方法getView的时候，如果整个列表中的Item View如果有多种类型布局



我们继续使用convertView来将数据从新填充貌似不可行了，因为每次返回的convertView类型都不一样，无法重用。

Android在设计上的时候，也想到了这点。所以，在adapter中预留的两个方法。public int getItemViewType(int position) ;public int getViewTypeCount();

只需要重新这两个方法，设置一下ItemViewType的个数和判断方法，Recycler就能有选择性的给出不同的convertView了。



这种问题的本质就是：ListView的Item包含多个类型的布局，如何进行优化，使滑动时不出现卡顿。

简单说几点，后面大家的再补充：

每个Item布局尽量简单，View层级尽量浅，不要有过渡绘制出现

UI线程(即getView)中尽量少做事情，图片加载什么的，放到另一个线程去做，或者用第三方图片加载库

最基本的，使用ListView的ViewHolder模式



动态获取view种类数量的话是不是就不能使用viewHolder进行优化？



固定显示view如果不存在该种view就不显示的方法是否太耗内存

### 6. ListView的Adapter的getView具体是什么机制？

http://www.trinea.cn/android/android-listview-display-error-image-when-scroll/

http://blog.csdn.net/guolin_blog/article/details/44996879

1. Adapter.getView()

public View getView(int position, View convertView , ViewGroup parent){...}


这个方法就是用来获得指定位置要显示的View。官网解释如下：


Get a View that displays the data at the specified position in the data set. You can either create a View manually or inflate it from an XML layout file.




当要显示一个View就调用一次这个方法。这个方法是ListView性能好坏的关键。方法中有个convertView，这个是Android在为我们而做的缓存机制。


ListView中每个item都是通过getView返回并显示的，假如item有很多个，那么重复创建这么多对象来显示显然是不合理。因此，Android提供了Recycler，将没有正在显示的item放进RecycleBin，然后在显示新视图时从RecycleBin中复用这个View。



### 7. Recycler的工作原理大致如下：

假设屏幕最多能看到11个item，那么当第1个item滚出屏幕，这个item的View进入RecycleBin中，第12个要出现前，通过getView从回收站（RecycleBin）中重用这个View，然后设置数据，而不必重新创建一个View。
10.9 ListView在数据量很大图片很多的情况下怎么优化？假如一个图片，轻轻的向上滑动一丢丢，那么需要重绘吗？（什么鬼。。。）

### 8. 假如你要记录ListView滚动到的位置，要记录什么信息，view怎样获得坐标信息

http://www.cnblogs.com/trinea/archive/2012/04/11/2443093.html


记录listView滚动到的位置的坐标（推荐）

```java
public void onScrollStateChanged(AbsListView view, int scrollState) {     
  // 不滚动时保存当前滚动到的位置     
  if (scrollState == OnScrollListener.SCROLL_STATE_IDLE){           
    if (currentMenuInfo != null) {               
      scrolledX = listView.getScrollX();               
      scrolledY = listView.getScrollY();          
    }     
  }
}
```


记录listView显示在屏幕上的第一个item的位置



### 9. 了不了解Scrollview scrollview和ListView有什么相似点有什么不同 那如果这两个是继承关系 那应该是谁继承谁？

相同点：


这两种容器组件都可以通过竖向滚动的方式显示容器中的内容。


不同点：


ListView容器组件是用来显示一组相同类型的数据。

ScrollView组件可以直接让其子组件进行滚动显示。Android文档中特别提醒开发者，不要将一个ListView容器组件作为ScrollView容器组件的子组件，因为这样会破坏系统对ListView容器组件的性能优化。

### 10. 怎么实现一个部分更新的 ListView？

[http://blog.csdn.net/linglongxin24/article/details/53020164](http://blog.csdn.net/linglongxin24/article/details/53020164)

**方法一：**更新对应的View的内容这种方法先通过listView.getChildAt（position）拿到要更新的对应的item布局文件,然后再通过findViewById找到对应的控件进行设置。

```java
/在看见范围内才更新，不可见的滑动后自动会调用getView方法更新/    
  if (position >= firstVisiblePosition && position <= lastVisiblePosition) {
    /获取指定位置view对象/        
    View view = listView.getChildAt(position - firstVisiblePosition);      
    TextView textView = (TextView) view.findViewById(R.id.textView);
    textView.setText(datas.get(position));    
  }
```

**方法二：**通过ViewHolder去设置值通过Item找出对应的ViewHolder，然后通过ViewHolder去设置值

```java
if (position >= firstVisiblePosition && position <= lastVisiblePosition) {        
  /获取指定位置view对象/        
  View view = listView.getChildAt(position - firstVisiblePosition);        
  /通过ViewHolder找出缓存的对应控件/        
  TextView textView = CommonViewHolder.get(view, R.id.textView);        
  textView.setText(datas.get(position));    
}
```


**方法三：**调用一次getView( )方法这种方法是调用适配器对应的getView方法，用它里面的代码对界面进行刷新。这也是google在IO大会上推荐的做法

```java
/在看见范围内才更新，不可见的滑动后自动会调用getView方法更新/
if (position >= firstVisiblePosition && position <= lastVisiblePosition) {
    /获取指定位置view对象/
    View view = listView.getChildAt(position - firstVisiblePosition);    
    commonAdapter.getView(position, view, listView);
}
```

### 11. 怎么实现ListView多种布局？

[http://www.jianshu.com/p/b7091e43c971](http://www.jianshu.com/p/b7091e43c971)在Adapter类中，实现getItemViewType和getViewTypeCount方法，需要多个ViewHolder 和Item布局getViewTypeCount()返回布局种类的数量
getItemViewType(int position)返回当前Item布局类型，可以在这里实现我们的逻辑getView中我们根据当前Item布局类型，加载对应的布局。

### 12. ListView与数据库绑定的实现？

一般两种方式，第一种将数据库读出的数据，放入到HashMap或者List对象中，然后进行绑定第二种使用SimpleCursorAdapter

### 13. Item复用--出现错乱---解决错乱

[http://blog.csdn.net/guolin_blog/article/details/45586553](http://blog.csdn.net/guolin_blog/article/details/45586553)

编写adapter时一般会重写他的个getView（）这个方法

![](http://om4rextnc.bkt.clouddn.com/17-8-4/56272158.jpg)

初始化时ListView先请求一个type1视图(getView)，这时的convertView在getView中是空(null)。当item1滑动出屏幕，并且一个新的项目从屏幕低端上来时，ListView再请求一个type1视图。这个时候convertView此时不是空值了，它的值是item1。你只需设定新的数据然后返回convertView，不必重新创建一个视图。如下图所示

![](http://om4rextnc.bkt.clouddn.com/17-8-4/46069470.jpg)

为什么会错乱明白了item复用的原理后我们就来探究一下错乱的原因吧，通过查看源码我们可以得知原来AbListView中获取getView()和滑动操作是异步进行的，其中滑动操作在一个FlingRunnable的支线程中运行，所以这就导致了在ListView在滑动时可能已经滑动到了第十行，但可能第二行的数据这时就被直接使用了，这就是导致数据加载错乱的根本原因。由于滑动和加载是异步进行的，这样就会出现一些意想不到的问题：a. 行item图片显示重复b. 行item图片显示错乱c. 行item图片显示闪烁**解决错乱**可以给图片设置tag来避免数据错乱

![](http://om4rextnc.bkt.clouddn.com/17-8-4/61240885.jpg)