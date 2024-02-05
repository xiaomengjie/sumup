#### dispatchTouchEvent@View

事件消费

```java
----> dispatchTouchEvent@View:

public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;
		...
      	
    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        //通知父View停止嵌套滑动
        stopNestedScroll();
    }

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //优先会将事件交给mOnTouchListener处理
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
				
      	//如果onTouch方法返回false，才会执行onTouchEvent方法
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
      	//通知父View停止嵌套滑动
        stopNestedScroll();
    }

    return result;
}
```

#### onTouchEvent@View

```java
----> onTouchEvent@View:

public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

  	//当前view是否可点击（CLICKABLE，LONG_CLICKABLE或者CONTEXT_CLICKABLE）
  	//android:contextClickable 配置CONTEXT_CLICKABLE
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
		
  	//view允许点击
    if ((viewFlags & ENABLED_MASK) == DISABLED
            && (mPrivateFlags4 & PFLAG4_ALLOW_CLICK_WHEN_DISABLED) == 0) {
      	//抬起时，取消按下状态
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        //可点击，消费事件
      	//不可点击，不消费
        return clickable;
    }
  	//mTouchDelegate：添加代理，增加点击有效接收面积（小按钮时可以用）
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
		
  	//TOOLTIP：长按提示功能（android:tooltipText设置）
  	//view可点击或者设置了tooltipText时，才会判断处理事件
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
            		//取消PFLAG3_FINGER_DOWN
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                  	//如果设置了tooltip，1500ms后隐藏
                    handleTooltipUp();
                }
            		//不可点击时
                if (!clickable) {
                  	//移除按下计时器
                    removeTapCallback();
                  	//移除长按计时器
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
            		//prepressed是否为预按下状态
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {//按下状态或者预按下状态
                    boolean focusTaken = false;
                  	//请求焦点（比如EditText）
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }
										
                    if (prepressed) {
                      	//如果是预按下状态设置为按下状态
                        setPressed(true, x, y);
                    }

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        //长按未触发，移除长按计时器
                        removeLongPressCallback();

                        //没有获取焦点，执行点击监听
                        if (!focusTaken) {
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
                      	//UnsetPressedState == Runnable
                      	//预按下状态时，发送64ms延时消失取消按下状态
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }
										//移除PFLAG_PREPRESSED标记
                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
            		//手指触摸屏幕，添加标记PFLAG3_FINGER_DOWN
            		//当显示tooltip时离触摸点隔一段距离，防止被手指挡住
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                mHasPerformedLongPress = false;

            		//不可点击，说明是设置了tooltip
                if (!clickable) {
                  	//长按计时器
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),//400ms（android11以前好像是500ms）
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    break;
                }
								
            		//鼠标右键点击处理处理
                if (performButtonActionOnTouchDown(event)) {
                    break;
                }
								
                //是否在滑动空间中
            		//依次调用ViewGroup的shouldDelayChildPressedState方法是否返回true
            		//shouldDelayChildPressedState默认返回true
            		//自定义不可滑动的ViewGroup，应该重写返回false
                boolean isInScrollingContainer = isInScrollingContainer();

                if (isInScrollingContainer) {
                  	//设置预按下状态
                  	//避免如果是滑动操作时，状态误变
                    mPrivateFlags |= PFLAG_PREPRESSED;
                  	
                    if (mPendingCheckForTap == null) {
                      	//CheckForTap == Runnable
                      	//run方法中，设置为按下状态和添加长按计时器（延时时间 = LongPressTimeout - TapTimeout）
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                  	//按下计时器
                  	//默认100ms
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // 不在滑动控件中，设置为按下状态
                    setPressed(true, x, y);
                  	//设置长按计时器
                    checkForLongClick(
                            ViewConfiguration.getLongPressTimeout(),
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                if (clickable) {
                  	//可点击，取消按下状态
                    setPressed(false);
                }
            		//移除各种计时器，重置状态变量
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;

            case MotionEvent.ACTION_MOVE:
                if (clickable) {
                  	//重绘Ripple Effect（背景水波纹？？？）
                    drawableHotspotChanged(x, y);
                }
            		
            		//不确定
            		//猜测是手指按在屏幕边缘时
            		//增加长按计时器的计时时常
                final int motionClassification = event.getClassification();
                final boolean ambiguousGesture =
                        motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
                int touchSlop = mTouchSlop;
                if (ambiguousGesture && hasPendingLongPressCallback()) {
                    if (!pointInView(x, y, touchSlop)) {
                        removeLongPressCallback();
                        long delay = (long) (ViewConfiguration.getLongPressTimeout()
                                * mAmbiguousGestureMultiplier);
                        // Subtract the time already spent
                        delay -= event.getEventTime() - event.getDownTime();
                        checkForLongClick(
                                delay,
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    }
                    touchSlop *= mAmbiguousGestureMultiplier;
                }

                //触摸移动到了view的外面
                if (!pointInView(x, y, touchSlop)) {
                  	//移除按下计时器和长按计时器，清除PFLAG3_FINGER_DOWN，设置为未按下状态
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }

            		//用力按下
                final boolean deepPress =
                        motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
                if (deepPress && hasPendingLongPressCallback()) {
                    //移除长按计时器，马上执行长按
                    removeLongPressCallback();
                    checkForLongClick(
                            0 /* send immediately */,
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
                }

                break;
        }
        return true;
    }
    return false;
}
```

##### checkForLongClick@View

```java
private void checkForLongClick(long delay, float x, float y, int classification) {
  	//检查是否支持长按或者设置了tooltip
    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE || (mViewFlags & TOOLTIP) == TOOLTIP) {
        mHasPerformedLongPress = false;

        if (mPendingCheckForLongPress == null) {
          	//CheckForLongPress == Runnable
          	//run方法中，会调用到performLongClickInternal方法
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        mPendingCheckForLongPress.setAnchor(x, y);
        mPendingCheckForLongPress.rememberWindowAttachCount();
        mPendingCheckForLongPress.rememberPressedState();
        mPendingCheckForLongPress.setClassification(classification);
      	//发送延时Message
        postDelayed(mPendingCheckForLongPress, delay);
    }
}
```

##### performLongClickInternal@View

```java
private boolean performLongClickInternal(float x, float y) {
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

    boolean handled = false;
    final ListenerInfo li = mListenerInfo;
    boolean shouldPerformHapticFeedback = true;
  	//首先处理mOnLongClickListener，返回值为false时才会执行下面的代码
    if (li != null && li.mOnLongClickListener != null) {
        handled = li.mOnLongClickListener.onLongClick(View.this);
        if (handled) {
            shouldPerformHapticFeedback =
                    li.mOnLongClickListener.onLongClickUseDefaultHapticFeedback(View.this);
        }
    }
    if (!handled) {
        final boolean isAnchored = !Float.isNaN(x) && !Float.isNaN(y);
        handled = isAnchored ? showContextMenu(x, y) : showContextMenu();
    }
  	//TOOLTIP处理，显示tooltip
    if ((mViewFlags & TOOLTIP) == TOOLTIP) {
        if (!handled) {
            handled = showLongClickTooltip((int) x, (int) y);
        }
    }
    if (handled && shouldPerformHapticFeedback) {
        performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
    }
    return handled;
}
```

#### 总结

- 当用户按下（ACTION_DOWN）
  - 如果在滑动控件中，设置预按下状态，并注册按下计时器（100ms）
    - 按下计时器如果触发，会设置为按下状态，并设置长按计时器（400 - 100 = 300ms）
  - 如果不在滑动控件中，设置为按下状，并注册长按计时器（400ms）
- 当进入按下状态并移动（ACTION_MOVE）
  - 重绘Ripple Effect
  - 如果移出自己的范围，自我标记本次事件失效，忽略后续事件（移除计时器，设置为未按下，清除PFLAG3_FINGER_DOWN）
- 当用户抬起（ACTION_UP）
  - 如果还是预按下状态（滑动控件中按下且100ms内抬起），马上设置为按下状态，64ms后设置为未按下
  - 如果是用户按下状态并未触发长按（400ms内抬起），设置为未按下状态，触发点击事件并清除一切状态
  - 如果长按已经触发，设置为未按下且清除一切状态
- 当事件意外结束（ACTION_CANCEL）- 父view将事件拦截
  - 设置为未按下，清除一切状态
