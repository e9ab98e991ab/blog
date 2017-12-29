# 通过KeyEvent选中控件

## 知识准备

要准确的理解这篇文章，首先需要理解<a href="https://github.com/wslaimin/blog/blob/master/Android%20KeyEvent%E5%88%86%E5%8F%91%E6%9C%BA%E5%88%B6.md">Android KeyEvent分发机制</a>

## 需求说明

 1. 通过上、下、左、右四个方向KeyEvent选中区块。
 2. 自定义KeyCode为300、301两个KeyEvent。300时在区块内顺时针寻找下一个可获取焦点控件，301时在区块内逆时针寻找上一个可获取焦点控件。
 3. 支持ListView,GridView等集合类控件。KeyCode为300时在item内顺时针查找下一个可获取焦点控件，若无下一个控件，则向下一个item内查找；keyCode为301时在item内逆时针查找上一个可获取焦点控件，若无上一个，则向上一个item内查找。

## 实现过程

### 定义区块FocusViewGroup

FocusViewGroup继承FrameLayout,重写dispatchKeyEvent()方法，如果当前区块内有控件获取焦点，处理KeyCode为300、301的事件。

区块内顺时针查找，是控件在布局内的排列顺序；反之，逆时针查找为排列的你顺序。

FocusViewGroup的实现如下：

```java
public class FocusViewGroup extends FrameLayout {
    private final static String TAG = "FocusViewGroup";

    public FocusViewGroup(@NonNull Context context) {
        super(context);
    }

    public FocusViewGroup(@NonNull Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchKeyEvent(KeyEvent event) {
        boolean handled=super.dispatchKeyEvent(event);
        if(!handled){
            View next;
            View directFocused=findFocus();
            if(directFocused !=null&&event.getAction()==KeyEvent.ACTION_DOWN){
                switch (event.getKeyCode()) {
                    case 300:
                        next=findNextFocused(directFocused);
                        if(next!=null){
                            handled=next.requestFocus();
                        }
                        break;
                    case 301:
                        next=findBeforeFocused(directFocused);
                        if(next!=null){
                            handled=next.requestFocus();
                        }
                        break;
                    default:
                        handled=super.dispatchKeyEvent(event);
                        break;
                }
            }else {
                handled=super.dispatchKeyEvent(event);
            }
        }

        return handled;
    }

    /**
     *
     * @param focused 当前焦点View
     * @return 下一个获取焦点View
     */
    private View findNextFocused(View focused) {
        View next=null;
        ArrayList<View> views=new ArrayList<>();
        addFocusables(views,FOCUS_RIGHT);
        for(int i=0;i<views.size();i++){
            if(views.get(i)==focused){
                next=views.get((i+1)%views.size());
                break;
            }
        }
        return next;
    }

    /**
     ** @param focused 当前焦点View
     * @return 前一个获取焦点View
     */
    private View findBeforeFocused(View focused){
        View next=null;
        ArrayList<View> views=new ArrayList<>();
        addFocusables(views,FOCUS_LEFT);
        for(int i=0;i<views.size();i++){
            if(views.get(i)==focused){
                next=views.get((i-1+views.size())%views.size());
                break;
            }
        }
        return next;
    }
}
```
 
### 定义BigFocusViewGroup
BigFocusViewGroup继承FrameLayout,BigFocusViewGroup的作用是识别上、下、左、右事件，并且根据方向找到下一个区块。

查找区块的方案参考FocusFinder，但是与FocusFinder并不完全一样，findNextFocusInAbsoluteDirection()方法的第一个参数需要的是FocusViewGroup的集合。findNextFocusInAbsoluteDirection()为private方法，无法重写，所以，复制FocusFinder的部分代码，findNextFocusInAbsoluteDirection()方法的传入参数。

获取FocusViewGroup集合的方法如下：

```java
private ArrayList<View> findCandidates(ViewGroup root){
    ArrayList<View> candidates = new ArrayList<>();
    Queue<View> queue = new LinkedList<>();
    View child;
    for (int i = 0; i < root.getChildCount(); i++) {
        child = root.getChildAt(i);
        if (child instanceof ViewGroup) {
            queue.add(root.getChildAt(i));
            if (child instanceof FocusViewGroup) {
                candidates.add(child);
            }
        }
        while (!queue.isEmpty()) {
            ViewGroup viewGroup = (ViewGroup) queue.poll();
            for (int j = 0; j < viewGroup.getChildCount(); j++) {
                child = viewGroup.getChildAt(j);
                if (child instanceof ViewGroup) {
                    queue.add(child);
                }
                if (child instanceof FocusViewGroup) {
                    candidates.add(child);
                }
            }
        }
    }
    return candidates;
}
```
 
 >ps:实现类似于图的广度优先遍历

BigFocusViewGroup定义一个成员变量mFocused保存当前获取焦点区块。mFocused的赋值很有趣。

当View调用requestFocus()方法时，会调用父容器的requestChildFocus()方法，层层向上，通知上层容器获取焦点child是哪一个和实际获取焦点的控件是哪一个。

根据上面焦点控件向上传递的机制，可以重写BigFocusViewGroup的requestChildFocus()方法：

```java
@Override
public void requestChildFocus(View child, View focused) {
    super.requestChildFocus(child, focused);

    ViewParent parent = focused.getParent();
    while (parent != null) {
        if (parent instanceof FocusViewGroup) {
            mFocused = (ViewGroup) parent;
            break;
        }
        parent = parent.getParent();
    }
}
```

BigFocusViewGroup重写dispatchKeyEvent()方法处理上、下、左、右事件：

```java
public boolean dispatchKeyEvent(KeyEvent event) {
    boolean handled = false;
    View next;
    if (event.getAction() == KeyEvent.ACTION_DOWN) {
        switch (event.getKeyCode()) {
            case KeyEvent.KEYCODE_DPAD_UP:
            next = FocusViewGroupFinder.getInstance().findNextFocus(this, mFocused, FOCUS_UP);
            if (next != null) {
                next.requestFocus();
                }
            handled = true;
            break;
            
            ......
        }
    } else {
        handled = super.dispatchKeyEvent(event);
    }
    return handled;
}
```

### 支持ListView

继承ListView重写onKeyDown()方法，当ListView里有已获取焦点控件时处理KeyCode为300和301的事件:

```java
public boolean onKeyDown(int keyCode, KeyEvent event) {
    boolean result = false;
    if (getFocusedChild() != null && event.getAction() == KeyEvent.ACTION_DOWN) {
        result = true;
        switch (event.getKeyCode()) {
            case 300:
                requestNextFocus();
                break;
            case 301:
                requestBeforeFocused();
                break;
            default:
                break;
        }
    }
    return result;
}
```

以KeyCode=300为例，查找下一个可获取焦点控件。在item里按控件顺序查找下一个可获取焦点控件，找到则控件获取焦点，没找到则下一个item第一个可获取焦点控件获取焦点。代码如下：

```java
private void requestNextFocus() {
    View next;
    View focusedChild = getFocusedChild();
    View focused = focusedChild.findFocus();
    ArrayList<View> focusableList = findFocusableInChild(focusedChild,FOCUS_RIGHT);
    //现在child下查找下一个获取焦点控件
    for (int i = 0; i < focusableList.size(); i++) {
        if (focusableList.get(i) == focused && (i + 1) < focusableList.size()) {
            next = focusableList.get(i + 1);
            next.requestFocus();
            //View不完全显示滚动知道完全显示
            scrollToNext(getPositionForView(focusedChild), next,FOCUS_DOWN);
            return;
        }
    }

    //在child中未找到下一个焦点控件，焦点移动到下一个child
    View nextFocusedChild;
    int position = getPositionForView(focusedChild);
    if (position + 1 >= getCount()) {
        return;
    }
    if ((position + 1 - getFirstVisiblePosition()) < getChildCount()) {
        nextFocusedChild = getChildAt(position + 1 - getFirstVisiblePosition());
        //要处理控件是否在屏幕上情况，如果不在，需要滚动ListView
        nextFocusedChild.requestFocus();
        scrollToNext(position + 1, nextFocusedChild.findFocus(),FOCUS_DOWN);
    } else {
        scrollToNext(position+1,null,FOCUS_DOWN);
    }
}
```

虽然下一个控件获取了焦点，但是还要考虑控件是否完全显示在屏幕上，如果没有完全显示，要滚动ListView使控件完全显示出来。代码如下：

```java
private void scrollToNext(final int position, View nextFocused,int direction) {
    //直接向上或向下滚动ListView一个child
    if(nextFocused==null){
        switch (direction){
            case FOCUS_UP:
                if(position>=0){
                    smoothScrollToPosition(position);
                    postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            View focusedChild=getChildAt(position-getFirstVisiblePosition());
                            if(focusedChild!=null){
                                ArrayList<View> focusableList=findFocusableInChild(focusedChild,FOCUS_LEFT);
                                    focusableList.get(focusableList.size()-1).requestFocus();
                            }
                        }
                    },200);
                }
                break;
            case FOCUS_DOWN:
                if(position<getCount()){
                    smoothScrollToPosition(position);
                        /*smoothScrollToPosition在UI线程执行，需要时间SCROLL_DURATION / viewTravelCount详情查看AbsListView*/
                    postDelayed(new Runnable() {
                        @Override
                        public void run() {
                            View focusedChild=getChildAt(position-getFirstVisiblePosition());
                            if(focusedChild!=null){
                                focusedChild.requestFocus();
                            }
                        }
                    },200);
                }
                break;
            default:
                break;
        }
    return;
    }

    Rect focusedRect = new Rect();
    nextFocused.getFocusedRect(focusedRect);
    //focused平移到ListView坐标系下
    offsetDescendantRectToMyCoords(nextFocused, focusedRect);
    //top是否超过ListView顶部
    switch (direction){
        case FOCUS_UP:
            if(focusedRect.top < 0){
                smoothScrollBy(focusedRect.top,0);
            }
            break;
        case FOCUS_DOWN:
            //判断focused的bottom是否超过ListView的底部
            if(focusedRect.bottom > getMeasuredHeight() - getListPaddingBottom()){
                    //不能用scrollBy,因为ListView里的View没有更新坐标(left,top,right,bottom),下一次依旧会滚动
                    smoothScrollBy(focusedRect.bottom-getMeasuredHeight()-getListPaddingBottom(),0);

            }
            break;
        default:
            break;
    }
}
```

## 效果图
![](https://www.github.com/wslaimin/blog/raw/master/pics/FocusViewGroup.gif)

## <a href="https://github.com/wslaimin/FocusViewGroup.git">Demo链接</a>