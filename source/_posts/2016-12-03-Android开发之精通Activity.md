---
title: Android开发之从源码深入Activity
tags: Android
grammar_cjkRuby: true
catalog: true
layout:  post
categories: Android
date: 2016-12-03
---

很多人提到Activity就知道其7大生命周期，以及各个方法的使用，但是Activity到底是怎么工作的呢？
本篇文章带你学习Activity到底是什么。

----------


### Activity相关Framework类

- Context是什么？

  Context在我们的[Android开发之深入Context][1]文章中已经介绍过

- ActivityThread是什么？

ActivityThread不是一个线程，是应用的主线程，在Activity中有个

>  ActivityThread mMainThread;

ActivityThread有个main方法

``` java
     public static void main(String[] args) {
        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);    //与AMS进行通信，交互
        ...
        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();   //轮询消息队列
    }
```

- ApplicationThread？

负责与AMS进行通信的一个帮助类<br>
在ActivityThread中作为一个成员变量进行初始化

``` java
final ApplicationThread mAppThread = new ApplicationThread();
```

- ActivityClientRecord

一个记录类，客户端负责记录Activity的各种信息

- ActivityRecord
ActivityManagerService端负责记录每个注册过来的Activity的信息

---------- 

### Activity与其他类的区别

Activity作为整体框架，控制界面，有其生命周期，但是其生命周期的回调等都是由Framework控制
如果从本质上看，与普通的类没什么区别

----------

### Activity的生命方法是什么时候回调的

关键点：
1.Instrumentation （生命周期的管理，还可以启动Activity，创建Application）<br>
2.ActivityManagerNative AMS的远程代理<br>
3.ActivityThread中的final H mH = new H();这个handler类<br>

``` java
public final class ActivityThread {    
    private class H extends Handler {
        ...
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                /*拿到启动avtivity的请求*/
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
                case RESUME_ACTIVITY:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
                    SomeArgs args = (SomeArgs) msg.obj;
                    handleResumeActivity((IBinder) args.arg1, true, args.argi1 != 0, true,
                            args.argi3, "RESUME_ACTIVITY");
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ...
        }
    }
        
        
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
        Activity a = performLaunchActivity(r, customIntent);
    ...
    }
    
    
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
        
        /*创建application，整个应用程序就这个地方创建app 
        * 里面会调用LoadedApk的  public Application makeApplication(boolean forceDefaultAppClass,Instrumentation instrumentation)
        ---->instrumentation.callApplicationOnCreate(app);
        ---->mActivityThread.mAllApplications.add(app);
        回调application的oncreate方法*/
    
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        
        /*使用classloader创建class*/
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        
        /*使得activity与windows对象进行关联 同时对Activity初始化相关的内容
          比如：ContextImpl，Theme，Configuration，title*/
        activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);
        
        /*让mInstrumentation回掉自己的oncreate*/
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
  
    }
    
```

从上面我们看出Instrumentation是负责回掉activity生命周期的工具类，而ActivityThread是控制着这些执行的类。<br>

#### 其实代码继续跟下去，最终会发现，我们的应用无非是在由前台的ActivityThread，ContextImpl，后台的ActivityManagerService，想要显示到界面上最后都是拿到WindowManager设置参数后addView来干的。




 ----------
 ### 谢谢大家阅读，如有帮助，来个喜欢或者关注吧！

 ----------
 本文作者：Anderson/Jerey_Jobs

 简书地址：[Anderson大码渣][2]

 github地址：[Jerey_Jobs][3]


  [1]: http://www.jianshu.com/p/25613ae8a88e
  [2]: http://www.jianshu.com/users/016a5ba708a0/latest_articles
  [3]: https://github.com/Jerey-Jobs