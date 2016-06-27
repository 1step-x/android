# ActivityManagerService(AMS)

ActivityManagerService(AMS)在system_server进程中启动，首先会创建一个系统级的system context，建立好上下文。然后加载相应的provider。在setSystemProcess中将`activity`服务加入到`service manager`中，方便其他进程获取AMS，并且创建`system_server`的进程管理对象。最后调用setSystemReady，启动AMS服务，创建系统桌面。


# Activity启动流程(以Home为例)

![HomeActivity](https://raw.githubusercontent.com/1step-x/android/master/Android%E6%96%87%E6%A1%A3%E6%80%BB%E7%BB%93%E5%BD%92%E7%BA%B3/ActivityManagerService/HomeActivity.png)

`4.startActivityLocked`会创建一个ActivityRecord，记录的是Home activity的信息。注意在ASS的setWindowManager函数被调用时会创建一个ActivityStack，并且将stack id设置 为`HOME_STACK_ID = 0`。

`5.startActivityUncheckedLocked`是一个关键函数，在这个函数中通过activity的标志位，找到或者新建合适的stack和task。

`11.resumeTopActivityInnerLocked`这个函数会和`resumeTopActivityLocked`构成一个迭代循环，需要ASS的inResumeTopActivity标志位来控制是否退出循环。



startProcessLocked会创建一个进程，通过socket向zygote通过fork出一个子进程。fork后的子进程执行的第一个函数是ActivityThread的main函数，它是一个静态函数。
在main函数里，会创建一个ActivityThread对象，并且调用attach函数。(getDefault返回的是gDefault成员，它这是一个模板类对象，实际上得到的是AMS的IBinder对象)
```
    final IActivityManager mgr = ActivityManagerNative.getDefault(); //得到activity服务的客户端
    try {
        mgr.attachApplication(mAppThread); //关键点，mAppThread是一个ApplicationThread对象,它是应用程序的。
    } catch (RemoteException ex) {
         // Ignore
    }
```

最终调用到的是ActivityManagerProxy的attachApplication方法。ActivityManagerProxy是一个代理类，用于应用程序和AMS通信。

```
public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
注意看它传输的数据，一个是descriptor =“android.app.IActivityManager”,另一个是binder对象本身。在AMS服务端，因为AMS是继承自ActivityManagerNative的，所以会调用AMN的onTransact方法，并且走到下面的分支：
```
case ATTACH_APPLICATION_TRANSACTION: {
    data.enforceInterface(IActivityManager.descriptor);
    IApplicationThread app = ApplicationThreadNative.asInterface(
            data.readStrongBinder()); //注意此处读到的应用程序的binder对象，获得的app对象也属于应用端。
    if (app != null) {
        attachApplication(app);
    }
    reply.writeNoException();
    return true;
        }
```

最终调用到AMS的`attachApplicationLocked(IApplicationThread thread,int pid)`，其中thread是应用端的对象，pid是应用端进程的pid。attachApplicationLocked函数首先通过pid查找ProcessRecord(因为之前创建过，所以一定会存在)，并且将应用端的thread对象赋值给process的成员变量thread，这样由AMS管理的processRecord就可以通过IApplictionThread和对应的进程通信了。然后调用thread.bindApplication函数，这个函数主要是初始化应用线程的一些变量。最后会通知应用进程启动Activity和Service等组件，其中启动Activity的函数是调用ASS的attachApplicationLocked函数，并最终会调用到realStartActivityLocked函数。

```
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
```
`realStartActivityLocked`函数中会调用`app.thread.scheduleLaunchActivity`，通知应用进程lauchActivity，之前说过thread是属于应用进程，因此最终会走到ActivityThread的scheduleLaunchActivity函数。
```
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    ... //初始化r的一些信息，这些信息完全都是从参数中获得的。ActivityClientRecord可以看成是ActivityRecord的一个镜像。

    sendMessage(H.LAUNCH_ACTIVITY, r);
}

```
然后调用向主线程发送LAUCH_ACTIVITY消息，调用ActivityThread的handleLaunchActivity方法,该方法中有两个比较重要的函数，一个是performLaunchActivity，利用反射机制创建目标Activity，完成Activity生命周期的onCreate函数。另一个是handleResumeActivity，会调用Activity生命周期的onStart和onResume函数。


# ActivityContainer、ActivityStack、TaskRecord、AcitivtyRecord 四者的关系：
ActivityStack管理所有的activity，但它并不是直接操作ActivityRecord，而是通过TaskRecord对象。每个ActivityStack都有唯一的stackId，并且有一个`ArrayList<TaskRecod>`类型的数组，该数组就是就是该 ActivityStack下的所有Task。一般来讲对于只有一个显示屏的系统而言，只有两个ActivityStack，一个是home acitivtystack，它有两个TaskRecord，分别管理launcher和 recent activities。另外一个是app activitystack，管理所有其他应用的TaskRecord。每个TaskRecord从用户角度看就是一个"应用"，用户打开一个app，然后在此app"内部"打开的activity都在此task中(一般而言)，每个task都有一个taskID标识唯一的task。

ActivityStack 并不是直接在ASS中新建，而是作为ActivityContainer的成员，包含在ActivityContainer中， 在ActivityContainer的构造函数中新建。

**导出所有的activity关系**

>dumpsys activity activities  可以导出所有的activity的结构关系