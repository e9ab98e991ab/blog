﻿#Android事件分发机制源码分析
[TOC]
##Part1:事件来源以及传递顺序
###Activity分发事件源码
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (ev.getAction() == MotionEvent.ACTION_DOWN) {
		onUserInteraction();
	}
	if (getWindow().superDispatchTouchEvent(ev)) {
		return true;
	}
	return onTouchEvent(ev);
}
```
当触屏、按下Home键、back键、menu键等都会触发onUserInteraction()，可以重写这个方法处理一些用户交互。可以看到Activity将事件交给Window来处理。如果Window不能消费事件Activity调用onTouchEvent()自行处理事件。PhoneWindow是Window的唯一实现类，接下来分析PhoneWindow分发事件过程。

###PhoneWindow分发事件源码
```java
public boolean superDispatchTouchEvent(MotionEvent event) {
	return mDecor.superDispatchTouchEvent(event);
}
```
mDecor是DecorView类的对象，DecorView继承FrameLayout，PhoneWindow只是把事件交由顶级DecorView处理。由于DecorView继承FrameLayout，FrameLayout继承ViewGroup，所以，之后的事件分发过程与ViewGroup事件分发过程一样。接下来Part2会介绍这部分。

###小结
事件首先传递给Activity，Activity将事件传递给Window（PhoneWindow是其实现类），Window把事件交给顶级DecorView处理，如果Window没有消费这个事件则Activity调用oTouchEvent()自行处理事件。
 事件传递顺序：
```flow
op=>operation: Activity
op1=>operation: Window
op2=>operation: DecorView

op->op1->op2
```

##Part2:ViewGroup事件分发过程
只针对主要流程以及相应代码进行分析，不会贴出完整代码。ViewGroup的事件分发过程主要在dispatchTouchEvent()方法中。

首先，对MotionEvent做个简单介绍。事件序列开始于ACTION_DOWN，终于ACTION_UP。对于单指操作有ACTION_DOWN、ACTION_MOVE、ACTION_UP等事件序列组成。多指操作由ACTION_DOWN、ACTION_POINTER_DOWN、ACTION_MOVE、ACTION_POINTER_UP、ACTION_UP事件序列组成。pointer可以理解为触摸点。多指对应的pointerId不变，pointerIndex在事件序列中是变化的。
更多参考：http://www.jianshu.com/p/0c863bbde8eb

事件序列的起始动作是ACTION_DOWN，在新事件序列到达时要做一些状态清除操作。
```java
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
	// Throw away all previous state when starting a new touch gesture.
	// The framework may have dropped the up or cancel event for the previous gesture
	// due to an app switch, ANR, or some other state change.
	/**由于一些特殊原因丢失ACTION_UP或者ACTION_CANCEL，
	*导致事件序列结束时mFirstTouchTarget(TouchTarget链表)未被清空，
	* 新事件序列到达时，要先清空mFirstTouchTarget
	*/
	cancelAndClearTouchTargets(ev);
	//主要设置 mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT
	resetTouchState();
}
```
看看cancelAndClearTouchTargets()和resetTouchState()的具体实现。
```java
/**
* Cancels and clears all touch targets.
*/
private void cancelAndClearTouchTargets( MotionEvent event )
{
	if ( mFirstTouchTarget != null )
	{
		boolean syntheticEvent = false;
		if ( event == null )
		{
			final long now = SystemClock.uptimeMillis();
			event = MotionEvent.obtain( now, now,
						    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0 );
			event.setSource( InputDevice.SOURCE_TOUCHSCREEN );
			syntheticEvent = true;
		}

		for ( TouchTarget target = mFirstTouchTarget; target != null; target = target.next )
		{
			resetCancelNextUpFlag( target.child );
			dispatchTransformedTouchEvent( event, true, target.child, target.pointerIdBits );
		}
		clearTouchTargets();

		if ( syntheticEvent )
		{
			event.recycle();
		}
	}
}
```
dispatchTransformedTouchEvent()后面再分析，先来看clearTouchTargets()方法。
```java
/**
 * Clears all touch targets.
 */
private void clearTouchTargets()
{
	TouchTarget target = mFirstTouchTarget;
	if ( target != null )
	{
		do
		{
			TouchTarget next = target.next;
			target.recycle();
			target = next;
		}
		while ( target != null );
		mFirstTouchTarget = null;
	}
}
```
显而易见，clearTouchTargets()对mFirstTouchTarget指向的链表进行了清空操作。
接下来看看resetTouchState()方法的实现：
```java
private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        mNestedScrollAxes = SCROLL_AXIS_NONE;
    }
```
同样对mFirstTouchTarget指向的链表进行了清空，更重要的是设置了~FLAG_DISALLOW_INTERCEPT标志位。引出一个结论，子View在ACTION_DOWN时调用ViewGroup的requestDisallowInterceptTouchEvent()方法是无效的。

接下来ViewGroup检测是否要拦截事件：
```java 
/* Check for interception. */
final boolean intercepted;
/* ACTION_DOWN或者mFirstTouchTarget!=null时检测是否要拦截事件 */
if ( actionMasked == MotionEvent.ACTION_DOWN
     || mFirstTouchTarget != null )
{
	final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
	if ( !disallowIntercept )
	{
		intercepted = onInterceptTouchEvent( ev );
		/* restore action in case it was changed */
		ev.setAction( action );  
	} else {
		intercepted = false;
	}
} else {
	/*
	 * There are no touch targets and this action is not an initial down
	 * so this view group continues to intercept touches.
	 */
	intercepted = true;
}
```
通过上面代码，可以看出，ViewGroup在ACTON_DOWN或mFirstTouchTarget!=null条件时都会检测是否需要拦截事件；在mFirstTouchTarget!=null的情况下，可以通过设置FLAG_DISALLOW_INTERCEPT或~FLAG_DISALLOW_INTERCEPT标记位来决定ViewGroup是否允许拦截ACTION_DOWN之后的事件，在允许拦截的情况下是否拦截还取决于onInterceptTouchEvent()的返回值。对于滑动冲突，方案一：使用onInterceptTouchEvent()拦截事件；方案二：使用onInterceptTouchEvent()和requestDisallowInterceptTouchEvent()一起拦截事件。个人觉得方案一更为简单实用。

接下来ViewGroup会遍历Children，寻找能消费事件的Child。实现代码如下：
```java
/*
 * Check for cancelation.
 * PFLAG_CANCEL_NEXT_UP_EVENT标记位文档解释是Indicates whether the view is temporarily detached
 */
final boolean canceled = resetCancelNextUpFlag( this )
			 || actionMasked == MotionEvent.ACTION_CANCEL;

/* Update list of touch targets for pointer down, if needed. */
final boolean	split					= (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
TouchTarget	newTouchTarget				= null;
boolean		alreadyDispatchedToNewTouchTarget	= false;

/* View detached或者event类型为ACTION_CANCEL或者未被拦截 */
if ( !canceled && !intercepted )
{
	/*
	 * If the event is targeting accessiiblity focus we give it to the
	 * view that has accessibility focus and if it does not handle it
	 * we clear the flag and dispatch the event to all children as usual.
	 * We are looking up the accessibility focused host to avoid keeping
	 * state since these events are very rare.
	 */
	View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
					   ? findChildWithAccessibilityFocus() : null;

	if ( actionMasked == MotionEvent.ACTION_DOWN
	     || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
	     || actionMasked == MotionEvent.ACTION_HOVER_MOVE )
	{
		final int actionIndex = ev.getActionIndex(); /* always 0 for down */

		/* 这里处理多触控情况，一个View如果有多指触摸，用32位的int记录不同Pointer */
		final int idBitsToAssign = split ? 1 << ev.getPointerId( actionIndex )
					   : TouchTarget.ALL_POINTER_IDS;

		/*
		 * Clean up earlier touch targets for this pointer id in case they
		 * have become out of sync.
		 */
		removePointersFromTouchTargets( idBitsToAssign );

		final int childrenCount = mChildrenCount;
		if ( newTouchTarget == null && childrenCount != 0 )
		{
			final float	x	= ev.getX( actionIndex );
			final float	y	= ev.getY( actionIndex );
			/*
			 * Find a child that can receive the event.
			 * Scan children from front to back.
			 */
			final ArrayList<View>	preorderedList	= buildOrderedChildList();
			final boolean		customOrder	= preorderedList == null
								  && isChildrenDrawingOrderEnabled();


			final View[] children = mChildren;
			for ( int i = childrenCount - 1; i >= 0; i-- )
			{
				final int childIndex = customOrder
						       ? getChildDrawingOrder( childrenCount, i ) : i;
				final View child = (preorderedList == null)
						   ? children[childIndex] : preorderedList.get( childIndex );

				/*
				 * If there is a view that has accessibility focus we want it
				 * to get the event first and if not handled we will perform a
				 * normal dispatch. We may do a double iteration but this is
				 * safer given the timeframe.
				 */
				if ( childWithAccessibilityFocus != null )
				{
					if ( childWithAccessibilityFocus != child )
					{
						continue;
					}
					childWithAccessibilityFocus	= null;
					i				= childrenCount - 1;
				}
				/* Child不可见并且无动画直接跳过，或者Point不在child范围内 */
				if ( !canViewReceivePointerEvents( child )
				     || !isTransformedTouchPointInView( x, y, child, null ) )
				{
					ev.setTargetAccessibilityFocus( false );
					continue;
				}

				/* mFirstTouchTarget链表已经存在消费该事件的Child，用于多点触控 */
				newTouchTarget = getTouchTarget( child );
				if ( newTouchTarget != null )
				{
					/*
					 * Child is already receiving touch within its bounds.
					 * Give it the new pointer in addition to the ones it is handling.
					 */
					newTouchTarget.pointerIdBits |= idBitsToAssign;
					break;
				}

				resetCancelNextUpFlag( child );
				/* 如果Child能消费事件，Child加入到mFirstTouchTarget链表 */
				if ( dispatchTransformedTouchEvent( ev, false, child, idBitsToAssign ) )
				{
					/* Child wants to receive touch within its bounds. */
					mLastTouchDownTime = ev.getDownTime();
					if ( preorderedList != null )
					{
						/* childIndex points into presorted list, find original index */
						for ( int j = 0; j < childrenCount; j++ )
						{
							if ( children[childIndex] == mChildren[j] )
							{
								mLastTouchDownIndex = j;
								break;
							}
						}
					}else  {
						mLastTouchDownIndex = childIndex;
					}
					mLastTouchDownX				= ev.getX();
					mLastTouchDownY				= ev.getY();
					newTouchTarget				= addTouchTarget( child, idBitsToAssign );
					alreadyDispatchedToNewTouchTarget	= true;
					break;
				}

				/*
				 * The accessibility focus didn't handle the event, so clear
				 * the flag and do a normal dispatch to all children.
				 */
				ev.setTargetAccessibilityFocus( false );
			}
			if ( preorderedList != null )
				preorderedList.clear();
		}

		if ( newTouchTarget == null && mFirstTouchTarget != null )
		{
			/*
			 * Did not find a child to receive the event.
			 * Assign the pointer to the least recently added target.
			 */
			newTouchTarget = mFirstTouchTarget;
			while ( newTouchTarget.next != null )
			{
				newTouchTarget = newTouchTarget.next;
			}
			newTouchTarget.pointerIdBits |= idBitsToAssign;
		}
	}
}
```
在ACTION_DOW或者ACTION_POINTER_DOWN时ViewGroup遍历Children，寻找能够消费事件的Child。Child不在TouchTarget链表中，addTouchTarget(child, idBitsToAssign)；Child已经存在TouchTarget链表中，多指触摸同一View情况，newTouchTarget.pointerIdBits |= idBitsToAssign。ViewGroup对于多指触控不同View的解决方案是使用链表，View对于多指触控的方案是使用32位int来记录每个Pointer。

找到能够消费事件序列的Child后，ACTION_DOWN或ACTION_POINTER_DOWN之后的事件，在ViewGroup不拦截的情况下，直接交由Child处理；一旦被拦截，在dispatchTransformedTouchEventChild()方法中eventAction会置为ACTION_CANCEL，并且Child会从TouchTarget链表中清除，因此接收不到后续事件序列，都将交给ViewGroup处理。实现代码如下：
```java
/*
 * Dispatch to touch targets.
 * Child不能消费事件序列，交由ViewGroup处理
 */
if ( mFirstTouchTarget == null )
{
	/* No touch targets so treat this as an ordinary view. */
	handled = dispatchTransformedTouchEvent( ev, canceled, null,
						 TouchTarget.ALL_POINTER_IDS );
} else {
	/*
	 * Dispatch to touch targets, excluding the new touch target if we already
	 * dispatched to it.  Cancel touch targets if necessary.
	 */
	TouchTarget	predecessor	= null;
	TouchTarget	target		= mFirstTouchTarget;
	while ( target != null )
	{
		final TouchTarget next = target.next;
		/* 除去ACTION_DOWN或ACTION_POINTER_DOWN事件，因为在寻找过程中已经处理过 */
		if ( alreadyDispatchedToNewTouchTarget && target == newTouchTarget )
		{
			handled = true;
		} else {
			final boolean cancelChild = resetCancelNextUpFlag( target.child )
						    || intercepted;
			if ( dispatchTransformedTouchEvent( ev, cancelChild,
							    target.child, target.pointerIdBits ) )
			{
				handled = true;
			}
			/* PFLAG_CANCEL_NEXT_UP_EVENT重置或者ViewGroup拦截ACTION_DOWN或ACTION_POINTER_DOWN  之后的事件，清除相应TouchTarget */
			if ( cancelChild )
			{
				if ( predecessor == null )
				{
					mFirstTouchTarget = next;
				} else {
					predecessor.next = next;
				}
				target.recycle();
				target = next;
				continue;
			}
		}
		predecessor	= target;
		target		= next;
	}
}
```
ViewGroup通过dispatchTransformedTouchEvent()方法将事件分发给Child。
```java
/* Perform any necessary transformations and dispatch. */
if ( child == null )
{
	handled = super.dispatchTouchEvent( transformedEvent );
} else {
	/* 将event坐标转换成Child坐标系内坐标 */
	final float	offsetX = mScrollX - child.mLeft;
	final float	offsetY = mScrollY - child.mTop;
	transformedEvent.offsetLocation( offsetX, offsetY );
	if ( !child.hasIdentityMatrix() )
	{
		transformedEvent.transform( child.getInverseMatrix() );
	}

	handled = child.dispatchTouchEvent( transformedEvent );
}
```
未找到可消费事件的Child，ViewGroup自行处理事件序列；否则，将event坐标转换成Child坐标系内坐标交由Child处理。

ACTION_UP触发事件序列结束时清空TouchTarget，ACTION_PONITER_UP触发时，清空相应pointer的target。
```java
/* Update list of touch targets for pointer up or cancel, if needed. */
if ( canceled
     || actionMasked == MotionEvent.ACTION_UP
     || actionMasked == MotionEvent.ACTION_HOVER_MOVE )
{
	resetTouchState();
} else if ( split && actionMasked == MotionEvent.ACTION_POINTER_UP )
{
	final int	actionIndex	= ev.getActionIndex();
	final int	idBitsToRemove	= 1 << ev.getPointerId( actionIndex );
	removePointersFromTouchTargets( idBitsToRemove );
}
```
###小结
ViewGroup事件分发的流程图整理如下：
```flow
st=>start: 开始
cond=>condition: ACTION_DOWN
op=>operation: 清空mFirstTouchTarget链表，设置~FLAG_DISALLOW_INTERCEPT标记
cond1=>condition: ACTION_DOWN或者mFirstTouchTarget!=null
cond2=>condition: disallowIntercept
cond3=>condition: onInterceptTouchEvent()
op1=>operation: intercepted=true
op2=>operation: intercepted=false
cond4=>condition: ACTION_DOWN或者ACTION_POINTER_DOWN
op3=>operation: 寻找能消费事件的Child
cond5=>condition: mFirstTouchTarget==null
op4=>operation: ViewGroup处理事件
cond6=>condition: 事件未被分发到Child
op5=>operation: Child处理
cond7=>condition: intercepted
op6=>operation: 从TouchTarget链表清除Child对应Target
cond8=>condition: ACTION_UP或者ACTION_POINTER_UP
op7=>operation: 作相应清除操作
e=>end: 结束
st->cond
cond(yes)->op->cond1
cond(no)->cond1
cond1(yes)->cond2
cond1(no)->op1
cond2(yes)->op2
cond2(no)->cond3
cond3(yes)->op1->cond4
cond3(no)->op2->cond4
cond4(yes)->op3->cond5
cond4(no)->cond5
cond5(yes)->op4->cond8
cond5(no)->cond6
cond6(yes)->op5->cond7
cond7(yes)->op6->cond8
cond7(no)->cond8
cond6(no)->cond8
cond8(yes)->op7->e
cond8(no)->e
```
ViewGroup的事件分发过程就分析完了，接下来分析View的事件分发过程，相对ViewGroup来说相对简单。
##Part3:View事件分发过程
>注意：View只处理了单指触控的情况，未实现多指触控，如果有需要可以自己实现。针对View的事件分发只涉及单指情况。View的事件分发过程同样在dispatchTouchEvent()方法中，主要对这个方法进行分析即可。

View的事件分发过程同样在dispatchTouchEvent()方法中，主要对这个方法进行分析即可。
```java
final int actionMasked = event.getActionMasked();
	if (actionMasked == MotionEvent.ACTION_DOWN) {
		// Defensive cleanup for new gesture
		stopNestedScroll();
}
```
上面代码表示，ACTION_DOWN事件会使View停止滚动(如果View是能够滚动的，比如ListView)。
接下来View就要开始处理事件了，代码如下：
```java
if (onFilterTouchEventForSecurity(event)) {
	//noinspection SimplifiableIfStatement
	ListenerInfo li = mListenerInfo;
	if (li != null && li.mOnTouchListener != null
		&& (mViewFlags & ENABLED_MASK) == ENABLED
		&& li.mOnTouchListener.onTouch(this, event)) {
                result = true;
	}

	if (!result && onTouchEvent(event)) {
		result = true;
	}
}
```
View状态是ENABLE并且调用过setOnTouchListener()方法，事件是否能被OnTouchListener消费取决于onTouch()的返回值。未调用过setOnTouchListener()方法或者OnTouchListener未消费事件，由onTouchEvent()方法来处理事件，事件是否能被消费取决于onTouchEvent()的返回值。接下来看看onTouchEvent()具体实现。
```java
if ( (viewFlags & ENABLED_MASK) == DISABLED )
{
	if ( event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0 )
	{
		setPressed( false );
	}
	/*
	 * A disabled view that is clickable still consumes the touch
	 * events, it just doesn't respond to them.
	 */
	return( ( (viewFlags & CLICKABLE) == CLICKABLE ||
		  (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) );
}
```
其实，View是CLICKABLE或者LONG_CLICKABLE时返回结果都为true(具体可查看源码)，也就是View能够消费事件。上面的情况是View是DISABLED状态时，会在ACTION_UP或者(mPrivateFlags & PFLAG_PRESSED) != 0设置mPrivateFlags &= ~PFLAG_PRESSED。长按以及点击事件执行前都会先对这个标记位进行判断。View处于DISABLED状态可以消费事件，但是单击和长按事件不会执行。
```java
if (mTouchDelegate != null) {
	if (mTouchDelegate.onTouchEvent(event)) {
		return true;
	}
}
```
这段代码表示可以给View设置一个代理对象(别的View)，使用代理对象的onTouchEvent()来处理事件。比如扩大View的接触面积、几个View同步处理事件都可以用到。

对于View的事件处理，主要分析对ACTION_DOWN和ACTION_UP进行分析。先来看对ACTION_DOWN事件的处理：
```java
/*
 * For views inside a scrolling container, delay the pressed feedback for
 * a short period in case this is a scroll.
 */
if ( isInScrollingContainer )
{
	mPrivateFlags |= PFLAG_PREPRESSED;
	if ( mPendingCheckForTap == null )
	{
		mPendingCheckForTap = new CheckForTap();
	}
	mPendingCheckForTap.x	= event.getX();
	mPendingCheckForTap.y	= event.getY();
	postDelayed( mPendingCheckForTap, ViewConfiguration.getTapTimeout() );
} else {
	/* Not inside a scrolling container, so show the feedback right away */
	setPressed( true, x, y );
	checkForLongClick( 0 );
}
```
在滚动容器中的操作只是增加了个延时操作，本质还是和不在滚动容器中一样的。来看看checkForLongClick()方法的实现：
```java
private void checkForLongClick( int delayOffset )
{
	if ( (mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE )
	{
		mHasPerformedLongPress = false;

		if ( mPendingCheckForLongPress == null )
		{
			mPendingCheckForLongPress = new CheckForLongPress();
		}
		mPendingCheckForLongPress.rememberWindowAttachCount();
		postDelayed( mPendingCheckForLongPress,
			     ViewConfiguration.getLongPressTimeout() - delayOffset );
	}
}
```
postDelayed()向Handler的消息队列插入一个待处理的Runable对象，并且设置延时，这也是为什么需要长按一段时间，长按操作才会执行。长按操作的具体实现都在CheckForLongPress里了。
```java
private final class CheckForLongPress implements Runnable {
	private int mOriginalWindowAttachCount;

	@Override
	public void run()
	{
		if ( isPressed() && (mParent != null)
		     && mOriginalWindowAttachCount == mWindowAttachCount )
		{
			if ( performLongClick() )
			{
				mHasPerformedLongPress = true;
			}
		}
	}


	public void rememberWindowAttachCount()
	{
		mOriginalWindowAttachCount = mWindowAttachCount;
	}
}
```
看到重点了，performLongClick()会执行setOnLongClickListener()方法设置的OnLongClickListener的onLongClick()方法。

最后来看看View对ACTION_UP事件的处理：
```java
if ( !mHasPerformedLongPress )
{
	/* This is a tap, so remove the longpress check */
	removeLongPressCallback();

	/* Only perform take click actions if we were in the pressed state */
	if ( !focusTaken )
	{
		/*
		 * Use a Runnable and post this rather than calling
		 * performClick directly. This lets other visual state
		 * of the view update before click actions start.
		 */
		if ( mPerformClick == null )
		{
			mPerformClick = new PerformClick();
		}
		if ( !post( mPerformClick ) )
		{
			performClick();
		}
	}
}
```
如果mHasPerformedLongPress为false (可能OnLongClickListener为空或者onLongCkcik()方法返回false)，移除队列中的CheckForLongPress对象，然后如果OnClickListener不为空执行onClick()方法。

>注意：给一个Button设置OnLongClickListener和OnClickListener，onLongClick()方法返回false。这种情况长按和点击都会执行，验证方法不能使用System.out.print()来进行输出验证，因为System.out是一个有缓存的输出流，print()并不会立即输出，使用println()才会立即输出。
###小结
View的状态是CLICKABLE或者LONG_CLICKABLE都能够消费事件，如果是DISABLED状态则不会触发长按和点击事件。单击事件优先级最低，因为最后才会处理单击事件。

至此，Android事件分发机制分析完毕。