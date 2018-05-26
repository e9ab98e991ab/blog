# ListView滚动机制分析

## 目的
了解ListView的滚动机制，在滚动过程中View如何添加到ListView。

## 实现过程
先简单介绍下ListView滚动的大概算法。

算法实现：

 1. ACTION_MOVE动作触发，滚出屏幕外的View添加进scrp views，并且从ListView里detach
 2. 还在屏幕上的View，向滚动方向移动滚动距离
 3. 填充views
 4. 修正多余的滚动距离

图示：
![](https://www.github.com/wslaimin/blog/raw/master/pics/listview_scroll.png)

## 源码分析
下面分析ACTION_MOVE的方向为从下往上。

1.ACTION_MOVE动作触发
ListView没有重写onTouchEvent(MotionEvent ev)方法，追溯到父类AbsListView，最终调用关键的scrollIfNeeded(int x, int y, MotionEvent vtev)方法。
scrollIfNeeded(int x,int y,MotionEvent vtev)方法中有段代码：

```java
// No need to do all this work if we're not going to move anyway
boolean atEdge = false;
if (incrementalDeltaY != 0) {
    atEdge = trackMotionScroll(deltaY, incrementalDeltaY);
}
```

trackMotionScroll(int deltaY, int incrementalDeltaY)方法开始对ListView里的views进行处理。deltaY是ACTION_DOWN触发开始到现在移动的距离，incrementalDeltaY是每次ACTION_MOVE之间移动的距离。

2.滚动到屏幕外的views添加进scrap views
下面代码是从下往上滚动：

```java
......
final boolean down = incrementalDeltaY < 0;
......
if (down) {
    //移动views距离和改变top距离来判断是否滚出屏幕是等效的
    int top = -incrementalDeltaY;
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
    top += listPadding.top;
    }
    for (int i = 0; i < childCount; i++) {
        final View child = getChildAt(i);
        if (child.getBottom() >= top) {
            break;
        } else {
            count++;
            int position = firstPosition + i;
            if (position >= headerViewsCount && position < footerViewsStart) {
                // The view will be rebound to new data, clear any
                // system-managed transient state.
                child.clearAccessibilityFocus();
                mRecycler.addScrapView(child, position);
            }
        }
    }
}
......
```

3.移动还在屏幕上views

```java
offsetChildrenTopAndBottom(incrementalDeltaY);
```
 
4.填充views
```java
final int absIncrementalDeltaY = Math.abs(incrementalDeltaY);
if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
    fillGap(down);
}
```

从上往下滑动，第一个child在屏幕外高度>absIncrementalDeltaY,不用填充view;从下往上滑动，最后一个child在屏幕外高度>absIncrementalDeltaY，不用填充view。

fillGap(boolean down)方法由ListView实现。分析往下方向填充的过程。
```java
@Override
void fillGap(boolean down) {
    final int count = getChildCount();
    if (down) {
        int paddingTop = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            paddingTop = getListPaddingTop();
        }
        final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :paddingTop;
        fillDown(mFirstPosition + count, startOffset);
        correctTooHigh(getChildCount());
    }else{
        ......
    }
}
```

5.修正多余滚动距离
fillDown(int pos, int nextTop)方法做了填充工作。填充完考虑下，如果滚动距离太大，可能最后一个view填充完后下面还有一段距离,这就需要后面的correctTooHigh(int childCount)来修正。

```java
private void correctTooHigh(int childCount) {
    // First see if the last item is visible. If it is not, it is OK for the
    // top of the list to be pushed up.
    int lastPosition = mFirstPosition + childCount - 1;
    if (lastPosition == mItemCount - 1 && childCount > 0) {

        // Get the last child ...
        final View lastChild = getChildAt(childCount - 1);

        // ... and its bottom edge
        final int lastBottom = lastChild.getBottom();

        // This is bottom of our drawable area
        final int end = (mBottom - mTop) - mListPadding.bottom;

        // This is how far the bottom edge of the last view is from the bottom of the
        // drawable area
        int bottomOffset = end - lastBottom;
        View firstChild = getChildAt(0);
        final int firstTop = firstChild.getTop()    
        // Make sure we are 1) Too high, and 2) Either there are more rows above the
        // first row or the first row is scrolled off the top of the drawable area
        if (bottomOffset > 0 && (mFirstPosition > 0 || firstTop < mListPadding.top))  {
            if (mFirstPosition == 0) {
                // Don't pull the top too far down
                bottomOffset = Math.min(bottomOffset, mListPadding.top - firstTop);
            }
            // Move everything down
            offsetChildrenTopAndBottom(bottomOffset);
            if (mFirstPosition > 0) {
                // Fill the gap that was opened above mFirstPosition with more rows, if possible
                fillUp(mFirstPosition - 1, firstChild.getTop() - mDividerHeight);
                // Close up the remaining gap
                adjustViewsUpOrDown();
            }

        }
    }
}
```

上面代码的思路：
1.最后一个view的bottom小于end说明需要修正
2.如果需要修正
    a.如果第一个view的绝对position=0，那么修正距离为mListPadding.top-firstTop
    b.如果第一个view的绝对position>0，那么先将所有views向下移动bottomOffset，然后向上填充views，填充完可能上部有未填满部分，所以调用adjustViewsUpOrdown()向上平移。
    
```java
private void adjustViewsUpOrDown() {
    final int childCount = getChildCount();
    int delta;
    if (childCount > 0) {
        View child;
        if (!mStackFromBottom) {
            // Uh-oh -- we came up short. Slide all views up to make them
            // align with the top
            child = getChildAt(0);
            delta = child.getTop() - mListPadding.top;
            if (mFirstPosition != 0) {
                // It's OK to have some space above the first item if it is
                // part of the vertical spacing
                delta -= mDividerHeight;
            }
            if (delta < 0) {
                // We only are looking to see if we are too low, not too high
                delta = 0;
            }
        } else{
            ......
        }
        if (delta != 0) {
            offsetChildrenTopAndBottom(-delta);
        }
}
```

mStackFromBottom表示的是view数量不够的时候是吸顶还是吸底，默认为false。

上面的算法思路：第一个view的绝对position为0，并且delta大于0，那么所有views移动-delta。