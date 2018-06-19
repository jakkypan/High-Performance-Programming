API 11后，开始支持硬件加速，在API 14后，系统默认是开启硬件加速的。

## 硬件加速的级别

硬件加速可以在不同的地方进行开启/关闭，小粒度的将覆盖大粒度的设置。

> Application > Activity > Window > View

### Application

```xml
<application 
    android:hardwareAccelerated="false" 
...>
</application>
```

### Activity

```xml
<activity android:hardwareAccelerated="false"
...>
</activity>
```

### window

注意这个请求不能保证硬件加速一定会发生。同时这个只能开启，不能关闭由Application或activity开启的硬件加速。

必须在activity或dialog设置content view之前设置。

```java
Window w = activity.getWindow();
w.setFlags(WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED, WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
```

### View

```java
View.setLayerType(View.LAYER_TYPE_HARDWARE, null);
```

这里是是否使用硬件图层，并不是说开启硬件加速。如果硬件加速没有开启，那么这个硬件图层就无法使用，转成使用软件图层。

> 建议用法

在动画启动前，开启硬件层，在动画结束后，立即关闭，释放被占用的资源。

```java
view.setLayerType(View.LAYER_TYPE_HARDWARE, null)
val animator:ObjectAnimator = ObjectAnimator.ofFloat(view, "rotationX", 180)
animator.addListener(AnimatorListenerAdapter() {
    override fun onAnimationEnd(animation: Animator) {
         view.setLayerType(View.LAYER_TYPE_NONE, null)
    }
})
```

### 检测是否开启硬件加速

指示该view是不是attached到开启了硬件加速的window上。即使返回true，也不意味着每次的draw都会进行硬件加速。

```java
View.isHardwareAccelerated()
```

这个更加准确，如果要真的判断的话，建议采用这个方法。
```java
Canvas.isHardwareAccelerated()
```

## 关于硬件加速使用过程中的建议

* 减少布局中View的数量，这个很好理解，view越多绘制时消耗越多
* 

## 引申阅读

[屏幕刷新的深度分析](./屏幕刷新的深度分析.md)
