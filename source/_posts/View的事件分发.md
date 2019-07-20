---
title: View的事件分发
date: 2016-9-22
tags:
  - View
categories: Android
---


## 事件分发的源码解析

### 1.Activity对点击事件的分发过程

点击事件用`MotionEvent`来表示，当一个点击操作发生的时候，事件最先传递给当前`Activity`，由`Activity`的`dispatchTouchEvent`来进行事件的派发，具体的工作是由Activity内部的window来完成的，window会将事件传递给`decor view`,`decor view`一般都是当前界面的底层容器，通过Activity.getWindow.getDecorView()获得，我们可用先从`Activity`的`dispatchTouchEvent`的源码看起：

```java
/**
 *处理触摸屏事件。 
 *可以覆盖此方法在分发所有触摸屏事件之前拦截所有事件
 *当然最后还要调用此方法一下,否则事件无法正常分发传递给view。
 *
 * @param ev触摸屏事件。
 *
 * @return boolean如果消耗此事件，则返回true。
*/
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
<!-- more -->

先分析下上面的代码,首先事件交给Activity的dispatchTouchEvent进行分发,然后在通过`getWindow().superDispatchTouchEvent(ev)`把事件分发给activity所依附的window，如果返回true那就结束了，如果返回false的话就没人处理，那么Activity的`onTouchEvent`就会被调用

接下来我们看下`getWindow().superDispatchTouchEvent(ev)`里面window是如何将事件传递给ViewGroup的，通过源码我们知道，window时一个抽象类，而window的super.dispatchTouchEvent(ev)方法也是抽象的，因此我们必须找到window的实现类

```java
public abstract boolean superDispatchTouchEvent(MotionEvent event);
```

那么window的实现类是什么呢？就是phonewindow，这点源码中有一段注释就说明了

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {
    Window类的代码
}
```

上面的意思大概就是winodw类控制顶级的View的外观和行为机制，他的唯一实现位于`android.policy.PhoneWinodw`中

由于Window的唯一实现是PhoneWindow，那我们看一下PhoneWindow是如何处理点击事件的

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

我们进入`mDecor.superDispatchTouchEvent(event);`方法中

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

此处调用了`super.dispatchTouchEvent(event);`那就要看看这个`Decorview`的父类是谁了,看下面源码:

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
    ...
    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;
    
     @Override
    public final View getDecorView() {
        if (mDecor == null || mForceDecorInstall) {
            installDecor();
        }
        return mDecor;
    }
}
```

很明显`DecorView`继承自`FrameLayou`，所以最终事件会传递给`ViewGroup`的`dispatchTouchEvent`。不过上面只是分析了事件从`Activity`传递到`ViewGroup`的过程,但这不是我们的重点，重点是事件到了`ViewGroup`以后应该如何传递，从这里开始，事件已经传递到顶级`ViewGroup`了，接下来就开始分析具体的事件传递了接！

### 顶级ViewGroup对事件的分发过程

点击事件达到顶级`ViewGroup`以后，会调用`ViewGroup`的`dispatchTouchEvent`方法

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    // If the event targets the accessibility focused view and this is it, start
    // normal event dispatch. Maybe a descendant is what will handle the click.
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

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

        // If intercepted, start normal event dispatch. Also if there is already
        // a view that is handling the gesture, do normal event dispatch.
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {

            // If the event is targeting accessibility focus we give it to the
            // view that has accessibility focus and if it does not handle it
            // we clear the flag and dispatch the event to all children as usual.
            // We are looking up the accessibility focused host to avoid keeping
            // state since these events are very rare.
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

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
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

由于这个方法比较长，这里分段说明。先看下面一段

```java
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
```

很显然，它描述的是当ViewGroup是否拦截点击事情这个逻辑。

从上面代码我们可以看出，ViewGroup在如下两种情况下会判断是否要拦截当前事件：`事件类型为ACTION_DOWN`或者`mFirstTouchTarget!=null`,`ACTION_DOWN`事件好理解，那么`mFirstTouchTarget!=null`是什么意思呢？这个从后面的代码逻辑可以看出来

```java
if (!canceled && !intercepted) {
    详细逻辑可以看上面代码
}
```

当ViewGroup不拦截事件时`intercepted = onInterceptTouchEvent(ev);`中的`intercepted`为false，那下面那段代码就会执行`mFirstTouchTarget`会被赋值并指向子元素，也就是说，当ViewGroup不拦截事件并将事件交由子元素处理时`mFirstTouchTarget != null`。

反过来，一旦事件由当前ViewGroup拦截时`intercepted`就为true，那`if (!canceled && !intercepted)`就不成立,那么mFirstTouchTarget就不会被赋值为子元素,那 `mFirstTouchTarget != null`就不成立。那么当`ACTION_MOVE和ACTION_UP`事件到来时，由于`actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null`这个条件为false，将导致`ViewGroup` 的`onInterceptTouchEvent`不会再被调用，那也就是说同一序列中的其他事件都会默认交给此ViewGroup去处理了。

当然，这里有一种特殊情况，就是上面代码中`final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0`这个判断,如果`disallowIntercept`为false那就走父类拦截方法,如果为true那就不走父类拦截方法,这个`mGroupFlags & FLAG_DISALLOW_INTERCEPT`是否等于0,有以下两个地方决定

第一个地方是每次DOWN事件来的时候都会执行的下面的一段代码

```java
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
```

还有就是子类调用`requestDisallowInterceptTouchEvent(true)`告诉父类不拦截事件,`requestDisallowInterceptTouchEvent(false)`允许父类拦截事件

```java
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        // We're already in this state, assume our ancestors are too
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    // Pass it up to our parent
    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

通过上面两段代码我们知道

- `mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT`执行后再执行`mGroupFlags & FLAG_DISALLOW_INTERCEPT`一定为0
- `mGroupFlags |= FLAG_DISALLOW_INTERCEPT`执行后再执行`mGroupFlags & FLAG_DISALLOW_INTERCEPT`一定不为0

为什么呢?

我们都知道

```java
protected static final int FLAG_DISALLOW_INTERCEPT = 0x80000;
```

```java
FLAG_DISALLOW_INTERCEPT的值为0x80000，化成二进制，就是1000000000000000000
所以经过ACTION_DOWN执行了mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT之后再执行
(mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0 这个结果是false的

因为~FLAG_DISALLOW_INTERCEPT是01111111111111111111
mGroupFlags&01111111111111111111最高位是0,
然后mGroupFlags & FLAG_DISALLOW_INTERCEPT也就是
mGroupFlags & 1000000000000000000 最后一定是0  
```

此处我只分析这一种另一种自己分析

有了上面的分析我们可以知道:ViewGroup在事件分发时,如果是ACTION_DOWN就会执行`mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT`使其 `final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0` 中的disallowIntercept为false,也就是当DOWN来的时候ViewGroup总是会调用自己的`onInterceptTouchEvent`方法询问自己是否拦截事件,而在子类设置了`requestDisallowInterceptTouchEvent(true)`后`final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0`中的disallowIntercept为true结果就是除了DOWN的其他事件都不会被父类拦截了

**结论:**
当ViewGroup决定拦截事件后,那么后续的点击事件将默认交给它处理并不会再调用它的`onInterceptTouchEvent`方法,`FLAG_DISALLOW_INTERCEPT`标志位的作用是让ViewGroup不再拦截事件,当然前提是ViewGroup不拦截ACTION_DOWN事件,通过上面的分析我们还可以得到:

- 第一点:`onInterceptTouchEvent`不是每次事件都会调用,如果我们想提前处理所有的点击事件,要选择`dispatchTouchEvent`方法,只有这个方法能保证每次都会调用,当然前提是事件能够传递到当前的ViewGroup;
- 另一点:`FLAG_DISALLOW_INTERCEPT`标志位的作用可以解决一些滑动冲突,后面会分析.

接下来我们再看下ViewGroup不拦截事件的时候，事件会向下分发由他的子View进行处理：

```java
final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = getAndVerifyPreorderedIndex(
            childrenCount, i, customOrder);
    final View child = getAndVerifyPreorderedView(
            preorderedList, children, childIndex);

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
```

上面的代码逻辑还是比较清晰的，首先遍历的是ViewGroup的所有子元素，然后判断子元素是否能接受这个点击事件主要是两点来衡量，子元素是否在播动画和点击是按的坐标是否落在子元素的区域内，如果某子元素满足这两个条件，那么事件就会传递给他处理，可以看到，`dispatchTransformedTouchEvent`实际上调用的就是子元素的`dispatchTouchEvent`方法，在他的内部有下面的一段代码，

```java
if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}
```

而在上面的代码`dispatchTransformedTouchEvent`中child传递的不是null，因此他会直接调用子元素的`dispatchTouchEvent`方法，这样事件就交由子元素处理了，这就从而完成这一轮事件分发。

如果子元素的`dispatchTouchEvent`返回true，这时我们暂时不考虑事件在子元素的怎么分发的，那么`mFirstTouchTarget`就会被赋值同时跳出for循环：

```java
newTouchTarget = addTouchTarget(child, idBitsToAssign);
alreadyDispatchedToNewTouchTarget = true;
break;
```

这几行代码就完成了`mFirstTouchTarget`的赋值并且并终止对子元素的遍历，如果子元素的`dispatchTouchEvent`返回false，ViewGroup就会把事件分给下一个子元素(如果还有下一个子元素的话)

其实mFirstTouchTarget真正的赋值过程是在addTouchTarget内部完成的，从下面的addTouchTarget的内部结构就可以看出，mFirstTouchTarget其实是一种单链表的结构，mFirstTouchTarget是否被赋值，将直接影响到ViewGroup对事件的拦截机制，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截下来同一序列中所有的点击事件

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

如果遍历所有的子元素后事件都没有被合适的处理，这包含两种情况，第一是Viewgroup没有子元素，第二是子元素处理了点击事件，但是在`dispatchTouchEvent`中返回false，这一般是因为子元素在`onTouchEvent`返回了false，这两种情况下，ViewGroup会自己处理点击事件，看代码：

```java
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
} 
```

注意上面这段话，这里的第三个参数child为null，从上面的分析我们可用知道，他会调用`supe.dispatchTouchEvent(event)`，很显然，这里就转到了View的`dispatchTouchEvent`方法，就是点击事件开始给View处理了,下面开始分析View对点击事件的处理

### View对点击事件的处理过程

View对点击事件的处理过程稍微简单一些，这里注意，这里的View不包含ViewGroup，先看他的`dispatchTouchEvent`方法

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    // If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
        // We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Defensive cleanup for new gesture
        stopNestedScroll();
    }

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

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

View点击事件的处理就比较简单了，因为他只是一个View，他没有子元素所以无法向下传递，所以只能自己处理点击事件，从上面的源码可以看出View对点击事件的处理过程，首选会判断你有没有设置`onTouchListener`，如果`onTouchListener`中的`onTouch`为true，那么`onTouchEvent`就不会被调用，可见`onTouchListener`的优先级高`于onTouchEvent`，这样做到好处就是方便在外界通过设置`onTouchListener`来处理点击事件;

接着我们再来分析下`onTouchEvent`的实现

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

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

    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                   }

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

                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }

                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(0);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_MOVE:
                drawableHotspotChanged(x, y);

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, mTouchSlop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();

                        setPressed(false);
                    }
                }
                break;
        }

        return true;
    }

    return false;
}
```

先看当View处于不可用的状态下点击事件的处理过程，如下，很显然，不可用状态下的View照样会消耗点击事件，尽管他看起来不可用：

```java
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

接着，如果View设置有代理，那么还会执行TouchDelegate的onTouchEvent方法，这个onTouchEvent的工作机制看起来和onTouchListener类似，这里就不深究了

```java
if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
```

下面再看一下onTouchEvent中点击事件的具体处理，如下所示：

```java
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                   }

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
             black;
         }
    ....
return true;
}
```

从上面的代码来看，只要View的CLICKABLE和LONG_CLICKABLE有一个为true，那么他就会消耗这个事件，即onTouchEvent返回true，不管他是不是DISABLE状态，然后就是当ACTION_UP事件发生时，会触发`performClick`方法，如果View设置了`onClickListener`，那么`performClick`方法内部就会调用他的onClick方法

```java
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

View的LONG_CLICKABLE属性默认为false，而CLICKABLE属性是否为false和具体的View有关，确切的说是可点击的View其CLICKABLE为true，不可点击的为false，比如button是可点击的，textview是不可点击的，通过`setClickable`和`setLongClickable`可以分别改变View的CLICKABLE和LONG_CLICKABLE属性，另外,setOnClickListener会自动的将View的CLICKABLE设为true,setOnLongClickListener也会自动将LONG_CLICKABLE属性设为true,这点我们看源码：

```java
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}

public void setOnLongClickListener(@Nullable OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
```
到这里，点击事件的分发机制源码就分析完成了

## 总结

所谓点击事件的事件分发，其实就是对`MotionEvent`事件的分发过程，即当一个`MotionEvent`产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递的过程就是分发过程。点击事件的分发过程由三个很重要的方法来共同完成:`dispatchTouchEvent`，`onInterceptTouchEvent`和`onTouchEvent`

- puhlic boolean dispatchTouchEvent(MotionEvent ev)

用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

- public boolean onInterceptTouchEven(MotionEvent event)

在`dispatchTouchEvent`方法内部调用，用来判断是否拦截某个事件，如果当前ViewGroup拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

- public boolean onTouchEvent(MotionEvent event)

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

对于一个根`ViewGroup`来说，点击事件产生以后，首先传递给`ViewGroup`，这时它的dispatchTouchEvent就会被调用，如果这个ViewGroup的`onIntereptTouchEvent`方法返回true就表示它要控截当前事件，接着事件就会交给这个`ViewGroup`处理，则他的`onTouchEvent`方法就会被调用；如果这个ViewGroup的`onIntereptTouchEvent`方法返回`false`就表示不需要拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的`dispatchTouchEvent`方法就会被调用，如此反复直到事件被最终处理。

当一个View需要处理事件时，如果它设置了`OnTouchListener`，那么`OnTouchListener`中的`onTouch`方法会被回调。这时事件如何处理还要看onTouch的返回值，如果返回`false`,那当前的View的方法`onTouchEvent`会被调用；如果返回`true`，那么`onTouchEvent`方法将不会被调用。由此可见，给View设置的`OnTouchListener`优先级比`onTouchEvent`要高，在`onTouchEvent`方法中，如果当前设置的有`OnClickListener`，那么它的`onClick`方法会被调用。可以看出，平时我们常用的`OnClickListener`，其优先级最低，即处于事件尾端。

当一个点击事件产生后，它的传递过程遵循如下顺序：`Activity-->Window-->View`，即事件总是先传递给`Activity`,`Activity`再传递给`Window`，最后`Window`再传递给顶级`View`。顶级`View`接收到事件后，就会按照事件分发机制去分发事件。考虑一种情况，如果一个View的`onTouchEvent`返回`false`，那么它的父容器的`onTouchEvent`将会被调用，依此类推,如果所有的元素都不处理这个事件，那么这个事件将会最终传递给`Activity`处理，即`Activity`的`onTouchEvent`方法会被调用。

关于事件传递的机制，这里先给出一些结论，具体分析上文已经结合源码进行了讲解

- （1）同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏慕的那一刻结束，在这个过程中会产生产生的一系列事件，这个事件序列以`down`事件开始，中间含有数量不定的`move`事件，最后以`up`事件结束

- （2）正常情况下，一个事件序列只能被一个Visw拦截且消耗。这一条的原因可以参考（3），因为一旦一个元素拦截了某此事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个Vew将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。

- （3)某个View一旦决定拦截，那么这一个事件序列都只能由它来处理（如果事件序列能够传递给它的话)，并且它的onInterceprTouchEvent不会再被调用。这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。

- (4）某个View一旦开始处理事件，如果它不消耗ACTON_DOWN事件(onTouchEvent返回了false)，那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。意思就是事件一旦交给一个View处理，那么它就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理了，这就好比上级交给程序员一件事，如果这件事没有处理好，短期内上级就不敢再把事情交给这个程序员做了，二者是类似的道理。

- （5）如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。

- （6)ViewGroup默认不拦截任何事件。Android源码中ViewGroup的onInterceptTouchEvent方法默认返回false

- （7）View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。

- （8）view的onTouchEvent默认都会消耗事件（返回true)，除非它是不可点击的(clickable和longClickable同时为false)，View的longClickable属性默认都为false,clickable属性要分情况，比如Button的clickable属性默认为true，而TextView 的clickable属性默认为false

- （9)view 的enable.属性不影响onTouchEvent的默认返回值。哪怕一个View是disable状态的，只要它的clickable或者longclickable有一个为true，那么它的onTouchEvent就返会true。

- （10）onclick会发生的前提实际当前的View是可点击的，并且他收到了down和up的事件

- (11)事件传递过程是由外到内的，理解就是事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterptTouchEvent方法可以再子元素中干预元素的事件分发过程，但是ACTION_DOWN除外。
