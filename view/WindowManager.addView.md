#### handleResumeActivity@ActivityThread

```java
----> handleResumeActivity@ActivityThread:

@Override
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
    ...
    //执行onResume
    if (!performResumeActivity(r, finalStateRequest, reason)) {
        return;
    }
    ...

    final Activity a = r.activity;
    ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
      	//从window拿到在setContentView时创建的DecorView
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
      	//获得wm（在attach时赋值，WindowManagerImpl）
        ViewManager wm = a.getWindowManager();
      	//拿到DecorView的LayoutParams（LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT）
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
      	//设置窗口类型为应用窗口TYPE_BASE_APPLICATION = 1
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
              	//WindowManagerImpl.addView，decorView添加到wm
                wm.addView(decor, l);
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    } else if (!willBeVisible) {
        if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
        r.hideForNow = true;
    }
		...
  	//添加Idler
    Looper.myQueue().addIdleHandler(new Idler());
}
```

#### performResumeActivity@ActivityThread

```java
----> performResumeActivity@ActivityThread:

public boolean performResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        String reason) {
    ...
    try {
        ...
        //执行activity的onResume方法
        r.activity.performResume(r.startsNotResumed, reason);

        r.state = null;
        r.persistentState = null;
      	//状态设置为ON_RESUME
        r.setState(ON_RESUME);

        reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
    } catch (Exception e) {
        if (!mInstrumentation.onException(r.activity, e)) {
            throw new RuntimeException("Unable to resume activity "
                    + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
        }
    }
    return true;
}
```

#### addView@WindowManagerImpl

```java
----> addView@WindowManagerImpl:

private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyTokens(params);
  	//mGlobal = WindowManagerGlobal(单例)
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}
```

#### addView@WindowManagerGlobal

```java
----> addView@WindowManagerGlobal:

public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
  	..
  	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    ...

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ...

        IWindowSession windowlessSession = null;
        // If there is a parent set, but we can't find it, it may be coming
        // from a SurfaceControlViewHost hierarchy.
        if (wparams.token != null && panelParentView == null) {
            for (int i = 0; i < mWindowlessRoots.size(); i++) {
                ViewRootImpl maybeParent = mWindowlessRoots.get(i);
                if (maybeParent.getWindowToken() == wparams.token) {
                    windowlessSession = maybeParent.getWindowSession();
                    break;
                }
            }
        }

      	//创建ViewRootImpl
        if (windowlessSession == null) {
            root = new ViewRootImpl(view.getContext(), display);
        } else {
            root = new ViewRootImpl(view.getContext(), display,
                    windowlessSession, new WindowlessWindowLayout());
        }
				//设置DecorView的layoutParams（MARTCH_PARENT,MARTCH_PARENT）
        view.setLayoutParams(wparams);
				
      	//DecorView，ViewRootImpl，LayoutParams分别保存到mViews，mRoots，mParams中
      	//通过反射WindowManagerGlobal可以获取到所有窗口的DecorView，ViewRootImpl和LayoutParams
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        try {
          	//root == DecorView
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
            final int viewIndex = (index >= 0) ? index : (mViews.size() - 1);
            // BadTokenException or InvalidDisplayException, clean up.
            if (viewIndex >= 0) {
                removeViewLocked(viewIndex, true);
            }
            throw e;
        }
    }
}
```

#### setView@ViewRootImpl

```java
----> setView@ViewRootImpl:

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
    synchronized (this) {
        if (mView == null) {
          	//mView赋值为DecorView
            mView = view;

            ...
            //测量布局
            requestLayout();
            ...

            try {
                mOrigWindowType = mWindowAttributes.type;
                mAttachInfo.mRecomputeGlobalAttributes = true;
                collectViewAttributes();
                adjustLayoutParamsForCompatibility(mWindowAttributes);
                controlInsetsForCompatibility(mWindowAttributes);

                Rect attachedFrame = new Rect();
                final float[] compatScale = { 1f };
              	//跨进程调用，将窗口添加到WMS（WindowManagerService）
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId,
                        mInsetsController.getRequestedVisibleTypes(), inputChannel, mTempInsets,
                        mTempControls, attachedFrame, compatScale);
                ...
            } catch (RemoteException | RuntimeException e) {
                mAdded = false;
                mView = null;
                mAttachInfo.mRootView = null;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                throw new RuntimeException("Adding window failed", e);
            } finally {
                if (restore) {
                    attrs.restore();
                }
            }
						...
          	//处理wms的返回结果
            if (res < WindowManagerGlobal.ADD_OKAY) {
                mAttachInfo.mRootView = null;
                mAdded = false;
                mFallbackEventHandler.setView(null);
                unscheduleTraversals();
                setAccessibilityFocus(null, null);
                switch (res) {
                    case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                    case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not valid; is your activity running?");
                    case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- token " + attrs.token
                                + " is not for an application");
                    case WindowManagerGlobal.ADD_APP_EXITING:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- app for token " + attrs.token
                                + " is exiting");
                    case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                        throw new WindowManager.BadTokenException(
                                "Unable to add window -- window " + mWindow
                                + " has already been added");
                    case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                        // Silently ignore -- we would have just removed it
                        // right away, anyway.
                        return;
                    case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- another window of type "
                                + mWindowAttributes.type + " already exists");
                    case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                        throw new WindowManager.BadTokenException("Unable to add window "
                                + mWindow + " -- permission denied for window type "
                                + mWindowAttributes.type);
                    case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified display can not be found");
                    case WindowManagerGlobal.ADD_INVALID_TYPE:
                        throw new WindowManager.InvalidDisplayException("Unable to add window "
                                + mWindow + " -- the specified window type "
                                + mWindowAttributes.type + " is not valid");
                    case WindowManagerGlobal.ADD_INVALID_USER:
                        throw new WindowManager.BadTokenException("Unable to add Window "
                                + mWindow + " -- requested userId is not valid");
                }
                throw new RuntimeException(
                        "Unable to add window -- unknown error code " + res);
            }

            ...
            if (inputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
              	//创建输入事件接受器
                mInputEventReceiver = new WindowInputEventReceiver(inputChannel,
                        Looper.myLooper());

                if (ENABLE_INPUT_LATENCY_TRACKING && mAttachInfo.mThreadedRenderer != null) {
                    InputMetricsListener listener = new InputMetricsListener();
                    mHardwareRendererObserver = new HardwareRendererObserver(
                            listener, listener.data, mHandler, true /*waitForPresentTime*/);
                    mAttachInfo.mThreadedRenderer.addObserver(mHardwareRendererObserver);
                }
                // Update unbuffered request when set the root view.
                mUnbufferedInputSource = mView.mUnbufferedInputSource;
            }
						//将ViewRootImpl设置为DecorView的parent
            view.assignParent(this);
            ...
        }
    }
}
```

#### requestLayout@ViewRootImpl

```java
----> requestLayout@ViewRootImpl:

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
      	//检查请求更新UI的线程是否为创建ViewRootImpl的线程
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

scheduleTraversals@ViewRootImpl

```java
----> scheduleTraversals@ViewRootImpl:

void scheduleTraversals() {
  	//一次调用就行
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
      	//发送同步屏障
       	mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
      	//发送异步Message（保证Message优先执行）
      	//message执行时，会调用mTraversalRunnable.run方法，继而调用doTraversal
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
      	//硬件绘制相关？？？
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

#### doTraversal@ViewRootImpl（后续执行已经不在同一个Message了）

```java
----> doTraversal@ViewRootImpl:

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //从MessageQueue中移除同步屏障，保证后续消息的正常执行
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
				//Measure，layout，draw
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

#### performTraversals@ViewRootImpl

```java
----> performTraversals@ViewRootImpl:

private void performTraversals() {
    //host == DecorView
    final View host = mView;
		...
		
    //requestLayout时，mLayoutRequested = true
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        ...
        // Ask host how big it wants to be
        // 测量DecorView，你有多大
        //（宽度为WRAP_CONTENT时可能测量3次）
        //（宽度为MARCH_PARENT时测量1次） 
        windowSizeMayChange |= measureHierarchy(host, lp, mView.getContext().getResources(),
                desiredWindowWidth, desiredWindowHeight, shouldOptimizeMeasure);
    }

   ...
		//measureHierarchy测量完成，mLayoutRequested设置为false，layoutRequested = true
    if (layoutRequested) {
        // Clear this now, so that if anything requests a layout in the
        // rest of this function we will catch it and re-run a full
        // layout pass.
        mLayoutRequested = false;
    }
		
  	//判断window大小是否需要调整
    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.height() < desiredWindowHeight && frame.height() != mHeight));
    windowShouldResize |= mDragResizing && mPendingDragResizing;
    ...

    if (mFirst || windowShouldResize || viewVisibilityChanged || params != null
            || mForceNextWindowRelayout) {
        ...

        try {
            ...
          	//布局窗口（WMS相关）
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
           
        } catch (RemoteException e) {
        } finally {
            if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }

        if (!mStopped || mReportNextDraw) {
          	//窗口宽高与测量宽高不一致时
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()
                    || dispatchApplyInsets || updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width,
                        lp.privateFlags);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height,
                        lp.privateFlags);

                //再次测量
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                // Implementation of weights from WindowManager.LayoutParams
                // We just grow the dimensions as needed and re-measure if
                // needs be
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;
								
              	//设置了weight，还会再次测量
                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {
                    //再次测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }
                layoutRequested = true;
            }
        }
    } else {
      	...
    }
		...
    if (!isViewVisible) {
        mLastPerformTraversalsSkipDrawReason = "view_not_visible";
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }

        if (mActiveSurfaceSyncGroup != null) {
            mActiveSurfaceSyncGroup.markSyncReady();
        }
    } else if (cancelAndRedraw) {
        mLastPerformTraversalsSkipDrawReason = cancelDueToPreDrawListener
            ? "predraw_" + mAttachInfo.mTreeObserver.getLastDispatchOnPreDrawCanceledReason()
            : "cancel_" + cancelReason;
        // Try again
        scheduleTraversals();
    } else {
      	//过渡动画？？？
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
      	//performDraw执行绘制
        if (!performDraw() && mActiveSurfaceSyncGroup != null) {
            mActiveSurfaceSyncGroup.markSyncReady();
        }
    }
  ...
}
```

#### measureHierarchy@ViewRootImpl

```java
----> measureHierarchy@ViewRootImpl:

private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight,
        boolean forRootSizeOnly) {
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;

    boolean goodMeasure = false;
  	//宽度设置为WRAP_CONTENT时
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
        final DisplayMetrics packageMetrics = res.getDisplayMetrics();
        res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
      	//得到dialog的默认宽度
        int baseSize = 0;
        if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
            baseSize = (int)mTmpValue.getDimension(packageMetrics);
        }
        //期望宽度大于默认宽度时
        if (baseSize != 0 && desiredWindowWidth > baseSize) {
          	//确定对子View的测量要求MeasureSpec
          	//宽度进行多次测量决定，高度是想要多少是多少？？？
            childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width, lp.privateFlags);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height,
                    lp.privateFlags);
          	//执行测量
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            //测量完后，宽度没有MEASURED_STATE_TOO_SMALL标识，测量有效goodMeasure = true
            if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                goodMeasure = true;
            } else {
              	//baseSize没有达到要求，扩大baseSize
                baseSize = (baseSize+desiredWindowWidth)/2;
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width, lp.privateFlags);
              	//再次测量
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
              	//再次判断测量是否有效
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    if (DEBUG_DIALOG) Log.v(mTag, "Good!");
                    goodMeasure = true;
                }
            }
        }
    }
		//如果宽度设置为MARCH_PARENT
  	//或者WRAP_CONTENT时前两次测量都没有达到要求
    if (!goodMeasure) {
      	//宽度给你期望的尺寸，再次测量
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width,
                lp.privateFlags);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height,
                lp.privateFlags);
        if (!forRootSizeOnly || !setMeasuredRootSizeFromSpec(
                childWidthMeasureSpec, childHeightMeasureSpec)) {
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        } else {
            // We already know how big the window should be before measuring the views.
            // We can measure the views before laying out them. This is to avoid unnecessary
            // measure.
            mViewMeasureDeferred = true;
        }
      	//窗口宽高和测量宽高不一样，windowSizeMayChange设置为true
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
		
  	//返回尺寸是否改变
    return windowSizeMayChange;
}
```

```java
----> getRootMeasureSpec@ViewRootImpl:

//确定子View的measureSpec
//MATCH_PARENT -> EXACTLY + windowSize
//WRAP_CONTENT -> AT_MOST + windowSize
//else -> EXACTLY + measurement（自己定的尺寸）
private static int getRootMeasureSpec(int windowSize, int measurement, int privateFlags) {
    int measureSpec;
    final int rootDimension = (privateFlags & PRIVATE_FLAG_LAYOUT_SIZE_EXTENDED_BY_CUTOUT) != 0
            ? MATCH_PARENT : measurement;
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
    }
    return measureSpec;
}
```

#### performMeasure@ViewRootImpl：

```java
----> performMeasure@ViewRootImpl:

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
      	//mView == DecorView自我测量
      	//measure方法为整体的调度方法，具体的测量逻辑在对应的onMeasure方法中
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
  	//得到DecorView测量后的结果
    mMeasuredWidth = mView.getMeasuredWidth();
    mMeasuredHeight = mView.getMeasuredHeight();
    mViewMeasureDeferred = false;
}
```

performLayout@ViewRootImpl：

```java
----> performLayout@ViewRootImpl:

private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mScrollMayChange = true;
    mInLayout = true;
		
    final View host = mView;
		
    try {
      	//DecorView的layout调用，传入测量的宽高。
      	//在onLayout方法中还需要调用子view的layout方法对子View布局
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

        mInLayout = false;
        ...
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

#### performDraw@ViewRootImpl

```java
----> performDraw@ViewRootImpl:

private boolean performDraw() {
    ...

    try {
      	//draw方法调用
        boolean canUseAsync = draw(fullRedrawNeeded, usingAsyncReport && mSyncBuffer);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

   	...
    return true;
}
```

#### draw@ViewRootImpl

```java
----> draw@ViewRootImpl:

private boolean draw(boolean fullRedrawNeeded, boolean forceDraw) {
    ...

    boolean useAsyncReport = false;
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
      	//硬件加速情况下
        if (isHardwareEnabled()) {
            ...
          	//硬件绘制
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            ...
						//软件绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
    return useAsyncReport;
}
```
