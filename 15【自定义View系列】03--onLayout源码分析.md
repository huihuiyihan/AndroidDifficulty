
本文转载自：http://blog.csdn.net/lfdfhl/article/details/51393131

----------

## 一、源码分析

在经过measure阶段以后，系统确定了View的测量大小，接下来就进入到layout的过程。

在该过程中会确定视图的显示位置，即子View在其父控件中的位置。

先看View的layout( )方法：


``` Java
//l, t, r, b分别表示子View相对于父View的左、上、右、下的坐标
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this,l,t,r,b,oldL,oldT,oldR,oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
```

1 确定该View在其父View中的位置。
在该处调用setFrame()方法，在该方法中把l，t， r， b分别与之前的mLeft，mTop，mRight，mBottom一一作比较，假若其中任意一个值发生了变化，那么就判定该View的位置发生了变化

2 若View的位置发生了变化则调用onLayout()方法

View中的onLayout是一个空方法，ViewGroup中的onLayout是一个抽象方法，实际上是ViewGroup的子类实现了onLayout：

``` Java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

我们来看看LinearLayout的onLayout( )方法：

``` Java
	protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```

``` Java
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;
    int childTop;
    int childLeft;
    final int width = right - left;
    int childRight = width - mPaddingRight;

    int childSpace = width - paddingLeft - mPaddingRight;

    final int count = getVirtualChildCount();

    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

    switch (majorGravity) {
          case Gravity.BOTTOM:
              childTop = mPaddingTop + bottom - top - mTotalLength;
               break;

          case Gravity.CENTER_VERTICAL:
              childTop =mPaddingTop+(bottom-top-mTotalLength) / 2;
              break;

          case Gravity.TOP:
          default:
              childTop = mPaddingTop;
              break;
    }

    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();

            final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();

            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                + lp.leftMargin - lp.rightMargin;
                    break;

                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;

                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }

            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            setChildFrame(child,childLeft,childTop+ getLocationOffset(child),
                        childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```

***第一步：***
计算child可使用空间的大小，请参见代码第8行

***第二步：***
获取子View的个数，请参见代码第10行

***第三步：***
计算childTop从而确定子View的开始布局位置，请参见代码第12-28行

***第四步：***
确定每个子View的位置，请参见代码第30-74行：

1 得到子View测量后的宽和高，请参见代码第35-36行.
这里获取到的childWidth和childHeight就是在measure阶段所确立的宽和高

2 得到子View的LayoutParams，请参见代码第38-39行.

3 依据子View的LayoutParams确定子View的位置，请参见代码第41-69行.

我们可以发现在setChildFrame()中又调用了View的layout()方法来确定子View的位置。

到这我们就可以理清楚思路了：

ViewGroup首先调用了layout()确定了自己本身在其父View中的位置，然后调用onLayout()确定每个子View的位置，每个子View又会调用View的layout()方法来确定自己在ViewGroup的位置。

概况地讲：View的layout()方法用于View确定自己本身在其父View的位置 ;ViewGroup的onLayout()方法用于确定子View的位置


----------
## 二、注意事项

***1 获取View的测量大小measuredWidth和measuredHeight的时机。***

在某些复杂或者极端的情况下系统会多次执行measure过程，所以在onMeasure()中去获取View的测量大小得到的是一个不准确的值。为了避免该情况，最好在onMeasure()的下一阶段即onLayout()中去获取。

***2 getMeasuredWidth()和getWidth()的区别***

在绝大多数情况下这两者返回的值都是相同的，但是结果相同并不说明它们是同一个东西。

首先，它们的获取时机是不同的。

在measure()过程结束后就可以调用getMeasuredWidth()方法获取到View的测量大小，而getWidth()方法要在layout()过程结束后才能被调用从而获取View的实际大小。

其次，它们返回值的计算方式不同。

getMeasuredWidth()方法中的返回值是通过setMeasuredDimension()方法得到的，这点我们之前已经分析过，在此不再赘述；而getWidth()方法中的返回值是通过View的右坐标减去其左坐标(right-left)计算出来的。

***3 刚才说到了关于View的坐标，在这就不得不提一下：***

view.getLeft()，view.getRight()，view.getBottom()，view.getTop();

这四个方法用于获取子View相对于父View的位置。
但是请注意:

getLeft( )表示子View的左边距离父View的左边的距离

getRight( )表示子View的右边距离父View的左边的距离

getTop( )表示子View的上边距离父View的上边的距离

getBottom( )表示子View的下边距离父View的上边的距离
