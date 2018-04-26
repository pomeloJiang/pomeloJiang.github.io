---
title: Android Framework之Activity启动流程(三)
urlname: android_framework_start_activity_3
categories:
- android
tags: [Frameworks,Android]
---
各位看官好，本文是Android Framework之Activity启动流程的第三篇，本篇将分析Activity生命周期的回调，新世界的大门就在眼前,走起。

第一篇：[Android Framework之Activity启动流程(一)](https://pomelojiang.github.io/android_framework_start_activity_1)
第二篇：[Android Framework之Activity启动流程(二)](https://pomelojiang.github.io/android_framework_start_activity_2)

执行完 **ApplicationThread# handleBindApplication ()** 之后，这时候新进程已经启动。
回到AMS# attachApplicationLocked(),代码逻辑来到了上篇文章的注释5处，调用了ActivityStackSupervisor# attachApplicationLocked,在里面又执行了realStartActivityLocked(),这样就回到了第一篇末尾所说到的第二种情况，接下来分析第二种情况。
<!-- more -->


## ApplicationThread#scheduleLaunchActivity()
```java
public final void scheduleLaunchActivity(/*参数省略*/){
	//……省略代码
	sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
sendMessage内部使用了Hanlder，发送了一个H.LAUNCH_ACTIVITY类型的消息，然后调用handleLaunchActivity()->performLaunchActivity()方法。

## ActivityThread#handleLaunchActivity()
看看这两个方法到底干了什么事。
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // 当前进程正活跃，避免GC
    unscheduleGcIdler();
    //确保使用的是最近的配置
    handleConfigurationChanged(null, null);
    //在创建Activity之前初始化WindowManagerService
    if (!ThreadedRenderer.sRendererDisabled) {
        GraphicsEnvironment.earlyInitEGL()
    }
    WindowManagerGlobal.initialize();
    //======== 注释9 ========
    //执行performLaunchActivity(),并返回Activity对象
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        //======== 注释10=========
        //启动成功后，恢复Activity
        handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        //……省略代码
    }
}
```
参数列表中的ActivityClientRecord可以看做是一个Activity的实例，同时还包括了其他一些属性，主要是对Activity状态进行记录。
**handleLaunchActivity()** 主要有两个操作：
1.注释9:performLaunchActivity()
该方法主要是调用Activity的onCreate(),onStart(),onRestoreInstance(),onPostCreate()生命周期
2.注释10:handleResumeActivity()
该方法会调用Activity的onResume()生命周期


## ActivityThread#performLaunchActivity()
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        //从PackageManagerService获取应用包信息
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ComponentName component = r.intent.getComponent();
    if (component == null) {
        //获取组件信息
        component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }
    //如果提前设置好了目标Activity，则重新设置组件信息
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        //利用ClassLoader去加载Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //利用Instrumentation创建Activity实例
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        //……省略代码
    }
    try{
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if (activity != null) {
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,r.embeddedID, r.lastNonConfigurationInstances, config,r.referrer, r.voiceInteractor, window, r.configCallback);
        }
        ……
        //回调Activity的onCreate()方法
        //这里回调的重载函数由ActivityInfo的persistableMode参数决定
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        r.activity = activity;
        r.stopped = true;
        if (!r.activity.mFinished) {
            //回调Activity的onStart()方法，同时会改变FragmentManager的状态信息
            // mInstrumentation.callActivityOnStart(this);
            activity.performStart();
            r.stopped = false;
        }

        //回调Activity的onRestoreInstanceState()方法
        //这里的回调方法同样由ActivityInfo的persistableMode参数决定
        if (!r.activity.mFinished) {
            if (r.isPersistable()) {
                if (r.state != null || r.persistentState != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
                }
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        //回调Activity的OnPostCreate()方法
        if (!r.activity.mFinished) {
            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnPostCreate(activity, r.state,
                        r.persistentState);
            } else {
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                                " did not call through to super.onPostCreate()");
            }
        }
        r.paused = true;
        mActivities.put(r.token, r);
    } catch(SuperNotCalledException e){
        ……
    }
    return activity;
}
```

## ActivityThread#handleResumeActivity()
```java
final void handleResumeActivity(IBinder token,
                                boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    //======== 注释11=========
    //回调Activity的onResume()方法
    r = performResumeActivity(token, clearHide, reason);
    if (r != null) {
        final Activity a = r.activity;
        final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
        //显示window
        if (r.window == null && !a.mFinished && willBeVisible) {
	        ……
            ViewManager wm = a.getWindowManager();
	        ……
            //将decorView添加到WindowManager中
            wm.addView(decor, l);
        }
        ……
        //更新布局
        wm.updateViewLayout(decor, l);
        ……
        if (reallyResume) {
            try {
                //通知AMS已经Resume了
                ActivityManager.getService().activityResumed(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } else {
            try {
                //如果在onResume之前抛出异常了,则通知AMS结束该Activity
                ActivityManager.getService()
                        .finishActivity(token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            }
        }
    }
}
```
该方法主要是回调Activity的onResume()方法(实际上是调用Instrumentation.callActivityOnResume(this)来执行)，并将DecorView添加到WindowManager中，这里的WindowManager是a.getWindowManager()得到的，其实现是WindowManagerImpl，这步操作在onResume()方法之后执行。


**至此，Activity的启动流程分析完毕。**
看完之后感觉如何，是否恍然大悟？![全TM是套路](https://img-blog.csdn.net/20180412213610531?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3VuY2xlUG9tZWxv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


</br>
</br>
## 总结
1、	startActivity()通过Instrumentation向AMS进程发起startActivity()请求
2、	AMS收到启动请求后由ActivityStarter处理Flags和Intent信息,然后再由ActivityStackSupervisor和ActivityStack处理Task和Stack流程
3、	在ActivityStackSupervisor中判断app进程是否存在，若不存在则请求AMS进程创建新进程，app进程存在的情况请直接跳到第7步
4、	AMS进程通过Socket方式请求Zygote fork新进程
5、	在新进程里创建ActivityThread(主线程)并开启Looper循环，同时将ApplicationThread绑定到AMS
6、	AMS回调ApplicationThread的bindApplication()方法将自身与新进程绑定,并在ActivityThread的handleBindApplication()方法中创建应用的Application
7、	Application创建成功后，AMS调用ActivityStackSupervisor# realStartActivityLocked向ActivityThread发送创建Activity请求
8、	ActivityThread利用ClassLoader去加载Activity,并回调其生命周期方法。
