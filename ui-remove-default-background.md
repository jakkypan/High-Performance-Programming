
DecorView为我们设置了默认背景，根据theme的不同会不同。但是这个背景其实没啥用，因为我们的布局通常会带上背景，所以DecorView的背景可以不要绘制。

方法是：

```xml
<style name="Theme.NoBackground" parent="android:Theme">
    <item name="android:windowBackground">@null</item>
</style>
```

我们可以更进一步，我们可以将activity的背景放到窗口的DecorView上进行绘制，这样做的依据DecorView的背景是早于其他任何布局的，这样用户可以更早的看到我的activity的背景。

具体做法很简单，就是将`android:windowBackground`替换成activity的背景即可：

```xml
<style name="Theme.SpecialBackground" parent="android:Theme">
    <item name="android:windowBackground">@drawable/my_background</item>
</style>
```

> 引申阅读

[DecorView是怎么创建出来的？](./DecorView是怎么创建出来的？.md)
