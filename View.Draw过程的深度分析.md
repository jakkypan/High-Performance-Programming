# View.Draw过程的深度分析

在完成了[View.Measure过程的深度分析](./View.Measure过程的深度分析.md)和[View.Layout过程的深度分析](./View.Layout过程的深度分析.md)之后，进入到draw的阶段。

还是从`performTraversals()`开始：

![](./imgs/draw_trigger.png)

`performDraw()`：

```java
private void performDraw() {
    if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
        return;
    } else if (mView == null) {
        return;
    }

    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
    try {
        draw(fullRedrawNeeded);
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    
    ... ...
}
```

## ViewRootImpl.draw()

`draw()`方法，参数表示是否是重新绘制全部视图。



