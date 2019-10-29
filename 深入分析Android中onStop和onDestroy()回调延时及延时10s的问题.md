### 一、起因
1. 很久以前接手的一个比较老的项目中，是使用Activity的名字作为tag来标识网络请求的。在Activity的onDestroy回调中根据这个标识取消所有的网络请求。但是在部分页面，出现了比较奇怪的问题：从Activity A打开Activity B，然后finish掉Activity B回到Activity A，这时候再次打开Activity B，Activity B中的网络请求会出现概率性无回调导致页面加载失败，且毫无规律可循。后经过仔细排查，原来是第一次打开的Activity B实例onDestroy回调延时造成的。也就是说，由于第一次打开的Activity B实例的onDestroy回调延时，恰好在第二次打开的Activity B实例启动后正在请求网络的时候被执行，取消了全部以Activity B名字作为tag的网络请求，当然会导致第二次打开的Activity B实例中网络请求无回调了。(此处暂不讨论这种取消网络请求的不合理性)
2. 在某个项目中，产品经理要求实现应用内浮窗，根据各种需求细节，最终敲定的技术方案为：使用WindowManager来加载一个自定义浮窗，并根据Activity的生命周期回调来确定当前App是在前台还是后台。最先想到的自然是维护一个计数器，在Activity的onStart处将计数器+1，然后再onStop处将计算器-1。然而实际情况中，经常发现onStop延时调用，且延时时间不确定，导致浮窗关闭与App切换前后台时间不同步。最终只能改用onResume和onPause，外加一大堆冗余判断来保证浮窗能正常显示关闭。
3. 很多时候由于项目开发周期紧，碰到问题没时间深究，为了赶工，只是采取替代方案快速解决/绕开了存在的问题。这段时间需求少了一点，开始对项目中的一些重点页面做性能分析和优化。当对一个复杂的列表页和详情页进行检查的时候，再一次发现了onStop和onDestroy回调延时的问题。简单分析无果，加之又回想起那些被未知原因的bug支配的恐惧，于是下定决心，要趁着现在有空来深入挖掘下回调延时的背后究竟隐藏着哪些不为人知的秘密。
### 二、原因/结论
> Activity调用流程：打开Activity A -> 打开Activity B -> 关闭Activity B，回到Activity A

1. 回调延时：
    由于**要关闭的Activity B或者将要打开的Activity A往主线程的MessageQueue中连续不断的post了大量的msg**，导致主线程一直在不断的进行消息循环处理这些消息而没有得到停歇。因此，ActivityThread中Idler.queueIdle方法没有被调用的机会，App侧也就不能向ActivityManagerService(AMS)发起IPC告知自己有空闲时间来处理AMS侧的任务，也就是AMS向App侧发起IPC来进行Activity B的销毁的正常流程被阻断了。所以，finish Activity B后，onDestroy不会被及时回调。具体延时多久，要看主线程中堆积的msg什么时候被处理完，也就是说要看主线程什么时候闲下来。
2. 延时10s：
    除正常流程外，Android系统另行安排了一套流程来保证即使正常流程被阻断以后，Activity B还是能被销毁。这个流程就是用来预防正常流畅中App不会闲下来处理AMS侧的IPC任务而设计的(*源码注释：Schedule an idle timeout in case the app doesn't do it for us*)。流程具体为：在关闭Activity B返回Activity A的时候，当AMS侧发起IPC通知App侧的Activity A进行resume的时候，同时也会向AMS自己的主线程发送一个msg，该msg延时10s后执行。该msg的处理方为ActivityStackSupervisor#ActivityStackSupervisorHandler，msg的具体内容与正常流程App段空闲时需要执行的任务内容一致，当然也包括销毁Activity B。这样，Activity B的onDestroy方法也就在延时10s后被调用了。
### 三、问题复现
由于实际项目中出现onStop和onDestroy回调延时的页面代码较为复杂，难以抽取展示。因而这里只给出一个简单的示例来演示、复现问题。我们创建两个Activity，分别为FirstActivity和SecondActivity，并在SecondActivity创建的时候开始往主线程不断的post空msg，如下：
```Java
// in SecondActivity.java
public class SecondActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);

        ...
        postMsg();
    }

    private void postMsg() {
       view.post(new Runnable() {
            @Override
            public void run() {
                try {
                    // 在主线程中休眠一小段时间
                    // 用来模拟主线程中诸如复杂的绘制、复杂数据处理、帧动画等等操作
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
       view.postDelayed(new Runnable() {
            @Override
            public void run() {
                postMsg();
            }
        }, 10);
    }
}
```
代码非常简单。我们知道，view.post()方法底层就是往主线程的消息队列MessageQueue发送消息(这里这么写只是为了少写两行代码)，因为整段代码逻辑就是，在SecondActivity中不断向主线程post空msg，来模拟复杂页面中不停的重绘、数据流处理、帧动画处理等。
测试流程：我们首先打开FirstActivity，然后在FirstActivity中通过StartActivity方法来打开SecondActivity，接着我们按下回退键，关闭SecondActivity回到FirstActivity。如果我们打印出FirstActivity和SecondActivity的生命周期回调，就不难发现，按下回退键后，在FirstActivity的onResume方法被调用后，SecondActivity的onStop和onDestroy方法并没有紧接着被回调，而是恰好在10s后才被回调。
### 四、源码探究
既然出现了问题，当然要想办法弄清楚才行。于是我们熟练的打开百度，输入onDestroy回调延时，随便找两篇博客看看也就知道答案了。
本文分析到此为止。
开个玩笑，要是常规操作能解决问题，我当然也就不用多此一举来写这么一篇问题分析的博文了。这里随便贴几篇百度搜到的博文：
* [Activity中OnDestory()回调慢的原因及解决办法](https://blog.csdn.net/Heijinbaitu/article/details/79153635)
* [Activity销毁延迟10s执行回调的踩坑之路](https://www.jianshu.com/p/a7a35daf0d6b)
* [Android Activity的onStop()与onDestroy() 回调缓慢，延时调用的问题解决方案](https://blog.csdn.net/huahuaxiaolian/article/details/85231832)
* [...]()

看完很绝望有没有啊。虽然很多博文都写着原因分析，但完全没有普适性。给出的解决方案也是要么绕开这个坑，要么明着告诉你不要在onDestroy回调中去做balabala...。既然百度不行，那咱Google一下吧...Google好像也不太好使(或者说咱用的不好?)...还是不行，去stackoverflow看看，鉴于咱这英文阅读速度和搜出来的质量，在换了几个关键词之后也就放弃了。得，还是自己动手，丰衣足食吧。
首先从内存泄漏的角度考察了一番，什么LeakCanary、Android Profile和大杀器MAT，通通祭出来。结果闹了半天，仍然没有什么重大发现。看来是个棘手得问题，真叫人头秃啊，差点泄气要放弃了。可是留下这个问题，又让人恨得牙痒痒，甚至就此留下阴影，成为以后职业生涯修炼时走火入魔的导火索。哎，既然捷径都走不通，那看来只能笨办法，源码分析走起，我倒要看看系统这个鬼究竟搞了什么骚操作。
我们就以Activity的finish方法作为入口，一步一步，深入分析其流程。
本次分析，**源码基于Android 9.0**。其中会有基于Binder的进程间通信和基于Handler的进程内线程间通信相关知识，因为不是分析重点，一笔带过。另外onStop方法延迟调用与onDestroy方法延迟调用的原因有关联，分析流程也基本一致，所以这里以分析onDestroy方法延迟为主线。
```Java
// in Activity.java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient {

    public void finish() {
        finish(DONT_FINISH_TASK_WITH_ACTIVITY);
    }

    private void finish(int finishTask) {
        if (mParent == null) {
            ...
            try {
                if (resultData != null) {
                    resultData.prepareToLeaveProcess(this);
                }
                if (ActivityManager.getService()
                        .finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
                }
            } catch (RemoteException e) {
                // Empty
            }
            ...
        }
    }
}
```
这个流程还是比较简单的，finish方法内部调用了一个带参数的私有finish方法，参数`DONT_FINISH_TASK_WITH_ACTIVITY`从字面意思也可以看出来，就是只关闭Activity而不关闭Activity所在的任务栈。带参数的finish方法中调用了`ActivityManager.getService().finishActivity(mToken, resultCode, resultData, finishTask)`，通过IPC向AMS发起调用关闭当前Activity。
```Java
// in ActivityManagerService
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    public final boolean finishActivity(IBinder token, int resultCode, Intent resultData,
            int finishTask) {
            ...
            try {
                boolean res;
                final boolean finishWithRootActivity =
                        finishTask == Activity.FINISH_TASK_WITH_ROOT_ACTIVITY;
                if (finishTask == Activity.FINISH_TASK_WITH_ACTIVITY
                        || (finishWithRootActivity && r == rootR)) {
                    res = mStackSupervisor.removeTaskByIdLocked(tr.taskId, false,
                            finishWithRootActivity, "finish-activity");
                    if (!res) {
                        Slog.i(TAG, "Removing task failed to finish activity");
                    }
                } else {
                    // 由于finishtask标志为DONT_FINISH_TASK_WITH_ACTIVITY
                    // 因此这里进入else分支
                    res = tr.getStack().requestFinishActivityLocked(token, resultCode,
                            resultData, "app-request", true);
                    if (!res) {
                        Slog.i(TAG, "Failed to finish by app-request");
                    }
                }
                return res;
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            ...
    }
}
```
```Java
// in ActivityStack.java
class ActivityStack<T extends StackWindowController> extends ConfigurationContainer
        implements StackWindowListener {

    final boolean requestFinishActivityLocked(IBinder token, int resultCode,
            Intent resultData, String reason, boolean oomAdj) {
        ...
        finishActivityLocked(r, resultCode, resultData, reason, oomAdj);
        ...
    }

    final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
            String reason, boolean oomAdj) {
        //PAUSE_IMMEDIATELY定义在ActivityStackSupervisor中
        //static final boolean PAUSE_IMMEDIATELY = true;
        //即不要立即执行暂停该Activity后的流程，而是延时500ms后再进行暂停Activity后的相关工作
        //延时500ms其实也是一种保险机制，确保后续流程一定会被执行。正常的话App侧在onPause后会发起IPC来告知AMS执行后续流程
        return finishActivityLocked(r, resultCode, resultData, reason, oomAdj, !PAUSE_IMMEDIATELY);
    }

    final boolean finishActivityLocked(ActivityRecord r, int resultCode, Intent resultData,
            String reason, boolean oomAdj, boolean pauseImmediately) {
        if (r.finishing) {
            Slog.w(TAG, "Duplicate finish request for " + r);
            return false;
        }

        mWindowManager.deferSurfaceLayout();
        try {
            //设置r.finishing=true; 标记当前Activity为finishing
            r.makeFinishingLocked();
            final TaskRecord task = r.getTask();
            EventLog.writeEvent(EventLogTags.AM_FINISH_ACTIVITY,
                    r.userId, System.identityHashCode(r),
                    task.taskId, r.shortComponentName, reason);
            final ArrayList<ActivityRecord> activities = task.mActivities;
            final int index = activities.indexOf(r);
            if (index < (activities.size() - 1)) {
                task.setFrontOfTask();
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {
                    // If the caller asked that this activity (and all above it)
                    // be cleared when the task is reset, don't lose that information,
                    // but propagate it up to the next activity.
                    ActivityRecord next = activities.get(index+1);
                    next.intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET);
                }
            }
            //停止向当前Activity分发按键消息事件
            r.pauseKeyDispatchingLocked();
            //调整当前的活动的任务栈
            adjustFocusedActivityStack(r, "finishActivity");
            //记录当前Activity要发送出去的结果数据
            finishActivityResultsLocked(r, resultCode, resultData);

            final boolean endTask = index <= 0 && !task.isClearingToReuseTask();
            final int transit = endTask ? TRANSIT_TASK_CLOSE : TRANSIT_ACTIVITY_CLOSE;
            if (mResumedActivity == r) {//此处，当前Activity即为活动Activity，进入if分支
                if (DEBUG_VISIBILITY || DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare close transition: finishing " + r);
                if (endTask) {
                    mService.mTaskChangeNotificationController.notifyTaskRemovalStarted(
                            task.taskId);
                }
                mWindowManager.prepareAppTransition(transit, false);

                // Tell window manager to prepare for this one to be removed.
                //将当前Activity设置为不可见
                r.setVisibility(false);

                if (mPausingActivity == null) {//还未暂停当前Activity，因此mPausingActivity为null
                    //开始暂停当前Activity
                    startPausingLocked(false, false, null, pauseImmediately);
                }

                if (endTask) {
                    mService.getLockTaskController().clearLockedTask(task);
                }
            } else if (!r.isState(PAUSING)) {
                ...
            } else {
                ...
            }

            return false;
        } finally {
            ...
        }
    }

    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity != null) {
            if (!shouldSleepActivities()) { //很明显，正常情况这里为false
                completePauseLocked(false, resuming);
            }
        }

        ActivityRecord prev = mResumedActivity;
        ...
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        //设置当前Activity的状态为pausing
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);
        mStackSupervisor.getLaunchTimeTracker().stopFullyDrawnTraceIfNeeded(getWindowingMode());
        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
            try {
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);
                mService.updateUsageStats(prev, false);
                //发起IPC通信，通知App侧暂定当前Activity，最终会回调Activity的onPause方法
                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                ...
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        if (mPausingActivity != null) { //前面已经将当前Activity赋值给mPausingActivity了，因此这里不为null
            ...
            if (pauseImmediately) {//前面我们传进来的参数为!PAUSE_IMMEDIATELY，因此进入else分支
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;
            } else {
                //向AMS主线程发送一个延时500ms的消息，
                schedulePauseTimeout(prev);
                return true;
            }
        } else {
            ...
        }
    }
}
```
为了使逻辑看起来更清晰，这里删减了一部分源码，并根据调用流程，调整了方法的排列顺序，同时对部分较为重要的源码进行了注释。
整个流程方法调用链为：在AMS中发起调用finishActivity方法，进入ActivityStack中，接着依次调用了` requestFinishActivityLocked -> finishActivityLocked -> finishActivityLocked -> startPausingLocked `这几个方法。在第一个调用方法finishActivityLocked中，我们详细注释了pauseImmediately参数的意义。最后，在startPausingLocked方法中，我们我们看到了这么一句代码：
```Java
mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
```
其实就是AMS通过Binder与App侧进行IPC，App侧ActivityThread中的ApplicationThread即为本次IPC通信的对端。然后App侧再通过Handler进行进程内线程间通信，通知ActivityThread中的H(继承Handler)来处理消息。H中TransactionExecutor作为真正的msg执行者，统一处理消息中所携带的任务。处理方式即执行PauseActivityItem中的execute和postExecute方法：
```Java
// in TransactionExecutor.java
public class TransactionExecutor {
    private void executeLifecycleState(ClientTransaction transaction) {
        //这里的lifecycleItem即为前面AMS传递过来的PauseActivityItem
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ...
        // Execute the final transition with proper parameters.
        //调用PauseActivityItem中的execute和postExecute方法
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
}
```
我们进入PauseActivityItem看看AMS发起IPC究竟要干什么：
```Java
// in PauseActivityItem.java
public class PauseActivityItem extends ActivityLifecycleItem {
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        //调用ActivityThread的handlePauseActivity方法，最终会回调当前Activity的onPause
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        if (mDontReport) {
            return;
        }
        try {
            // TODO(lifecycler): Use interface callback instead of AMS.
            // 向AMS发起IPC，告知AMS当前Activity已经暂停，可以执行后续流程了
            // IPC对端是ActivityManagerService，最终AMS中的activityPaused方法会被调用
            ActivityManager.getService().activityPaused(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```
PauseActivityItem一共干了两件事，首先是回调当前Activity的onPause，然后发起IPC，告诉AMS可以执行后续流程了。我们进入ActivityManagerService来看看activityPaused方法：
```Java
// in ActivityManagerService
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    @Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
}
```
```Java
// in ActivityStack.java
class ActivityStack<T extends StackWindowController> extends ConfigurationContainer
        implements StackWindowListener {

    final void activityPausedLocked(IBinder token, boolean timeout) {
        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            //前面我们发送了一个延时500ms的消息来确保后续流程一定能够得以执行
            //现在正常流程没问题，因此移除延时消息
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                mService.mWindowManager.deferSurfaceLayout();
                try {
                    //开始执行当前Activity暂停后的流程
                    //参数resumeNext为true，表示需要唤起下一个Activity
                    completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
                } finally {
                    mService.mWindowManager.continueSurfaceLayout();
                }
                return;
            } else {
                ...
            }
        }
        //确保全部Activity的可见性都处于正确的状态
        mStackSupervisor.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
    }

    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev = mPausingActivity;

        if (prev != null) {
            prev.setWillCloseOrEnterPip(false);
            final boolean wasStopping = prev.isState(STOPPING);
            prev.setState(PAUSED, "completePausedLocked");
            if (prev.finishing) { //早在finishActivityLocked方法中就通过r.makeFinishingLocked()将finishing置为true了
                //finish当前Activity
                prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false,
                        "completedPausedLocked");
            } else if (prev.app != null) {
                ...
            } else {
                ...
            }
            if (prev != null) {
                prev.stopFreezingScreenLocked(true /*force*/);
            }
            mPausingActivity = null;
        }

        if (resumeNext) { //传递进来的resumeNext为true
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!topStack.shouldSleepOrShutDownActivities()) { //看字面意思也可猜到，正常情况下进入if分支
                mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
            } else {
                ...
            }
        }
        ...
        //再次确保全部Activity的可见性都处于正确的状态
        mStackSupervisor.ensureActivitiesVisibleLocked(resuming, 0, !PRESERVE_WINDOWS);
    }

    final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj,
            String reason) {
        //获取任务栈中栈顶Activity作为接下来即将要显示的Activity
        final ActivityRecord next = mStackSupervisor.topRunningActivityLocked(
                true /* considerKeyguardState */);
        ...
        final ActivityState prevState = r.getState();
        r.setState(FINISHING, "finishCurrentActivityLocked");
        //很明显，当前任务栈即为前台活动任务栈，因此finishingActivityInNonFocusedStack为false
        final boolean finishingActivityInNonFocusedStack
                = r.getStack() != mStackSupervisor.getFocusedStack()
                && prevState == PAUSED && mode == FINISH_AFTER_VISIBLE;
        if (mode == FINISH_IMMEDIATELY // mode为FINISH_AFTER_VISIBLE false
                || (prevState == PAUSED
                    && (mode == FINISH_AFTER_PAUSE || inPinnedWindowingMode())) // inPinnedWindowingMode() 为false
                || finishingActivityInNonFocusedStack //false
                || prevState == STOPPING //prevState应该为PAUSED
                || prevState == STOPPED
                || prevState == ActivityState.INITIALIZING) {
            //看起来是要销毁当前Activity了，好开心...然而，判断结果为false，压根不会进到if分支里面来
            r.makeFinishingLocked();
            boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm:" + reason);
            ...
        }
        //虽然因为条件不符合，这里还没办法直接销毁当前Activity。但是，我们先将当前Activity放到一个列表中，该列表存储着全部即将要销毁的Activity
        mStackSupervisor.mFinishingActivities.add(r);
        r.resumeKeyDispatchingLocked();
        mStackSupervisor.resumeFocusedStackTopActivityLocked();
        return r;
    }
}
```
经过IPC，AMS侧收到消息，调用ActivityStack的activityPausedLocked方法开始继续后续流程。我们看到，这里ActivityStack中方法调用链为：`activityPausedLocked ->  completePauseLocked -> finishCurrentActivityLocked`。本来追踪到finishCurrentActivityLocked方法，看到destroyActivityLocked这句代码时挺开心的，以为这就是终点了。然而，经过仔细分析，条件根本不符合，整个if分支根本得不到执行。因此，finishCurrentActivityLocked方法中仅仅是将当前Activity放到一个存储着全部要销毁的Activity列表里。再次回completePauseLocked方法中，由于需要唤起下一个Activity，因此，调用了
```Java
//注意参数。第一个参数为当前前台活动的任务栈，第二个参数为当前正在关闭的Activity
mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
```
这行代码，我们进入StackSupervisor，看看resumeFocusedStackTopActivityLocked方法都干了啥：
```Java
// in StackSupervisor.java
public class ActivityStackSupervisor extends ConfigurationContainer implements DisplayListener,
        RecentTasks.Callbacks {

    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        ...
        if (targetStack != null && isFocusedStack(targetStack)) {
            //根据传进来的参数，很容易知道进入if分支
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        ...
    }
}
```
```Java
// in ActivityStack.java
class ActivityStack<T extends StackWindowController> extends ConfigurationContainer
        implements StackWindowListener {

    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            //唤起/显示栈顶的Activity
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        ...
    }

    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        //方法很长，内容也很多，限于篇幅，这里就不仔细分析了。
        //主要是调用了：
        if (next.app != null && next.app.thread != null) {
            ...
            synchronized(mWindowManager.getWindowManagerLock()) {
                try {
                    ...
                    //发起IPC，通知App侧要打开的Activity，最终会回调下一个要打开的Activity的onResume
                    transaction.setLifecycleStateRequest(
                            ResumeActivityItem.obtain(next.app.repProcState,
                                    mService.isNextTransitionForward()));
                    mService.getLifecycleManager().scheduleTransaction(transaction);
                } catch (Exception e) {
                    ...
                }
            }

            try {
                // 除了通知App侧下一个要打开的Activity，AMS侧也要完成其他状态设置相关的流程
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
            ...
        }
        ...
    }
}
```
至此，我们需要兵分两路。一路是追踪AMS向App发起的IPC，唤起下一个要显示的Activity；一路是追踪唤起下一个Activity后AMS本身要进行的状态设置相关工作。我们先跟踪IPC流程，ResumeActivityItem与前面我们分析的PauseActivityItem一样，都是先执行execute方法，再执行postExecute方法：
```Java
// in ResumeActivityItem.java
public class ResumeActivityItem extends ActivityLifecycleItem {

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        //调用ActivityThread的handleResumeActivity方法，最终会回调下一个要显示的Activity的onResume
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        try {
            // TODO(lifecycler): Use interface callback instead of AMS.
            // 向AMS发起IPC，告知AMS下一个要显示的Activity已经resume，可以执行后续流程了
            // IPC对端是ActivityManagerService，最终AMS的activityResumed方法会被调用
            ActivityManager.getService().activityResumed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```
ResumeActivityItem也就干了两件事，首先是回调下一个要显示的Activity的onResume，然后发起IPC，告诉AMS可以执行后续流程了。我们再次进入ActivityManagerService来看看activityResumed方法：
```Java
// in ActivityManagerService
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    @Override
    public final void activityResumed(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityRecord.activityResumedLocked(token);
            mWindowManager.notifyAppResumedFinished(token);
        }
        Binder.restoreCallingIdentity(origId);
    }
}
```
如果从AMS再继续跟踪下去，线索就断了。千辛万苦分析到此处，结果还是没能挖出真相？心里一万头草泥马呼啸而过。没办法，生活还是要继续，让我们收拾好心情，回溯到上一个方法，看看ActivityThread的handleResumeActivity方法都干了些啥：
```Java
// in ActivityThread.java
public final class ActivityThread extends ClientTransactionHandler {

    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...
        往主线程的消息队列添加了一个空闲处理器，当主线程空闲时就会回调这个处理器来执行一些优先级较低的任务
        Looper.myQueue().addIdleHandler(new Idler());
    }

    private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            boolean stopProfiling = false;
            if (mBoundApplication != null && mProfiler.profileFd != null
                    && mProfiler.autoStopProfiler) {
                stopProfiling = true;
            }
            if (a != null) {
                mNewActivities = null;
                IActivityManager am = ActivityManager.getService();
                ActivityClientRecord prev;
                do {
                    if (localLOGV) Slog.v(
                        TAG, "Reporting idle of " + a +
                        " finished=" +
                        (a.activity != null && a.activity.mFinished));
                    if (a.activity != null && !a.activity.mFinished) {
                        try {
                            //发起IPC，告诉AMS侧，我(App侧)空闲了，你有什么事情需要我做的，赶紧扔过来吧
                            //最终会调用ActivityManagerService的activityIdle方法
                            am.activityIdle(a.token, a.createdConfig, stopProfiling);
                            a.createdConfig = null;
                        } catch (RemoteException ex) {
                            throw ex.rethrowFromSystemServer();
                        }
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            ...
        }
    }
}
```
关键的代码我已经注释起来了，总而言之就是App在下一个要显示的Activity调用完onResume后往自己的主线程消息队列注册一个空闲处理器，以便App主线程的消息队列中全部msg处理完了以后能去处理一些低优先级的任务。这个空闲处理器的调用是一次性的，也就是说调用一次之后就不会再被调用了。而这个所谓的低优先级任务，其实就是App侧向AMS侧发起IPC，告诉AMS，我(App)闲下来了，你有什么需要我干的任务没有。进入ActivityManagerService，看看activityIdle方法：
```Java
//in ActivityManagerService
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    @Override
    public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
        final long origId = Binder.clearCallingIdentity();
        synchronized (this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                //App侧空闲了，要干点什么好呢？
                ActivityRecord r =
                        mStackSupervisor.activityIdleInternalLocked(token, false /* fromTimeout */,
                                false /* processPausingActivities */, config);
                if (stopProfiling) {
                    if ((mProfileProc == r.app) && mProfilerInfo != null) {
                        clearProfilerLocked();
                    }
                }
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
}
```
```Java
//in StackSupervisor
public class ActivityStackSupervisor extends ConfigurationContainer implements DisplayListener,
        RecentTasks.Callbacks {
    final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            boolean processPausingActivities, Configuration config) {
        ...
        // Finish any activities that are scheduled to do so but have been
        // waiting for the next one to start.
        //还记得前面我们分析的Activity销毁流程吧，
        //在ActivityStack的finishCurrentActivityLocked方法中，眼看就要成功，却因条件不符功亏一篑的地方吗？
        //虽然条件不符，未能调用destroyActivityLocked去销毁Activity，但当时却将需要待销毁的Activity加入到一个列表中了。
        //此处的finishes就是那个待销毁的Activity列表
        for (int i = 0; i < NF; i++) {
            r = finishes.get(i);
            final ActivityStack stack = r.getStack();
            if (stack != null) {
                activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
            }
        }
        ...
    }
}
```
分析这么久，终于看到希望的曙光了。再接再厉，进入ActivityStack，速度瞟一眼destroyActivityLocked方法吧：
```Java
// in ActivityStack.java
class ActivityStack<T extends StackWindowController> extends ConfigurationContainer
        implements StackWindowListener {
    final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
        ...
        final boolean hadApp = r.app != null;
        if (hadApp) {
            ...
            boolean skipDestroy = false;
            try {
                //发起IPC，通知App侧销毁Activity！！！
                mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken,
                        DestroyActivityItem.obtain(r.finishing, r.configChangeFlags));
            } catch (Exception e) {
                ...
            }

            r.nowVisible = false;

            // If the activity is finishing, we need to wait on removing it
            // from the list to give it a chance to do its cleanup. During
            // that time it may make calls back with its token so we need to
            // be able to find it on the list and so we don't want to remove
            // it from the list yet. Otherwise, we can just immediately put
            // it in the destroyed state since we are not removing it from the
            // list.
            if (r.finishing && !skipDestroy) {
                r.setState(DESTROYING,
                        "destroyActivityLocked. finishing and not skipping destroy");
                //向AMS主线程发起一个延时10s的消息，保证即使正常流程失败，也能进行Activity销毁的后续流程
                //这里也是10s，颇具误导性。
                //在这里绕了好久时间，最终发现，这个分支流程与我们要分析的问题关系不大
                Message msg = mHandler.obtainMessage(DESTROY_TIMEOUT_MSG, r);
                mHandler.sendMessageDelayed(msg, DESTROY_TIMEOUT);
            } else {
                r.setState(DESTROYED,
                        "destroyActivityLocked. not finishing or skipping destroy");
                r.app = null;
            }
        } else {
            ...
        }
    }
}
```
老规矩，让我们看看AMS发起IPC，通知App侧都干了啥。进入DestroyActivityItem，看看execute方法和postExecute方法：
```Java
public class DestroyActivityItem extends ActivityLifecycleItem {
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        //调用ActivityThread的handleDestroyActivity，最终会调用Activity的onDestroy方法
        client.handleDestroyActivity(token, mFinished, mConfigChanges,
                false /* getNonConfigInstance */, "DestroyActivityItem");
    }
}
```
So easy~。呃，postExecute方法哪去了？简单，我们一路跟踪DestroyActivityItem的父类，会发现postExecute默认实现为一个空方法。
那么，整个分析算是完成了吧。
终于完了啊，让我放松下喘口气先~~
等等！！！onStop方法怎么没被调用？？为什么回调会延时？？？为什么会延时10s？？？
苍天啊，搞了这么久，原来问题一个都还没解决呢？？哈哈，其实不是，现在我们离真相只有一步之遥了，一个一个来。

1. 首先说下onStop方法什么时候被调用的。
    DestroyActivityItem的execute和postExecute方法谁调用的啊？前面的分析还没忘吧，是在TransactionExecutor的executeLifecycleState方法中调用的：
    ```Java
    // in TransactionExecutor.java
    private void executeLifecycleState(ClientTransaction transaction) {
        // Cycle to the state right before the final requested state.
        //根据Activity当前的状态和最终要到达的状态，计算中间需要经过哪些步骤
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
    ```
    看到没，关键代码是`cycleToPath()`，只要稍微一跟踪就可以知道在Pause和Destroy之间会执行Stop了，这个流程简单没有其他分支，因此就不详写了。
2. onStop和onDestroy回调为什么会延时
    前面我们已经分析完了调用finish后，到回调onStop和onDestroy之间的整个正常流程。这中间经历了多次App侧和AMS侧的进程间通信，以及两端各自进程内通信。正常流程中较为关键的一点是，新的要显示的Activity在resume后，App侧的主线程空闲下来才会通知AMS执行后续流程，将关闭的Activity销毁。那要是新的Activity在resume后，主线程一直在循环处理MessageQueue中堆积的msg呢？很显然，要是这样的话，通知AMS执行后续流程自然被延迟了，因此，Activity的onStop和onDestroy生命周期回调自然也就被延迟了。
3. 延时10s是为什么
    作为一个健壮的操作系统，当然要有一定的容错机制，不能说因为App侧主线程一直忙，AMS侧就不去销毁/回收已经死亡的Activity。要不然新手开发者开发的App会因为内存泄漏等问题分分钟玩死系统。那么这个容错机制是怎么设计的呢？
    前面我们说兵分两路，一路是追踪AMS向App发起的IPC，唤起下一个要显示的Activity，一路是追踪唤起下一个Activity后AMS本身要进行的状态设置相关工作。分析完了AMS侧向App侧发起IPC后的正常流程后，我们继续分析AMS本身还做了哪些工作：
    ```Java
    //in ActivityStack.java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        next.completeResumeLocked();
    }
    ```
    ```Java
    // in ActivityRecord.java
    final class ActivityRecord extends ConfigurationContainer implements AppWindowContainerListener {
        void completeResumeLocked() {
            ...
            // Schedule an idle timeout in case the app doesn't do it for us.
            // 设置一个超时的空闲时间，以便在App没有通知我们其有空的情形下也能执行相关流程
            mStackSupervisor.scheduleIdleTimeoutLocked(this);
            ...
        }
    }
    ```
    ```Java
    // in StackSupervisor.java
    public class ActivityStackSupervisor extends ConfigurationContainer implements DisplayListener,
            RecentTasks.Callbacks {
        void scheduleIdleTimeoutLocked(ActivityRecord next) {
            //真相就在这里，发送一个延时10s的消息，确保正常流程行不通的情况下也能销毁Activity
            Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
            mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);
        }

        private final class ActivityStackSupervisorHandler extends Handler {
            void activityIdleInternal(ActivityRecord r, boolean processPausingActivities) {
                synchronized (mService) {
                    //在前面我们分析正常流程的时候，已经将该方法的执行流程分析完了，不再赘述
                    activityIdleInternalLocked(r != null ? r.appToken : null, true /* fromTimeout */,
                            processPausingActivities, null);
                }
            }
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case IDLE_TIMEOUT_MSG: {
                        //延时10s后，就当Activity侧已经空闲下来了，执行后续流程
                        activityIdleInternal((ActivityRecord) msg.obj,
                                true /* processPausingActivities */);
                    } break;
                    ...
                }
            }
    }
    ```

好了。这下是真的分析完了。
因为源码是在太长了，因此省略了很多分支流程和与主题无关的代码。
说是分析回调延时的问题，其实也即是分析finish的源码调用流程。通过分析，我们知道，要想使得Activity的onStop和onDestroy尽快得到回调，我们就该在写代码的时候，及时关闭、清理、移除不必要的主线程消息，并且尽可能的保证每个消息处理时间不要太长。
经过这一波分析，该对Framework层App和AMS的交互有了更清晰的理解了吧。
也算是给自己一个交代，完美！下班~
