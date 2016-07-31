title: 「Android-Art」理解事件分发机制
date: 2016-07-12 01:15:32
tags: [Android,Notes,Android开发艺术探索]
---


之前一直对Android的事件分发机制似懂非懂，所以拿起「Android开发艺术探索」决定这两天好好学习这方面的知识顺便总结下，以便后面的复习。

<!--more-->

> 在看源码之前，我首先看的是这篇文章[「可能是讲解Android事件分发最好的文章」](http://www.jianshu.com/p/2be492c1df96)，讲得很好，虽然文章内没有源码，当时看完后对看源码有很大的裨益。
> .
> 这边文章主要讲到了一个很重要的点：若将布局看成一个树形结构，先是
> * `onInterceptTouchEvent()`方法经历了从顶层向下
> * `onTouchEvent(`方法经历了从底层向上(若所有的View都不拦截不消耗的情况下)
> * 若其中一个ViewGroup的`onInterceptTouchEven`t return true拦截，直接跳转到此ViewGroup的父类View的`dispatchOnTouchEvent`方法中，表示ViewGroup自己处理事件。
> 
> 这些都会在下面的源码分析中能得到验证


#### 点击事件的分发过程由三个很重要的方法完成：

* `public boolean dispatchTouchEvent(MotionEvent ev) {}`
	如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响

* `public boolean onInterceptTouchEvent(MotionEvent ev) {}`
	在上面的dispatchTouchEvent方法内调用，此方法只有ViewGroup有，View没有，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用(后面的源码中可以很清楚的验证)，返回结果表示是否拦截当前事件

* `public boolean onTouchEvent(MotionEvent event) {}`
	也是在dispatchTouchEvent方法内调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗(return false)，则在同一事件序列中，当前View无法再次接收到事件


##### 下面是三者关系的伪代码：

``` java
public boolean dispatchTouchEvent(MotionEvent ev){
	boolean consume = false;
	if(onInterceptTouchEvent(ev)){
		consume = onTouchEvent(ev);
	}else{
		consume = child.dispatchTouchEvent(ev);
	}
	return consume;
}
```


##### 当一个点击事件产生后，它的传递过程遵循如下顺序：Activity -> Window -> View

## Activity -> Window

源码: Activity#dispatchTouchEvent()
``` java
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

## Window -> View
源码: PhoneWindow#superDispatchTouchEvent()
``` java
public boolean superDispatchTouchEvent(MotionEvent ev){
	return mDecor.superDispatchTouchEvent(ev);
}
```
这里直接将事件传给了DecorView，而DecorView是我们setContentView的父布局，肯定会传递到我们的顶级View的。


## 顶级View对点击事件的分发过程

``` java

###PART ONE 源码: ViewGroup的dispatchTouchEvent()方法的部分代码片段###

... 

// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}

...


```

* mFirstTouchTarget的赋值情况：若ViewGroup的子元素成功处理时，mFirstTouchTarget会被赋值并指向子元素(后面会验证)，一旦事件由当前ViewGroup处理时，mFirstTouchTarget为null，所以后续的MOVE,UP事件到来时，由于`actionMasked == MotionEvent.ACTION_DOWN||mFirstTouchTarget != null`都不成立，所以intercepted = true;所以onInterceptTouchEvent将不会再调用到，验证了上面提到的结论
* 还有种特殊情况`FLAG_DISALLOW_INTERCEPT`这个标记位，这个标记位可以通过`requestDisallowInterceptTouchEvent()`这个方法设置，一般子元素调用这个方法后，disallowIntercept为true了，所以父View的`onInterceptTouchEvent()`也执行不到了，所以父View就不能拦截事件了,**但是ACTION_DOWN事件除外**，父View还是会执行到`onInterceptTouchEvent()`来决定是否拦截的，因为看下面的源码片段,源码中就位于PART ONE的上面

``` java
###PART TWO 源码: ViewGroup的dispatchTouchEvent()方法的部分代码片段###

...

// Handle an initial down.  处理最初的DOWN事件
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}

后面紧接PART ONE片段


```
* 在`resetTouchState()`的方法内对`FLAG_DISALLOW_INTERCEPT`的标记位进行了重置，所以`DOWN`事件到来时`FLAG_DISALLOW_INTERCEPT`的标记位进行了重置，所以验证了上面提到的点：`requestDisallowInterceptTouchEvent()`方法并不能影响到ViewGroup对`ACTION_DOWN`事件的处理


##### 到这里，intercepted值有两种结果，要么为false，要么为true。

## intercepted 为 true的情况

所以，先来看为true的这种情况的源码，**这部分源码应该是表示ViewGroup自己处理**

在源码中若intercepted为tru，则mFirstTouchTarget为null(前面提过了)直接执行到这里：

``` java
###PART THREE 源码: ViewGroup的dispatchTouchEvent()方法的部分代码片段###

// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
} else {
	...
}

```

跟进dispatchTransformedTouchEvent()方法代码片段
``` java
...

if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}

...
```

`dispatchTransformedTouchEvent()`第三个参数是child参数，这里上面传了null，所以执行`handled = super.dispatchTouchEvent(event)`,因为ViewGroup继承自View，所以ViewGroup要想自己处理事件，肯定要调用自己的onTouch、onTouchEvent、onClick方法，所以要想ViewGroup处理自己，就调用父类View的`dispatchTouchEvent()`方法，把自己当做一个View来处理这些事件，所以这里调用了`super.dispatchTouchEvent(event)`，至于View的`dispatchTouchEvent()`事件怎么处理的，后面会分析到。


再看刚才的另一种结果，intercepted为false，这种结果的代码片段在PART THREE的上面，因为刚才intercepted为true，所以跳过了下面的代码，来看下面的代码片段,下面的代码的执行条件是**<font color=red>DOWN事件</font>并且 intercepted为false，意味着ViewGroup不拦截事件，应向子元素分发事件**

``` java
###PART FOUR 源码: ViewGroup的dispatchTouchEvent()方法的部分代码片段###

final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = customOrder
            ? getChildDrawingOrder(childrenCount, i) : i;
    final View child = (preorderedList == null)
            ? children[childIndex] : preorderedList.get(childIndex);

    // If there is a view that has accessibility focus we want it
    // to get the event first and if not handled we will perform a
    // normal dispatch. We may do a double iteration but this is
    // safer given the timeframe.
    if (childWithAccessibilityFocus != null) {
        if (childWithAccessibilityFocus != child) {
            continue;
        }
        childWithAccessibilityFocus = null;
        i = childrenCount - 1;
    }

    if (!canViewReceivePointerEvents(child)
            || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
        continue;
    }

    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {
        // Child is already receiving touch within its bounds.
        // Give it the new pointer in addition to the ones it is handling.
        newTouchTarget.pointerIdBits |= idBitsToAssign;
        break;
    }
	
    resetCancelNextUpFlag(child);
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        // Child wants to receive touch within its bounds.
        mLastTouchDownTime = ev.getDownTime();
        if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
                if (children[childIndex] == mChildren[j]) {
                    mLastTouchDownIndex = j;
                    break;
                }
            }
        } else {
            mLastTouchDownIndex = childIndex;
        }
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }

    // The accessibility focus didn't handle the event, so clear
    // the flag and do a normal dispatch to all children.
    ev.setTargetAccessibilityFocus(false);
}

```

代码比较长，不过不是很难理解，大致逻辑是这样的：
遍历ViewGroup的所有元素，如果触摸的事件落在遍历到的view(你都没touch到的view，还传递分发给他事件干嘛，对吧？)，并且当前遍历到的元素正在播放动画(有点奇怪，不过不影响)，满足这两个条件，这个元素才能接收到父元素传递给他的事件。若两个条件有一个不满足就continue，继续遍历。假如遍历到了能够接收到事件的子元素时，便会执行到上面代码PART FOUR的这里：
``` java
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
	...
}
```
这个方法刚才也看到过了：
``` java
...

if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}

...
```

这里注意了，刚才ViewGroup自己处理的时候，第三个参数child传的是null，不过，这里是ViewGroup不处理，传递给子View，所以第三个参数child传递的是刚才遍历到的将要接收到传递的事件的子元素，所以执行了这句`handled = child.dispatchTouchEvent(event);`，很明显，执行了子元素的`dispatchTouchEvent()`，子元素可能是View也可能是ViewGroup，如果是ViewGroup的话，就跟上面分析父元素的处理过程一样，可能这样层层传递下去....，如果子View是View，那就更简单了，因为View的`dispatchTouchEvent()`方法内`没有onInterceptTouchEvent`方法，所以`dispatchTouchEvent()`方法处理要简单的多。

最后，if判断的`dispatchTransformedTouchEvent()`，会有一个返回值，看源码发现，那个`dispatchTransformedTouchEvent()`的返回值就是那个handled，即`child.dispatchTouchEvent(event)`的返回值，如果返回了true，表示子元素来处理这个事件了，就会执行到了if判断里面的这一句:`newTouchTarget = addTouchTarget(child, idBitsToAssign);`，在`addTouchTarget()`中完成了对`mFirstTouchTarget`的赋值(验证前面反复提到的结论)，然后最后一句`break;`，打断for循环，因为已经有子元素处理了，所以不需要遍历了。如果`child.dispatchTouchEvent(event)`的返回值返回了false，表示这个子元素也不处理，所以mFirstTouchTarget无法赋值，即为null(验证前面反复提到的结论)，接着继续for循环去遍历下一个子元素....

刚才提到，如果遍历到的子元素是一个View，因为View的`dispatchTouchEvent()`内没有`onInterceptTouchEvent`方法，所以`dispatchTouchEvent()`方法处理要简单的多，下面立马分析View对点击事件的处理过程。

## View对点击事件的处理过程

* 下面是**View的dispatchTouchEvent()方法**部分代码

``` java
    public boolean dispatchTouchEvent(MotionEvent event) {

		...		

        boolean result = false;

		...

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

		...

        return result;
    }
```

如果有设置过onTouchListener，那么`mOnTouchListener.onTouch`将会执行，**如果onTouch返回true**，则resulr为true，所以下面的那个if判断内的**`onTouchEvent(event)`不会执行到**。相反，**若onTouch返回false，`onTouchEvent(event)`会执行得到**，所以onTouch的优先级高于onTouchEvent，这样做的好处是**方便在外界处理点击事件**。

* 接着分析onTouchEvent()方法内的代码

``` java

if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    // A disabled view that is clickable still consumes the touch
    // events, it just doesn't respond to them.
    return (((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
}

```

先看当View处于不可用的状态下点击事件的处理过程，很显然，不可用状态下的View照样会消耗点击事件，尽管它看起来不可用。

继续看:

``` java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
        (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
        (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
    switch (action) {
        case MotionEvent.ACTION_UP:
            boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
            if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
				
				...
				
                if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                    // This is a tap, so remove the longpress check
                    removeLongPressCallback();

                    // Only perform take click actions if we were in the pressed state
                    if (!focusTaken) {
                        // Use a Runnable and post this rather than calling
                        // performClick directly. This lets other visual state
                        // of the view update before click actions start.
                        if (mPerformClick == null) {
                            mPerformClick = new PerformClick();
                        }
                        if (!post(mPerformClick)) {
                            performClick();
                        }
                    }
                }

				...

            }

			...

            break;
    }

    return true;
}
```

从上面的代码来看，只要View的`CLICKABLE`和`LONG CLICKABLE`有一个为true，那么他就会消耗这个事件，因为return了true，不管他是不是DISABLE状态。然后当`ACTION_UP`触发时，会执行`performClick()`，在`performClick()`的内部，如果有设置onClickListener，那么`performClick()`方法内部会调用他的onClick方法，如下所示：
``` java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```
View的`LONG_CLICKABLE`默认是false，而`CLICKABLE`属性则要看具体的View了，例如，Button的`CLICKABLE`默认为true，TextView的`CLICKABLE`默认为false，只要执行了`setOnClickListener()`或者`setOnLongClickListener()`都会将`CLICKABLE`或者`LONG_CLICKABLE`置为true，看源码就知道了：
``` java
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
```
``` java
public void setOnLongClickListener(@Nullable OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
```

---

##### 下结论都是书上总结的：

1. 同一事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。
2. 正常情况下，一个事件序列只能被一个View拦截且消耗。这一条原因可以参考(3)，因为一旦一个元素拦截了某此事件，那么同一事件序列内的所有事件都会直接交给它处理，因此同一事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过`onTouchEvent`强行传递给其他View处理
3. 某个View一旦决定拦截，那么这一个事件序列都只能由它处理（如果事件能够传递给他的话），并且它的`onInterceptTouchEvent`不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一事件序列内的其他方法都直接交给他来处理，因此不会再调用这个View的`onInterceptTouchEvent`去询问它是否要拦截了
4. 某个View一旦开始处理事件，**如果它不消耗`ACTION_DOWN`事件(`onTouchEvent`返回了false)，那么同一事件序列中的其他事件都不会再交给他来处理**，并且事件将重新交给它的父元素处理，即父元素的`onTouchEvent`会被调用。意思是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给他来处理了。
5. 如果View不消耗除`ACTION_DOWN`以外的其他事件，那么这个点击事件会消失，此时父元素的`onTouchEvent`并不会被调用，并且当前View可以持续收到后续的点击事件，最终这些消失的点击事件会传递给Activity处理
6. ViewGroup默认不拦截任何点击事件。Android源码中ViewGroup的`onInterceptTouchEven`t方法默认返回false
7. View没有`onInterceptTouchEvent`方法，一旦有点击事件传递给他，那么它的`onTouchEvent`方法就会被调用
8. View的`onTouchEvent`默认都会消耗事件(返回true)，除非它是不可点击的(clickable和longClickable同时为false)。View的longClickable属性默认都为false，clickable属性要分情况，不必多说
9. View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longClickable有一个为true，那么它的`onTouchEvent`就返回true
10. onClick会发生的前提是当前View可点击的，并且他收到了down和up事件
11. 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再有父元素分发给子View，通过`requestDisallowInterceptTouchEvent`方法可以在子元素中干预父元素的事件分发过程，当然`ACTION_DOWN`不能干预







