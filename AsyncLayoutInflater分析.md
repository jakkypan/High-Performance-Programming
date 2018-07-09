# AsyncLayoutInflater分析

`LayoutInflater`的改进类，因为`LayoutInflater`的inflate()都是在UI线程中同步加载布局的，这个是个很耗时的操作，在布局很复杂的时候，很容易造成页面卡顿甚至ANR，并且阻塞了后面的一些事情。

而`AsyncLayoutInflater`可以帮助你在非UI线程中加载layout，然后将加载好的布局通过接口回调的形式同步给UI线程。

## 基本使用

**重点：调用的时候必须在主线程**

```java
new AsyncLayoutInflater(this).inflate(  
                         R.layout.activity_main,  
                         null,  
                         new AsyncLayoutInflater.OnInflateFinishedListener() {  
@Override  
public void onInflateFinished(View view, int resid, ViewGroup parent) {  
    //Do something with view  
}});
```

## 原理分析

主要是3个方面的内容：

* **InflateThread**
* **InflateRequest**
* **BasicInflater**

### InflateThread

进行inflate实际操作的线程。它采用单例模式：

```java
private static final InflateThread sInstance;
static {
    sInstance = new InflateThread();
    sInstance.start();
}

public static InflateThread getInstance() {
    return sInstance;
}
```

操作的核心是：

```java
public void runInner() {
    InflateRequest request;
    try {
        request = mQueue.take();
    } catch (InterruptedException ex) {
        // Odd, just continue
        Log.w(TAG, ex);
        return;
    }

    try {
        request.view = request.inflater.mInflater.inflate(
                request.resid, request.parent, false);
    } catch (RuntimeException ex) {
        // Probably a Looper failure, retry on the UI thread
        Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                + " thread", ex);
    }
    Message.obtain(request.inflater.mHandler, 0, request)
            .sendToTarget();
}

@Override
public void run() {
    while (true) {
        runInner();
    }
}
```

这里使用到了阻塞队列和线程池：

```java
private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);
```

在上面`request = mQueue.take();`，如果没有数据在阻塞队列中，那么这里将会被阻塞（而不是一直在死循环）。

数据被添加到队列中是在`inflate()`方法里：

```java
@UiThread
public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
        @NonNull OnInflateFinishedListener callback) {
    if (callback == null) {
        throw new NullPointerException("callback argument may not be null!");
    }
    InflateRequest request = mInflateThread.obtainRequest();
    request.inflater = this;
    request.resid = resid;
    request.parent = parent;
    request.callback = callback;
    mInflateThread.enqueue(request);
}

// InflateThread
public void enqueue(InflateRequest request) {
    try {
        mQueue.put(request);
    } catch (InterruptedException e) {
        throw new RuntimeException(
                "Failed to enqueue async inflate request", e);
    }
}
```

### InflateRequest

这个只是个对象类，保存一些数据给到InflateThread：

```java
private static class InflateRequest {
        AsyncLayoutInflater inflater;
        ViewGroup parent;
        int resid;
        View view;
        OnInflateFinishedListener callback;

        InflateRequest() {
        }
    }
```

### BasicInflater

这个类和`PhoneLayoutInflater`一样，搞不清楚为什么要重写一个。

## 限制条件

1. parent的`generateLayoutParams()`函数必须是线程安全的
 * 这个可以理解，毕竟是在独立线程操作inflate
2. 不支持LayoutInflater.Factory也不支持LayoutInflater.Factory2
3. 不支持包含`<fragment>`标签的inflate操作
 * 这个是因为条件3决定的
4. 所有正在构建的views一定不能创建任何Handlers或者调用Looper.myLooper函数

## 扩展阅读

上面说到阻塞队列，典型的“生产者-消费者”模型，特点是队列为空的时候，线程会被阻塞，当有数据的时候会被唤醒，从而保证资源可以及时的被释放。

在`AsyncLayoutInflater`里用的是`ArrayBlockingQueue`，基于数组的阻塞队列，还有个`LinkedBlockingQueue`，基于链表的阻塞队列。

他们都是基于`AbstractQueue`抽象类，共同的操作有：

* **add()**：添加元素到队列里，添加成功返回true，由于容量满了添加失败会抛出IllegalStateException异常，add()的内部其实调用的是offer的方法
* **offer()**：添加元素到队列里，添加成功返回true，添加失败返回false
* **put()**：添加元素到队列里，如果容量满了会阻塞直到容量不满
* **poll()**：删除并返回队列头部元素，如果队列为空，返回null。否则返回元素。
* **peak()**：类似于poll()的操作，但是只是返回对头元素，不会删除，类似堆栈里的操作
* **take()**：删除并返回队列头部元素，如果队列为空，则会一直阻塞，知道队列里有元素了
* **remove()**：基于对象找到对应的元素，并删除。删除成功返回true，否则返回false

他们的差异是：

1. 队列大小初始化方式不同（**PS：两者都是不进行扩容**）
 * ArrayBlockingQueue队列是必须指定队列的大小的
 * LinkedBlockingQueue可以不指定队列的大小，默认采用`Integer.MAX_VALUE`
2. 锁的实现方式不同
 * ArrayBlockingQueue实现的队列中的锁是没有分离的，即生产和消费用的是同一个锁
 
 ```java
 /** Main lock guarding all access */
 final ReentrantLock lock;

 /** Condition for waiting takes */
 private final Condition notEmpty;

 /** Condition for waiting puts */
 private final Condition notFull;
 ```
 * LinkedBlockingQueue实现的队列中的锁是分离的，即生产用的是putLock，消费是takeLock
 
 ```java
 /** Lock held by take, poll, etc */
 private final ReentrantLock takeLock = new ReentrantLock();

 /** Wait queue for waiting takes */
 private final Condition notEmpty = takeLock.newCondition();

 /** Lock held by put, offer, etc */
 private final ReentrantLock putLock = new ReentrantLock();

 /** Wait queue for 3、waiting puts */
 private final Condition notFull = putLock.newCondition();
 ```
3. 生成消费数据时操作不同
 * ArrayBlockingQueue是直接将元素插入到数组中的，读取也是直接从数组中拿出来
 * LinkedBlockingQueue在插入数据时会将元素转成Node对象，然后插入到队列，取出是就是解开操作
 