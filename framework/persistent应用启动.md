# persistent应用启动

## 1 启动persistent应用

​    在Android系统中，有一种永久性应用。它们对应的AndroidManifest.xml文件里，会将persistent属性设为true，比如：

```xml
<application android:name="PhoneApp"
            android:persistent="true"
            android:label="@string/dialerIconLabel"
            android:icon="@drawable/ic_launcher_phone">
```

在系统启动之时，AMS的systemReady()会加载所有persistent为true的应用。

```java
public void systemReady(final Runnable goingCallback) 
{
	......
    try{
        List apps = AppGlobals.getPackageManager().
            getPersistentApplications(STOCK_PM_FLAGS);
        if(apps != null)
        {
            int N = apps.size();
            int i;
            for(i=0; i<N; i++) 
            {
                ApplicationInfo info = (ApplicationInfo)apps.get(i);
                if(info != null&& !info.packageName.equals("android"))
                {
                    addAppLocked(info,false);
                }
            }
         }
    }
    catch(RemoteException ex) {
        // pm is in same process, this will never happen.
    }
}
```

其中的STOCK_PM_FLAGS的定义如下：

```java
// The flags that are set for all calls we make to the package manager.
static final int STOCK_PM_FLAGS = PackageManager.GET_SHARED_LIBRARY_FILES;
```

上面代码中的getPersistentApplications()函数的定义如下：

```java
public List<ApplicationInfo> getPersistentApplications(int flags) 
{
    final ArrayList<ApplicationInfo> finalList = new ArrayList<ApplicationInfo>();
    // reader
    synchronized (mPackages) 
    {
        final Iterator<PackageParser.Package> i = mPackages.values().iterator();
        final int userId = UserId.getCallingUserId();
        while (i.hasNext()) 
        {
            final PackageParser.Package p = i.next();
            if (p.applicationInfo != null
                && (p.applicationInfo.flags & ApplicationInfo.FLAG_PERSISTENT) != 0
                && (!mSafeMode || isSystemApp(p))) 
            {
                PackageSetting ps = mSettings.mPackages.get(p.packageName);
                finalList.add(PackageParser.generateApplicationInfo(p, flags,
                        ps != null ? ps.getStopped(userId) : false,
                        ps != null ? ps.getEnabled(userId) : COMPONENT_ENABLED_STATE_DEFAULT, userId));
            }
        }
    }

    return finalList;
}
```
​		在PKMS中，有一个记录所有的程序包信息的哈希表（mPackages），每个表项中含有ApplicationInfo信息，该信息的flags（int型）数据中有一个专门的bit用于表示persistent。getPersistentApplications()函数会遍历这张表，找出所有persistent包，并返回ArrayList<ApplicationInfo>

​		在AMS中，所谓的“add App”主要是指“添加一个与App进程对应的ProcessRecord节点”。当然，如果该节点已经添加过了，那么是不会重复添加的。在添加节点的动作完成以后，addAppLocked()还会检查App进程是否已经启动好了，如果尚未开始启动，此时就会调用startProcessLocked()启动这个进程。既然addAppLocked()试图确认App“正在正常运作”或者“将被正常启动”，那么其对应的package就不可能处于stopped状态，这就是上面代码调用setPackageStoppedState(...,false,...)的意思。

​		现在，我们就清楚了，那些persistent属性为true的应用，基本上都是在系统启动伊始就启动起来的。

​		因为启动进程的过程是异步的，所以我们需要一个缓冲列表（即上面代码中的mPersistentStartingProcesses列表）来记录那些“正处于启动状态，而又没有启动完毕的”ProcessRecord结点。一旦目标进程启动完毕后，目标进程会attach系统，于是走到AMS的attachApplicationLocked()，在这个函数里，会把目标进程对应的ProcessRecord结点从mPersistentStartingProcesses缓冲列表里删除。

```java
private final boolean attachApplicationLocked(IApplicationThread thread, intpid) {
	// Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
	......
    thread.asBinder().linkToDeath(adr,0);
    ......
    thread.bindApplication(processName, appInfo, providers,
                app.instrumentationClass, profileFile, profileFd, profileAutoStop,
                app.instrumentationArguments, app.instrumentationWatcher, testMode,
                enableOpenGlTrace, isRestrictedBackupMode || !normalMode, 
                app.persistent,
                newConfiguration(mConfiguration), app.compat, 
                getCommonServicesLocked(),
                mCoreSettingsObserver.getCoreSettingsLocked());
    ......
    // Remove this record from the list of starting applications.
    mPersistentStartingProcesses.remove(app);
```
## 2 如何保证应用的持久性（persistent）

​		我们知道，persistent一词的意思是“持久”，那么persistent应用的意思又是什么呢？简单地说，这种应用会顽固地运行于系统之中，从系统一启动，一直到系统关机。

​		为了保证这种持久性，persistent应用必须能够在异常出现时，自动重新启动。在Android里是这样实现的。每个ActivityThread中会有一个专门和AMS通信的binder实体——final ApplicationThread mAppThread。这个实体在AMS中对应的代理接口为IApplicationThread。

​		当AMS执行到attachApplicationLocked()时，会针对目标用户进程的IApplicationThread接口，注册一个binder讣告监听器，一旦日后用户进程意外挂掉，AMS就能在第一时间感知到，并采取相应的措施。如果AMS发现意外挂掉的应用是persistent的，它会尝试重新启动这个应用。

注册监听器的代码如下：

```java
AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
thread.asBinder().linkToDeath(adr,0);
app.deathRecipient = adr;
```

其中的thread就是IApplicationThread代理。AppDeathRecipient的定义如下：

```java
private final class AppDeathRecipient implements IBinder.DeathRecipient
{
    final ProcessRecord mApp;
    final int mPid;
    final IApplicationThread mAppThread;

    AppDeathRecipient(ProcessRecord app, intpid,IApplicationThread thread) 
    {
        if(localLOGV)
            Slog.v(TAG,"New death recipient " + this+" for thread " + thread.asBinder());
        mApp = app;
        mPid = pid;
        mAppThread = thread;
    }

    public void binderDied() 
    {
        if(localLOGV)
            Slog.v(TAG,"Death received in " + this
                   +" for thread " + mAppThread.asBinder());
        synchronized(ActivityManagerService.this)
        {
            appDiedLocked(mApp, mPid, mAppThread);
        }
    }
}
```
当其监听的binder实体死亡时，系统会回调AppDeathRecipient的binderDied()。这个回调函数会辗转重启persistent应用，调用关系如下：

![20150102104749511](D:\myFile\Project\MarkDown\assets\20150102104749511.jpg)

​		一般情况下，当一个应用进程挂掉后，AMS当然会清理掉其对应的ProcessRecord，这就是cleanUpApplicationRecordLocked()的主要工作。然而，对于persistent应用，cleanUpApplicationRecordLocked() 会尝试再次启动对应的应用进程。代码截选如下：

```java

private final void cleanUpApplicationRecordLocked(ProcessRecord app,
                                                  boolean restarting,
                                                  boolean allowRestart,int index)
{
	......
    if (!app.persistent || app.isolated) 
    {
		......
        mProcessNames.remove(app.processName, app.uid);
        mIsolatedProcesses.remove(app.uid);
        ......
    }
    else if(!app.removed) 
    {
		if(mPersistentStartingProcesses.indexOf(app) < 0) {
            mPersistentStartingProcesses.add(app);
            restart = true;
        }
    }
    ......
    if (restart && !app.isolated) 
    {
        mProcessNames.put(app.processName, app.uid, app);
        startProcessLocked(app,"restart", app.processName);
    }
    else if(app.pid > 0&& app.pid != MY_PID) 
    {
		......
    }
	......
}
```

现在我们可以画一张关于“启动persistent应用”的示意图：

![20150102104807718](D:\myFile\Project\MarkDown\assets\20150102104807718.jpg)

 

## 3 补充知识点

3.1 persistent应用可以在系统未准备好时启动
		在AMS中，有一个isAllowedWhileBooting()函数，其代码如下：

```java
boolean isAllowedWhileBooting(ApplicationInfo ai) 
{
    return (ai.flags & ApplicationInfo.FLAG_PERSISTENT) != 0;
}
```

​		从这个函数可以看到，将persistent属性设为true的应用，是允许在boot的过程中启动的。我们可以查看前文提到的startProcessLocked()函数：

```java

final ProcessRecord startProcessLocked(String processName,
                                       ApplicationInfo info, boolean knownToBeDead,
                                       int intentFlags,
                                       String hostingType, ComponentName hostingName, 
                                       boolean allowWhileBooting,
                                       boolean isolated)
{
    ProcessRecord app;

    if(!isolated)
    {
        app = getProcessRecordLocked(processName, info.uid);
    }
    else
    {
        // If this is an isolated process, it can't re-use an existing process.
        app = null;
    }
    ......    
    if(!mProcessesReady
        && !isAllowedWhileBooting(info)
        && !allowWhileBooting) {
        if(!mProcessesOnHold.contains(app)) {
            mProcessesOnHold.add(app);
        }
        if(DEBUG_PROCESSES) Slog.v(TAG, "System not ready, putting on hold: " + app);
        return app;
    }
 
    startProcessLocked(app, hostingType, hostingNameStr);
    return (app.pid != 0) ? app : null;
}
```

其中的最后几句可以改写为以下更易理解的形式：

```java
if (mProcessesReady || isAllowedWhileBooting(info) || allowWhileBooting) 
{
    startProcessLocked(app, hostingType, hostingNameStr);
    return (app.pid != 0) ? app : null;
}
else
{
    . . . . . .
    return app;
}
```

也就是说，当系统已经处于以下几种情况时，多参数的startProcessLocked()会进一步调用另一个只有三个参数的startProcessLocked()：

1）系统已经处于ready状态；
2）想要启动persistent应用；
3）参数中明确指定可以在boot过程中启动应用。

​		补充说一下，一般情况下，当AMS调用startProcessLocked()时，传入的allowWhileBooting参数都为false。比如说，当系统需要启动“某个content provider或者某个service或者某个特定activity”时，此时传给startProcessLocked()的allowWhileBooting参数是写死为false的。只有一种特殊情况下会在该参数中传入true，那就是当系统发出的广播intent中携带有Intent.FLAG_RECEIVER_BOOT_UPGRADE标记时，此时允许在系统未ready时，启动接受广播的目标进程。