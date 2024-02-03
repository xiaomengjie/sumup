#### 事件传递到DecorView

在ViewRootImpl中setView方法中，会创建InputEventReceiver接受输入事件

```java
mInputEventReceiver = new WindowInputEventReceiver(inputChannel, Looper.myLooper());

//stage（责任链模式）
mSyntheticInputStage = new SyntheticInputStage();
InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
      "aq:native-post-ime:" + counterSuffix);

//earlyPostImeStage
InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
InputStage imeStage = new ImeInputStage(earlyPostImeStage,
      "aq:ime:" + counterSuffix);
InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);

//nativePreImeStage
InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
      "aq:native-pre-ime:" + counterSuffix);

mFirstInputStage = nativePreImeStage;
mFirstPostImeInputStage = earlyPostImeStage;
```

接收到事件回调onImputEvent方法

##### onInputEvent@WindowInputEventReceiver:

```java
----> onInputEvent@WindowInputEventReceiver:

public void onInputEvent(InputEvent event) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "processInputEventForCompatibility");
    List<InputEvent> processedEvents;
    try {
        processedEvents =
            mInputCompatProcessor.processInputEventForCompatibility(event);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
  	//将所有的事件入队
    if (processedEvents != null) {
        if (processedEvents.isEmpty()) {
            // InputEvent consumed by mInputCompatProcessor
            finishInputEvent(event, true);
        } else {
            for (int i = 0; i < processedEvents.size(); i++) {
                enqueueInputEvent(
                        processedEvents.get(i), this,
                        QueuedInputEvent.FLAG_MODIFIED_FOR_COMPATIBILITY, true);
            }
        }
    } else {
        enqueueInputEvent(event, this, 0, true);
    }
}
```

##### enqueueInputEvent@ViewRootImpl

```java
----> enqueueInputEvent@ViewRootImpl:

//processImmediately == true
void enqueueInputEvent(InputEvent event,
        InputEventReceiver receiver, int flags, boolean processImmediately) {
  	//从QueuedInputEvent的缓存池中获取QueuedInputEvent
    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
    ...
    //将QueuedInputEvent添加到事件队列的尾部
    QueuedInputEvent last = mPendingInputEventTail;
    if (last == null) {
        mPendingInputEventHead = q;
        mPendingInputEventTail = q;
    } else {
        last.mNext = q;
        mPendingInputEventTail = q;
    }
  	//数量+1
    mPendingInputEventCount += 1;
    
    if (processImmediately) {
        doProcessInputEvents();
    } else {
        scheduleProcessInputEvents();
    }
}
```

##### doProcessInputEvents@ViewRootImpl

```java
----> doProcessInputEvents@ViewRootImpl:

void doProcessInputEvents() {
  	//从队列头依次取出QueuedInputEvent处理
    while (mPendingInputEventHead != null) {
        QueuedInputEvent q = mPendingInputEventHead;
        mPendingInputEventHead = q.mNext;
        if (mPendingInputEventHead == null) {
            mPendingInputEventTail = null;
        }
        q.mNext = null;

        mPendingInputEventCount -= 1;
        mViewFrameInfo.setInputEvent(mInputEventAssigner.processEvent(q.mEvent));
				//转发
        deliverInputEvent(q);
    }
		...
}
```

##### deliverInputEvent@ViewRootImpl

```java
----> deliverInputEvent@ViewRootImpl:

private void deliverInputEvent(QueuedInputEvent q) {
    ...
    try {
        ...

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }
				...
				//在经过ViewPostImeInputStage时，会给activity处理
        if (stage != null) {
            handleWindowFocusChanged();
          	//deliver中会调用onProcess处理QueuedInputEvent
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

##### onProcess@ViewPostImeInputStage

```java
----> onProcess@ViewPostImeInputStage:

protected int onProcess(QueuedInputEvent q) {
    if (q.mEvent instanceof KeyEvent) {
        return processKeyEvent(q);
    } else {
        final int source = q.mEvent.getSource();
        if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
          	//这里处理
            return processPointerEvent(q);
        } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
            return processTrackballEvent(q);
        } else {
            return processGenericMotionEvent(q);
        }
    }
}
```

##### processPointerEvent@ViewPostImeInputStage

```java
----> processPointerEvent@ViewPostImeInputStage:

private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent)q.mEvent;
    boolean handled = mHandwritingInitiator.onTouchEvent(event);

    mAttachInfo.mUnbufferedDispatchRequested = false;
    mAttachInfo.mHandlingPointerEvent = true;
  	//mView = DecorView
    handled = handled || mView.dispatchPointerEvent(event);
    maybeUpdatePointerIcon(event);
    maybeUpdateTooltip(event);
    mAttachInfo.mHandlingPointerEvent = false;
    if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
        mUnbufferedInputDispatch = true;
        if (mConsumeBatchedInputScheduled) {
            scheduleConsumeBatchedInputImmediately();
        }
    }
    return handled ? FINISH_HANDLED : FORWARD;
}
```

##### dispatchPointerEvent@DecorView

```java
----> dispatchPointerEvent@DecorView:
//在父类View中定义

public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```

##### dispatchTouchEvent@DecorView

```java
----> dispatchTouchEvent@DecorView:

//在activity的attach方法中
//mWindow.setCallback(this);
//所以cb = activity

@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```

##### dispatchTouchEvent@Activity

```java
----> dispatchTouchEvent@Activity:

public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
  	//处理分发
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
  	//都没处理事件，自己处理
    return onTouchEvent(ev);
}
```

##### superDispatchTouchEvent@PhoneWindow

```java
----> superDispatchTouchEvent@PhoneWindow:

@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

##### superDispatchTouchEvent@DecorView

```java
----> superDispatchTouchEvent@DecorView:

public boolean superDispatchTouchEvent(MotionEvent event) {
  	//ViewGroup的dispatchTouchEvent
    return super.dispatchTouchEvent(event);
}
```

