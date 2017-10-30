[Butter Knif](https://github.com/JakeWharton/butterknife)



### ButterKnife使用中有哪些注意的点呢？

> **注意：**
>
> 1. Activity ButterKnife.bind(this);必须在setContentView();之后，且父类bind绑定后，子类不需要再bind
> 2. Fragment ButterKnife.bind(this, mRootView);
> 3. 属性布局不能用private or static 修饰，否则会报错
> 4. setContentView()不能通过注解实现。
> 5. ButterKnife已经更新到版本7.0.1了，以前的版本中叫做**@InjectView**了，而现在改用叫**@Bind**，更加贴合语义。
> 6. 在Fragment生命周期中，onDestoryView也需要Butterknife.unbind(this)



由于每次都要在 Activity 中的 onCreate 绑定 Activity，所以个人建议写一个 BaseActivity 完成绑定，子类继承即可
注：ButterKnife.bind(this)；绑定Activity 必须在setContentView之后：


#### 视图绑定

```java
class ExampleActivity extends Activity {
  @BindView(R.id.title) TextView title;
  @BindView(R.id.subtitle) TextView subtitle;
  @BindView(R.id.footer) TextView footer;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```

生成的代码

```java
public void bind(ExampleActivity activity) {
  activity.subtitle = (android.widget.TextView) activity.findViewById(2130968578);
  activity.footer = (android.widget.TextView) activity.findViewById(2130968579);
  activity.title = (android.widget.TextView) activity.findViewById(2130968577);
}
```

#### RESOURCE BINDING

使用`@BindBool，@BindColor，@BindDimen，@BindDrawable，@BindInt，@BindString`绑定预定义的资源，将R.bool ID（或指定的类型）绑定到其对应的字段。

```java
class ExampleActivity extends Activity {
  @BindString(R.string.title) String title;
  @BindDrawable(R.drawable.graphic) Drawable graphic;
  @BindColor(R.color.red) int red; // int or ColorStateList field
  @BindDimen(R.dimen.spacer) Float spacer; // int (for pixel size) or float (for exact value) field
  // ...
}
```

#### NON-ACTIVITY BINDING

```java
public class FancyFragment extends Fragment {
  @BindView(R.id.button1) Button button1;
  @BindView(R.id.button2) Button button2;

  @Override public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View view = inflater.inflate(R.layout.fancy_fragment, container, false);
    ButterKnife.bind(this, view);
    // TODO Use fields...
    return view;
  }
}
```

另一个用途是简化列表适配器内的视图保持器模式。

```java
public class MyAdapter extends BaseAdapter {
  @Override public View getView(int position, View view, ViewGroup parent) {
    ViewHolder holder;
    if (view != null) {
      holder = (ViewHolder) view.getTag();
    } else {
      view = inflater.inflate(R.layout.whatever, parent, false);
      holder = new ViewHolder(view);
      view.setTag(holder);
    }

    holder.name.setText("John Doe");
    // etc...

    return view;
  }

  static class ViewHolder {
    @BindView(R.id.title) TextView name;
    @BindView(R.id.job_title) TextView jobTitle;

    public ViewHolder(View view) {
      ButterKnife.bind(this, view);
    }
  }
}
```

使用Activity为根视图绑定任意对象时，如果你使用类似MVC的设计模式你可以在Activity 调用 `ButterKnife.bind(this, activity)`来绑定Controller。

使用`ButterKnife.bind(this)` 绑定子 View。如果在布局中使用`<merge>`标签，并在自定义视图构造函数中展开，可以立即调用它。或者，从XML扩展的自定义视图类型可以在onFinishInflate（）回调中使用它。

#### VIEW LISTS

对一组 View 进行同意操作，可以将多个视图分组到列表或数组。

```java
@BindViews({ R.id.first_name, R.id.middle_name, R.id.last_name })
List<EditText> nameViews;
```

`apply` 方法允许一次对列表中的所有视图执行操作。

```java
ButterKnife.apply(nameViews, DISABLE);
ButterKnife.apply(nameViews, ENABLED, false);
```

`Action` 和 `Setter` 可以实现简单的行为

```java
static final ButterKnife.Action<View> DISABLE = new ButterKnife.Action<View>() {
  @Override public void apply(View view, int index) {
    view.setEnabled(false);
  }
};
static final ButterKnife.Setter<View, Boolean> ENABLED = new ButterKnife.Setter<View, Boolean>() {
  @Override public void set(View view, Boolean value, int index) {
    view.setEnabled(value);
  }
};
```

安卓的属性也可以使用在 `apply` 方法中

```java
ButterKnife.apply(nameViews, View.ALPHA, 0.0f);
```

#### LISTENER BINDING

监听器，点击事件的绑定

```java
@OnClick(R.id.submit)
public void submit(View view) {
  // TODO submit data to server...
}
```

监听器所有的参数是可选的

```java
@OnClick(R.id.submit)
public void submit() {
  // TODO submit data to server...
}
```

定义一个特定的类型，它将自动被转换。

```java
@OnClick(R.id.submit)
public void sayHi(Button button) {
  button.setText("Hello!");
}
```

多个view统一处理同一个点击事件，在单个绑定中指定多个ID以进行常规事件处理。

```java
@OnClick({ R.id.door1, R.id.door2, R.id.door3 })
public void pickDoor(DoorView door) {
  if (door.hasPrizeBehind()) {
    Toast.makeText(this, "You win!", LENGTH_SHORT).show();
  } else {
    Toast.makeText(this, "Try again", LENGTH_SHORT).show();
  }
}
```

自定义视图可以通过不指定ID来绑定到自己的侦听器。

```java
public class FancyButton extends Button {
  @OnClick
  public void onClick() {
    // TODO do something!
  }
}
```

给EditText加addTextChangedListener（即添加多回调方法的监听的使用方法），利用指定回调，实现想回调的方法即可，哪个注解不会用点进去看下源码上的注

```java

@OnTextChanged(value = R.id.mobileEditText, callback = OnTextChanged.Callback.BEFORE_TEXT_CHANGED)
void beforeTextChanged(CharSequence s, int start, int count, int after) {

}
@OnTextChanged(value = R.id.mobileEditText, callback = OnTextChanged.Callback.TEXT_CHANGED)
void onTextChanged(CharSequence s, int start, int before, int count) {

}
@OnTextChanged(value = R.id.mobileEditText, callback = OnTextChanged.Callback.AFTER_TEXT_CHANGED)
void afterTextChanged(Editable s) {

}
```



#### BINDING RESET

Fragments 与activity 具有不同的生命周期。在 `onCreateView` 绑定，需要在 `onDestroyView` 中设置为 `null` ，Butter Knife 会返回 `Unbinder` 实例，当调用 bind 来执行此操作时。在适当的生命周期回调中调用其unbind方法。

```java
public class FancyFragment extends Fragment {
  @BindView(R.id.button1) Button button1;
  @BindView(R.id.button2) Button button2;
  private Unbinder unbinder;

  @Override public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View view = inflater.inflate(R.layout.fancy_fragment, container, false);
    unbinder = ButterKnife.bind(this, view);
    // TODO Use fields...
    return view;
  }

  @Override public void onDestroyView() {
    super.onDestroyView();
    unbinder.unbind();
  }
}
```

#### OPTIONAL BINDINGS

默认情况下，@Bind和监听器绑定都是必需的。如果找不到目标视图，将抛出异常。要抑制此行为并创建可选绑定，请将@Nullable注释添加到字段或将@Optional注释添加到方法中。

```java
Nullable @BindView(R.id.might_not_be_there) TextView mightNotBeThere;

@Optional @OnClick(R.id.maybe_missing) void onMaybeMissingClicked() {
  // TODO ...
}
```

#### MULTI-METHOD LISTENERS

对应的监听器有多个回调的方法注释可用于绑定到任何一个。每个注释都具有绑定到的默认回调。使用回调参数指定备用。

```java
@OnItemSelected(R.id.list_view)
void onItemSelected(int position) {
  // TODO ...
}

@OnItemSelected(value = R.id.maybe_missing, callback = NOTHING_SELECTED)
void onNothingSelected() {
  // TODO ...
}
```

#### BONUS

还包括findById方法，它简化了仍然必须在View，Activity或Dialog上查找视图的代码。它使用泛型来推断返回类型并自动执行转换。

```java
View view = LayoutInflater.from(context).inflate(R.layout.thing, null);
TextView firstName = ButterKnife.findById(view, R.id.first_name);
TextView lastName = ButterKnife.findById(view, R.id.last_name);
ImageView photo = ButterKnife.findById(view, R.id.photo);
```



### 有一点需要注意

通过ButterKnife来注入View时，ButterKnife有`bind(Object, View)` 和 `bind(View)`两个方法，有什么区别呢？

如果你自定义了一个View，比如`public class BadgeLayout extends Fragment`，那么你可以可以通过`ButterKnife.bind(BadgeLayout)`来注入View的

如果你在一个ViewHolder中inflate了一个xml布局文件，得到一个`View`对象，并且这个View是`LinearLayout`或`FrameLayout`等系统自带View，那么不是不能用`ButterKnife.bind(View)`来注入View的，因为ButterKnife认为这些类的包名以`com.android`开头的类是没有注解功能的（-。- 这不是废话吗？），所以这种情况你需要使用`ButterKnife.bind(ViewHolder，View)`来注入View。

这表示**你是把@Bind、@OnClick等注解写到了这个ViewHolder类中，ViewHolder中的View呢需要从后面那个View中去找**， 大概就是这么个意思

### References

[深入理解 ButterKnife](http://dev.qq.com/topic/578753c0c9da73584b025875)

[浅析ButterKnife](https://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651232205&idx=1&sn=6c24e6eef2b18f253284b9dd92ec7efb&chksm=f1d9eaaec6ae63b82fd84f72c66d3759c693f164ff578da5dde45d367f168aea0038bc3cc8e8&mpshare=1&scene=1&srcid=02275mb9yhPes4V8sUPfrh10#rd)

[Java注解处理器](https://race604.com/annotation-processing/)















