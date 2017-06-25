[Java集合框架分析-LinkedHashMap](http://www.jianshu.com/p/1182237a1940)





一个有序的Map接口实现，这里的有序指的是元素可以按插入顺序或访问顺序排列；与HashMap相比，因为LinkedHashMap是继承自HashMap，因此LinkedHashMap：

1. 同样是基于散列表实现。
2. 同时实现了Serializable 和 Cloneable接口，支持序列化和克隆。
3. 并且同样不是线程安全的。

区别是其内部维护了一个双向循环链表，该链表是有序的，可以按元素插入顺序或元素最近访问顺序(LRU)排列。

1、由于LinkedHashMap继承自HashMap，所以它不仅像HashMap那样对其进行基于哈希表和单链表的Entry数组+ next链表的存储方式，而且还结合了LinkedList的优点，为每个Entry节点增加了前驱和后继，并增加了一个为header头结点，构造了一个双向循环链表。**（多一个以header为头结点的双向循环链表，也就是说，每次put进来KV，除了将其保存到对哈希表中的对应位置外，还要将其插入到双向循环链表的尾部。）**

![img](http://upload-images.jianshu.io/upload_images/676457-acf608231b3ff031?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、LinkedHashMap的属性比HashMap多了一个accessOrder属性。**当它false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序**，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序；**当它为true时，表示双向链表中的元素按照访问的先后顺序排列**，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用recordAccess方法（put方法在key相同，覆盖原有的Entry的情况下调用recordAccess方法），该方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用addEntry，它会调用creatEntry，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了），否则，什么也不做。

3、构造函数中有设置accessOrder的方法，如果我们需要实现LRU算法时，就需要将accessOrder的值设定为TRUE。

4、在HashMap的put方法中，如果key不为null时且哈希表中已经在存在时，循环遍历table[i]中的链表时会调用recordAccess方法，而在HashMap中这个方法是个空方法，在LinkedHashMap中则实现了该方法，该方法会判断accessOrder是否为true，如果为true，它会将当前访问的Entry（在这里指put进来的Entry）移动到双向循环链表的尾部，从而实现双向链表中的元素按照访问顺序来排序（最近访问的Entry放到链表的最后，这样多次下来，前面就是最近没有被访问的元素，在实现、LRU算法时，当双向链表中的节点数达到最大值时，将前面的元素删去即可，因为前面的元素是最近最少使用的），否则什么也不做。

















