---
title: 一文搞懂事件分发，手势冲突和滑动冲突
date: 2020-10-04 12:00:00 +0800
categories: [Android, Framework]
tags: [TouchEvent, Motion, NestedScrolling]
---

手势冲突是 android 开发中经常遇到的一类问题了，网上讲解此问题的文章也很多，但是大都浅显地过一遍事件分发的调用栈，然后给出一个调用栈流程图；要不就是使用日志大法，用日志来验证自己的想法，完全没有参考价值；这里根据事件分发相关源码，记录下我的理解。

`MotionEvent` 里定义的 `ACTION_XXX` 还不少有 10 多个，看起来情况很复杂的样子，实际上只需要关注三个：`ACTION_DOWN`，`ACTION_MOVE` 和 `ACTION_UP`，而且在一个手势里它们的顺序是：`ACTION_DOWN` → `ACTION_MOVE` → `ACTION_MOVE` → ... → `ACTION_UP`。

## 跟踪源码的调用栈

- `Window.Callback.dispatchTouchEvent`（`Activity.dispatchTouchEvent` 实现了它，在 `Activity.attach` 里通过 `Window.setCallback` 设置进去）
- `Window.superDispatchTouchEvent`（实现在 `PhoneWindow.superDispatchTouchEvent`）
- `DecorView.superDispatchTouchEvent`
- `ViewGroup.dispatchTouchEvent`
- `ViewGroup.onInterceptTouchEvent`
- `View.dispatchTouchEvent`
- `View.onTouchEvent`
- `ViewGroup.onTouchEvent`
- `Activity.onTouchEvent`

网上大部分文章到此就结束了，实际上重点应该在 `ViewGroup.dispatchTouchEvent`，里面是事件分发的核心逻辑，我把它切分为三个阶段：

### 拦截

```java
// DOWN 会被拦截，后续的 MOVE 和 UP 如果有 touch target 也会被拦截
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    // 可以通过 requestDisallowInterceptTouchEvent 跳过此阶段，
    // 一般是 child 调用 parent.requestDisallowInterceptTouchEvent 来阻止 parent 拦截 touch event
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // 后续的 MOVE 和 UP 没有 touch target 则直接走向 onTouchEvent 也就不需要拦截了
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}
```

### 分发

```java
// 收到 CANCEL 或者 onInterceptTouchEvent 返回 true，则不分发 DOWN 给 children
// 导致 children 收不到 DOWN 以及没有 touch target
final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
// ...
if (!canceled && !intercepted) {
    // ...
    if (actionMasked == MotionEvent.ACTION_DOWN
        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
        // ... 按顺序分发 ACTION_DOWN，child index(in children array) 越大优先级越高，child z value 越大优先级越高
        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
        for (int i = childrenCount - 1; i >= 0; i--) {
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            // ... touch 是否落在 child 的矩形区域内
            if (!child.canReceivePointerEvents()
                || !isTransformedTouchPointInView(x, y, child, null)) {
                ev.setTargetAccessibilityFocus(false);
                continue;
            }
            // ... 将 touch event 坐标转换为 child 区域坐标，分发给 child；当有第一个 child 消费时，记录起来并中断剩下的分发过程
            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                // ...
                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                alreadyDispatchedToNewTouchTarget = true;
                break;
            }
        // ...
}
```

### 处理

```java
// 如果没有 touch target，则走自己的 View.dispatchTouchEvent 流程（相当于流向 onTouchEvent）
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
} else {
    // 分发给 touch target
    // 但如果 onInterceptTouchEvent 返回 true，则发送 CANCEL 给 touch target，后续将不再流向 touch target，而是直接流向 onTouchEvent
    // onInterceptTouchEvent 拦截的那个 touch 不会流向 onTouchEvent
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
```

## 总结 `dispatchTouchEvent`

- `onInterceptTouchEvent` 总是会收到 DOWN，但不一定会收到后续的 MOVE 和 UP（没有 touch target 的话，就不需要拦截了，直接走到 `onTouchEvent` 去了）
- 只有在第一个 DOWN 时，才会分发给所有的 children，找到第一个消费的 child（就是 touch target，后续的 MOVE 和 UP 只分发给它）
- `onInterceptTouchEvent` 返回 true 会导致 touch target 置空（并收到 CANCEL），这样后续的 MOVE 和 UP 因为没有 touch target 而直接走向 `onTouchEvent`
- child 通过 `requestDisallowInterceptTouchEvent` 告知 parent 不要拦截事件流，交由 child 处理

## 常见滑动效果的实现

了解 `dispatchTouchEvent` 后，我们看看常用的具有滑动效果的 widget 是怎么处理 touch event 的，参考 `ViewPager` 和 `RecyclerView`，代码比较多，这里就不贴了，总结下其套路：

- 在 `onInterceptTouchEvent` 和 `onTouchEvent` 这两个方法里介入
- `onInterceptTouchEvent` 只监听不拦截 DOWN；拦截 DOWN 会导致 children 接收不到 DOWN，那么它们的 OnClick 和 OnLongClick 就无法触发；更复杂的情况是 children 里也包含具有滑动效果的 widget
- `onTouchEvent` 返回 true 做一个兜底方案；万一没有 child 消费 touch（一般情况是没有 `OnClickListener`），而自己也不消费 touch 的话，就会没有 touch target，后续的 touch event 会直接流向 parent.`onTouchEvent`，我们想拦截也拦截不了
- 在 `onInterceptTouchEvent` 里监听和拦截（满足情况下）MOVE，`onTouchEvent` 里也要消费 MOVE（没有 child 消费 touch 的话，后续的 touch 就直接流向 `onTouchEvent` 了）
- `ACTION_MOVE` 不会直接触发滑动，而是与 `ACTION_DOWN` 的点有了一定长度的距离后才触发，这个距离叫 touch slop（`ViewConfiguration#getScaledTouchSlop`），用以消除抖动，使滑动效果更加顺滑

记住上面的关键点，基本上就可以解决大部分手势冲突，并能够开发稳健的具有手势的 `ViewGroup` 了；但实际开发中，有一个问题是更加常见的：滑动冲突。在引入嵌套滑动之前，说说为什么 `dispatchTouchEvent` 很难解决滑动冲突。滑动一般是由 MOVE > touch slop 触发，这样 parent 总是会优先触发而 child 总是会被屏蔽，而且只能由 child 发起 `requestDisallowInterceptTouchEvent` 告知 parent 不要拦截，从 parent 的视角看，没有其他办法主动得知 child 的需求，而大多数滑动冲突是需要在 parent 里解决的（脑补淘宝、京东这类 app 的首页）。

## 引入 `NestedScrollingChild` 和 `NestedScrollingParent` 解决滑动冲突

上面我们在分析 `ViewPager` 和 `RecyclerView` 的与滚动相关的代码时了解到，`ACTION_MOVE` 会产生 scroll 和 fling 的动量 dy（垂直方向），全部由自己消费掉（View.scrollBy）就会产生滑动的效果；而嵌套滑动引入了协商机制，对于动量 dy：

1. 先给 parent 消费，parent 可以根据自身情况，选择不消费、消费一部分或者消费全部，此时 dy = dy - consumed
2. child 根据滋生情况，消费剩下的 dy（全部 or 部分），此时 dy = dy - consumed
3. 如果还有 dy 剩余（dy > 0），则把剩余的 dy 的处置权交由 parent

对于上述流程，android 提供了实现：NestedScrollingChildHelper，我们在实现 NestedScrollingChild 时，只需把方法代理至 helper 即可实现通用的 child 逻辑

具体 api 调用流如下图：

![01.jpg](../../../../image/2020-10-04-dispatchtouchevent/01.jpg)

## 实战

### 三段式布局

![02.jpeg](../../../../image/2020-10-04-dispatchtouchevent/02.jpeg)

这里以常见的三段式布局为例子看下嵌套滑动怎么用；上图是淘宝首页，由 head、bar 和 list 三个部分组成，bar 在滑动时会粘连在顶部

1. 当 touch 落在 list 上时，由 list 产生动量 dy，在 list 消费 dy 前，先给 parent 消费，parent 完全消费 dy，scroll 三个 child 直到 head 不可见
2. list 因为 dy 被 parent 消费掉而不产生滑动，直到 head 不可见才有剩下的 dy 用以消费，产生滑动
3. 当 touch 落在 head 和 bar 上时，由 parent 产生动量 dy；此时 parent 才是 child，而 list 是 parent（明确谁是 child，谁是 parent，view tree 结构上的父子关系不一定是嵌套滑动里的父子关系，动量产生者才是 child，child 主动分发动量）
4. parent 先消费 dy 直到 head 不可见，然后分发给 list
5. fling 同理

### 更复杂的布局

![03.jpg](../../../../image/2020-10-04-dispatchtouchevent/03.jpg)

结合上述的所有办法，解决更加复杂的页面；上图是糖纸的首页，最外层是 `ViewPager`，然后是 refresh layout 加上三段式的布局，head 里又有可以左右滑动的 banner 和卡片列表，还有可以上下滑动的滚动资讯条，我们一个个解决：

- banner 是可以左右滑动的，会与 `ViewPager` 冲突；我的期望是当 touch 落在 banner 上时，左右滑动完全交由 banner 处理，所以给 `ViewPager` 添加一个 freeze 方法（当 ViewPager is freeze 时不拦截 touch，此时不能使用 `requestDisallowInterceptTouchEvent` 否则上下滑动被屏蔽）；而上下滑动距离又会引起 refresh 和三段布局的滑动，需在 banner 触发左右滑动时，调用 `requestDisallowInterceptTouchEvent` 让 parents 不在拦截 touch
- 上下滚动的咨询条，同 banner 的处理方式，只不过它只需解决 refresh 和三段布局的上下滑动冲突即可：它自己完全消费动量 dy
- 左右滑动的卡片列表同 banner
- `SwipeRefreshLayout` 看下述源码可以看到是由 touch event 和 nested scroll 两种方式触发，touch event 不够灵活屏蔽掉，选用 nested scroll；`dispatchNestedPreScroll`、`dispatchNestedScroll`、`dispatchNestedPreFling` 和 `dispatchNestedFling` 正常触发，`startNestedScroll` 则在 scrollY == 0 时触发，这样就只在滚动到顶的时候才触发 refresh。

```java
// 显示 loading
private void startDragging(float y) {
    final float yDiff = y - mInitialDownY;
    if (yDiff > mTouchSlop && !mIsBeingDragged) {
        mInitialMotionY = mInitialDownY + mTouchSlop;
        mIsBeingDragged = true;
        mProgress.setAlpha(STARTING_PROGRESS_ALPHA);
    }
}

// 由 touch event 触发 loading
case MotionEvent.ACTION_MOVE: {
    pointerIndex = ev.findPointerIndex(mActivePointerId);
    if (pointerIndex < 0) {
        Log.e(LOG_TAG, "Got ACTION_MOVE event but have an invalid active pointer id.");
        return false;
    }

    final float y = ev.getY(pointerIndex);
    startDragging(y);

    if (mIsBeingDragged) {
        final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
        if (overscrollTop > 0) {
            moveSpinner(overscrollTop);
        } else {
            return false;
        }
    }
    break;
}

// 由 nested scroll 触发 loading
public void onNestedScroll(final View target, final int dxConsumed, final int dyConsumed,
                           final int dxUnconsumed, final int dyUnconsumed) {
    // Dispatch up to the nested parent first
    dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed,
            mParentOffsetInWindow);

    // This is a bit of a hack. Nested scrolling works from the bottom up, and as we are
    // sometimes between two nested scrolling views, we need a way to be able to know when any
    // nested scrolling parent has stopped handling events. We do that by using the
    // 'offset in window 'functionality to see if we have been moved from the event.
    // This is a decent indication of whether we should take over the event stream or not.
    final int dy = dyUnconsumed + mParentOffsetInWindow[1];
    if (dy < 0 && !canChildScrollUp()) {
        mTotalUnconsumed += Math.abs(dy);
        moveSpinner(mTotalUnconsumed);
    }
}
```