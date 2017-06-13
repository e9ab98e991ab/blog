#View的工作流程
##View的measure过程
View的measure过程由measure方法来完成，其中会调用onMeasure方法来确定View的大小。一般来说，继承View的子类都会重写onMeasure方法来确定大小。

首先，看看View的onMeasure方法。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

onMeasure有两个参数：widthMeasureSpec，heightMeasureSpec。这两个参数是由父容器和View自身LayoutParams确定的，用来对View宽/高的限制进行说明(ViewGroup的measure过程会有详细介绍)。

getDefaultSize方法的实现：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    
    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
        break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

getDefaultSize方法的作用就是根据measureSpec参数返回一个宽/高值。要注意，measureSpec的specMode为MeasureSpec.AT_MOST时返回宽/高值都为measureSpec的specSize。通常，自定义View需要对specMode为MeasureSpec.AT_MOST这种情况进行处理。

getDefaultSize方法的第一个参数由getSuggestedMinimumWidth/getSuggetedMinimumHeight方法决定。两个方法相类似，只分析getSuggestedMinimumWidth实现：

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

```java
public int getMinimumWidth() {
    final int intrinsicWidth = getIntrinsicWidth();
    return intrinsicWidth > 0 ? intrinsicWidth : 0;
}
```

没有设置背景设置背景时，返回mMinWidth，这个值对应android:minWidth属性，或通过setMinimumWidth方法设置。getMinimumWidth方法获取背景的固有宽高。但是要注意，不是所有Drawable都有原始宽高，比如ShapeDrawable没有原始宽高，BitmapDrawable有原始宽高。

##ViewGroup的measure过程
ViewGroup没有重写onMeasure方法，因为ViewGroup是个抽象类，不能定义每种布局的测量方式。ViewGroup的子类都有重写onMeasure方法。接下来的分析以LinearLayout为例。

LinearLayout的onMeasure方法实现：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

LinearLayout对横向和纵向布局方式进行了不同实现，但是实现过程都相类似，选取纵向作分析，横向可以类推。开始逐步分析measureVertical方法的实现过程。由于源码较长，只截取重要部分进行分析。

```java
LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
totalWeight += lp.weight;
if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
    // Optimization: don't bother measuring children who are going to use
    // leftover space. These views will get measured again down below if
    // there is any leftover space.
    final int totalLength = mTotalLength;
    mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
    skippedMeasure = true;
    } else {
    ......
    }
```

这是一个判断，当LinearLayout高度的模式为MeasureSpec.EXACTLY并且child高度为0并且设置的weight>0时，不会对这个child测量大小。通俗点说，LinearLayout高度为定值或者"match_parent"，child的高度设置为0dp，且weight值>0，该child就会跳过测量。相反于上面情况，也就是在else语句块中会测量child大小,实现过程如下：

```java
int oldHeight = Integer.MIN_VALUE;
if (lp.height == 0 && lp.weight > 0) {
// heightMode is either UNSPECIFIED or AT_MOST, and this
// child wanted to stretch to fill available space.
// Translate that to WRAP_CONTENT so that it does not end up
// with a height of 0
oldHeight = 0;
lp.height = LayoutParams.WRAP_CONTENT;
}
// Determine how big this child would like to be. If this or
// previous children have given a weight, then we allow it to
// use all available space (and we will shrink things later
// if needed).
measureChildBeforeLayout(child, i, widthMeasureSpec, 0, heightMeasureSpec,totalWeight == 0 ? mTotalLength : 0);
if (oldHeight != Integer.MIN_VALUE) {
    lp.height = oldHeight;
}

final int childHeight = child.getMeasuredHeight();
final int totalLength = mTotalLength;
mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +lp.bottomMargin + getNextLocationOffset(child));
```

为了使child高度为0，并且weight>0的情况也能够测量child的高度，暂时先给child的高度赋值为wrap_content。measureChildBeforeLayout方法对child进行测量。measureChildBeforeLayout方法的最后一个参数表示被使用的高度值，这个参数由一个三目表达式决定，totalWeight!=0表示之前有child设置了weight属性，那么之前设置了weight属性的child虽然可能测量过，但是并不是最终的宽/高，也就是说大小仍未确定，所以对于之后的child测量直接传0。measureChildBeforeLayout的实现过程:

```java
void measureChildBeforeLayout(View child, int childIndex,
    int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
    int totalHeight) {
        measureChildWithMargins(child, widthMeasureSpec,totalWidth,
            heightMeasureSpec, totalHeight);
    }
```

直接调用了measureChildWithMargins方法:

```java
protected void measureChildWithMargins(View child,
    int parentWidthMeasureSpec, int widthUsed,
    int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams)child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
        mPaddingLeft + mPaddingRight + lp.leftMargin + 
        lp.rightMargin + widthUsed, lp.width);
    final int childHeightMeasureSpec=getChildMeasureSpec(parentHeightMeasureSpec,mPaddingTop + 
        mPaddingBottom + lp.topMargin + lp.bottomMargin+ heightUsed,lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

主要是通过getChildMeasureSpec方法确定对child进行限制的widthMeasureSpec和heightMeasureSpec。确定好child的widthMeasureSpec和heightMeasureSpec,child调用measure方法，之后的过程就属于View的measure过程了。getChildMeasureSpec方法实现:

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
        break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
        break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
        break;
        }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

上面算是LinearLayout完成了第一次测量，如果child有被跳过的或者有些child有设置weight属性的，那么第一次的测量结果是不准确的，所以要再进行一次测量：

```java
int heightSize = mTotalLength;

// Check against our minimum height
heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
        
// Reconcile our calculated size with the heightMeasureSpec
int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
// Either expand children with weight to take up available space or
// shrink them if they extend beyond our current bounds. If we skipped
// measurement on any children, we need to measure them now.
int delta = heightSize - mTotalLength;
if (skippedMeasure || delta != 0 && totalWeight > 0.0f) {
    float weightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

    mTotalLength = 0;

    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
                
        if (child.getVisibility() == View.GONE) {
            continue;
        }
                
        LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
                
        float childExtra = lp.weight;
        if (childExtra > 0) {
            // Child said it could absorb extra space -- give him his share
            int share = (int) (childExtra * delta / weightSum);
            weightSum -= childExtra;
            delta -= share;

            final int childWidthMeasureSpec =getChildMeasureSpec(
                widthMeasureSpec,mPaddingLeft + mPaddingRight +
                lp.leftMargin + lp.rightMargin, lp.width);

            // TODO: Use a field like lp.isMeasured to figure out if this
            // child has been previously measured
            if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY)) {
                // child was measured once already above...
                // base new measurement on stored values
                int childHeight = child.getMeasuredHeight() + share;
                if (childHeight < 0) {
                    childHeight = 0;
                }
                        
                child.measure(childWidthMeasureSpec,MeasureSpec.
                makeMeasureSpec(childHeight,MeasureSpec.EXACTLY));
            } else {
                // child was skipped in the loop above.
                // Measure for this first time here      
                child.measure(childWidthMeasureSpec,MeasureSpec.
                makeMeasureSpec(share > 0 ? share : 0,
                MeasureSpec.EXACTLY));
            }

            // Child may now not fit in vertical dimension.
            childState = combineMeasuredStates(childState, child.
            getMeasuredState()&(MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
        }

        final int margin =  lp.leftMargin + lp.rightMargin;
        final int measuredWidth = child.getMeasuredWidth() + margin;
        maxWidth = Math.max(maxWidth, measuredWidth);

        boolean matchWidthLocally = widthMode !=MeasureSpec.EXACTLY
        &&lp.width == LayoutParams.MATCH_PARENT;

        alternativeMaxWidth = Math.max(alternativeMaxWidth,
            matchWidthLocally ? margin : measuredWidth);

        allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

        final int totalLength = mTotalLength;
        mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +lp.topMargin +lp.bottomMargin + getNextLocationOffset(child));
    }

    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;
    // TODO: Should we recompute the heightSpec based on the new total length?
}
```

resolveSizeAndState方法根据第一次高度测量结果，父容器对LinearLayout的限制来进一步确定高度。 这个高度是LinearLayout的最终高度。之前child有测量过的会根据已测量高度、delta偏移量和weight属性占总weight比重新确定child的heightMeasureSpec，之后再重新测量。之前没有测量过的child则根据自身weight占总weight比重和delta确定heightMeasureSpec，再进行测量。

LinearLayout的measure过程:
![](https://github.com/wslaimin/blog/raw/master/pics/LinearLayout_measure.png)

##View的layout过程
View的layout方法最终都会调用setFrame方法来设置四个顶点的位置。onLayout方法默认是空实现。
##ViewGroup的layout过程
LinearLayout的onLayout方法的实现逻辑与onMeasure的实现逻辑类似，onLayout方法中会调用layoutVertical方法对children进行遍历，调用child的layout方法。确定childTop的过程：

```java
switch (majorGravity) {
    case Gravity.BOTTOM:
    // mTotalLength contains the padding already
    childTop = mPaddingTop + bottom - top - mTotalLength;
    break;

    // mTotalLength contains the padding already
    case Gravity.CENTER_VERTICAL:
    childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
    break;

    case Gravity.TOP:
    default:
        childTop = mPaddingTop;
    break;
}
```

bottom-top表示LinearLayout测量后的高度，mTotalLength是LinearLayout的真实高度，两者可能会不相等，从LinearLayout的onMeasure过程也可以看出来。

##总结
![](https://github.com/wslaimin/blog/raw/master/pics/measure_flow.png)





