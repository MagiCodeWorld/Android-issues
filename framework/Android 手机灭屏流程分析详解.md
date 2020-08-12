本篇文章主要介绍 `Android` 开发中的部分知识点，通过阅读本篇文章，您将收获以下内容:

> 1.前言
> 2.Power键灭屏
> 3.超时灭屏
> 4.PSensor灭屏


**PowerManagerService** 之前系列文章请参考如下
1.[PowerManagerService分析(一)之PMS启动](https://mp.weixin.qq.com/s/Mgi1W9mmrCUPkASTp3LPrA)
2.[PowerManagerService分析(二)之updatePowerStateLocked()核心](https://mp.weixin.qq.com/s/P3IvBrYt7afEa4XyEd3BQg)
3.[PowerManagerService分析(三)之WakeLock机制](https://mp.weixin.qq.com/s/q3NLI_o3db7wRZ8ThKW3cQ)
4.[Android手机亮屏流程分析](https://mp.weixin.qq.com/s/ErdSuXNeLGt81s5cBocaJQ)



#### 前言

在之前的`PMS`文章分析中知道，`PMS`中定义了四种屏幕状态：

- Awake状态：表示唤醒状态
- Dream状态：表示处于屏保状态
- Doze状态：表示处于Doze状态
- Asleep状态：表示处于休眠状态

#### Power键灭屏

当`power`键灭屏时，会在`PhoneWindowManager`中处理按键事件后，调用到`PMS`的`gotoSleep()`进行灭屏处理，下面直接看看`PhoneWindowManger`中对`Power`键灭屏的处理以及和`PMS`的交互。

在按`power`后，`PWS`中如下：

```java
case KeyEvent.KEYCODE_POWER: {
    .......
    if (down) {//按下时
        //处理按下事件
        interceptPowerKeyDown(event, interactive);
    } else //抬起时
        //处理抬起事件
        interceptPowerKeyUp(event, interactive, canceled);
    }
    break;
}
```

在处理`Power`键`interceptPowerKeyUp`抬起事件时，开始了灭屏流程：

```java
private void interceptPowerKeyUp(KeyEvent event, boolean interactive, boolean canceled) {
   
    .......
        if (!handled) {
          
            // No other actions.  Handle it immediately.开始灭屏流程
            powerPress(eventTime, interactive, mPowerKeyPressCounter);
        }

        // Done.  Reset our state.
        finishPowerKeyPress();
    }
```

`powerPress`灭屏流程

```java
private void powerPress(long eventTime, boolean interactive, int count) {
    if (mScreenOnEarly && !mScreenOnFully) {
        Slog.i(TAG, "Suppressed redundant power key press while "
                + "already in the process of turning the screen on.");
        return;
    }
    if (count == 2) {
       ......
    } else if (interactive && !mBeganFromNonInteractive) {
        switch (mShortPressOnPowerBehavior) {
            //灭屏
            case SHORT_PRESS_POWER_GO_TO_SLEEP:
                goToSleep(eventTime, PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON, 0);
                break;
            //灭屏，直接跳过Doze状态
            case SHORT_PRESS_POWER_REALLY_GO_TO_SLEEP:
                goToSleep(eventTime, PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON,
                        PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE);
                break;
                } else {
                    shortPressPowerGoHome();
                }
                break;
            }
        }
    }
}
```

在这里调用了`goToSleep()`方法，该方法如下：

```java
private void goToSleep(long eventTime, int reason, int flags) {
    mRequestedOrGoingToSleep = true;
    mPowerManager.goToSleep(eventTime, reason, flags);
}
```

最终，`PhoneWindowManager`中调用了`PowerManager`的`goToSleep()`方法来灭屏。

现在我们进入到`PowerManager.goToSleep()`方法：

```java
public void goToSleep(long time, int reason, int flags) {
    try {
        mService.goToSleep(time, reason, flags);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

可以看到，在`PowerManger`中开始向下调用到了`PoweManagerService`(以下简称PMS)中的`goToSleep()`中。
我们进`入PMS`中，就需要详细分析其中的方法了，先来看看`goToSleep()`方法：

```java
/**
 * @param eventTime 时间
 * @param reason 原因，Power键灭屏则是PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON
 * @param flags 目前只有两个值：0和1(GO_TO_SLEEP_FLAG_NO_DOZE)
 */
@Override // Binder call
public void goToSleep(long eventTime, int reason, int flags) {
    if (eventTime > SystemClock.uptimeMillis()) {
        throw new IllegalArgumentException("event time must not be in the future");
    }
    //检查权限
    mContext.enforceCallingOrSelfPermission(
            android.Manifest.permission.DEVICE_POWER, null);

    final int uid = Binder.getCallingUid();
    final long ident = Binder.clearCallingIdentity();
    try {
        //调用gotToSleepInternal
        goToSleepInternal(eventTime, reason, flags, uid);
    } finally {
        Binder.restoreCallingIdentity(ident);
    }
}
```

这个方法的参数和`PowerManager,PhoneWindowManager`中的同名方法对应，需要注意的是第二个参数和第三个参数；
第二个参数：表示灭屏原因，在PowerManager中定义了一些常量值来表示；
第三个参数：是一个标识，用来表示是否直接进入灭屏，一般的灭屏流程，都会先进入Doze状态，然后才会进入Sleep状态，如果将flag设置为1，则将会直接进入Sleep状态，这部分会在下文中逐渐分析到。

在`goToSleep()`方法中，检查权限之后，开始调用了`goToSleepInternal()`方法，该方法如下：

```java
private void goToSleepInternal(long eventTime, int reason, int flags, int uid) {
    synchronized (mLock) {
        if (goToSleepNoUpdateLocked(eventTime, reason, flags, uid)) {
            updatePowerStateLocked();
        }
    }
}
```

这个方法逻辑很简单，首先是调用了`goToSleepNoUpdateLocked()`方法，并根据该方法返回值来决定是否调用`updatePowerStateLocked()`方法。

一般来说，`goToSleepNoUpdateLocked()`都会返回true，现在看看该方法：

```java
@SuppressWarnings("deprecation")
private boolean goToSleepNoUpdateLocked(long eventTime, int reason, int flags, int uid) {

    if (eventTime < mLastWakeTime
            || mWakefulness == WAKEFULNESS_ASLEEP
            || mWakefulness == WAKEFULNESS_DOZING
            || !mBootCompleted || !mSystemReady) {
        return false;
    }
    try {
        switch (reason) {
            case PowerManager.GO_TO_SLEEP_REASON_DEVICE_ADMIN:
                Slog.i(TAG, "Going to sleep due to device administration policy "
                        + "(uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_TIMEOUT:
                Slog.i(TAG, "Going to sleep due to screen timeout (uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_LID_SWITCH:
                Slog.i(TAG, "Going to sleep due to lid switch (uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON:
                Slog.i(TAG, "Going to sleep due to power button (uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_SLEEP_BUTTON:
                Slog.i(TAG, "Going to sleep due to sleep button (uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_HDMI:
                Slog.i(TAG, "Going to sleep due to HDMI standby (uid " + uid +")...");
                break;
            case PowerManager.GO_TO_SLEEP_REASON_ACCESSIBILITY:
                Slog.i(TAG, "Going to sleep by an accessibility service request (uid "
                        + uid +")...");
                break;
            default:
                Slog.i(TAG, "Going to sleep by application request (uid " + uid +")...");
                reason = PowerManager.GO_TO_SLEEP_REASON_APPLICATION;
                break;
        }
        //标记最后一次灭屏时间
        mLastSleepTime = eventTime;
        //用于判定是否进入屏保
        mSandmanSummoned = true;
        //设置wakefulness值为WAKEFULNESS_DOZING，因此先进Doze状态
        setWakefulnessLocked(WAKEFULNESS_DOZING, reason);

        // Report the number of wake locks that will be cleared by going to sleep.
        //灭屏时，将清除以下三种使得屏幕保持亮屏的wakelock锁，numWakeLocksCleared统计下个数
        int numWakeLocksCleared = 0;
        final int numWakeLocks = mWakeLocks.size();
        for (int i = 0; i < numWakeLocks; i++) {
            final WakeLock wakeLock = mWakeLocks.get(i);
            switch (wakeLock.mFlags & PowerManager.WAKE_LOCK_LEVEL_MASK) {
                case PowerManager.FULL_WAKE_LOCK:
                case PowerManager.SCREEN_BRIGHT_WAKE_LOCK:
                case PowerManager.SCREEN_DIM_WAKE_LOCK:
                    numWakeLocksCleared += 1;
                    break;
            }
        }
        // Skip dozing if requested.
        //如果带有PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE的flag，则直接进入Sleep状态，不再进入Doze状态
        if ((flags & PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE) != 0) {
            //该方法才会真正地进入睡眠
            reallyGoToSleepNoUpdateLocked(eventTime, uid);
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_POWER);
    }
    return true;
}
```

在这个方法中：
首先，是判断调用该方法的原因并打印log，该log在日常分析问题时非常有用；
然后，通过`setWakefulnessLocked()`将当前`wakefulness`设置为`Doze`状态；
最后，通过flag判断，如果flag为1,则调用`reallyGoToSleepNoUpdateLocked()`方法直接进入`Sleep`状态。
因此，系统其他模块在调用`PM.goToSleep()`灭屏时，在除指定flag为`PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE`的情况外，都会首先进入`Doze`，再由Doze进入Sleep。

`setWakefulnessLocked()`方法用来设置`wakefulness`值，同时将会调用`Notifier`中`wakefulness`相关的逻辑，这部分在之前的流程分析中也分析过，这里再来看下：

```java
@VisibleForTesting
void setWakefulnessLocked(int wakefulness, int reason) {
    if (mWakefulness != wakefulness) {
        //设置mWakefulness
        mWakefulness = wakefulness;
        mWakefulnessChanging = true;
        mDirty |= DIRTY_WAKEFULNESS;
        if (mNotifier != null) {
            //调用Notifier中的方法，做wakefulness改变开始时的工作
            mNotifier.onWakefulnessChangeStarted(wakefulness, reason);
        }
    }
}
```

我们跟着执行流程来进行分析，`Notifier`是`PMS`模块中用于进行“通知”的一个组件类，比如发送亮灭屏广播就是它来负责，具体详细的分析请[点击这里](https://mp.weixin.qq.com/s/ErdSuXNeLGt81s5cBocaJQ) 查看。这里针对于灭屏场景，再来看下其中的逻辑：

```java
public void onWakefulnessChangeStarted(final int wakefulness, int reason) {
    //由于wakefulness为Doze，故interactive为false
    final boolean interactive = PowerManagerInternal.isInteractive(wakefulness);
    // ............................................
    // Handle any early interactive state changes.
    // Finish pending incomplete ones from a previous cycle.
    if (mInteractive != interactive) {
        // Finish up late behaviors if needed.
        if (mInteractiveChanging) {
            handleLateInteractiveChange();
        }
        // Handle early behaviors.
        mInteractive = interactive;
        mInteractiveChangeReason = reason;
        mInteractiveChanging = true;
        //处理早期工作
        handleEarlyInteractiveChange();
    }
}
```

在这个方法中，首先根据`wakefulness`值判断了系统当前的交互状态，如果是处于`Awake`状态和`Dream`状态，则表示可交互；如果处于`Doze`和`Asleep`状态，则表示不可交互；
由于在`setWakefulnessLocked()`中已经设置了`wakefulness为DOZE`状态，因此此时处于不可交互状态，接下来开始执行`handleEarlyInteractiveChange()`方法：

```java
private void handleEarlyInteractiveChange() {
    synchronized (mLock) {
        //此时为false
        if (mInteractive) {
            // Waking up...
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    // Note a SCREEN tron event is logged in PowerManagerService.
                    mPolicy.startedWakingUp();
                }
            });
            // Send interactive broadcast.
            mPendingInteractiveState = INTERACTIVE_STATE_AWAKE;
            mPendingWakeUpBroadcast = true;
            updatePendingBroadcastLocked();
        } else {
            final int why = translateOffReason(mInteractiveChangeReason);
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    //通过PhoneWindowManager设置锁屏
                    mPolicy.startedGoingToSleep(why);
                }
            });
        }
    }
}
```

在这个方法中，将调用`mPolicy.startedGoingToSleep(why)`进行锁屏流程(Keyguard的绘制)。

回到`PMS`中，在处理完`setWakefulnessLocked()`方法后，由于没有`PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE`，所以不会立即执行`reallyGoToSleepNoUpdateLocked()`方法，此时`goToSleepNoUpdateLocked()`方法完毕并返回true。

之后开始执行`updatePowerStateLocked()`方法了，这个方法对于熟悉PMS模块的人来说再熟悉不过了，它是整个PMS的核心，详细的分析请[点击这里](https://www.jianshu.com/p/PowerManagerService分析(二)之updatePowerStateLocked()核心) , 在这里我们只看其灭屏时的一些处理。

在`updatePowerStateLocked()`方法中，和灭屏直接相关的有如下部分：

```java
// 更新屏幕状态
boolean displayBecameReady = updateDisplayPowerStateLocked(dirtyPhase2);
//更新屏保信息
updateDreamLocked(dirtyPhase2, displayBecameReady);
// 收尾工作
finishWakefulnessChangeIfNeededLocked();
//释放锁
updateSuspendBlockerLocked();
```

- `updateDisplayPowerStateLocked()`将会向`DisplayPowerController`请求新的屏幕状态，完成屏幕的更新；
- `updateDreamLocked()`方法用来更新屏保信息，除此之外还有一个任务
  调用`reallyGoToSleep()`方法进入休眠，即由DOZE状态进入Sleep状态。
- `finishWakefulnessChangeIfNeededLocked()`方法用来做最后的收尾工作，当然，在这里会调用到`Notifier`中进行收尾。
- `updateSuspendBlockerLocked()`方法将用来更新`SuspendBlocker`锁，会根据当前的`WakeLock`类型以及屏幕状态来决定是否需要申请`SuspendBlocker`锁。

在`updateDreamLocked()`中更新屏保状态时，如果此时处于Doze状态且没有进行屏保，则将调用`reallyGoToSleepNoUpdateLocked()`方法，将`wakefulness`值设置为了`Sleep`,部分代码如下：

```java
else if (wakefulness == WAKEFULNESS_DOZING) {
                if (isDreaming) {
                    return; // continue dozing
                }

                // Doze has ended or will be stopped.  Update the power state.
                reallyGoToSleepNoUpdateLocked(SystemClock.uptimeMillis(), Process.SYSTEM_UID);
                updatePowerStateLocked();
            }
```

再来看看该方法：

```java
private boolean reallyGoToSleepNoUpdateLocked(long eventTime, int uid) {
    if (eventTime < mLastWakeTime || mWakefulness == WAKEFULNESS_ASLEEP
            || !mBootCompleted || !mSystemReady) {
        return false;
    }

    try {
        //设置为ASLEEP状态
        setWakefulnessLocked(WAKEFULNESS_ASLEEP, 
                  PowerManager.GO_TO_SLEEP_REASON_TIMEOUT);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_POWER);
    }
    return true;
}
```

以上就是整个`Power`键灭屏`PMS`部分的流程，其时序图如下：

![img](D:\myFile\Project\MarkDown\assets\1684c10b305e6910)



#### 超时灭屏

经过上面的分析，我们知道了`Power键灭屏`由`PhoneWindowManager`发起了`goToSleep`，现在来看看超时灭屏是如何实现的。

超时灭屏主要有`两个影响因素`：`休眠时间`和`用户活动`。休眠时间在`Settings`中进行设置，用户活动是指当手机处于亮屏状态，都会调用`userActivityNoUpdateLocked()`方法去更新用户活动时间。接下来我们就从`userActivityNoUpdateLocked()`方法开始分析其超时灭屏的流程。

首先来看该方法：

```
private boolean userActivityNoUpdateLocked(long eventTime, int event, int flags, int uid) {

        if (eventTime < mLastSleepTime || eventTime < mLastWakeTime
                || !mBootCompleted || !mSystemReady) {
            return false;
        }

            mNotifier.onUserActivity(event, uid);
           
            if (mUserInactiveOverrideFromWindowManager) {
                mUserInactiveOverrideFromWindowManager = false;
                mOverriddenTimeout = -1;
            }
            //如果wakefulness为Asleep或Doze，不再计算超时时间，直接返回
            if (mWakefulness == WAKEFULNESS_ASLEEP
                    || mWakefulness == WAKEFULNESS_DOZING
                    || (flags & PowerManager.USER_ACTIVITY_FLAG_INDIRECT) != 0) {
                return false;
            }
            //如果带有该flag，则会小亮一会儿再灭屏
            if ((flags & PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS) != 0) {
                if (eventTime > mLastUserActivityTimeNoChangeLights
                        && eventTime > mLastUserActivityTime) {
                    //将当前时间赋值给mLastUserActivityTimeNoChangeLights
                    mLastUserActivityTimeNoChangeLights = eventTime;
                    mDirty |= DIRTY_USER_ACTIVITY;
                    if (event == PowerManager.USER_ACTIVITY_EVENT_BUTTON) {
                        mDirty |= DIRTY_QUIESCENT;
                    }

                    return true;
                }
            } else {
                if (eventTime > mLastUserActivityTime) {
                    //将当前时间赋值给mLastUserActivityTime
                    mLastUserActivityTime = eventTime;
                    mDirty |= DIRTY_USER_ACTIVITY;
                    if (event == PowerManager.USER_ACTIVITY_EVENT_BUTTON) {
                        mDirty |= DIRTY_QUIESCENT;
                    }
                    return true;
                }
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_POWER);
        }
        return false;
    }
```

在这个方法中，如果传入的参数flag为`PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS`，则将事件时间赋值给`mLastUserActivityTimeNoChangeLights`,否则将事件时间赋值给`mLastUserActivityTime`。这个flag标志用于延长亮屏或Dim的时长一小会儿。

当这个方法执行之后，就得到了`mLastUserActivityTime`或者`mLastUserActivityTimeNoChangeLights`的值，然后经过一些调用后，又会进入`updatePowerStateLocked()`方法中。在这个方法中，和超市灭屏直接相关的就是for循环部分：

```
for (;;) {
                int dirtyPhase1 = mDirty;
                dirtyPhase2 |= dirtyPhase1;
                mDirty = 0;

                updateWakeLockSummaryLocked(dirtyPhase1);
                updateUserActivitySummaryLocked(now, dirtyPhase1);
                if (!updateWakefulnessLocked(dirtyPhase1)) {
                    break;
                }
            }
```

其中`updateWakeLockSummaryLocked()`用来统计`WakeLock`，这里就不分析该方法了，详细的分析请[点击这里](https://mp.weixin.qq.com/s/P3IvBrYt7afEa4XyEd3BQg)，现在从`updateUserActivitySummaryLocked()`方法开始分析，该方法如下：

```
private void updateUserActivitySummaryLocked(long now, int dirty) {
        // Update the status of the user activity timeout timer.
        if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY
                | DIRTY_WAKEFULNESS | DIRTY_SETTINGS)) != 0) {
            mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);

            long nextTimeout = 0;
            if (mWakefulness == WAKEFULNESS_AWAKE
                    || mWakefulness == WAKEFULNESS_DREAMING
                    || mWakefulness == WAKEFULNESS_DOZING) {
                //获取睡眠时长，为Settings.Secure.SLEEP_TIMEOUT的值和最小休眠时间的最大值，Settings.Secure.SLEEP_TIMEOUT一般为-1，
                //表示禁用，因此该值默认为-1
                final int sleepTimeout = getSleepTimeoutLocked();
                //获取休眠时长，在Settings中设置的值
                final int screenOffTimeout = getScreenOffTimeoutLocked(sleepTimeout);
                //获取Dim时长，由休眠时长剩Dim百分比得到
                final int screenDimDuration = getScreenDimDurationLocked(screenOffTimeout);
                //用户活动是否由Window覆盖
                final boolean userInactiveOverride = mUserInactiveOverrideFromWindowManager;
                //该值用来统计用户活动状态，每次进入该方法，置为0
                mUserActivitySummary = 0;
                //上次用户活动时间>=上次唤醒时间
                if (mLastUserActivityTime >= mLastWakeTime) {
                    //下次超时时间为上次用户活动时间+休眠时间-Dim时间，到达这个时间后，将进入Dim状态
                    nextTimeout = mLastUserActivityTime
                            + screenOffTimeout - screenDimDuration;
                    //如果当前时间<nextTimeout,则此时处于亮屏状态，标记mUserActivitySummary为USER_ACTIVITY_SCREEN_BRIGHT
                    if (now < nextTimeout) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                    } else {
                        //如果当前时间>nextTimeout，此时有两种情况，要么进入Dim要么进入Sleep
                        //将上次用户活动时间+灭屏时间赋值给nextTimeout，如果该值大于当前时间，则说明此时应该处于Dim状态
                        //因此将标记mUserActivitySummary为USER_ACTIVITY_SCREEN_DIM
                        nextTimeout = mLastUserActivityTime + screenOffTimeout;
                        if (now < nextTimeout) {
                            mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                        }
                    }
                }
                //判断和USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS标记相关,如果带有此标记，才会进入该if
                if (mUserActivitySummary == 0
                        && mLastUserActivityTimeNoChangeLights >= mLastWakeTime) {
                    //下次超时时间=上次用户活动时间+灭屏时间
                    nextTimeout = mLastUserActivityTimeNoChangeLights + screenOffTimeout;
                    //根据当前时间和nextTimeout设置mUserActivitySummary
                    if (now < nextTimeout) {
                        if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_BRIGHT
                                || mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_VR) {
                            mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                        } else if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_DIM) {
                            mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                        }
                    }
                }
                //不满足以上条件时，此时mUserActivitySummary为0，这种情况应该为当mUserActivitySummary经历了USER_ACTIVITY_SCREEN_BRIGHT
                //和USER_ACTIVITY_SCREEN_DIM之后才会执行到这里
                if (mUserActivitySummary == 0) {
                    if (sleepTimeout >= 0) {
                        //获取上次用户活动时间的最后一次时间
                        final long anyUserActivity = Math.max(mLastUserActivityTime,
                                mLastUserActivityTimeNoChangeLights);
                        if (anyUserActivity >= mLastWakeTime) {
                            nextTimeout = anyUserActivity + sleepTimeout;
                            //将mUserActivitySummary值置为USER_ACTIVITY_SCREEN_DREAM，表示屏保
                            if (now < nextTimeout) {
                                mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                            }
                        }
                    } else {
                        //将mUserActivitySummary值置为USER_ACTIVITY_SCREEN_DREAM，表示屏保
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                        nextTimeout = -1;
                    }
                }

                if (mUserActivitySummary != USER_ACTIVITY_SCREEN_DREAM && userInactiveOverride) {
                    if ((mUserActivitySummary &
                            (USER_ACTIVITY_SCREEN_BRIGHT | USER_ACTIVITY_SCREEN_DIM)) != 0) {
                        // Device is being kept awake by recent user activity
                        if (nextTimeout >= now && mOverriddenTimeout == -1) {
                            // Save when the next timeout would have occurred
                            mOverriddenTimeout = nextTimeout;
                        }
                    }
                    mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                    nextTimeout = -1;
                }
                if (mUserActivitySummary != 0 && nextTimeout >= 0) {
                    //发送一个异步Handler定时消息
                    Message msg = mHandler.obtainMessage(MSG_USER_ACTIVITY_TIMEOUT);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtTime(msg, nextTimeout);
                }
            } else {//当wakefulness=Sleep的时候，直接将mUserActivitySummary置为0
                mUserActivitySummary = 0;
            }
        }
    }
```

该方法用来更新用户活动状态，其中细节在代码中都进行了注释，该方法中来看，通过`Handler`多次再此进入`updatePowerStateLocked（）`从而调用`updateUserActivitySummaryLocked()`方法，直到`nextTime=-1`和`mUserActivitySummary=0`时将不再发送`Handler`，从而完成了`mUserActivitySummary`的更新。根据流程来看，当设备从亮屏到休眠时间到达灭屏，`mUserActivitySummary`的值的变化应为：
`USER_ACTIVITY_SCREEN_BRIGHT—>USER_ACTIVITY_SCREEN_DIM—>USER_ACTIVITY_SCREEN_DREAM—>0.`
Handler的调用处理逻辑如下：

```
@Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_USER_ACTIVITY_TIMEOUT:
                    handleUserActivityTimeout();
                    break;
            }
        }

    private void handleUserActivityTimeout() { // runs on handler thread
        synchronized (mLock) {
            mDirty |= DIRTY_USER_ACTIVITY;
            updatePowerStateLocked();
        }
    }
```

当执行到这个方法后，现在就统计得到了`mWakeLockSummary和mUserActivitySummary`的值，现在我们看下一个方法——`updateWakefulnessLocked（）`，在for循环中，会根据该方法返回值来决定是否进行循环，为何会如此设计呢？在分析完该方法后，就会有答案了，如下：

```
private boolean updateWakefulnessLocked(int dirty) {
        boolean changed = false;
        if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY | DIRTY_BOOT_COMPLETED
                | DIRTY_WAKEFULNESS | DIRTY_STAY_ON | DIRTY_PROXIMITY_POSITIVE
                | DIRTY_DOCK_STATE)) != 0) {
            //isItBedTimeYetLocked()判断是否需要"睡觉"了
            if (mWakefulness == WAKEFULNESS_AWAKE && isItBedTimeYetLocked()) {
                final long time = SystemClock.uptimeMillis();
                if (shouldNapAtBedTimeLocked()) {//进入屏保
                    changed = napNoUpdateLocked(time, Process.SYSTEM_UID);
                } else {//开始休眠
                    changed = goToSleepNoUpdateLocked(time,
                            PowerManager.GO_TO_SLEEP_REASON_TIMEOUT, 0, Process.SYSTEM_UID);
                }
            }
        }
        return changed;
    }
```

这个方法中可以看到，首先根据isItBedTimeYetLocked()和mWakefulness来决定是否执行，然后根据`shouldNapAtBedTimeLocked（）`决定进入屏保还是休眠。
该方法如果返回值为true，则说明此时屏幕状态发生改变(在`goToSleepNoUpdateLocked()`和`napNoUpdateLocked()`中会分别设置`mWakefulness`为DREAM和ASLEEP)，因此将不会跳出for循环，再次进行一次循环。这就是为何会设置一个死循环的目的，同时也说明只有超时灭屏才会循环两次，其他情况下都会只执行一次for循环就退出。

回到该方法中，我们继续看看`isItBedTimeYetLocked（）：`

```
private boolean isItBedTimeYetLocked() {
        return mBootCompleted && !isBeingKeptAwakeLocked();
    }

    private boolean isBeingKeptAwakeLocked() {
        return mStayOn//是否需要保持常亮
                || mProximityPositive//PSensor是否靠近
                || (mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) != 0//当前是否有Wakelock类型为屏幕相关的锁
                || (mUserActivitySummary & (USER_ACTIVITY_SCREEN_BRIGHT
                        | USER_ACTIVITY_SCREEN_DIM)) != 0//当前用户活动状态是否为Draem或者0
                || mScreenBrightnessBoostInProgress;//是否处于亮度增强过程中
    }
```

以上代码可以看出，如果有任意一个条件为true，那么就不能进入休眠或者屏保状态，因此只有全部为false时，才会返回false，从而说明需要“睡觉”了。

仔细看这个方法，这里正是`mWakeLockSummary`和`mUserActivitySummary`的作用体现之一。

在平时分析问题时，如果存在无法超时灭屏问题，就需要查看`mWakeLockSummary`和`mUserActivitySummary`的值了。前者查看是否存在亮屏锁，后者查看用户活动是否已经处于0了。
现在继续分析`updateWakfulnessLocked()`方法中的下一个逻辑，当进入if语句后，就开始判断是要进入屏保呢？还是要直接休眠呢？

如果`shouldNapAtBedTimeLocked（）`返回true，则开始屏保，否则直接休眠，这里对于屏保相关就不再分析了，以后的时间中如果有机会，会单独进行分析。

当开始休眠时，直接调用了`goToSleepNoUpdateLocked()`方法中了，于是开始走休眠流程，之后的逻辑和`Power键灭屏`一样了。

整个超时灭屏的流程分析就到这里了，从以上流程中可以看到，`mWakeLockSummary`和`mUserActivitySummayr`的作用相当重要，我之前在android4.4手机上遇到过一个问题就是到达休眠时间后不会灭屏，分析后发现有一个应用申请了一个`PowerManager.SCREEN_BRIGHT_WAKE_LOCK`锁，该锁导致`mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) != 0`，从而没有灭屏。
整个超时灭屏流程的时序图如下：

![img](D:\myFile\Project\MarkDown\assets\1684c10b3060ab20)

#### PSensor灭屏

**什么是PSensor灭屏呢？**
`Proximity Sensor`，即距离传感器，当通话或微信时，如果脸部靠近屏幕，将会灭屏，这就是通过`PSenso`r灭屏的。
**为何会有PSensor灭屏呢？**
为了防止脸部误触，有更好的用户体验。

在原生的`Android`系统中，`PSensor灭屏`不同于`Power键灭屏`和`超时灭屏`，前者仅仅是设置屏幕的状态和关闭背光，而后两者在设置屏幕的状态和关闭背光后，让CPU也进入了休眠状态`(如果不持有PowerManger.PARTIAL_WAKE_LOCK)。`

`PSensor`灭屏涉及到更多的是`DisplayPowerController`中的内容，因此，将会在之后的文章中进行分析。



##  dumpsys power解释

```shell
POWER MANAGER (dumpsys power)

Power Manager State:
  mDirty=0x0//
  mWakefulness=Dozing//设备的清醒状态。
  mWakefulnessChanging=false//设备清醒状态是否正在改变中
  mIsPowered=true//是否连接电源
  mPlugType=2//充电种类，1 ac充电，2 usb充电 3 无线充电
  mBatteryLevel=100//电池电量百分比
  mBatteryLevelWhenDreamStarted=100//屏保开始时电量百分比
  mDockState=0//插入基座类型，0 没有基座，1 桌面基座 2 车载基座 3 低端模拟桌面基座 4 高端模拟桌面基座
  mStayOn=false//
  mProximityPositive=false//距离传感器得到正数结果
  mBootCompleted=true//开机动作是否完成
  mSystemReady=true//系统加载完成
  mHalAutoSuspendModeEnabled=true//？
  mHalInteractiveModeEnabled=false//？
  mWakeLockSummary=0x40//？
  mUserActivitySummary=0x4//？
  mRequestWaitForNegativeProximity=false//？
  mSandmanScheduled=false//？
  mSandmanSummoned=false//？
  mLowPowerModeEnabled=false//设备是否开启低电量模式
  mBatteryLevelLow=false//电量是否很低
  mLastWakeTime=71986209 (11539 ms ago)//最后一次电量屏的时间
  mLastSleepTime=71996210 (1538 ms ago)//最后一次睡眠的时间
  mLastUserActivityTime=71986209 (11539 ms ago)//最后一次显示activity时间
  mLastUserActivityTimeNoChangeLights=70050061 (1947687 ms ago)//最后一次显示activity时间
  mLastInteractivePowerHintTime=71986209 (11539 ms ago)//上一次电源提示时间
  mLastScreenBrightnessBoostTime=0 (71997748 ms ago)//上一次亮度改变的时间
  mScreenBrightnessBoostInProgress=false//？
  mDisplayReady=true//display加载完成
  mHoldingWakeLockSuspendBlocker=false//？
  mHoldingDisplaySuspendBlocker=false//？

Settings and Configuration:
  mDecoupleHalAutoSuspendModeFromDisplayConfig=false//？
  mDecoupleHalInteractiveModeFromDisplayConfig=true//？
  mWakeUpWhenPluggedOrUnpluggedConfig=true//插拔usb power是否亮屏
  mWakeUpWhenPluggedOrUnpluggedInTheaterModeConfig=false//插拔usb 是否可以从剧场模式唤醒
  mTheaterModeEnabled=false//剧场模式是否可用
  mSuspendWhenScreenOffDueToProximityConfig=true//近距离产生的屏灭是否允许刷新display暂停
  mDreamsSupportedConfig=true//是否支持屏保模式
  mDreamsEnabledByDefaultConfig=true//默认是否支持屏保模式
  mDreamsActivatedOnSleepByDefaultConfig=false//睡眠和充电下是否支持屏保模式
  mDreamsActivatedOnDockByDefaultConfig=true//？
  mDreamsEnabledOnBatteryConfig=false
  mDreamsBatteryLevelMinimumWhenPoweredConfig=-1
  mDreamsBatteryLevelMinimumWhenNotPoweredConfig=15
  mDreamsBatteryLevelDrainCutoffConfig=5
  mDreamsEnabledSetting=true
  mDreamsActivateOnSleepSetting=false
  mDreamsActivateOnDockSetting=true
  mDozeAfterScreenOffConfig=true
  mLowPowerModeSetting=false//当前是否时低电量模式
  mAutoLowPowerModeConfigured=false//当前是否允许自动启动低电量模式
  mAutoLowPowerModeSnoozing=false//用户是否关闭了低功耗模式
  mMinimumScreenOffTimeoutConfig=10000//屏幕最小timeout时间
  mMaximumScreenDimDurationConfig=7000//屏幕变暗持续的最长时间
  mMaximumScreenDimRatioConfig=0.20000005//屏幕变暗为20%
  mScreenOffTimeoutSetting=300000//屏幕timeout默认时间
  mSleepTimeoutSetting=-1//？
  mMaximumScreenOffTimeoutFromDeviceAdmin=2147483647 (enforced=false)//？
  mStayOnWhilePluggedInSetting=0//？
  mScreenBrightnessSetting=255//当前屏亮度
  mScreenAutoBrightnessAdjustmentSetting=0.0//自动亮度 -1 到1，0表示没有效果
  mScreenBrightnessModeSetting=0//屏亮度模式0，没有打开自动亮度，1 打开自动亮度
  mScreenBrightnessOverrideFromWindowManager=-1//是否允许前台应用修改屏亮度
  mUserActivityTimeoutOverrideFromWindowManager=0//是否允许前台应用修改屏timeout -1 不允许，0 允许
  mTemporaryScreenBrightnessSettingOverride=-1//是否允许设置应用，临时调整屏亮度，知道下次刷新 -1 为不允许
  mTemporaryScreenAutoBrightnessAdjustmentSettingOverride=NaN//设置自动调整自动亮度，直到下次刷新 NaN为不允许
  mDozeScreenStateOverrideFromDreamManager=1//打瞌睡 低功耗屏的状态 0 位置，1 关闭，2 屏亮
  mDozeScreenBrightnessOverrideFromDreamManager=-1//打瞌睡 低功耗时屏的亮度 -1 为灭屏
  mScreenBrightnessSettingMinimum=60//设置屏暗最小百分比
  mScreenBrightnessSettingMaximum=255//设置屏亮的最大百分比
  mScreenBrightnessSettingDefault=160//默认屏亮度
  
Sleep timeout: -1 ms
Screen off timeout: 10000 ms
Screen dim duration: 2000 ms

Wake Locks: size=1//ap 层wake lock 列表
  DOZE_WAKE_LOCK                 'DreamManagerService' (uid=1000, pid=757, ws=null)


Suspend Blockers: size=4
  PowerManagerService.WakeLocks: ref count=0
  PowerManagerService.Display: ref count=0
  PowerManagerService.Broadcasts: ref count=0
  PowerManagerService.WirelessChargerDetector: ref count=0


Wireless Charger Detector State://无线充电信息
  mGravitySensor=null
  mPoweredWirelessly=false
  mAtRest=false
  mRestX=0.0, mRestY=0.0, mRestZ=0.0
  mDetectionInProgress=false
  mDetectionStartTime=0 (never)
  mMustUpdateRestPosition=false
  mTotalSamples=0
  mMovingSamples=0
  mFirstSampleX=0.0, mFirstSampleY=0.0, mFirstSampleZ=0.0
  mLastSampleX=0.0, mLastSampleY=0.0, mLastSampleZ=0.0
```

