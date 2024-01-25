##### main@ThreadActivity

```java
----> main@ThreadActivity: 

public static void main(String[] args) {
		...
	  //创建主线程Looper
    Looper.prepareMainLooper();

    ...
    //调用activityThread得attach方法，（system == false）
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

  	//主线程handler（final H mH = new H()）
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
		
  	...
    
    //进入loop死循环
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
//main方法中主要是创建Looper，调用attach方法后，进入loop循环读取message进行处理
```

##### attach@ActivithThread

```java
----> attach@ActivithThread: 

private void attach(boolean system, long startSeq) {
		...
    //system = false
    if (!system) {
        ...
        //通过Binder与AMS进行跨进程通信，最终调用客户端ApplicationThread.bindApplication方法
        final IActivityManager mgr = ActivityManager.getService();
        try {
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ...
        });
    } else {
        ...
    }
		...	
}
```

##### bindApplication@ApplicationThread

```java
----> bindApplication@ApplicationThread: ApplicationThread客户端的binder，用来跨进程通信

@Override
public final void bindApplication(String processName, ApplicationInfo appInfo,
        String sdkSandboxClientAppVolumeUuid, String sdkSandboxClientAppPackage,
        ProviderInfoList providerList, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
        String buildSerial, AutofillOptions autofillOptions,
        ContentCaptureOptions contentCaptureOptions, long[] disabledCompatChanges,
        SharedMemory serializedSystemFontMap,
        long startRequestedElapsedTime, long startRequestedUptime) {
    ...
    //发送message到主线程Handler
    sendMessage(H.BIND_APPLICATION, data);
}
```

##### handleMessage@H

```java
----> handleMessage@H：H继承Handler

public void handleMessage(Message msg) {
    switch (msg.what) {
        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData)msg.obj;
        		//调用ActivityThread中的handleBindApplication方法
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
				...
    }
}
```

##### handleBindApplication@ActivityThread:

```java
----> handleBindApplication@ActivityThread:

private void handleBindApplication(AppBindData data) {
    ...
    final InstrumentationInfo ii;
    ...
    //初始化Instrumentation
    if (ii != null) {
        initInstrumentation(ii, data, appContext);
    } else {
        mInstrumentation = new Instrumentation();
      	//传入activityThread保存在字段thread中
        mInstrumentation.basicInit(this);
    }
		...
    Application app;
		...
    try {
        //创建application，data.info = LoadApk（instrumentation == null）
        app = data.info.makeApplicationInner(data.restrictedBackupMode, null);
				...
        //调用application中的onCreate方法
        try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                  "Unable to create application " + app.getClass().getName()
                  + ": " + e.toString(), e);
            }
        }
    } finally {
        // If the app targets < O-MR1, or doesn't change the thread policy
        // during startup, clobber the policy to maintain behavior of b/36951662
        if (data.appInfo.targetSdkVersion < Build.VERSION_CODES.O_MR1
                || StrictMode.getThreadPolicy().equals(writesAllowedPolicy)) {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }

    // Preload fonts resources（提前加载字体资源）
    FontsContract.setApplicationContextForResources(appContext);
    if (!Process.isIsolated()) {
        try {
            final ApplicationInfo info =
                    getPackageManager().getApplicationInfo(
                            data.appInfo.packageName,
                            PackageManager.GET_META_DATA /*flags*/,
                            UserHandle.myUserId());
            if (info.metaData != null) {
                final int preloadedFontsResource = info.metaData.getInt(
                        ApplicationInfo.METADATA_PRELOADED_FONTS, 0);
                if (preloadedFontsResource != 0) {
                    data.info.getResources().preloadFonts(preloadedFontsResource);
                }
            }
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    try {
      	//通知AMS绑定application结束
        mgr.finishAttachApplication(mStartSeq);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```

##### makeApplicationInner@LoadApk: 

```java
----> makeApplicationInner@LoadApk: 

//instrumentation == null
private Application makeApplicationInner(boolean forceDefaultAppClass,
            Instrumentation instrumentation, boolean allowDuplicateInstances) {
  	//如果application已经存在直接返回
    if (mApplication != null) {
        return mApplication;
    }
    ...
    Application app = null;

  	//进程名
    final String myProcessName = Process.myProcessName();
  	//自定义application的类名
    String appClass = mApplicationInfo.getCustomApplicationClassNameForProcess(
            myProcessName);
    if (forceDefaultAppClass || (appClass == null)) {
      	//没有使用默认的application
        appClass = "android.app.Application";
    }

    try {
        final java.lang.ClassLoader cl = getClassLoader();
        ...
				//application上下文创建
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        ...
        //通过反射获取application
        //会回调application中的attach方法，继而调用attachBaseContext方法
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        ...
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + " package " + mPackageName + ": " + e.toString(), e);
        }
    }
  	//保存到mAllApplications集合中
    mActivityThread.mAllApplications.add(app);
  	//赋值给mApplication，避免重复创建
    mApplication = app;
    ...
    //不会执行
    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!instrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    }

    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    return app;
}
```
