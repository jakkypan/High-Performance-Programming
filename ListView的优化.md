# ListView的优化

ListView的优化有很多方面可以切入，我们会从最基本的一点点深入下去，同时也会讲到ListView的原理。

## 1. 入门级：使用ConvertView

这个就是利用到ListView自身对item的回收而产生的对item view的重复利用。

具体的操作：

```java
@Override
public View getView(int position, View convertView, ViewGroup parent) {
    View v = null;
    if (convertView == null) {
        v = View.inflate(Test3.this, R.layout.l, null);
    } else {
        v = convertView;
    }
    TextView tv = findViewById(R.id.tv);
    tv.setText(getItem(position) + "");
    return v;
}
```

关于ListView这个回收机制，请看：[ListView的回收机制](./ListView的回收机制.md)

## 2. 入门级：ViewHolder

这个是对优化1的进一步优化，主要是优化每次都需要进行`findViewById()`的操作，这个操作是从ViewGroup的子View里面循环遍历找id与给出的ID相同的子View，因此还是比较耗时的。

看`ViewGroup`里的`findViewTraversal()`：

```java
@Override
protected <T extends View> T findViewTraversal(@IdRes int id) {
    if (id == mID) {
        return (T) this;
    }

    final View[] where = mChildren;
    final int len = mChildrenCount;

    for (int i = 0; i < len; i++) {
        View v = where[i];

        if ((v.mPrivateFlags & PFLAG_IS_ROOT_NAMESPACE) == 0) {
            v = v.findViewById(id);

            if (v != null) {
                return (T) v;
            }
        }
    }

    return null;
}
```

而View里的对应的代码是：

```java
public final <T extends View> T findViewById(@IdRes int id) {
    if (id == NO_ID) {
        return null;
    }
    return findViewTraversal(id);
}

// 被ViewGroup继承
protected <T extends View> T findViewTraversal(@IdRes int id) {
    if (id == mID) {
        return (T) this;
    }
    return null;
}
```

ViewHolder模式的使用：

```java
@Override
public View getView(int position, View convertView, ViewGroup parent) {
    ViewHolder viewHolder = null;
    View v = null;
    if (convertView == null) {
        viewHolder = new ViewHolder();
        v = View.inflate(Test3.this, R.layout.l, null);
        viewHolder.tv= v.findViewById(R.id.tv);
        v.setTag(viewHolder);
    } else {
        v = convertView;
        viewHolder = (ViewHolder) v.getTag();
    }

    viewHolder.tv.setText(getItem(position) + "");
    return v;
}

class ViewHolder {
    TextView tv;
}
```

## 3. 入门级：关于`layout_weight`

item的高度尽量采用固定值或者`match_parent`，尽量不要使用`layout_weight`属性，这个会触发多次的measure。

## 4、进阶级：异步加载

开启异步线程来完成部分事情，让`getView()`做的事情尽可能的少，不阻塞视图的展现。

**需要注意的是异步加载出现的item错乱问题（见优化9），还要注意线程数的控制（见优化5）**

## 5、进阶级：滑动不加载

对于Item比较复杂的，为了使得滑动的更为流畅，可以使得滑动时不做加载，如item里有网络图片，这样的图片在idle的时候进行加载。

```java
listView.setOnScrollListener(new AbsListView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(AbsListView view, int scrollState) {
        switch (scrollState){
            case SCROLL_STATE_TOUCH_SCROLL:
            case SCROLL_STATE_FLING:
                break;
            case SCROLL_STATE_IDLE:
                int start = listView.getFirstVisiblePosition();
                int end = listView.getLastVisiblePosition();
                if(end >= listView.getCount()){
                    end = listView.getCount() - 1;
                }
                // 做自己要做的事情，如加载图片
                break;
        }
    }

    @Override
    public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount) {

    }
});
```

## 6、进阶级：item的事件监听优化

对item的点击事件等各种事件，不需要对每个item都设置，这样会导致不停的new监听器，可以重用一个监听器。

不好的写法：

```java
@Override
public View getView(final int position, View convertView, ViewGroup parent) {
    ViewHolder viewHolder = null;
    View v = null;
    if (convertView == null) {
        viewHolder = new ViewHolder();
        v = View.inflate(Test3.this, R.layout.l, null);
        viewHolder.tv = v.findViewById(R.id.tv);
        v.setTag(viewHolder);
        v.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(Test3.this, datas.get(position), Toast.LENGTH_SHORT).show();
            }
        });
    } else {
        v = convertView;
        viewHolder = (ViewHolder) v.getTag();
    }

    viewHolder.tv.setText(datas.get(position) + "");
    return v;
}
```

优化后的写法：

```java
@Override
public View getView(final int position, View convertView, ViewGroup parent) {
    ViewHolder viewHolder = null;
    View v = null;
    if (convertView == null) {
        viewHolder = new ViewHolder();
        v = View.inflate(Test3.this, R.layout.l, null);
        viewHolder.tv = v.findViewById(R.id.tv);
        viewHolder.position = position;
        v.setTag(viewHolder);
        v.setOnClickListener(this);
    } else {
        v = convertView;
        viewHolder = (ViewHolder) v.getTag();
        viewHolder.position = position;
        v.setTag(viewHolder);
    }

    viewHolder.tv.setText(datas.get(position) + "");
    return v;
}

@Override
public void onClick(View v) {
    ViewHolder viewHolder = (ViewHolder) v.getTag();
    Toast.makeText(Test3.this, datas.get(viewHolder.position), Toast.LENGTH_SHORT).show();
}
```

这里不好的一点是需要保存住position的值，这个只能通过tag去做。**这个在RecycleView里已经得到了优化，在Adapter里可以获取到位置。**

## 7、进阶级：一些高级属性的设置

> setAnimationCacheEnabled

这个是ViewGroup的属性。

**这个在SDK M的时候已经废弃，不再有效。**设置是否在布局做动画的时候保存drawing cache，缺省的是开启的，但是这将阻止嵌套的布局动画，如果非要支持，则必须关闭。

**建议：M以下关闭**

> setScrollingCacheEnabled

设置是否在scroll的时候保存drawing cache，缺省的是开启的，但是这个会使用更多的内存。

**建议：关闭**

> setSmoothScrollbarEnabled

默认是开启的，决定scrollbar位置的计算方式。当每个item的高度是一样的时候，这个建议打开，当item高度不一致的时候，则建议关闭。

**建议：item高度一致，开启，否则关闭**

## 8、进阶级：局部刷新

不要去调用`notifyDataSetChanged()`，而是自己实现内容的变更。

```java
private void updateSingle(int... positions) {
    int firstVisiblePosition = listView.getFirstVisiblePosition();
    int lastVisiblePosition = listView.getLastVisiblePosition();

    for (int position : positions) {
        if (position >= firstVisiblePosition && position <= lastVisiblePosition) {
            View view = listView.getChildAt(position - firstVisiblePosition);
            TextView textView = view.findViewById(R.id.tv);
            textView.setText(datas.get(position));
        }
    }
}

// 点击刷新
public void click(View view) {
    datas.set(1, "12333");
    datas.set(3, "3244445");
    updateSingle(1, 3);
}
```

还有一个优化的方法【**触发一次getView，通过getView的代码去刷新，这样就不用写重复代码了**】：

```java
private void updateSingle(int... positions) {
    int firstVisiblePosition = listView.getFirstVisiblePosition();
    int lastVisiblePosition = listView.getLastVisiblePosition();

    for (int position : positions) {
        if (position >= firstVisiblePosition && position <= lastVisiblePosition) {
            View view = listView.getChildAt(position - firstVisiblePosition);
            listView.getAdapter().getView(position, view, listView);
        }
    }
}
```

## 9、进阶级：图片错乱优化

> 问题：

问题的产生就是由于listview的view回收引起的。

当我们快速滑动ListView的时候就很有可能出现这样一种情况，某一个位置上的元素进入屏幕后开始从网络上请求图片，但是还没等图片下载完成，它就又被移出了屏幕。这样按照listview的recycle机制，被移出屏幕的控件将会很快被新进入屏幕的元素重新利用起来，而如果在这个时候刚好前面发起的图片请求有了响应，就会将刚才位置上的图片显示到当前位置上，因为虽然它们位置不同，但都是共用的同一个ImageView实例，这样就出现了图片乱序的情况。

> 解决方案1：使用findViewWithTag

由于ListView中的ImageView控件都是重用的，移出屏幕的控件很快会被进入屏幕的图片重新利用起来，那么getView()方法就会再次得到执行，而在getView()方法中会为这个ImageView控件设置新的Tag，这样老的Tag就会被覆盖掉，于是这时再调用findVIewWithTag()方法并传入老的Tag，就只能得到null了，而我们判断只有ImageView不等于null的时候才会设置图片，这样图片乱序的问题也就不存在了。

```java
public class ImageAdapter extends ArrayAdapter<String> {  

    private ListView mListView;   

    ......  

    @Override  
    public View getView(int position, View convertView, ViewGroup parent) {  
        if (mListView == null) {    
            mListView = (ListView) parent;    
        }   
        String url = getItem(position);  
        View view;  
        if (convertView == null) {  
            view = LayoutInflater.from(getContext()).inflate(R.layout.image_item, null);  
        } else {  
            view = convertView;  
        }  
        ImageView image = (ImageView) view.findViewById(R.id.image);  
        image.setImageResource(R.drawable.empty_photo);  
        image.setTag(url);  
        BitmapDrawable drawable = getBitmapFromMemoryCache(url);  
        if (drawable != null) {  
            image.setImageDrawable(drawable);  
        } else {  
            BitmapWorkerTask task = new BitmapWorkerTask();  
            task.execute(url);  
        }  
        return view;  
    }  

    ......  

    /** 
     * 异步下载图片的任务。 
     *  
     * @author guolin 
     */  
    class BitmapWorkerTask extends AsyncTask<String, Void, BitmapDrawable> {  

        String imageUrl;   

        @Override  
        protected BitmapDrawable doInBackground(String... params) {  
            imageUrl = params[0];  
            // 在后台开始下载图片  
            Bitmap bitmap = downloadBitmap(imageUrl);  
            BitmapDrawable drawable = new BitmapDrawable(getContext().getResources(), bitmap);  
            addBitmapToMemoryCache(imageUrl, drawable);  
            return drawable;  
        }  

        @Override  
        protected void onPostExecute(BitmapDrawable drawable) {  
            ImageView imageView = (ImageView) mListView.findViewWithTag(imageUrl);    
            if (imageView != null && drawable != null) {    
                imageView.setImageDrawable(drawable);    
            }   
        }  

        ......  

    }  

}
```

> 解决方案2：弱引用关联

就是建立ImageView和DownloadTask之间的关联关系，互相持有对方的引用，为了防止出现内存泄露的问题，双向关联采用弱引用建立。

这个的写法：

```java
public class ImageAdapter extends ArrayAdapter<String> {

    private ListView mListView;

    private Bitmap mLoadingBitmap;

    /**
     * 图片缓存技术的核心类，用于缓存所有下载好的图片，在程序内存达到设定值时会将最少最近使用的图片移除掉。
     */
    private LruCache<String, BitmapDrawable> mMemoryCache;

    public ImageAdapter(Context context, int resource, String[] objects) {
        super(context, resource, objects);
        mLoadingBitmap = BitmapFactory.decodeResource(context.getResources(),
                R.drawable.ic_launcher);
        // 获取应用程序最大可用内存
        int maxMemory = (int) Runtime.getRuntime().maxMemory();
        int cacheSize = maxMemory / 8;
        mMemoryCache = new LruCache<String, BitmapDrawable>(cacheSize) {
            @Override
            protected int sizeOf(String key, BitmapDrawable drawable) {
                return drawable.getBitmap().getByteCount();
            }
        };
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        if (mListView == null) {
            mListView = (ListView) parent;
        }
        String url = getItem(position);
        View view;
        if (convertView == null) {
            view = LayoutInflater.from(getContext()).inflate(R.layout.image_item, null);
        } else {
            view = convertView;
        }
        ImageView image = (ImageView) view.findViewById(R.id.image);
        BitmapDrawable drawable = getBitmapFromMemoryCache(url);
        if (drawable != null) {
            image.setImageDrawable(drawable);
        } else if (cancelPotentialWork(url, image)) {
            BitmapWorkerTask task = new BitmapWorkerTask(image);
            AsyncDrawable asyncDrawable = new AsyncDrawable(getContext()
                    .getResources(), mLoadingBitmap, task);
            image.setImageDrawable(asyncDrawable);
            task.execute(url);
        }
        return view;
    }

    /**
     * 自定义的一个Drawable，让这个Drawable持有BitmapWorkerTask的弱引用。
     */
    class AsyncDrawable extends BitmapDrawable {

        private WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

        public AsyncDrawable(Resources res, Bitmap bitmap,
                             BitmapWorkerTask bitmapWorkerTask) {
            super(res, bitmap);
            bitmapWorkerTaskReference = new WeakReference<BitmapWorkerTask>(
                    bitmapWorkerTask);
        }

        public BitmapWorkerTask getBitmapWorkerTask() {
            return bitmapWorkerTaskReference.get();
        }

    }

    /**
     * 获取传入的ImageView它所对应的BitmapWorkerTask。
     */
    private BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
        if (imageView != null) {
            Drawable drawable = imageView.getDrawable();
            if (drawable instanceof AsyncDrawable) {
                AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
                return asyncDrawable.getBitmapWorkerTask();
            }
        }
        return null;
    }

    /**
     * 取消掉后台的潜在任务，当认为当前ImageView存在着一个另外图片请求任务时
     * ，则把它取消掉并返回true，否则返回false。
     */
    public boolean cancelPotentialWork(String url, ImageView imageView) {
        BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);
        if (bitmapWorkerTask != null) {
            String imageUrl = bitmapWorkerTask.imageUrl;
            if (imageUrl == null || !imageUrl.equals(url)) {
                bitmapWorkerTask.cancel(true);
            } else {
                return false;
            }
        }
        return true;
    }

    /**
     * 将一张图片存储到LruCache中。
     *
     * @param key
     *            LruCache的键，这里传入图片的URL地址。
     * @param drawable
     *            LruCache的值，这里传入从网络上下载的BitmapDrawable对象。
     */
    public void addBitmapToMemoryCache(String key, BitmapDrawable drawable) {
        if (getBitmapFromMemoryCache(key) == null) {
            mMemoryCache.put(key, drawable);
        }
    }

    /**
     * 从LruCache中获取一张图片，如果不存在就返回null。
     *
     * @param key
     *            LruCache的键，这里传入图片的URL地址。
     * @return 对应传入键的BitmapDrawable对象，或者null。
     */
    public BitmapDrawable getBitmapFromMemoryCache(String key) {
        return mMemoryCache.get(key);
    }

    /**
     * 异步下载图片的任务。
     *
     * @author guolin
     */
    class BitmapWorkerTask extends AsyncTask<String, Void, BitmapDrawable> {

        String imageUrl;

        private WeakReference<ImageView> imageViewReference;

        public BitmapWorkerTask(ImageView imageView) {
            imageViewReference = new WeakReference<ImageView>(imageView);
        }

        @Override
        protected BitmapDrawable doInBackground(String... params) {
            imageUrl = params[0];
            // 在后台开始下载图片
            Bitmap bitmap = downloadBitmap(imageUrl);
            BitmapDrawable drawable = new BitmapDrawable(getContext().getResources(), bitmap);
            addBitmapToMemoryCache(imageUrl, drawable);
            return drawable;
        }

        @Override
        protected void onPostExecute(BitmapDrawable drawable) {
            ImageView imageView = getAttachedImageView();
            if (imageView != null && drawable != null) {
                imageView.setImageDrawable(drawable);
            }
        }

        /**
         * 获取当前BitmapWorkerTask所关联的ImageView。
         */
        private ImageView getAttachedImageView() {
            ImageView imageView = imageViewReference.get();
            BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask) {
                return imageView;
            }
            return null;
        }

        /**
         * 建立HTTP请求，并获取Bitmap对象。
         *
         * @param imageUrl
         *            图片的URL地址
         * @return 解析后的Bitmap对象
         */
        private Bitmap downloadBitmap(String imageUrl) {
            Bitmap bitmap = null;
            HttpURLConnection con = null;
            try {
                URL url = new URL(imageUrl);
                con = (HttpURLConnection) url.openConnection();
                con.setConnectTimeout(5 * 1000);
                con.setReadTimeout(10 * 1000);
                bitmap = BitmapFactory.decodeStream(con.getInputStream());
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (con != null) {
                    con.disconnect();
                }
            }
            return bitmap;
        }
    }
}
```

这里涉及到了图片的缓存，也算是一种优化措施，一般采用**内存->本地->网络**的读取方式。

## 10、高级版：RecycleView

请看[RecycleView使用与优化](./RecycleView使用与优化.md)