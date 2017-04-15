# 超简单实现Google+列表特效

## 特效动画

### from_bottom_to_top.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@android:anim/decelerate_interpolator"
    android:shareInterpolator="true">
    <translate
        android:fromXDelta="0%" android:toXDelta="0%"
        android:fromYDelta="100%" android:toYDelta="0%"
        android:duration="400" />
</set>
```

### from_top_to_bottom.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:interpolator="@android:anim/decelerate_interpolator"
    android:shareInterpolator="true">
    <translate
        android:fromXDelta="0%" android:toXDelta="0%"
        android:fromYDelta="-100%" android:toYDelta="0%"
        android:duration="400" />
</set>
```

## 加入动画

```java
private int mLastPosition = -1;
@Override
public View getView(int position, View convertView, ViewGroup parent) {
  View view = super.getView(position, convertView, parent);
  int animResId;
  if (position > mLastPosition) {
      animResId = R.anim.from_bottom_to_top;
  } else {
      animResId = R.anim.from_top_to_bottom;
  }
          
  Animation animation = AnimationUtils.loadAnimation(getContext(), animResId);
  view.startAnimation(animation);
  mLastPosition = position;
  return view;
}
```















