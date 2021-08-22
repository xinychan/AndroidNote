# View 事件分发机制

## MotionEvent

| MotionEvent   | 作用                                             |
| ------------- | ------------------------------------------------ |
| ACTION_DOWN   | 按下时触发                                       |
| ACTION_MOVE   | 移动时触发，会多次触发                           |
| ACTION_UP     | 抬起时触发                                       |
| ACTION_CANCEL | 事件被上层拦截时触发；滑动超出 View 边界不会触发 |

## 事件分发整体流程

Activity.dispatchTouchEvent()
==>
Window.superDispatchTouchEvent()
==>
PhoneWindow.superDispatchTouchEvent()
==>
DecorView.superDispatchTouchEvent()
==>
ViewGroup.dispatchTouchEvent()
==>
View.dispatchTouchEvent()
==>
View.onTouchEvent()

**Activity.dispatchTouchEvent()==>Window.superDispatchTouchEvent()**

// 每一个 Activity 都对应一个 Window，Window 是抽象类，其唯一实现类为 PhoneWindow
// 所以 Activity 对应的 Window 是一个 PhoneWindow 实例

**Window.superDispatchTouchEvent()==>PhoneWindow.superDispatchTouchEvent()**

// PhoneWindow 是 Window 的唯一实现类，Window 的具体实现都在 PhoneWindow 中完成

**PhoneWindow.superDispatchTouchEvent()==>DecorView.superDispatchTouchEvent()**

// DecorView 是 PhoneWindow 中的全局变量，继承自 FrameLayout
// DecorView 是一个顶级 View，内部会包含一个竖直方向的 LinearLayout，这个 LinearLayout 有上下两部分，分为 ActionBar 和 ContentParent 两个子元素
// Activity 的布局是 ContentParent 中的子元素

    // Activity 中的 setContentView
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID); // 初始化 ContentParent
        initWindowDecorActionBar(); // 初始化 ActionBar
    }

// View 层的所有事件都要先经过 DecorView 后才传递给我们的 View

**DecorView.superDispatchTouchEvent()==>ViewGroup.dispatchTouchEvent()**

// DecorView 继承自 FrameLayout，执行父类 ViewGroup 的 dispatchTouchEvent

**ViewGroup.dispatchTouchEvent()==>View.dispatchTouchEvent()**

// 事件一直传到，ViewGroup#dispatchTouchEvent，都是分发事件
// 直到 View#dispatchTouchEvent，开始处理事件

**View.dispatchTouchEvent()==>View.onTouchEvent()**

// 子类具体的事件处理逻辑

## View 事件处理流程

    View#dispatchTouchEvent
    ==>
    View.setOnTouchListener(new View.OnTouchListener() {
        @Override
        public boolean onTouch(View view, MotionEvent motionEvent) {
            return false;
        }
    });
    ==>
    View#onTouchEvent
    ==>
    // 在 MotionEvent.ACTION_UP 中执行了 View#performClick
    // View#performClick 中执行了 onClick()
    View.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
        }
    });

## View 主要方法

### View#dispatchTouchEvent

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

        if (onFilterTouchEventForSecurity(event)) { // 若当前 Window 被遮挡，则不会下发事件
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            // mOnTouchListener 代表 View.setOnTouchListener() 中的 OnTouchListener
            // mOnTouchListener.onTouch() 代表 View.OnTouchListener 中的 onTouch()
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            // onTouchEvent() 中处理 ACTION_UP/ACTION_DOWN/ACTION_MOVE/ACTION_CANCEL 事件
            // mOnClickListener.onClick 等事件也在 onTouchEvent() 中处理
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

### View#onTouchEvent

    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable;
        }
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                // ACTION_UP 事件
                case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                        handleTooltipUp();
                    }
                    if (!clickable) {
                        removeTapCallback();
                        removeLongPressCallback();
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
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

                            // PerformClick 中处理 mOnClickListener.onClick  事件
                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClickInternal();
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
                // ACTION_DOWN 事件
                case MotionEvent.ACTION_DOWN:
                    if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                        mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                    }
                    mHasPerformedLongPress = false;

                    if (!clickable) {
                        checkForLongClick(0, x, y);
                        break;
                    }

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
                        checkForLongClick(0, x, y);
                    }
                    break;
                // ACTION_CANCEL 事件
                case MotionEvent.ACTION_CANCEL:
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    break;
                // ACTION_MOVE 事件
                case MotionEvent.ACTION_MOVE:
                    if (clickable) {
                        drawableHotspotChanged(x, y);
                    }

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        // Remove any future long press/tap checks
                        removeTapCallback();
                        removeLongPressCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    }
                    break;
            }

            return true;
        }

        return false;
    }

### View#performClick

    private final class PerformClick implements Runnable {
        @Override
        public void run() {
            performClickInternal();
        }
    }

    private boolean performClickInternal() {
        // Must notify autofill manager before performing the click actions to avoid scenarios where
        // the app has a click listener that changes the state of views the autofill service might
        // be interested on.
        notifyAutofillManagerOnClick();

        return performClick();
    }

    public boolean performClick() {
        // We still need to call this method to handle the cases where performClick() was called
        // externally, instead of through performClickInternal()
        notifyAutofillManagerOnClick();

        final boolean result;
        final ListenerInfo li = mListenerInfo;
        // 处理 View.setOnClickListener() 中的 onClick()
        // 只要执行了 onClick 就会返回 true，事件被消费
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        notifyEnterOrExitForAutoFillIfNeeded(true);

        return result;
    }

### View#setOnClickListener

    public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }

## ViewGroup 事件分发流程

**分发流程规则：**
**1-如果分发链没有形成，事件链最底层 View 无法处理事件**
**2-分发链形成之后，同系列事件再次传递时，直接按分发链传递事件，不会再询问事件的分发**
**3-分发链形成之后，处理事件最底层 View，对上层 View 的事件分发有反向制约的权利**
**4-上层 View 有两次选择处理事件的机会；1-事件首次分发到 View 时，2-底层 View 都不处理事件时**

**ViewGroup 与 View 方法的区别：**

| 方法                  | 作用     | View | ViewGroup | Activity |
| --------------------- | -------- | ---- | --------- | -------- |
| dispatchTouchEvent    | 分发事件 | 有   | 有        | 有       |
| onInterceptTouchEvent | 拦截事件 | 无   | 有        | 无       |
| onTouchEvent          | 处理事件 | 有   | 有        | 有       |

### 事件拦截时流程

    分发ACTION_DOWN事件，并ViewGroup拦截事件
    ==>
    ViewGroup#dispatchTouchEvent
    ==>
    // 返回true；对事件进行拦截，自己处理事件
    ViewGroup#onInterceptTouchEvent
    ==>
    ViewGroup#dispatchTransformedTouchEvent
    ==>
    // 处理事件
    View#dispatchTouchEvent
    ==>
    ACTION_MOVE/ACTION_UP 等系列事件，也自己处理

### 事件不拦截时流程

    分发ACTION_DOWN事件，ViewGroup不拦截事件
    ==>
    ViewGroup#dispatchTouchEvent
    ==>
    //返回false；不对事件进行拦截，分发给下层子View处理事件
    ViewGroup#onInterceptTouchEvent
    ==>
    // 对下层子View进行排序；依据 View#getZ 排序
    ViewGroup#buildTouchDispatchChildList
    ==>
    // 判断是否接收点击事件
    ViewGroup#canViewReceivePointerEvents
    // 判断点击事件是否在View的范围
    ViewGroup#isTransformedTouchPointInView
    ==>
    // 给下层子View分发事件
    // 当下层View是ViewGroup时，dispatchTransformedTouchEvent返回false，继续分发事件
    // 当下层View是View时，并且处理事件时，dispatchTransformedTouchEvent返回true
    ViewGroup#dispatchTransformedTouchEvent
    ==>
    // 不对事件进行拦截时，child != null；会执行当前ViewGroup的下层子View的dispatchTouchEvent逻辑
    // 下层子View是ViewGroup时，重复执行 ViewGroup#dispatchTouchEvent 逻辑继续分发事件；
    // 下层子View是View时，执行 View#dispatchTouchEvent 处理事件；
    child.dispatchTouchEvent
    ==>
    // 下层子View处理事件，修改部分全部变量赋值，
    ViewGroup#addTouchTarget
    ==>
    // 当下层View是View时，并且处理事件时，在这里返回 true
    // 并将结果层层返回给上层父View；至此，MotionEvent.ACTION_DOWN 分发链形成
    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
        handled = true;
    }
    ==>
    分发ACTION_MOVE/ACTION_UP事件，ViewGroup不拦截事件
    ==>
    ViewGroup#dispatchTouchEvent
    ==>
    //返回false；不对事件进行拦截，分发给下层子View处理事件
    ViewGroup#onInterceptTouchEvent
    ==>
    // 直接调用 MotionEvent.ACTION_DOWN 时形成的 target.child 的分发链，不再询问其他View是否处理事件
    // 并将结果层层返回给上层父View；至此，MotionEvent.ACTION_XXX 事件处理完成
    if (dispatchTransformedTouchEvent(ev, cancelChild,
        target.child, target.pointerIdBits)) {
        handled = true;
    }

## ViewGroup 主要方法

### ViewGroup#dispatchTouchEvent

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
            // ACTION_DOWN 事件会初始化全局变量
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            // 当ViewGroup的下层子View处理事件时，MotionEvent.ACTION_MOVE事件下发时，mFirstTouchTarget != null
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                // 下层子View通过设定 FLAG_DISALLOW_INTERCEPT 值，来反向制约上层父View的拦截事件
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    // 判断是否拦截事件；拦截事件则自己处理，不拦截事件则分发给下层子View处理
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
            // 每次新事件下发，alreadyDispatchedToNewTouchTarget 都为 false
            boolean alreadyDispatchedToNewTouchTarget = false;
            // 拦截事件时，不执行此分支，自己处理事件
            // 不拦截事件时，执行此分支，对事件进行分发
            if (!canceled && !intercepted) {

                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                // 当 MotionEvent.ACTION_DOWN 事件时进入此条件；MotionEvent.ACTION_MOVE 事件直接跳过
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
                        // 对下层子View进行排序；依据 View#getZ 排序
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        // 对下层子View进行排序后，for循环遍历下层子View；轮询下层子View是否处理事件
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

                            // canViewReceivePointerEvents：判断是否接收点击事件
                            // isTransformedTouchPointInView：判断点击事件是否在View的范围
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
                            // 给下层子View分发事件
                            // 当下层View是ViewGroup时，dispatchTransformedTouchEvent返回false，继续分发事件
                            // 当下层View是View时，并且处理事件时，dispatchTransformedTouchEvent返回true
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
                                // 当下层View是View时，并且处理事件时；
                                // 赋值：target.next = null; mFirstTouchTarget = target != null;
                                // 赋值：alreadyDispatchedToNewTouchTarget = true;
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
            // 对事件进行拦截，mFirstTouchTarget == null
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                // 对事件进行拦截时，dispatchTransformedTouchEvent 会执行 View#dispatchTouchEvent 处理事件
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // 当下层View是View时，并且处理事件时；mFirstTouchTarget = target != null;alreadyDispatchedToNewTouchTarget = true;
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    // target.next = null;while循环只执行一次
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        // 当下层View是View时，并且处理事件时，在这里返回 true
                        // 并将结果层层返回给上层父View；至此，MotionEvent.ACTION_DOWN 分发链形成
                        handled = true;
                    } else {
                        // 每次新事件下发，alreadyDispatchedToNewTouchTarget 都为 false
                        // 当事件被上层父View拦截时，intercepted = true，cancelChild = true
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        // 直接调用 MotionEvent.ACTION_DOWN 时形成的 target.child 的分发链，不再询问其他View是否处理事件
                        // 并将结果层层返回给上层父View；至此，MotionEvent.ACTION_XXX 事件处理完成
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

### ViewGroup#onInterceptTouchEvent

    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }

### ViewGroup#dispatchTransformedTouchEvent

    // dispatchTransformedTouchEvent：对事件进行分发，或者处理
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        // 当前层级View拦截了下层子View事件时，cancel = true；会触发 MotionEvent.ACTION_CANCEL 事件
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            // 对事件进行拦截时，没有下层子View处理事件，child == null；会执行当前ViewGroup的View#dispatchTouchEvent处理事件
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
            // 不对事件进行拦截时，child != null；会执行当前ViewGroup的下层子View的dispatchTouchEvent逻辑
            // 下层子View是ViewGroup时，重复执行 ViewGroup#dispatchTouchEvent 逻辑继续分发事件；
            // 下层子View是View时，执行 View#dispatchTouchEvent 处理事件；
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }

### ViewGroup#buildTouchDispatchChildList

    public ArrayList<View> buildTouchDispatchChildList() {
        return buildOrderedChildList();
    }

    // 对下层子View进行排序；依据 View#getZ 排序
    ArrayList<View> buildOrderedChildList() {
        final int childrenCount = mChildrenCount;
        if (childrenCount <= 1 || !hasChildWithZ()) return null;

        if (mPreSortedChildren == null) {
            mPreSortedChildren = new ArrayList<>(childrenCount);
        } else {
            // callers should clear, so clear shouldn't be necessary, but for safety...
            mPreSortedChildren.clear();
            mPreSortedChildren.ensureCapacity(childrenCount);
        }

        final boolean customOrder = isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            // add next child (in child order) to end of list
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View nextChild = mChildren[childIndex];
            final float currentZ = nextChild.getZ();

            // insert ahead of any Views with greater Z
            int insertIndex = i;
            while (insertIndex > 0 && mPreSortedChildren.get(insertIndex - 1).getZ() > currentZ) {
                insertIndex--;
            }
            mPreSortedChildren.add(insertIndex, nextChild);
        }
        return mPreSortedChildren;
    }

### ViewGroup#addTouchTarget

    private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
        final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }

### ViewGroup#requestDisallowInterceptTouchEvent

    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
        // 下层子View通过 View#requestDisallowInterceptTouchEvent 反向制约上层父View的拦截事件
        // 通过设定 FLAG_DISALLOW_INTERCEPT 值，可以屏蔽来自上层父View的拦截事件
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
