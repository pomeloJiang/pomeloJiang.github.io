---
title: Android Framework之Activity启动流程(二)
urlname: android_framework_start_activity_2
categories:
- android
tags: [Frameworks,Android]
---

各位看官好，本文是Android Framework之Activity启动流程的第二篇，接下来将为大家带来开启Activity进程的流程。

第一篇:[Android Framework之Activity启动流程(一)](https://pomelojiang.github.io/android_framework_start_activity_1)
第三篇：[Android Framework之Activity启动流程(三)](https://pomelojiang.github.io/android_framework_start_activity_3)
</br>
</br>
<!-- more -->
## ActivityManagerService#startProcessLocked
这里又回到了ActivityManagerService。
我们先来看一下**ActivityManagerService# startProcessLocked()**
在ActivityStackSupervisor里调用的其实是另外一个startProcessLocked()方法，在这个方法里又调用了它的重载方法。所以我们直接来看看它的重载方法做了什么事。 
```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
                                       boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName, boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge, String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    long startTime = SystemClock.elapsedRealtime();
    ProcessRecord app;
    if (!isolated) {
        //根据进程名和UID查找相应的ProcessRecord，
        //当第一次启动app时这里返回值为null
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
        checkTime(startTime, "startProcess: after getProcessRecord");

        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            //如果当前进程处于后台进程，检查当前进程是否为bad进程
            if (mAppErrors.isBadProcessLocked(info)) {
                return null;
            }
        } else {
            //当用户明确要启动一个进程时，则清空它的crash次数
            //在看见crash对话框之前它才不会成为一个bad进程
            mAppErrors.resetProcessCrashTimeLocked(info);
            if (mAppErrors.isBadProcessLocked(info)) {
                EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                        UserHandle.getUserId(info.uid), info.uid,
                        info.processName);
                mAppErrors.clearBadProcessLocked(info);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        //如果它是一个孤立的进程，则它无法使用现存的进程
        app = null;
    }

    //当已经存在ProcessRecord且其pid大于0(app早已经运行或者正在启动)
    //则不会清理该进程
    if (app != null && app.pid > 0) {
        if ((!knownToBeDead && !app.killed) || app.thread == null) {
            // 如果它是新的包，则将其添加到列表中
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            return app;
        }

        //当ProcessRecord被attach到之前的进程，就清理它
        killProcessGroup(app.uid, app.pid);
        handleAppDiedLocked(app, true, true);
    }

    String hostingNameStr = hostingName != null
            ? hostingName.flattenToShortString() : null;
    if (app == null) {
        checkTime(startTime, "startProcess: creating new process record");
        //根据ApplicationInfo、processName、UID创建一个ProcessRecord对象
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
        if (app == null) {
            return null;
        }
        app.crashHandler = crashHandler;
        checkTime(startTime, "startProcess: done creating new process record");
    } else {
        //如果在进程中它是新的一个包，则添加它到列表里
        app.addPackage(info.packageName, info.versionCode, mProcessStats);
        checkTime(startTime, "startProcess: added package to existing proc");
    }

    //如果系统仍未准备好，则推迟启动它，将app加入hold列表
    if (!mProcessesReady
            && !isAllowedWhileBooting(info)
            && !allowWhileBooting) {
        if (!mProcessesOnHold.contains(app)) {
            mProcessesOnHold.add(app);
        }
        checkTime(startTime, "startProcess: returning with proc on hold");
        return app;
    }

    checkTime(startTime, "startProcess: stepping in to startProcess");
    //======== 注释2 ========调用重载方法启动进程
    startProcessLocked(app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
    return (app.pid != 0) ? app : null;
}
```
这个方法主要是处理一些情况下ProcessRecord中的数据，创建新进程的代码在它的重载方法里，继续分析。


## ActivityManagerService#startProcessLocked()
见注释2处
```java
private final void startProcessLocked(ProcessRecord app, String hostingType, String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    //如果ProcessRecord的pid>0且不为当前进程的pid
    //就从mPidsSelfLocked移除该pid
    //当进程不存在时，pid=0
    if (app.pid > 0 && app.pid != MY_PID) {
        synchronized (mPidsSelfLocked) {
            mPidsSelfLocked.remove(app.pid);
            mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        }
        app.setPid(0);
    }
    //从hold列表移除该ProcessRecord
    mProcessesOnHold.remove(app);
    //更新Cpu状态
    updateCpuStats();
    try {
        try {
            final int userId = UserHandle.getUserId(app.uid);
            //通过PMS检查待启动进程对应的package是否满足启动条件		AppGlobals.getPackageManager().checkPackageStartable(app.info.packageNam     e, userId);
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }
        //
        if (!app.isolated) {
            int[] permGids = null;
            try {
                checkTime(startTime, "startProcess: getting gids from package manager");
                final IPackageManager pm = AppGlobals.getPackageManager();
                //得到对应的GID
                permGids = pm.getPackageGids(app.info.packageName,
                        MATCH_DEBUG_TRIAGED_MISSING, app.userId);
                StorageManagerInternal storageManagerInternal = LocalServices.getService(
                        StorageManagerInternal.class);
                //获得进程对外部存储的访问模式
                mountExternal = storageManagerInternal.getExternalStorageMountMode(uid,
                        app.info.packageName);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

              ……
            //不同情况下设置debugFlags的值，具体的值请看Zygote类的static属性
             ……
            boolean isActivityProcess = (entryPoint == null);
            //当entryPoint为空的情况下，设置它的值
            //这里的entryPoint是第一个startProcessLocked()传进来的null值
            //这里是指定反射需要的className
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";

            ProcessStartResult startResult;
            if (hostingType.equals("webview_service")) {
              ……
            } else {
                //开启新进程的真正代码
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, debugFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith, entryPointArgs);
            }
        }
        ……
        // 启动进程后更新ProcessRecord
        app.setPid(startResult.pid);
        app.usingWrapper = startResult.usingWrapper;
        app.removed = false;
        app.killed = false;
        app.killedByAm = false;

        synchronized (mPidsSelfLocked) {
            //将启动结果的pid和PreocessRecord添加到mPidsSelfLocked
            this.mPidsSelfLocked.put(startResult.pid, app);
            if (isActivityProcess) {
                //发送一个延时消息
                // PROC_START_TIMEOUT 值为 10
                //在消息未被处理前，若新创建的进程没有和AMS交互，则该进程启动失败
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        } catch(RuntimeException e){
            //当创建进程出现异常的时候就会清理相关的记录forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid), false,    false, true, false, false, UserHandle.getUserId(app.userId), "start failure");
        }
    }
}
```
该方法里主要干了三件事：
1.设置了各种debug参数，若AndroidManifest.xml将android:debuggable设置为true，这些参数就会生效。
2.通过Process.start()开启一个新进程，实际上是通过Socket与Zygote通信，使用Zygote fork新进		   	程，同时将ActivityThread类加入到新进程,并调用ActivityThread.main()。
3.发送一条延时消息，若新创建的进程在消息接收前未与AMS交互，则进程启动失败
</br>
</br>
## ActivityThread、ApplicationThread
当Zygote启动进程时，传入的className为android.app.ActivityThread，因此，当Zygote反射调用进程的main()函数时，ActivityThread的main()将被启动。
ActivityThread是应用程序的真正入口，在其main()方法里会使用Looper. myLooper()开启消息循坏队列，主线程也在这里创建。
frameworks/base/core/java/android/app/ActivityThread.java
**ActivityThread#main()**
```java
public static void main(String[] args) {
    ……
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    //将应用进程绑定到ActivityManagerService
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    Looper.loop();
}
```
我们可以看到main()方法里先是准备好Looper和消息队列,然后调用ActivityThread.attach方法将应用进程绑定到ActivityManagerService()，然后调用Looper.loop()循环不断读取并分发消息队列的消息。

attach()的主要代码如下所示:
**ActivityThread#attach()**
```java
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    ……
    final IActivityManager mgr = ActivityManager.getService();
    try {
        //======== 注释4 ========
        mgr.attachApplication(mAppThread);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    ……
}
```
我们可以看到注释4处调用了ActivityManagerService的attachApplication()方法，将ApplicationThread绑定到AMS,这样AMS就可以通过ApplicationThread控制应用进程。
**ActivityManagerService# attachApplication()**
**ActivityManagerService# attachApplicationLocked()**
```java
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
    }
}
```
因为attachApplication里又调用了attachApplicationLocked()，所以直接看该方法。
```java
private final boolean attachApplicationLocked(IApplicationThread thread,
                                              int pid) {
    ProcessRecord app;
    if (id != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            //根绝pid获得PrecessRecord
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }

    //没有找到pid对应的ProcessRecord，就杀死该进程
    if (app == null) {
        ……
        if (pid > 0 && pid != MY_PID) {
            killProcessQuiet(pid);
        } else {
            try {
                //关闭新进程的Looper
                thread.scheduleExit();
            } catch (Exception e) {
                // Ignore exceptions.
            }
        }
        return false;
    }
    //如果PreocessRecord的ApplicationThread不为null，则表示它还绑定在之前的进程//中，需要清空
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }
    //新进程的名字
    final String processName = app.processName;

    try {
        //在这个地方会注册该进程的死亡回调
        //这里的Thread指的是ApplicationThread
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        //出现异常则重新开启一个进程
        startProcessLocked(app, "link fail", processName);
        return false;
    }
    //重置ProcessRecord
    app.makeActive(thread, mProcessStats); //给ProcessRecord的thread赋值
    app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
    app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
    app.forcingToImportant = null;
    updateProcessForegroundLocked(app, false, false);
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;
    app.killedByAm = false;
    app.killed = false;
    ……
    //移除startProcessLocked()中发出的延时消息
    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ?
            generateApplicationProvidersLocked(app) : null;
    //如果该进程存在正在启动的ContentProvider，则发送一个延时消息
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
    }
    ……
    //这里通过AIDL调用了ApplicationThread. bindApplication方法，
    //这里是将新进程的ApplicationThread对象绑定到AMS的真正操作
    //两个方法只是参数不同
    // app.instr 为ProcessRecord.ActiveInstrumentation对象
    if (app.instr != null) {
        thread.bindApplication(/*参数省略*/);
    }else{
        thread.bindApplication(/*参数省略*/);
    }
    //更新进程情况
    updateLruProcessLocked(app, false, null);
    //将ProcessRecord从正在启动列表和hold列表中移除
    mPersistentStartingProcesses.remove(app);
    mProcessesOnHold.remove(app);

    //检查最顶层可见的Activity是否等待运行在该进程中
    if (normalMode) {
        try {
            //======== 注释5 ========
            //调用ActivityStackSupervisor# attachApplicationLocked
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
    //查找所有可运行在该进程中的服务
    //检查这个进程中是否有下一个广播接收者
    //检查这个进程中是否有下一个备份代理
    //上述操作如果出现异常就杀死进程
    ……
}
```
该方法主要做了如下操作：
1.	重置ProcessRecord信息
2.	将进程的ApplicationThread绑定到AMS
3.	处理Activity、Service、Broadcast等存在的情况


## ApplicationThread#bindApplication()
```java
public final void bindApplication(/*省略参数*/){
	……
	sendMessage(H.BIND_APPLICATION, data);
}

```
在该方法里向Handler发送了一个类型为H.BIND_APPLICATION的消息，随后在Handler中调用了**ActivityThread#handleBindApplication(data);**

## ActivityThread#handleBindApplication(data);
```java
private void handleBindApplication(AppBindData data) {
    //注册UI线程到VMRuntime作为一个敏感线程
    VMRuntime.registerSensitiveThread();
    //设置进程的启动时间
    Process.setStartTimes(SystemClock.elapsedRealtime(), SystemClock.uptimeMillis());

    ……
    //设置进程名
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName, UserHandle.myUserId());
    //当app版本<= 3.1 时，设置AsyncTask使用线程池实现
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }
    //重置时区（跟随系统时区）
    TimeZone.setDefault(null);
    LocaleList.setDefault(data.config.getLocales());

    //更新系统配置
    synchronized (mResourcesManager) {
        mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;
        applyCompatConfiguration(mCurDefaultDisplayDpi);
    }
    //======== 注释6 ========  获得LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    //判断是否需要为进程设置新的分辨率密度
    if ((data.appInfo.flags& ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
            == 0) {
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    updateDefaultDensity();

    //======== 注释7 ========   StrictMode
    //表示只为系统应用(FLAG_SYSTEM, FLAG_UPDATED_SYSTEM_APP)开启了
    //StrictMode，其他应用还是需要自行开启
    if ((data.appInfo.flags &
            (ApplicationInfo.FLAG_SYSTEM |
                    ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
        StrictMode.conditionallyEnableDebugLogging();
    }
    //当api>=HONEYCOMB时,Android不允许在主线程中使用网络
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB) {
        StrictMode.enableDeathOnNetwork();
    }
    // Android 7.0以后，Android引入了FileProvider
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
        StrictMode.enableDeathOnFileUriExposure();
    }

    ……
    final InstrumentationInfo ii;
    // Instrumentation信息会影响类加载器,所以应该在设置app context之前加载它
    if (data.instrumentationName != null) {
        try {
            ii = new ApplicationPackageManager(null, getPackageManager())
                    .getInstrumentationInfo(data.instrumentationName, 0);
        } catch (PackageManager.NameNotFoundException e) {
            ……
        }
        //设置InstrumentationInfo信息
        ……
    }else{
        ii = null;
    }
    //在这里创建了ContextImpl对象
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    updateLocaleListFromAppContext(appContext,
            mResourcesManager.getConfiguration().getLocales());

    //继续加载Instrumentation对象
    if (ii != null) {
        ……
        try {
            final ClassLoader cl = instrContext.getClassLoader();
            //创建Instrumentation对象
            // 之前提到，Instrumentation的作用就是监控系统和应用的交互，
            // 因此Activity的生命周期也会被Instrumentation所监控
            mInstrumentation =(Instrumentation)cl.loadClass(
                    data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            ……
        }
        final ComponentName component = new ComponentName(ii.packageName, ii.name);
        //初始化Instrumentation参数
        mInstrumentation.init(this, instrContext, appContext, component,
                data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
        ……
    } else {
        mInstrumentation = new Instrumentation();
    }

    Application app;
    try{
        //======== 注释8 ========
        app = data.info.makeApplication(data.restrictedBackupMode, null);
        //设置ActivityThread.mInitialApplication
        mInitialApplication = app;
        ……
        try {
            //这里会调用到Application的onCreate()方法
            mInstrumentation.callApplicationOnCreate(app);
        } catch(Exception e){
            ……
        }
    }
}
```
注释6:返回LoadedApk对象
注释7:关于StrictMode，这里就不再进行分析。给一个链接：[Android性能调优利器StrictMode](https://droidyue.com/blog/2015/09/26/android-tuning-tool-strictmode/)
注释8:这里的data.info对象是注释6处返回的LoadedApk对象

frameworks/base/core/java/android/app/LoadedApk.java
## LoadedApk#makeApplication()
```java
public Application makeApplication(boolean forceDefaultAppClass,
                                   Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }
    Application app = null;
    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }
    try {
        if (!mPackageName.equals("android")) {
            设置Thread.getContextClassLoader()的值
            initializeJavaContextClassLoader();
        }
        //创建ContextImpl对象
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        //创建Application并将ContextImpl与Application绑定
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
    } catch (Exception e) {
	    ……
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;
    ……
    //重写APK library中的R常量
    SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }
        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }
    return app;
}
```
makeApplication所干的事主要是创建一个ContextImpl对象并将其与Application绑定，最后返回刚创建的Application对象。
**ActivityThread#handleBindApplication()**方法到此结束,至此Application对象创建完成。


下一篇将为您带来Activity的生命周期回调。
下一篇：[Android Framework之Activity启动流程(三)](https://pomelojiang.github.io/android_framework_start_activity_3)

</br>
参考：
https://blog.csdn.net/yangwen123/article/details/17261339
https://blog.csdn.net/gaugamela/article/details/53183216
