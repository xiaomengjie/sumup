#### GestureDetector.OnGestureListener

##### onDown(e: MotionEvent): Boolean

每次ACTION_DOWN事件出现时调用，返回值决定是否消费当前事件序列（好像只能返回true）

##### onShowPress(e: MotionEvent)

按下 100ms 后，不松开时触发（好像长按时就会触发）

##### onSingleTapUp(e: MotionEvent): Boolean

单击时触发（按下+抬起）

- 长按抬起时不会触发
- 双击时，第一次按下抬起会触发，第二次按下抬起不会触发

返回值用于系统记录，不表示当前事件序列是否消费（其他的也是）

##### onScroll(e1: MotionEvent?, e2: MotionEvent, distanceX: Float, distanceY: Float): Boolean

滑动时调用

- e1：按下时的事件
- e2：当前事件
- distanceX：X轴位移（旧位置 - 新位置）
- distanceY：Y轴位移（旧位置 - 新位置）

##### onFling(e1: MotionEvent?, e2: MotionEvent, velocityX: Float, velocityY: Float): Boolean

松手后的惯性滑动

- e1：按下时的事件
- e2：当前事件
- velocityX：x轴速率
- velocityY：y轴速率

##### onLongPress(e: MotionEvent)

长按时触发

- GestureDetector 500ms触发
- GestureDetectorCompat 100 + 500 = 600ms触发

#### GestureDetector.OnDoubleTapListener

##### onDoubleTap(e: MotionEvent): Boolean

双击时触发（按下+抬起+按下时）

- GestureDetector 300ms以内，40ms以上
- GestureDetectorCompat 300ms以内

##### onDoubleTapEvent(e: MotionEvent): Boolean

双击时触发，但是触发后的后续MOVE，UP事件都会收到

##### onSingleTapConfirmed(e: MotionEvent): Boolean

单击触发

- 和 onSingltTapUp() 的区别在于，⽤户的⼀次点击不会⽴即调⽤这个⽅法，⽽是在⼀定时间后（300ms），确认⽤户没有进⾏双击，这个⽅法才会被调⽤

```java
----> onTouchEvent@GestureDetector:

public boolean onTouchEvent(@NonNull MotionEvent ev) {
    boolean handled = false;

    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_DOWN:
            if (mDoubleTapListener != null) {
                boolean hadTapMessage = mHandler.hasMessages(TAP);
              //先移除TAP监听器
                if (hadTapMessage) mHandler.removeMessages(TAP);
              //isConsideredDoubleTap双击触发的判断方法
                if ((mCurrentDownEvent != null) && (mPreviousUpEvent != null)
                        && hadTapMessage
                        && isConsideredDoubleTap(mCurrentDownEvent, mPreviousUpEvent, ev)) {
                    mIsDoubleTapping = true;
                  //双击onDoubleTap回调
                    handled |= mDoubleTapListener.onDoubleTap(mCurrentDownEvent);
                  //双击onDoubleTapEvent回调
                    handled |= mDoubleTapListener.onDoubleTapEvent(ev);
                } else {
                  //TAP单击监听器，判断onSingleTapConfirmed是否触发
                  //DOUBLE_TAP_TIMEOUT = 300ms
                  //双击没触发再添加
                    mHandler.sendEmptyMessageDelayed(TAP, DOUBLE_TAP_TIMEOUT);
                }
            }

            if (mIsLongpressEnabled) {
              //长按监听器（ViewConfiguration.getLongPressTimeout() = 500ms）
              //在GestureDetectorCompat中是（TAP_TIMEOUT（100ms） + ViewConfiguration.getLongPressTimeout()）
                mHandler.removeMessages(LONG_PRESS);
                mHandler.sendMessageAtTime(
                        mHandler.obtainMessage(
                                LONG_PRESS,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS,
                                0 /* arg2 */),
                        mCurrentDownEvent.getDownTime()
                                + ViewConfiguration.getLongPressTimeout());
            }
        //showPress监听器（TAP_TIMEOUT = 100ms）
            mHandler.sendEmptyMessageAtTime(SHOW_PRESS,
                    mCurrentDownEvent.getDownTime() + TAP_TIMEOUT);
        //onDown回调
            handled |= mListener.onDown(ev);
            break;

        case MotionEvent.ACTION_MOVE:
            if (mIsDoubleTapping) {
                //双击按下后的后续事件，onDoubleTapEvent回调
                handled |= mDoubleTapListener.onDoubleTapEvent(ev);
            } else if (mAlwaysInTapRegion) {
                final int deltaX = (int) (focusX - mDownFocusX);
                final int deltaY = (int) (focusY - mDownFocusY);
                int distance = (deltaX * deltaX) + (deltaY * deltaY);
              //滑动距离大于mTouchSlopSquare
                if (distance > slopSquare) {
                    recordGestureClassification(
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__SCROLL);
                  //onScroll回调
                    handled = mListener.onScroll(mCurrentDownEvent, ev, scrollX, scrollY);
                    mLastFocusX = focusX;
                    mLastFocusY = focusY;
                    mAlwaysInTapRegion = false;
                  //移除各种监听器
                    mHandler.removeMessages(TAP);
                    mHandler.removeMessages(SHOW_PRESS);
                    mHandler.removeMessages(LONG_PRESS);
                }
            } else if ((Math.abs(scrollX) >= 1) || (Math.abs(scrollY) >= 1)) {
                recordGestureClassification(TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__SCROLL);
              //onScroll回调，好像只要move事件onScroll就会回调
                handled = mListener.onScroll(mCurrentDownEvent, ev, scrollX, scrollY);
                mLastFocusX = focusX;
                mLastFocusY = focusY;
            }
        //用力按下，马上触发长按
            final boolean deepPress =
                    motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
            if (deepPress && hasPendingLongPress) {
                mHandler.removeMessages(LONG_PRESS);
                mHandler.sendMessage(
                        mHandler.obtainMessage(
                              LONG_PRESS,
                              TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS,
                              0 /* arg2 */));
            }
            break;

        case MotionEvent.ACTION_UP:
            mStillDown = false;
            MotionEvent currentUpEvent = MotionEvent.obtain(ev);
            if (mIsDoubleTapping) {
                recordGestureClassification(
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DOUBLE_TAP);
              //双击触发后，抬手时onDoubleTapEvent回调
                handled |= mDoubleTapListener.onDoubleTapEvent(ev);
            } else if (mInLongPress) {
                mHandler.removeMessages(TAP);
                mInLongPress = false;
            } else if (mAlwaysInTapRegion && !mIgnoreNextUpEvent) {
                recordGestureClassification(
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__SINGLE_TAP);
              //onSingleTapUp单击触发
                handled = mListener.onSingleTapUp(ev);
              //mDeferConfirmSingleTap=true，表示达到300ms
                if (mDeferConfirmSingleTap && mDoubleTapListener != null) {
                  //onSingleTapConfirmed触发
                    mDoubleTapListener.onSingleTapConfirmed(ev);
                }
            } else if (!mIgnoreNextUpEvent) {
                if ((Math.abs(velocityY) > mMinimumFlingVelocity)
                        || (Math.abs(velocityX) > mMinimumFlingVelocity)) {
                  //onFling惯性滑动触发
                    handled = mListener.onFling(mCurrentDownEvent, ev, velocityX, velocityY);
                }
            }
            break;
    }
    return handled;
}
```

#### 回调顺序

```
/**
 * 一次单击触发：
 * onDown
 * onSingleTapUp
 * onSingleTapConfirmed（延时300ms）
 *
 * 一次双击触发：(DOWN时会优先判断是否为双击。之后在调用onDown)
 * onDown
 * onSingleTapUp
 *
 * onDoubleTap
 * onDoubleTapEvent
 * onDown
 * onDoubleTapEvent
 *
 * 长按时触发：
 * onDown
 * onShowPress
 * onLongPress
 *
 * 滑动时触发：
 * onDown
 * onScroll
 * 。。。
 * onScroll
 * onFling
 */
```
