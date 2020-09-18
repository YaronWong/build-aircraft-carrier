# RecyclerView性能优化及高级使用

最近研究应用流畅度专题时，发现RecyclerView里边的坑真多，有很多可以优化的点，在理解优化点之前，最好对RecyclerView的缓存机制有一些了解，比如得知道CacheView和RecycledViewPool的区别和联系，RecyclerView的绘制流程有一定了解，再来谈RecyclerView的性能提升。缓存机制可以看看这篇文章：[基于滑动场景解析RecyclerView的回收复用机制原理](https://www.jianshu.com/p/9306b365da57)

还有一篇外国人写的，[ViewHolder的探究](https://android.jlelse.eu/anatomy-of-recyclerview-part-1-a-search-for-a-viewholder-404ba3453714)，这篇文章把RecyclerView的各级缓存作用剖析得很清晰，以前看过很多人写的文章，感觉都是一知半解，总结下：

## 1、RecyclerView缓存

### 1.1 RecyclerView主要有三级缓存：

（1）Attached scrap & Changed scrap

```html
ArrayList<ViewHolder> mAttachedScrap 
    主要用在插入或是删除itemView时，先把屏幕内的ViewHolder保存至AttachedScrap中，作用在LayoutManager中，
    它仅仅把需要从ViewGroup中移除的子view设置它的父view为null，从而实现了从RecyclerView中移除操作detachView()。
    需要新插入的view从cacheView/Pool中找，没找到则createViewHolder。
    而从ViewGroup中移除的子view会放到Pool缓存池中,如下图中的itemView b。
```

![img](https://img-blog.csdnimg.cn/20190326092808357.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NtaWxlaWFt,size_16,color_FFFFFF,t_70)

```html
ArrayList<ViewHolder> mChangedScrap ：
    主要用到刷新屏幕上的itemView数据，它不需要重新layout，notifyItemChanged()或者notifyItemRangeChanged()
```

（2) cache Views ：保存最近移出屏幕的ViewHolder，包含数据和position信息，复用时必须是相同位置的ViewHolder才能复用，应用场景在那些需要来回滑动的列表中，当往回滑动时，能直接复用ViewHolder数据，不需要重新bindView。用一个数组保存ViewHolder，

```
实现是：ArrayList<ViewHolder> mCachedViews 
```

（3) RecyclerViewPool ：缓存池，当cacheView满了后，将cacheView中移出的ViewHolder放到Pool中，放之前会把ViewHolder数据清除掉，所以复用时需要重新bindView。实现是： SparseArray<ArrayList<ViewHolder>> mScrap;//按viewType来保存ViewHolder，每种类型最大缓存个数默认为5

### 1.2 RecyclerView缓存过程：

在滑动过程中，会先滑动的itemView保存到CacheView中，CacheView大小默认是2，如果超过了最大容量，则按FIFO,将队列头部的itemView出队，保存至缓存池RecyclerViewPool中，缓存池是按itemView的类型itemType来保存的，每种itemType默认缓存个数是5，超过了，则直接由GC回收。具体表现如下图：

![img](https://img-blog.csdnimg.cn/20190326091821406.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NtaWxlaWFt,size_16,color_FFFFFF,t_70)

可以看到CacheView缓存中蓝色的块一直最最近两个，而RecycledViewPool中，保存最大是5,超过5了后ViewHolder都被回收。

### 1.3 RecyclerView缓存寻找过程：

RecyclerView在找到可用ViewHodler的顺序是：如果在缓存CacheViews中找到，则直接复用；如果在缓存池RecycerViewPool找到，则需要bindView；如果没有找到可用的ViewHolder，则需要create新建一个ViewHolder，并bindView绑定view。

### 1.4 调用notifyDataSetChanged过程：

如果调用notifyDataSetChanged，每个itemView没有稳定的id的话，RecyclerView不知道接下来会发生什么，也不知道哪些改变，它假设所有都改变了，会将每一个ViewHolder设置成无效并且放到缓存池Pool中，如果我们仅是把屏幕上的第四条itemView移到第六条的位置，屏幕上所有itemView都会重新layout一遍，这样只能从缓存池RecycledViewPool池中取缓存的ViewHolder，如果不够时，需要重新create ViewHolder.具体实现如下：

![img](https://img-blog.csdnimg.cn/2019032609260765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NtaWxlaWFt,size_16,color_FFFFFF,t_70)

如果设置了Stable Ids，即每一个itemView都有一个唯一的id来标识，通过getItemId()来获取这个唯一标识id，当然我们不能用position来标识，因为itemView会复用，位置会乱序。当调用notifyDataSetChanged()方法时，ViewHolder会进入上面的一级缓存mAttachedScrap中，而不是进入缓存池pool中，这样的好处：1）不会存在缓存池pool满的问题，不需要重新createViewHolder; 2) 不需要重新bindView了。

![img](https://img-blog.csdnimg.cn/20190326094544879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NtaWxlaWFt,size_16,color_FFFFFF,t_70)

下面说说RecyclerView的一些优化方案和使用技巧：

## 1、recyclerView.setHasFixedSize(true);

当Item的高度如是固定的，设置这个属性为true可以提高性能，尤其是当RecyclerView有条目插入、删除时性能提升更明显。RecyclerView在条目数量改变，会重新测量、布局各个item，如果设置了setHasFixedSize(true)，由于item的宽高都是固定的，adapter的内容改变时，RecyclerView不会整个布局都重绘。具体可用以下伪代码表示：

```java
void onItemsInsertedOrRemoved() {



   if (hasFixedSize) layoutChildren();



   else requestLayout();



}
```

 

## 2、[使用getExtraLayoutSpace为LayoutManager设置更多的预留空间](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0209/2452.html)

在RecyclerView的元素比较高，一屏只能显示一个元素的时候，第一次滑动到第二个元素会卡顿。 

RecyclerView (以及其他基于adapter的view，比如ListView、GridView等)使用了缓存机制重用子 view（即系统只将屏幕可见范围之内的元素保存在内存中，在滚动的时候不断的重用这些内存中已经存在的view，而不是新建view）。

这个机制会导致一个问题，启动应用之后，在屏幕可见范围内，如果只有一张卡片可见，当滚动的时 候，RecyclerView找不到可以重用的view了，它将创建一个新的，因此在滑动到第二个feed的时候就会有一定的延时，但是第二个feed之 后的滚动是流畅的，因为这个时候RecyclerView已经有能重用的view了。

如何解决这个问题呢，其实只需重写getExtraLayoutSpace()方法。根据官方文档的描述 getExtraLayoutSpace将返回LayoutManager应该预留的额外空间（显示范围之外，应该额外缓存的空间）。

```java
LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this) {



    @Override



    protected int getExtraLayoutSpace(RecyclerView.State state) {



        return 300;



    }



};
```

## 3、[RecyclerView 数据预取](https://yq.aliyun.com/articles/226475)

android sdk>=21时，支持渲染（Render）线程，RecyclerView数据显示分两个阶段：

1）在UI线程，处理输入事件、动画、布局、记录绘图操作，每一个条目在进入屏幕显示前都会被创建和绑定view；

2）渲染（Render）线程把指令送往GPU。

数据预取的思想就是：将闲置的UI线程利用起来，提前加载计算下一帧的Frame Buffer

在新的条目进入视野前，会花大量时间来创建和绑定view，而在前一帧却可能很快完成了这些操作，导致前一帧的UI线程有一大片空闲时间。RecyclerView开发工程师将创建和绑定移到前一帧，使UI线程与渲染线程同时工作，在一个条目即将进入视野时预取数据。具体如下图，在前一帧的红色虚线圈中，UI线程有一定的空闲时间，可以把第二帧Create B的工作移到前一帧的空闲时间来完成。

![img](https://user-gold-cdn.xitu.io/2018/8/18/1654c887f6f67b48?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)具体实现方式是：在 RecyclerView 开始一个滚动时new Runnable对象，根据 layout manager 和滚动的方向预取即将进入视野的条目，可以同时取出一个或多个条目，例如在使用 GridLayoutManager 时新的一行马上要出现的时候。在 25.1 版本中，预取操作被分为单独的创建/绑定操作，比对整组条目操作更容易被纳入 UI 线程的空隙中。具体实现原理可参考：[RecyclerView预加载机制源码分析](https://juejin.im/post/5b77f1d0e51d4538900586fb)

![img](https://user-gold-cdn.xitu.io/2018/8/18/1654c887f70032f2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

完成这些工作基本上没有任何代价，因为 UI 线程在两帧之间的空隙不做任何工作。我们可以使用这些空闲时间来完成将来的工作，并使得未来的帧出现得更快，

如果使用 RecyclerView 提供的LayoutManager，自动使用了这种优化操作。如果使用嵌套 RecyclerView 或者自己实现Layout Manager，则需要在代码中设置。

1）对于嵌套 RecyclerView，要获取最佳的性能，在内部的 LayoutManager 中调用 LinearLayoutManager.[setInitialItemPrefetchCount()](https://developer.android.com/reference/android/support/v7/widget/LinearLayoutManager.html?spm=a2c4e.11153940.blogcont226475.24.12ed36e5YB6URE#setInitialPrefetchItemCount(int))方法（25.1版本起可用）。

例如：如果竖直方向的list至少展示三个条目，调用 setInitialItemPrefetchCount(4)。

2）如果自己实现了LayoutManager，需要重写 [LayoutManager.collectAdjacentPrefetchPositions()](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html#collectAdjacentPrefetchPositions(int, int, android.support.v7.widget.RecyclerView.State, android.support.v7.widget.RecyclerView.LayoutManager.LayoutPrefetchRegistry))方法。该方法在数据预取开启时被 RecyclerView 调用（LayoutManager 的默认实现什么都不做）。在嵌套的内层 RecyclerView 中，如果想让LayoutManager 预取数据，同样应当实现 [LayoutManager.collectInitialPrefetchPositions()](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html#collectInitialPrefetchPositions(int, android.support.v7.widget.RecyclerView.LayoutManager.LayoutPrefetchRegistry))。

## 4、避免创建过多对象

onCreateViewHolder 和 onBindViewHolder 对时间都比较敏感，尽量避免繁琐的操作和循环创建对象。例如创建 OnClickListener，可以全局创建一个。同时onBindViewHolder调用次数会多于onCreateViewHolder的次数，如从RecyclerViewPool缓存池中取到的View都需要重新bindView，所以我们可以把监听放到CreateView中进行。

优化前：

```java
@Override



public void onBindViewHolder(ViewHolder holder, int position) {



    holder.setOnClickListener(new View.OnClickListener() {



       @Override



       public void onClick(View v) {



         //do something



       }



    });



}
```

优化后：

```java
private class XXXHolder extends RecyclerView.ViewHolder {



        private EditText mEt;



        EditHolder(View itemView) {



            super(itemView);



            mEt = (EditText) itemView;



            mEt.setOnClickListener(mOnClickListener);



        }



    }



    private View.OnClickListener mOnClickListener = new View.OnClickListener() {



        @Override



        public void onClick(View v) {



            //do something



        }



    }
```

## 5、局部刷新

可以用一下一些方法，替代notifyDataSetChanged，达到局部刷新的目的。notifyDataSetChanged会触发所有item的detached回调再触发onAttached回调。

notifyItemChanged(int position)
notifyItemInserted(int position)
notifyItemRemoved(int position)
notifyItemMoved(int fromPosition, int toPosition) 
notifyItemRangeChanged(int positionStart, int itemCount)
notifyItemRangeInserted(int positionStart, int itemCount) 
notifyItemRangeRemoved(int positionStart, int itemCount) 
如果必须用 notifyDataSetChanged()，那么最好设置 mAdapter.setHasStableIds(true)

## 6、重写onScroll事件

对于大量图片的RecyclerView，滑动暂停后再加载；RecyclerView中存在几种绘制复杂，占用内存高的楼层类型，但是用户只是快速滑动到底部，并没有必要绘制计算这几种复杂类型，所以也可以考虑对滑动速度，滑动状态进行判断，满足条件后再加载这几种复杂的。

## 7、RecyclerView缓存

### 7.1 setItemViewCacheSize(int )

RecyclerView可以设置自己所需要的ViewHolder缓存数量，默认大小是2。cacheViews中的缓存只能position相同才可得用，且不会重新bindView，CacheViews满了后移除到RecyclerPool中，并重置ViewHolder，如果对于可能来回滑动的RecyclerView，把CacheViews的缓存数量设置大一些，可以减少bindView的时间，加快布局显示。

注：此方法是拿空间换时间，要充分考虑应用内存问题，根据应用实际使用情况设置大小。

网上大部分设置CacheView大小时都会带上：

setDrawingCacheEnabled(true)和setDrawingCacheQuality(View.DRAWING_CACHE_QUALITY_HIGH)

setDrawingCacheEnabled这个是View本身的方法，意途是开启缓存。通过setDrawingCacheEnabled把cache打开，再调用getDrawingCache就可以获得view的cache图片，如果cache没有建立，系统会自动调用buildDrawingCache方法来生成cache。一般截图会用到，这里的设置drawingcache，可能是在重绘时不需要重新计算bitmap的宽高等，能加快dispatchDraw的速度，但开启drawingcache，肯定也会耗应用的内存，所以也慎用。

### 7.2 复用RecycledViewPool

在TabLayout+ViewPager+RecyclerView的场景中，当多个RecyclerView有相同的item布局结构时，多个RecyclerView共用一个RecycledViewPool可以避免创建ViewHolder的开销，避免GC。RecycledViewPool对象可通过RecyclerView对象获取，也可以自己实现。

RecycledViewPool mPool = mRecyclerView1.getRecycledViewPool();
下一个RecyclerView可直接进行setRecycledViewPool

mRecyclerView2.setRecycledViewPool(mPool);

mRecyclerView3.setRecycledViewPool(mPool);

注意：

（1）RecycledViewPool是依据ItemViewType来索引ViewHolder的，必须确保共享的RecyclerView的Adapter是同一个，或view type 是不会冲突的。

（2）RecycledViewPool可以自主控制需要缓存的ViewHolder数量，每种type的默认容量是5，可通过setMaxRecycledViews来设置大小。mPool.setMaxRecycledViews(itemViewType, number); 但这会增大应用内存开销，所以也需要根据应用具体情况来使用。

（3）利用此特性一般建议设置layout.setRecycleChildrenOnDetach(true);此属性是用来告诉LayoutManager从RecyclerView分离时，是否要回收所有的item，如果项目中复用RecycledViewPool时，开启该功能会更好的实现复用。其他RecyclerView可以复用这些回收的item。

什么时候LayoutManager会从RecyclerView上分离呢，有两种情况：1）重新setLayoutManager()时，比如淘宝页面查看商品列表，可以线性查看，也可以表格形式查看，2）还有一种是RecyclerView从视图树上被remove时。但第一种情况，RecyclerView内部做了回收工作，设不设置影响不大，设置此属性作用主要针对第二种情况。

## 8、RecyclerView中的一些方法

**`onViewRecycled()`**：当 ViewHolder 已经确认被回收，且要放进 RecyclerViewPool 中前，该方法会被回调。移出屏幕的ViewHolder会先进入第一级缓存ViewCache中，当第一级缓存空间已满时，会考虑将一级缓存中已有的ViewHolder移到RecyclerViewPool中去。在这个方法中可以考虑图片回收。

**onViewAttachedFromWindow()**： RecyclerView的item进入屏幕时回调
**onViewDetachedFromWindow()**：RecyclerView的item移出屏幕时回调

**onAttachedToRecyclerView()** ：当 RecyclerView 调用了 `setAdapter()` 时会触发，新的 adapter 回调 onAttached。
**onDetachedFromRecyclerView()**：当 RecyclerView 调用了 `setAdapter()` 时会触发，旧的 adapter 回调 onDetached

**setHasStableIds()／getItemId()：**setHasStableIds用来标识每一个itemView是否需要一个唯一标识，当stableId设置为true的时候，每一个itemView数据就有一个唯一标识。getItemId()返回代表这个ViewHolder的唯一标识，如果没有设置stableId唯一性，返回NO_ID=-1。通过setHasStableIds可以使itemView的焦点固定，从而解决RecyclerView的notify方法使得图片加载时闪烁问题。注意：setHasStableIds()必须在 `setAdapter()` 方法之前调用，否则会抛异常。因为RecyclerView.setAdapter后就设置了观察者，设置了观察者stateIds就不能变了。具体案例可参考：[RecyclerView notifyDataSetChanged 导致图片闪烁的真凶](https://www.jianshu.com/p/29352def27e6)

 

### 9、更多高级用法

### 9.1 SnapHelper实现卡片效果或ViewPager效果

SnapHelper是一个抽象类，Google 内置了两个默认实现类，`LinearSnapHelper`和`PagerSnapHelper` 。

1）LinearSnapHelper可以使RecyclerView 的当前Item 居中显示（横向和竖向都支持）

2）PagerSnapHelper使RecyclerView 像ViewPager一样的效果，每次只能滑动一页（LinearSnapHelper支持快速滑动）, PagerSnapHelper也是Item居中对齐。

使用方法如下，想了解更多可参考[Android中使用RecyclerView + SnapHelper实现类似ViewPager效果](https://www.jianshu.com/p/ef3a3b8d0a77)：

```java
 LinearLayoutManager manager = new LinearLayoutManager(getContext());



 manager.setOrientation(LinearLayoutManager.VERTICAL);



 mRecyclerView.setLayoutManager(manager);



// 将SnapHelper attach 到RecyclrView



 LinearSnapHelper snapHelper = new LinearSnapHelper();



 snapHelper.attachToRecyclerView(mRecyclerView);
```

### 9.2 用SortedList实现添加删除ItemView自动更新

我们在给RecyclerView的ArrayList<Item> data添加一个Data数据时，一般需要自己通知RecyclerView更新，尤其是遇到去重操作，还需要遍历一次data，定位后再决定是插入还是更新现有数据，调用notifyItemInserted(pos)，Android Support Lirbrary中提供了一个SortedList工具类，它是一个有序列表，数据变动时会回调SortedList.Callback中方法。

具体使用：

```java
class SortedListAdapter extends RecyclerView.Adapter<TodoViewHolder> {



    final SortedList<Item> mData;



    final LayoutInflater mLayoutInflater;



    public SortedListAdapter(Context context) {



        mLayoutInflater = LayoutInflater.from(context);



        mData = new SortedList<Item>(Item.class, new SortedListAdapterCallback<Item>(this){



            @Override



            public int compare(Item t0, Item t1) {



                // 实现这个方法来定义Item的显示顺序



                int txtComp = t0.mText.compareTo(t1.mText);



                if (txtComp != 0) {



                    return txtComp;



                }



                if (t0.id < t1.id) {



                    return -1;



                } else if (t0.id > t1.id) {



                    return 1;



                }



                return 0;



            }



            @Override



            public boolean areContentsTheSame(Item oldItem,



                    Item newItem) {



                // 比较两个Item的内容是否一致，如不一致则会调用adapter的notifyItemChanged()



                return oldItem.mText.equals(newItem.mText);



            }



            @Override



            public boolean areItemsTheSame(Item item1, Item item2) {



                // 两个Item是不是同一个东西，



                // 它们的内容或许不一样，但id相同代表就是同一个



                return item1.id == item2.id;



            }



        });



    }



    public void addItem(Item item) {



        mData.add(item);



        // 会通过SortedListAdapterCallback自动通知更新



    }



    ...



    @Override



    public int getItemCount() {



        return mData.size();



    }



}
```

当数据发生改变时，例如删除，增加等，只需直接对mDataList进行相应操作，无需关心mAdapter内数据显示更新问题，不用再调用notifyDataChanged等函数，因为SortedListAdapterCallback内的回调函数自动完成了。

### 9.3 [详解7.0带来的新工具类：DiffUtil](https://blog.csdn.net/aiynmimi/article/details/70214513)

DiffUtil是support-v7:24.2.0中的新工具类，它用来比较两个数据集，寻找出旧数据集—>新数据集的最小变化量，它和`mAdapter.notifyDataSetChanged()最大不同在于`它会自动计算新老数据集的差异，并根据差异情况，自动调用以下四个方法：

```java
adapter.notifyItemRangeInserted(position, count);



adapter.notifyItemRangeRemoved(position, count);



adapter.notifyItemMoved(fromPosition, toPosition);



adapter.notifyItemRangeChanged(position, count, payload);
```

且调用`notifyDataSetChanged()`不会触发RecyclerView的动画（删除、新增、位移、change动画），其次性能较低，它不管数据是否一样都整个刷新了一遍整个RecyclerView 。

具体使用方法：

`DiffUtil.Callback`抽象类如下：

```java
    public abstract static class Callback {



        public abstract int getOldListSize();//老数据集size



 



        public abstract int getNewListSize();//新数据集size



 



        //新老数据集在同一个position的Item是否是一个对象，如果给itemView设置了stableIds，则仅比较它们单独的id（可能内容不同，如果这里返回true，会调用下面的方法）



        public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);



 



        //这个方法仅仅是上面方法返回true才会调用，判断item的内容是否有变化，类似于Object.equals(Object)



        public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);



 



        //当areItemsTheSame()返回true且areContentsTheSame()返回false，用下面的方法找出两个itemView的data不同之处



        @Nullable



        public Object getChangePayload(int oldItemPosition, int newItemPosition) {



            return null;



        }



    }
```

使用时需要实现Callback接口，再将差异结果赋值到我们写的Adapter上。

```java
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new ProductListDiffCallback(mProducts, newProducts));



diffResult.dispatchUpdatesTo(mProductAdapter);
```

有一篇外国文章介绍的也很好：[DiffUtil使用介绍](https://proandroiddev.com/diffutil-is-a-must-797502bc1149)

### 9.4 [NestedScrollView嵌套RecyclerView](https://www.cnblogs.com/fuyaozhishang/p/8232378.html)

1) **滑动lRecyclerView列表会出现强烈的卡顿感**

mRecyclerView.setNestedScrollingEnabled(false);//RecyclerView默认是setNestedScrollingEnabled(true),是支持嵌套滚动的,也就是说当它嵌套在NestedScrollView中时,默认会随着NestedScrollView滚动而滚动,放弃了自己的滚动。将该值置false可以让RecyclerView不支持嵌套滑动，这样RecyclerView可以自己响应滑动事件。

2）**每次打开界面都是定位在RecyclerView在屏幕顶端,列表上面的布局都被顶上去了**

RecyclerView抢占了焦点,自动滚动导致的.

RecyclerView会在构造方法中调用setFocusableInTouchMode(true), 抢占焦点后一定会定位到第一行的位置，可以在NestedScrollView中添加属性：android:focusableInTouchMode="true"，同时在RecyclerView中添加属性：android:descendantFocusability="blocksDescendants"或直接设置mRecyclerVIew.setFocusableInTouchMode(false)

## 10、别人遇到的问题

### 10.1 由于RecyclerView缓存view复用导致图片错乱

[Recyclerview的缓存机制](https://blog.csdn.net/feelinghappy/article/details/80604566)，作者主要在对RecyclerView的ItemView某些图片进行了属性动画变换，这样就改变了ViewHolder中ImageView的属性，在滑动时，RecyclerView的缓存复用机制可能导致ViewHolder不会重新创建，也不会重新bindView，这样某些ItemView的图片是View属性动画变换后的图片，导致不是自己想要的结果。

### 10.2 由于RecyclerView关联的GapWorker导致内存泄漏

[RecyclerView导致内存泄漏问题分析](https://blog.csdn.net/c16882599/article/details/60140312)，其实主要是RecyclerView关联的GapWorker中有一个静态的ThreadLocal对象，静态属性生命周期和应用进程生命周期一致，发生内存泄漏肯定是因为GapWorker的引用链一直关联到Activity中，且没有在相应的时候释放这条引用链。按道理RecyclerView内部onAttachedToWindow和onDetachedFromWindow分别进行了引用和释放引用，是不会发生内存泄漏的，但是由于开发者应对的环境不一样，遇到的坑也不一样。作者这种分析办法还是很值得学习。

## 后记：

RecyclerView的优化点肯定还有很多，坑也还有很多，这和应用的实际使用情况有很大关系。同时Google开发工程师也一直在优化RecyclerView，我们也要一直学习着。





出处: [CSDN- 潇潇凤儿](https://blog.csdn.net/smileiam/article/details/88396546)