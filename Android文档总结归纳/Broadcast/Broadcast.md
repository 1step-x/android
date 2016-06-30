## Broadcast 与 BroadcastReceiver
广播发送方：
>三种广播发送方式：
> * 普通广播方式，`sendBroadCast`，所有注册了该广播的接收方都可以收到。
> * 有序广播方式，`sendOrderedBroadcast`，按接收者优先级将广播向下派发，接收者可以中断广播继续往下发送。
> * sticky广播方式，`sendStickyBroadcast`，系统存储发送的广播，并且在新的广播接收者注册时，系统把sticky广播发给它。

广播接收者：静态注册与动态注册 
>静态注册是在AdndroidManifest.xml中声明`<receiver>`标签。动态注册是使用`registerReceiver`函数注册，并且可以使用`unregisterReceiver`撤销之前注册的实例。


### 注册广播
注册广播`registerReceiver`函数最终调用的是ContextImpl类的`registerReceiver`函数。函数原型如下：
```
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }
```
从函数原型就可以看到注册`receiver`时需要一个`BroadcastReceiver`对象和一个`IntentFilter`对象，这是最基本的，第一个参数表示接收者，一般我们会重载其`onReceive`函数。第二个参数是表示接收哪些广播，只有符合要求的广播才会被接收。

注册函数最终会走到`registerReceiverInternal`：
```
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();//找到主线程的handler
                }
                rd = mPackageInfo.getReceiverDispatcher( //得到一个ReceiverDispatcher对象
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            return ActivityManagerNative.getDefault().registerReceiver(//向AMS注册receiver
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }
```
上面代码中的`getReceiverDispatcher`创建一个新的`ReceiverDispatcher`对象，并且每个`BroadcastReceiver`和`ReceiverDispatcher`以key-value的形式保存在map中。返回的`rd`是IIntentReceiver类型，实际上接收来自AMS的广播就是通过这个`rd`，后续分析。

`ActivityManagerNative.getDefault().registerReceiver(...)`函数最终会走到AMS的`registerReceiver`函数。
在该函数中主要完成下面几个动作：
1. 从`mStickyBroadcasts`成员中得到符合`action`(注册时的IntentFilter中声明)要求的所有`intent`。
2. 调用filter.match(filter为注册时候的IntentFilter)方法匹配1中找到的所有`intent`，满足要求的保存在`allSticky`中。*为什么在这里又需要再匹配一次？*
3. 从`mRegisteredReceivers`成员中得到属于该`BroadcastReceiver`的`ReceiverList`，并将BroadcastFilter放到了该数组中，这样是因为每个`BroadcastReceiver`可以注册多次，每次注册时可以设置不同的intentfilter。
 ```
    ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
    if (rl == null) {
        rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                userId, receiver);
        if (rl.app != null) {
            rl.app.receivers.add(rl);
        } else {
            try {
                receiver.asBinder().linkToDeath(rl, 0);
            } catch (RemoteException e) {
                return sticky;
            }
            rl.linkedToDeath = true;
        }
        mRegisteredReceivers.put(receiver.asBinder(), rl);
    } 
    ...
    BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
            permission, callingUid, userId);
    rl.add(bf); //将BroadcastFilter放入到ReceiverList数组。
    if (!bf.debugCheck()) {
        Slog.w(TAG, "==> For Dynamic broadcast");
    }
    mReceiverResolver.addFilter(bf);
 ```
 4. 从`allSticky`中取出所有的`intent`,建立`BroadcastRecord`对象记录和本次广播相关的信息。将非order广播加入到`mParallelBroadcasts`中去，并且调用 `scheduleBroadcastsLocked`函数，开始派发广播。

注意：前面已经说过对于sticky广播在广播接收者注册时就会派发出去。

### 派发广播

看最简单的情况，在ContextImpl.java中`sendBroadcast(Intent intent)`函数:
```
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess();
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
    }
```
很明显地，这个函数最终会调用到AMS的`broadcastIntent`函数，`broadcastIntent`函数又会调用到`broadcastIntentLocked`函数。

broadcastIntentLocked函数会执行以下几个步骤：
1. 拦截那些从用户应用发过来的被保护起来的广播，有些广播只能由系统发出，应用程序发送些广播会抛出异常。
2. 处理特殊的广播：比如`Intent.ACTION_UID_REMOVE,Intent.ACTION_PACKAGE_REMOVED,ACTION_PACKAGE_CHANGED,ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE,ACTION_EXTERNAL_APPLICATIONS_AVAILABLE,Intent.ACTION_TIMEZONE_CHANGED`等。
3. 如果是`sticky`类型的广播，则将其添加到`mStickyBroadcasts`中，并且保存起来。(前面我们在注册的时候也会查找`mStickyBroadcasts`中的广播，如果匹配则立即将该广播派发给注册的对象)
4. 决策谁将接收发送的广播，找到所有并将其发送给对应的接收者。它会找到所有匹配的动态和静态注册的广播，在`registeredReceivers`保存的是所有动态接收者，并且会首先处理非order的广播。
5. 如果在4步骤中发现广播是order广播，即有序广播，则将动态注册的广播接收者添加到`receivers`对象中，并且按照优先级排序。然后处理所有`receivers`对象中的广播接收者。
6. 静态接收者对应的广播记录都保存在`mOrderedBroadcasts`对象中，`mParallelBroadcasts`中保存的广播记录对应的都是动态接收者。
7. 调用`scheduleBroadcastsLocked`函数，开始广播处理。
8. 最终会调用到BroadcastQueue的`processNextBroadcast`函数，该函数会首先处理压栈到`mParallelBroadcasts`中的广播接收者，之前说过这部分的广播接收者都是动态注册的，并且是无序的。将此数组中的每个对象都取出来，然后调用 `deliverToRegisteredReceiverLocked`函数派发广播。
```
private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered) {
        boolean skip = false;
        ... //处理各种权限

        if (!skip) {
            // If this is not being sent as an ordered broadcast, then we
            // don't want to touch the fields that keep track of the current
            // state of ordered broadcasts.
            if (ordered) { //如果广播是有序的，则设置一些状态
                r.receiver = filter.receiverList.receiver.asBinder();
                r.curFilter = filter;
                filter.receiverList.curBroadcast = r;
                r.state = BroadcastRecord.CALL_IN_RECEIVE;
                if (filter.receiverList.app != null) {
                    // Bump hosting application to no longer be in background
                    // scheduling class.  Note that we can't do that if there
                    // isn't an app...  but we can only be in that case for
                    // things that directly call the IActivityManager API, which
                    // are already core system stuff so don't matter for this.
                    r.curApp = filter.receiverList.app;
                    filter.receiverList.app.curReceiver = r;
                    mService.updateOomAdjLocked(r.curApp);
                }
            }
            try {
                if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                        "Delivering to " + filter + " : " + r);
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId); //真正的广播派发函数
                if (ordered) {
                    r.state = BroadcastRecord.CALL_DONE_RECEIVE;
                }
            } catch (RemoteException e) {
                ...
                }
            }
        }
    }
```

`performReceiveLocked`函数最终会调用receiver的`performReceive`函数。对于动态注册的广播，如果进程已经启动并且还没有消失，则直接调用到应用进程的scheduleRegisteredReceiver函数。如果是静态注册的广播，则调用IIntentReceiver的performReceive函数。
```
    private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null) {
            if (app.thread != null) { //对于动态注册者，大部分会执行此分支。
                // If we have an app thread, do the call through that so it is
                // correctly ordered with other one-way calls.
                app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                        data, extras, ordered, sticky, sendingUser, app.repProcState);
                        // 执行应用进程的`scheduleRegisteredReceiver`函数
            } else {
                // Application has died. Receiver doesn't exist.
                throw new RemoteException("app.thread must not be null");
            }
        } else { 
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser); // 调用IIntentReceiver的performReceive，
        }
    }
```
9. 继续在`processNextBroadcast`中处理pending的广播，对于有序广播来讲，只有当上一个广播接收者(优先级高)处理了广播之后，低优先级的广播接收者才可能接收到广播，并且这种优先级是跨进程的。
10. 处理超时的广播。
11. 处理该条广播剩下的广播接收者，如果是动态注册的广播接收者则直接调用`deliverToRegisteredReceiverLocked`函数。如果是静态注册的广播接收者并且该对象对应的进程已经存在，则调用`processCurBroadcastLocked`方法，如果该对象对应的进程不存在，则创建该进程。

### 应用程序的广播响应
由前面的分析可知，动态注册的广播接收者最后会调用下面这个方法。
```
    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
        data, extras, ordered, sticky, sendingUser, app.repProcState);
```
很明显，这是应用程序端的方法，继续跟踪此函数，在ActivityThread 类中。
```
    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
            int resultCode, String dataStr, Bundle extras, boolean ordered,
            boolean sticky, int sendingUser, int processState) throws RemoteException {
        updateProcessState(processState, false);
        receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                sticky, sendingUser);
    }
```
首先更新进程状态，然后再执行 `receiver`的`performReceive`函数，这里的`receiver`对象实际上就是我们在注册广播接收者时发送给AMS的，然后在这里又传了回来。

在`receiver`对象的`performReceive`函数中，会调用`LoadedApk.ReceiverDispatcher`的`performReceive`，我们直接看dispatcher的`performReceive`函数。
```
    public void performReceive(Intent intent, int resultCode, String data,
            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
        if (ActivityThread.DEBUG_BROADCAST) {
            int seq = intent.getIntExtra("seq", -1);
            Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction() + " seq=" + seq
                    + " to " + mReceiver);
        }
        Args args = new Args(intent, resultCode, data, extras, ordered,
                sticky, sendingUser); //新建一个Args对象，然后将此对象投递到主线程中执行。
        if (!mActivityThread.post(args)) {
            if (mRegistered && ordered) {
                IActivityManager mgr = ActivityManagerNative.getDefault();
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing sync broadcast to " + mReceiver);
                args.sendFinished(mgr);
            }
        }
    }
```
投递到主线程的Args对象是一个`runnable`对象，广播的接收在其`run`方法中，即调用`receiver`的`onReceive`方法，需要注意的是，这里的receiver是一个`BroadcastReceiver`类型对象，即注册时的BroadcastReceiver对象。

![BroacastReceiver类图](https://raw.githubusercontent.com/1step-x/android/master/Android%E6%96%87%E6%A1%A3%E6%80%BB%E7%BB%93%E5%BD%92%E7%BA%B3/Broadcast/BroadcastReceiver%E7%B1%BB%E5%9B%BE.png)

接收完广播后，会调用finish方法，向AMS发送广播派发结束的消息，AMS根据消息的返回值决定是否再继续向低优先级的广播接收者派发广播。


UID 起始值 10000，终止值 19999