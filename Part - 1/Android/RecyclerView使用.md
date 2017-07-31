## RecyclerView使用

- **添加分割线**
- **添加点按效果**
- **列表动画**
- **改变某个数据保持当前位置**
- **添加头部尾部**
- **列表分组**
- **各种效果集成Demo**
- **更灵活的RecyclerView**



### 使用步骤

* xml 配置

* Activity 代码

  ```java
  public class Main extends Activity {  
      @Override  
      protected void onCreate(Bundle savedInstanceState) {  
          // TODO Auto-generated method stub  
          super.onCreate(savedInstanceState);  
          setContentView(R.layout.activity_main);  
          // 获取RecyclerView对象  
          final RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recycler_view);  
          // 创建线性布局管理器（默认是垂直方向）  
          final LinearLayoutManager layoutManager = new LinearLayoutManager(this);  
          // 为RecyclerView指定布局管理对象  
          recyclerView.setLayoutManager(layoutManager);  
          // 创建Adapter  
          final SampleRecyclerAdapter sampleRecyclerAdapter = new SampleRecyclerAdapter();  
          // 填充Adapter  
          recyclerView.setAdapter(sampleRecyclerAdapter);  
      }  
  }  
  ```

* Adapter 代码

  ```java
  public class NormalAdapter extends RecyclerView.Adapter<NormalAdapter.VH> {

      private List<ObjectModel> mDatas;
      public NormalAdapter(List<ObjectModel> data) {
          this.mDatas = data;
      }
  	// 用于创建控件
      @Override
      public NormalAdapter.VH onCreateViewHolder(ViewGroup parent, int viewType) {
          View v = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_1, parent, false);
          return new VH(v);
      }
  	// 为控件设置数据
      @Override
      public void onBindViewHolder(NormalAdapter.VH holder, int position) {
          // 获取当前 item 中显示的数据
          ObjectModel model = mDatas.get(position);
          // 设置要显示的数据
          holder.number.setText(model.number + "");
          holder.title.setText(model.title);
        
          holder.itemView.setOnClickListener(new View.OnClickListener() {
              @Override
              public void onClick(View view) {
                  //item 点击事件
              }
          });
      }

      @Override
      public int getItemCount() {
          return mDatas.size();
      }

      public static class VH extends RecyclerView.ViewHolder{
          public final TextView title;
          public final TextView number;
          public VH(View itemView) {
              super(itemView);
              title = (TextView) itemView.findViewById(R.id.title);
              number = (TextView) itemView.findViewById(R.id.number);
          }
      }
  }

  ```

### 布局

核心关键在于 [RecyclerView.LayoutManager](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html) 类，RecyclerView 在使用过程中要比 ListView 多一个 setLayoutManager 步骤，这个 LayoutManager 就是用于控制 RecyclerView 最终的展示效果的。而 LayoutManager 只是一个抽象类而已，系统已经为提供了三个相关的实现类 **LinearLayoutManager（线性布局效果）**、**GridLayoutManager（网格布局效果）**、**StaggeredGridLayoutManager（瀑布流布局效果）**。如果想用 RecyclerView 来实现自己 DIY 的效果，则继承实现自己的 LayoutManager，并重写相应的方法，而不应该想着去改写 RecyclerView。

### 空数据处理

ListView 提供了 setEmptyView 这个 API 来让我们处理 Adapter 中数据为空的情况，只需轻轻一 set 就能搞定一切。代码设置和效果如下

```java
mListView = (ListView) findViewById(R.id.listview);
mListView.setEmptyView(findViewById(R.id.empty_layout));//设置内容为空时显示的视图
```

而 RecyclerView 并没有提供此类 API

### HeaderView 和 FooterView

在 ListView 的设计中，存在着 HeaderView 和 FooterView 两种类型的视图，并且系统也提供了相应的 API 来让我们设置

![](http://upload-images.jianshu.io/upload_images/912181-3cb42e73c46bfc46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方法一：在 Adapter 中提供三种类型（Header，Footer以及普通Item）的 Type 和 View，但是这种方法写起来很麻烦，对 Adapter 的影响很大，改动的代码量多并且也容易产生BUG。

方法二：鸿洋老师的解决方案：[优雅的为RecyclerView添加HeaderView和FooterView](http://blog.csdn.net/lmj623565791/article/details/51854533) 。他的实现思路是通过装饰者模式来扩充 Adapter 的功能，从而实现添加 HeaderView 和 FooterView，并且不影响 Adapter 的编写工作，牛逼的是还能支持多个 HeaderView 和 FooterView （虽然我暂时想不到有什么应用场景，哈哈，不过先记着，以后说不定有用）。

### 局部刷新

在 ListView 中，刷新可用 notifyDataSetChanged() 。在更新了 ListView 的数据源后，需要通过 Adapter 的 notifyDataSetChanged 来通知视图更新变化，这样做比较的好处就是调用简单，坏处就是它会**重绘每个 Item**，但实际上并不是每个 Item 都需要重绘。最常见的，例如：朋友圈点赞，点赞只是更新当前点赞的Item，并不需要每个 Item 都更新。然而 ListView 并没有提供局部刷新刷新某个 Item 的 API ，同样自己自足。

[RecyclerView.Adapter](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.Adapter.html)	 则我们提供了 notifyItemChanged 用于更新单个 Item View 的刷新，我们可以省去自己写局部更新的工作。

![img](http://upload-images.jianshu.io/upload_images/912181-b8051dce7450291e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 动画

推荐一个RecyclerView的动画库（[recyclerview-animators](https://github.com/wasabeef/recyclerview-animators)）

RecyclerView自带添加、删除动画，而ListView则需添加额外的代码才能实现。
**删除调用RecyclerView 的 adapter 的 notifyItemRemoved**
**添加调用RecyclerView 的 adapter 的 notifyItemInserted**



RecyclerView 则为我们提供了很多基本的动画 API ，如下方的**增删移改**

![img](http://upload-images.jianshu.io/upload_images/912181-af00cdaa75008b0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



RecyclerView.Adapter 和 BaseAdapter 相比，额外提供了一下这些方法：

```java
// 数据发生了改变，那调用这个方法，传入改变对象的位置。
public final void notifyItemChanged(int position);
// 可以刷新从positionStart开始itemCount数量的item了
public final void notifyItemRangeChanged(int positionStart, int itemCount);
// 添加，传入对象的位置。
public final void notifyItemInserted(int position);
// 删除，传入对象的位置。
public final void notifyItemRemoved(int position);
// 对象从fromPosition移动到toPosition 
public final void notifyItemMoved(int fromPosition, int toPosition); 
// 批量添加 
public final void notifyItemRangeInserted(int positionStart, int itemCount);
// 批量删除
public final void notifyItemRangeRemoved(int positionStart, int itemCount);
```



### Item拖拽和滑动删除

ItemTouchHelper 是系统为我们提供的一个用于滑动和删除 RecyclerView 条目的工具类，用起来也是非常简单的，大致两步：

- 创建 ItemTouchHelper 实例，同时实现 ItemTouchHelper.SimpleCallback 中的抽象方法，用于初始化 ItemTouchHelper
- 调用 ItemTouchHelper 的 attachToRecyclerView 方法关联上 RecyclerView 即可

```java
itemTouchHelper.attachToRecyclerView(recyclerview);
```

ItemTouchHelper 用于实现 RecyclerView Item 拖曳效果的类，主要重写的是 getMovementFlags 、 onMove 、 onSwiped 三个抽象方法，getMovementFlags 用于告诉系统，RecyclerView 到底是支持滑动还是拖曳。就是表示着同时支持上下左右四个方向的拖曳和左右两个方向的滑动效果。如果是滑动，则 onSwiped 会被回调，如果是拖曳 onMove 会被回调，我们再到其中实现相应的业务操作即可。

### 自动加载更多

要想实现滑动到列表某处时自动加载下一页（比如剩最后两个item时），可以通过对 RecyclerView 设置滑动监听，获取当前显示的最后一个 item在适配器中的位置，如果该 item 的位置小于或等于适配器 item 总个数减2，就加载下一页数据。

```java
//recyclerview滚动监听
	recyclerview.addOnScrollListener(new RecyclerView.OnScrollListener() {
        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            //0：当前屏幕停止滚动；1时：屏幕在滚动 且 用户仍在触碰或手指还在屏幕上；2时：随用户的操作，屏幕上产生的惯性滑动；
            // 滑动状态停止并且剩余少于两个item时，自动加载下一页
            if (newState == RecyclerView.SCROLL_STATE_IDLE
                    && lastVisibleItem +2>=mLayoutManager.getItemCount()) {
                new GetData().execute("http://gank.io/api/data/福利/10/"+(++page));
            }
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            //获取加载的最后一个可见视图在适配器的位置。
            lastVisibleItem = mLayoutManager.findLastVisibleItemPosition();
        }
    });
```

StaggeredGridLayoutManager 因为 item 的位置是交错的，findLastVisibleItemPosition() 方法返回的是一个数组，所以我们得先判断下这个数组的最大值，上面的 onScrolled 方法写成这样

```java
int[] positions= mLayoutManager.findLastVisibleItemPositions(null);
lastVisibleItem =Math.max(positions[0],positions[1]);//根据StaggeredGridLayoutMa
```



### 监听 Item 的事件

ListView 为我们准备了几个专门用于监听 Item 的回调接口，如单击、长按、选中某个 Item 等

![img](http://upload-images.jianshu.io/upload_images/912181-eb82b15ab33d5bc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

缺点：比如setOnItemClickListener ，在不添加 HeaderView 和 FooterView 的时候，可以通过回调参数中的 position 去拿到数据源列表中对应 Item 的数据。

![img](http://upload-images.jianshu.io/upload_images/912181-cc6951855859a171.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是，添加了 HeaderView 和 FooterView 之后就不一样了，ListView 会把 HeaderView 和 FooterView 算入 position 内。假设原先在 onItemClick 回调方法中写了 mDataList.get(position) 这样的业务代码并且这段代码运行良好许久，但在某天你突然加了个 HeaderView 后，这段代码就开始变的有问题了，此时因为 HeaderView 占用的位置算入了 position 之内，所以 position 的最大值实际上是大于 mDataList 包含元素的个数值的，因此代码会报数组越界的错误。当然，我们可以去避免这种问题的发生，就是不通过 position 来获取数据，二是通过回调方法中的 id 。

![img](http://upload-images.jianshu.io/upload_images/912181-477b7947bf87f374.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样就不会受到添加 HeaderView 和 FooterView 的影响了，这个 id 的值就是来自我们编写好的 Adapter 中的 getItemId 函数中返回的 id，使用 IDE 生成此函数时，默认是返回0，需要将 position 作为 Item 的 id 返回。

![img](http://upload-images.jianshu.io/upload_images/912181-83bc80b68fdda92a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

并同时在 onItemClick 中判断 id 是否值为 -1，因为 HeaderView 和 FooterView 的返回值就是 -1。前面讲到我并不大喜欢 setOnItemClickListener 这种设计，除了由这些因素的影响外，更关键的是个人认为针对 Item 的事件实际上写在 getView 方法中会更加合适，如 setOnItemClickListener 我更喜欢用在 getView 中为每个 convertView 设置 setOnClickListener 的方式去取代它。

而再来看看 RecyclerView ，它并没有像 ListView 提供太多关于 Item 的某种事件监听，唯一的就是 addOnItemTouchListener

![img](http://upload-images.jianshu.io/upload_images/912181-4801aa03c89c279f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

API 的名字言简意赅，就是监听 Item 的触摸事件。如果想要拥有 ListView 那样监听某个 Item 的某个操作方法，可以看看这篇文章 [RecyclerView无法添加onItemClickListener最佳的高效解决方案](http://blog.csdn.net/liaoinstan/article/details/51200600) ，作者的实现思路就是通过 addOnItemTouchListener 和系统提供的 GestureDetector 手势判断结合实现的。

但我认为根本没有必要费这么大劲对外暴露这个接口，因为我们完全可以把点击事件的实现写在Adapter的`onBindViewHolder()`中，不暴露出来。具体方法就是通过：

```java
public void onBindViewHolder(VH holder, int position) {
    holder.itemView.setOnClickListener(...);
}
```



### Click and LongClick

不过一个挺郁闷的地方就是，系统没有提供ClickListener和LongClickListener。 
不过我们也可以自己去添加，只是会多了些代码而已。 
实现的方式比较多，你可以通过mRecyclerView.addOnItemTouchListener去监听然后去判断手势， 
当然你也可以通过adapter中自己去提供回调，这里我们选择后者，前者的方式，大家有兴趣自己去实现。

那么代码也比较简单：

```java
class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder>{
//...
    public interface OnItemClickLitener
    {
        void onItemClick(View view, int position);
        void onItemLongClick(View view , int position);
    }

    private OnItemClickLitener mOnItemClickLitener;

    public void setOnItemClickLitener(OnItemClickLitener mOnItemClickLitener)
    {
        this.mOnItemClickLitener = mOnItemClickLitener;
    }

    @Override
    public void onBindViewHolder(final MyViewHolder holder, final int position)
    {
        holder.tv.setText(mDatas.get(position));

        // 如果设置了回调，则设置点击事件
        if (mOnItemClickLitener != null)
        {
            holder.itemView.setOnClickListener(new OnClickListener()
            {
                @Override
                public void onClick(View v)
                {
                    int pos = holder.getLayoutPosition();
                    mOnItemClickLitener.onItemClick(holder.itemView, pos);
                }
            });

            holder.itemView.setOnLongClickListener(new OnLongClickListener()
            {
                @Override
                public boolean onLongClick(View v)
                {
                    int pos = holder.getLayoutPosition();
                    mOnItemClickLitener.onItemLongClick(holder.itemView, pos);
                    return false;
                }
            });
        }
    }
//...
}
```

adapter中自己定义了个接口，然后在onBindViewHolder中去为holder.itemView去设置相应 
的监听最后回调我们设置的监听。

Activity中去设置监听：

```java
mAdapter.setOnItemClickLitener(new OnItemClickLitener(){
    @Override
    public void onItemClick(View view, int position)
    {
        Toast.makeText(HomeActivity.this, position + " click",
        Toast.LENGTH_SHORT).show();
    }

    @Override
    public void onItemLongClick(View view, int position)
    {
        Toast.makeText(HomeActivity.this, position + " long click",
        Toast.LENGTH_SHORT).show();
        mAdapter.removeData(position);
    }
 });
```



### 嵌套滚动机制

Touch 事件在进行分发的时候，由父 View 向它的子 View 传递，一旦某个子 View 开始接收进行处理，那么接下来所有事件都将由这个 View 来进行处理，它的 ViewGroup 将不会再接收到这些事件，直到下一次手指按下。而嵌套滚动机制（NestedScrolling）就是为了弥补这一机制的不足，为了让子 View 能和父 View 同时处理一个 Touch 事件。关于嵌套滚动机制（NestedScrolling），实现上相对是比较复杂的，此处就不去拓展说明，其关键在于 **NestedScrollingChild** 和 **NestedScrollingParent** 两个接口，以及系统对这两个接口的实现类 **NestedScrollingChildHelper** 和 **NestedScrollingParentHelper** 。

CollapsingToolbarLayout 、AppBarLayout 和 RecyclerView







[优雅的为RecyclerView添加HeaderView和FooterView](http://blog.csdn.net/lmj623565791/article/details/51854533) 

 [RecyclerView无法添加onItemClickListener最佳的高效解决方案](http://blog.csdn.net/liaoinstan/article/details/51200600) 













































