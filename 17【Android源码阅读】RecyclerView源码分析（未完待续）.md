
## 一、名词概述

 - Adapter : 是RecyclerView的内部类，根据DataSet为RecyclerView提供显示的view。

 - Recycle : View滑出屏幕后，会被存储在一个缓存区里，用来被具有相同ViewType的Item重用；通过减少layout inflation和construction来提升滑动性能。

 - Scrap (ViewHolder) : 临时脱离屏幕的ViewHolder，如果在脱离过程中没有被重用（没有被rebinding），可以被直接重用，分为AttachedScrap和ChangedScrap两种，后面详细介绍。

<!--more-->

 - Dirty（ViewHolder）: 一个ViewHolder在显示之前被新的item的数据rebinding了，就说它是Dirty（脏）的，就不能直接被使用。

 - RecycledViewPool : 用于多个RecyclerView共享ViewHolder的换存池，如果不手动设置，默认每个RecyclerView拥有一个独立的Pool

 - Position : Data item在DataSet中的位置
  - LayoutPosition : Item在布局中的位置
  - AdapterPositon : Item的数据在Adapter DataSet中的位置，二者通过notifyDataSetChange()来同步

 - Id : 如果设置了hasStableIds为true，RecyclerView会为每一个Item设置一个唯一标识符，就是id，则个id需要通过重写Adapter的getItemId来生成，任何情况下，就算Adapter里dataSet的顺序和内容发生变化，也会根据id来标识；如果不设置hasStableIds为true，Adapter就会根据position来标识Item。

----------

## 二、Recycler

Recycler是RecyclerView的一个内部类，负责管理 ViewHolder 的复用。

##### 下面是Recycler中几个重要的成员变量

 - 1、mAttachedScrap、mChangedScrap

 也就是上述提到的Scrap，存储与RecyclerView显示区域脱离的ViewHolder，如果viewHolder支持itemAnimation，就放入mAttachedScrap，否则放入mChangedScrap

 - 2、mCachedViews

  - 在View离开屏幕时，如果viewHolder绑定的数据还在adapter中，而且没有设置stableId，就放入CachedViews，Cache中的ViewHolder随时可能重新出现在屏幕上而不需要重新Bind；如果绑定的数据已经从Adapter中删除了，或者是没设置stableId，就会放入上面的Scrap中

  - 在只有一种item的情况下，缓存的ViewHolder的数目为RecyclerView在滑动过程中所能在一屏内容纳的最大item个数+2。比如，在一个屏幕中只有item A可以显示，在滑动的过程最多可以出现6个item（这个最多是指所有item的个数，包括显示完全和显示不完全的总数），那么ViewHolder的缓存个数将会是8；而有至少两种item显示的情况下，每种item的ViewHolder的缓存个数为单种item在一屏内最大显示个数+1；比如，有3种item A，B，C，在滑动的过程，item A最多的情况下一屏显示了2个，那个item A对应的ViewHolder的缓存个数就是3个。

 - 3、RecycledViewPool

  - mCachedViews里的ViewHolder会在Cache满了之后，不断的放入RecycledViewPool中。

  - RecyclerViewPool里有两个成员变量：SparseArray<ArrayList<ViewHolder>> mScrap 和 SparseIntArray mMaxScrap，mScrap根据ViewType做了ViewHolder的二级索引；mMaxScrap表示每一类ViewType的允许保存scrapView的最大数量，当同一类ViewType的ViewHolder数量超过mMaxScrap时，会舍弃掉最末端的ViewHolder。


- 下面是将ViewHolder放入Scrap和Cache的部分代码

``` Java

//在这里决定是将ViewHolder放入Cache还是Scrap
private void scrapOrRecycleView(Recycler recycler, int index, View view) {
            final ViewHolder viewHolder = getChildViewHolderInt(view);
            if (viewHolder.shouldIgnore()) {
                if (DEBUG) {
                    Log.d(TAG, "ignoring view " + viewHolder);
                }
                return;
            }
            //如果viewHolder上次绑定的数据还在adapter中，而且没有设置stableId，就调用recycleViewHolderInternal放入cache
            //cache中的viewHolder在复用的时候只检查viewType
            if (viewHolder.isInvalid() && !viewHolder.isRemoved() &&
                    !mRecyclerView.mAdapter.hasStableIds()) {
                removeViewAt(index);
                //放入Cache
                recycler.recycleViewHolderInternal(viewHolder);
             } else {
                //如果viewHolder绑定的数据已经不在adapter中了，或者是设置了stableId的，则调用scrapView放入scrap中
                //scrap中缓存的viewHolder要同时检查viewType和id
                detachViewAt(index);
                //放入Scrap
                recycler.scrapView(view);
            }
}

//在这里决定是将ViewHolder放到哪一类Scrap
void scrapView(View view) {
            final ViewHolder holder = getChildViewHolderInt(view);
            holder.setScrapContainer(this);
            //如果该viewHolder支持itemAnimation，就放入mAttachedScrap
            if (!holder.isChanged() || !supportsChangeAnimations()) {
                if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
                    throw new IllegalArgumentException("Called scrap view with an invalid view."
                            + " Invalid views cannot be reused from scrap, they should rebound from"
                            + " recycler pool.");
                }
                mAttachedScrap.add(holder);
            } else { //如果不支持ItemAnimation，则放入mChangedScrap
                if (mChangedScrap == null) {
                    mChangedScrap = new ArrayList<ViewHolder>();
                }
                mChangedScrap.add(holder);
            }
}

```

 - 上面只讲了如何缓存ViewHolder，如果取出ViewHolder复用之后有时间再详细讲解~

##### 下面是Recycler中几个重要的方法


 - 这些方法以后有时间再详细讲解~

 - public void bindViewToPosition(View view, int position)  

 - public View getViewForPosition(int position)

 - onCreateViewHolder(ViewGroup viewGroup, int i)

  - 在任何ViewHolder被实例化的时候，OnCreateViewHolder将会被触发

 - onBindViewHolder(MyHolder myHolder, int i)

  - OnCreateViewHolder创建了一个ViewHolder的实例，之后，onBindViewHolder方法则负责将数据与ViewHolder绑定：


----------

## 三、LayoutManager

  - 这个以后有时间再详细讲解~

  LayoutManager 主要是用于测量和摆放 RecyclerView 中 ItemView，以及当 Item 不可见时循环复用处理。RecyclerView 提供了三种样式的 LayoutManager，这里先不赘述。

  LayoutManager 获取 Adapter 某一项的 ViewHolder 时会先去 Recycler的Scrap里查找，当 ViewHolder还没有被Rebind时，ViewHolder会直接显示出来，不需要重新measure和layout；如果View是Dirty的，则需要重新绑定数据，需要重新measure和layout；如果Scrap里找不到，就会到RecycledViewPool里找，如果还找不到ViewType相同的ViewHolder，就会创建一个新的ViewHolder。

----------

## 四、ViewHolder

以前写ListView的时候会自己去实现一个ViewHolder，RecyclerView直接将ViewHolder内置了，使用上来说更方便。

``` Java
public static abstract class ViewHolder {
    View itemView;
    int mPosition;
    int mOldPosition;
    long mItemId;//getItemId()返回的id
    int mItemViewType;//getItemViewType()
    int mPreLayoutPosition;
    int mFlags;
    int mIsRecyclableCount;
    Recycler mScrapContainer;
｝
```

查找ViewHolder的步骤：
0、在Cache中找，Cache中的ViewHolder一般来说不需要重新bind数据，可以直接显示
1、在changeScrap里找(getChangedScrapViewForPosition),mAttachedScrap里找（getScrapViewForPosition，getScrapViewForId）
2、在RecyclerViewPool里找

下面看看mFlags的值：

 - FLAG_BOUND

 ViewHolder已经绑定到某个位置，并且mPostion，mItemId，mItemViewType是有效的

 - FLAG_UPDATE

 ViewHolder绑定的数据过时了，需要重新绑定，但是mPosition，mItemId是不变的

 - FLAG_INVALID

 ViewHolder绑定的View对应的数据无效，需要重新绑定不同的数据  

 - FLAG_REMOVED

 ViewHolder对应的数据已经从数据集删除

 - FLAG_NOT_RECYCLABLE

 ViewHolder不能复用

 - FLAG_RETURNED_FROM_SCRAP

 这个状态的ViewHolder会加到scrap集合

 - FLAG_IGNORE

 ViewHolder完全由LayoutManager管理，不能复用，不能删除

 - FLAG_TMP_DETACHED

 ViewHolder从父RecyclerView临时分离的标志，便于后续移除或添加回来  
 - FLAG_ADAPTER_POSITION_UNKNOWN

 ViewHolder不知道对应的Adapter的位置，直到绑定到一个新位置

----------
