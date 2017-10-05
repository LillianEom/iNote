# MVP





### MVC

当用户出发事件的时候，view层会发送指令到controller层，接着controller去通知model层更新数据，model层更新完数据以后直接显示在view层上，这就是MVC的工作原理。

对于原生的Android项目来说，layout.xml里面的xml文件就对应于MVC的view层，里面都是一些view的布局代码，而各种java bean，还有一些类似repository类就对应于model层，至于controller层嘛，当然就是各种activity咯。大家可以试着套用我上面说的MVC的工作原理是理解。比如你的界面有一个按钮，按下这个按钮去网络上下载一个文件，这个按钮是view层的，是使用xml来写的，而那些和网络连接相关的代码写在其他类里，比如你可以写一个专门的networkHelper类，这个就是model层，那怎么连接这两层呢？是通过button.setOnClickListener()这个函数，这个函数就写在了activity中，对应于controller层。是不是很清晰。

问题就在于xml作为view层，控制能力实在太弱了，你想去动态的改变一个页面的背景，或者动态的隐藏/显示一个按钮，这些都没办法在xml中做，只能把代码写在activity中，造成了activity既是controller层，又是view层的这样一个窘境。大家回想一下自己写的代码，如果是一个逻辑很复杂的页面，activity或者fragment是不是动辄上千行呢？这样不仅写起来麻烦，维护起来更是噩梦。（当然看过Android源码的同学其实会发现上千行的代码不算啥，一个RecyclerView.class的代码都快上万行了呢。。）

**MVC还有一个重要的缺陷**，view层和model层是相互可知的，这意味着两层之间存在耦合，耦合对于一个大型程序来说是非常致命的，因为这表示开发，测试，维护都需要花大量的精力。

- Activity的臃肿：xml作为view层，控制能力太弱，无法动态的改变页面的内容，只能把代码写在activity中，造成了activity既是Controller层，又是View层的问题。
- 耦合度较高，需求变化改动大，后续维护成本高。
- Controller混杂着Android代码无法Junit。

### MVP

最明显的差别就是view层和model层不再相互可知，完全的解耦，取而代之的presenter层充当了桥梁的作用，用于操作view层发出的事件传递到presenter层中，presenter层去操作model层，并且将数据返回给view层，整个过程中view层和model层完全没有联系。看到这里大家可能会问，虽然view层和model层解耦了，但是view层和presenter层不是耦合在一起了吗？其实不是的，对于view层和presenter层的通信，我们是可以通过接口实现的，具体的意思就是说我们的activity，fragment可以去实现实现定义好的接口，而在对应的presenter中通过接口调用方法。不仅如此，我们还可以编写测试用的View，模拟用户的各种操作，从而实现对Presenter的测试。这就解决了MVC模式中测试，维护难的问题。



**当然，其实最好的方式是使用fragment作为view层，而activity则是用于创建view层(fragment)和presenter层(presenter)的一个控制器。**



- 减少各层之间耦合，易于后续的需求变化，降低维护成本。
- Presenter层独立于Android代码之外，可以进行Junit测试。
- 接口和类较多，互相做回调，代码臃肿。
- Presenter层与View层是通过接口进行交互的，接口粒度不好控制。

MVP的问题在于，由于我们使用了接口的方式去连接view层和presenter层，这样就导致了一个问题，如果你有一个逻辑很复杂的页面，你的接口会有很多，十几二十个都不足为奇。想象一个app中有很多个这样复杂的页面，维护接口的成本就会非常的大。







### MVVM

- 和MVP比较像，主要区别在于View和ViewModel / Presenter之间的通信
- 相比MVP优势是通过DataBinding技术为VM和V层进行数据绑定，提高开发效率，由于目前绑定技术的局限，V层一些界面的处理还是需要Activity的辅助。
- VM层掺杂Android代码无法进行Junit测试。



























