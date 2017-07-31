## ListView 中 RecycleBin机制



RecycleBin机制是ListView能够实现成百上千条数据都不会OOM最重要的一个原因。RecycleBin是AbsListView的一个内部类。

```java
/** 
 * The RecycleBin facilitates reuse of views across layouts. The RecycleBin 
 * has two levels of storage: ActiveViews and ScrapViews. ActiveViews are 
 * those views which were onscreen at the start of a layout. By 
 * construction, they are displaying current information. At the end of 
 * layout, all views in ActiveViews are demoted to ScrapViews. ScrapViews 
 * are old views that could potentially be used by the adapter to avoid 
 * allocating views unnecessarily. 
 *  
 * @see android.widget.AbsListView#setRecyclerListener(android.widget.AbsListView.RecyclerListener) 
 * @see android.widget.AbsListView.RecyclerListener 
 */  
class RecycleBin {  
    private RecyclerListener mRecyclerListener;  
  
    /** 
     * The position of the first view stored in mActiveViews. 
     */  
    private int mFirstActivePosition;  
  
    /** 
     * Views that were on screen at the start of layout. This array is 
     * populated at the start of layout, and at the end of layout all view 
     * in mActiveViews are moved to mScrapViews. Views in mActiveViews 
     * represent a contiguous range of Views, with position of the first 
     * view store in mFirstActivePosition. 
     */  
    private View[] mActiveViews = new View[0];  
  
    /** 
     * Unsorted views that can be used by the adapter as a convert view. 
     */  
    private ArrayList<View>[] mScrapViews;  
  
    private int mViewTypeCount;  
  
    private ArrayList<View> mCurrentScrap;  
  
    /** 
     * Fill ActiveViews with all of the children of the AbsListView. 
     *  
     * @param childCount 
     *            The minimum number of views mActiveViews should hold 
     * @param firstActivePosition 
     *            The position of the first view that will be stored in 
     *            mActiveViews 
     */  
    void fillActiveViews(int childCount, int firstActivePosition) {  
        if (mActiveViews.length < childCount) {  
            mActiveViews = new View[childCount];  
        }  
        mFirstActivePosition = firstActivePosition;  
        final View[] activeViews = mActiveViews;  
        for (int i = 0; i < childCount; i++) {  
            View child = getChildAt(i);  
            AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();  
            // Don't put header or footer views into the scrap heap  
            if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {  
                // Note: We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in  
                // active views.  
                // However, we will NOT place them into scrap views.  
                activeViews[i] = child;  
            }  
        }  
    }  
  
    /** 
     * Get the view corresponding to the specified position. The view will 
     * be removed from mActiveViews if it is found. 
     *  
     * @param position 
     *            The position to look up in mActiveViews 
     * @return The view if it is found, null otherwise 
     */  
    View getActiveView(int position) {  
        int index = position - mFirstActivePosition;  
        final View[] activeViews = mActiveViews;  
        if (index >= 0 && index < activeViews.length) {  
            final View match = activeViews[index];  
            activeViews[index] = null;  
            return match;  
        }  
        return null;  
    }  
  
    /** 
     * Put a view into the ScapViews list. These views are unordered. 
     *  
     * @param scrap 
     *            The view to add 
     */  
    void addScrapView(View scrap) {  
        AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();  
        if (lp == null) {  
            return;  
        }  
        // Don't put header or footer views or views that should be ignored  
        // into the scrap heap  
        int viewType = lp.viewType;  
        if (!shouldRecycleViewType(viewType)) {  
            if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {  
                removeDetachedView(scrap, false);  
            }  
            return;  
        }  
        if (mViewTypeCount == 1) {  
            dispatchFinishTemporaryDetach(scrap);  
            mCurrentScrap.add(scrap);  
        } else {  
            dispatchFinishTemporaryDetach(scrap);  
            mScrapViews[viewType].add(scrap);  
        }  
  
        if (mRecyclerListener != null) {  
            mRecyclerListener.onMovedToScrapHeap(scrap);  
        }  
    }  
  
    /** 
     * @return A view from the ScrapViews collection. These are unordered. 
     */  
    View getScrapView(int position) {  
        ArrayList<View> scrapViews;  
        if (mViewTypeCount == 1) {  
            scrapViews = mCurrentScrap;  
            int size = scrapViews.size();  
            if (size > 0) {  
                return scrapViews.remove(size - 1);  
            } else {  
                return null;  
            }  
        } else {  
            int whichScrap = mAdapter.getItemViewType(position);  
            if (whichScrap >= 0 && whichScrap < mScrapViews.length) {  
                scrapViews = mScrapViews[whichScrap];  
                int size = scrapViews.size();  
                if (size > 0) {  
                    return scrapViews.remove(size - 1);  
                }  
            }  
        }  
        return null;  
    }  
  
    public void setViewTypeCount(int viewTypeCount) {  
        if (viewTypeCount < 1) {  
            throw new IllegalArgumentException("Can't have a viewTypeCount < 1");  
        }  
        // noinspection unchecked  
        ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];  
        for (int i = 0; i < viewTypeCount; i++) {  
            scrapViews[i] = new ArrayList<View>();  
        }  
        mViewTypeCount = viewTypeCount;  
        mCurrentScrap = scrapViews[0];  
        mScrapViews = scrapViews;  
    }  
  
}  
```

- RecycleBin当中使用mActiveViews这个数组来存储View，调用这个方法后就会根据传入的参数来将ListView中的指定元素存储到mActiveViews中。
- mActiveViews当中所存储的View，一旦被获取了之后就会从mActiveViews当中移除，下次获取同样位置的时候将会返回null，所以mActiveViews不能被重复利用。
- addScrapView()用于将一个废弃的View进行缓存，该方法接收一个View参数，当有某个View确定要废弃掉的时候（比如滚动出了屏幕）就应该调用这个方法来对View进行缓存，RecycleBin当中使用mScrapV
- iews和mCurrentScrap这两个List来存储废弃View。
- getScrapView 用于从废弃缓存中取出一个View，这些废弃缓存中的View是没有顺序可言的，因此getScrapView()方法中的算法也非常简单，就是直接从mCurrentScrap当中获取尾部的一个scrap view进行返回。
- 我们都知道Adapter当中可以重写一个getViewTypeCount()来表示ListView中有几种类型的数据项，而setViewTypeCount()方法的作用就是为每种类型的数据项都单独启用一个RecycleBin缓存机制。



最主要的几个方法提了出来。那么我们先来对这几个方法进行简单解读，这对后面分析ListView的工作原理将会有很大的帮助。

- **fillActiveViews()** 这个方法接收两个参数，第一个参数表示要存储的view的数量，第二个参数表示ListView中第一个可见元素的position值。RecycleBin当中使用mActiveViews这个数组来存储View，调用这个方法后就会根据传入的参数来将ListView中的指定元素存储到mActiveViews数组当中。
- **getActiveView()** 这个方法和fillActiveViews()是对应的，用于从mActiveViews数组当中获取数据。该方法接收一个position参数，表示元素在ListView当中的位置，方法内部会自动将position值转换成mActiveViews数组对应的下标值。需要注意的是，mActiveViews当中所存储的View，一旦被获取了之后就会从mActiveViews当中移除，下次获取同样位置的View将会返回null，也就是说mActiveViews不能被重复利用。
- **addScrapView()** 用于将一个废弃的View进行缓存，该方法接收一个View参数，当有某个View确定要废弃掉的时候(比如滚动出了屏幕)，就应该调用这个方法来对View进行缓存，RecycleBin当中使用mScrapViews和mCurrentScrap这两个List来存储废弃View。
- **getScrapView** 用于从废弃缓存中取出一个View，这些废弃缓存中的View是没有顺序可言的，因此getScrapView()方法中的算法也非常简单，就是直接从mCurrentScrap当中获取尾部的一个scrap view进行返回。
- **setViewTypeCount()** 我们都知道Adapter当中可以重写一个getViewTypeCount()来表示ListView中有几种类型的数据项，而setViewTypeCount()方法的作用就是为每种类型的数据项都单独启用一个RecycleBin缓存机制。实际上，getViewTypeCount()方法通常情况下使用的并不是很多，所以我们只要知道RecycleBin当中有这样一个功能就行了。






**RecycleBin中有两个重要的View数组，分别是mActiveViews和mScrapViews。这两个数组中所存储的View都是用来复用的，只不过mActiveViews中存储的是OnScreen的View，这些View很有可能被直接复用；而mScrapViews中存储的是OffScreen的View，这些View主要是用来间接复用的。**



ListView的layoutChildren方法代码比较多，我们只研究和View增删相关的关键代码，主要分以下三个阶段：

1. ListView的children->RecycleBin
2. ListView清空children
3. RecycleBin->ListView的children






[源码解析ListView中的RecycleBin机制](http://blog.csdn.net/iispring/article/details/50967445)


