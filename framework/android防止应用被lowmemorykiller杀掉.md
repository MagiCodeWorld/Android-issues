系统会不断更新进程的adj值，然后在内存紧张的情况下，adj越大的应用越可能被杀，那么我们要防止被杀，要么是给我们的应用设置比较小的adj值，要么是要杀的时候过滤我们的应用，因为杀进程是比较偏底层做的，不太熟悉．所以优先考虑，系统计算adj值的时候直接给我们的应用adj值赋为-1.

　　直接说方法，系统计算过adj之后会通过下属方法写adj的值，我们只要在其中判断我们有应用的包名，然后更改adj的值就可以，该方法在AMS中

## 方法1：

```java
 private final boolean applyOomAdjLocked(ProcessRecord app, boolean wasKeeping,
            ProcessRecord TOP_APP, boolean doingAll, boolean reportingProcessState, long now) {
        boolean success = true;

        if (app.curRawAdj != app.setRawAdj) {
            if (wasKeeping && !app.keeping) {
                // This app is no longer something we want to keep.  Note
                // its current wake lock time to later know to kill it if
                // it is not behaving well.
                BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
                synchronized (stats) {
                    app.lastWakeTime = stats.getProcessWakeTime(app.info.uid,
                            app.pid, SystemClock.elapsedRealtime());
                }
                app.lastCpuTime = app.curCpuTime;
            }

            app.setRawAdj = app.curRawAdj;
        }

        //add for cmcc apk white list begin
        if(isInCmccWhiteList(app.processName) == true && app.curAdj != CMCC_ADJ) {
            app.curAdj = CMCC_ADJ;
            app.maxAdj = CMCC_ADJ;
        }
        //add for cmcc apk white list end

        //add for telecom apk white list begin
        if(isInTelecomWhiteList(app.processName) == true && app.curAdj != TELECOM_ADJ) {
            app.curAdj = TELECOM_ADJ;
            app.maxAdj = TELECOM_ADJ;
        }
        //add for telecom apk white list end

        if (app.curAdj != app.setAdj) {
            if (Process.setOomAdj(app.pid, app.curAdj)) {
                if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(
                    TAG, "Set " + app.pid + " " + app.processName +
                    " adj " + app.curAdj + ": " + app.adjType);
                app.setAdj = app.curAdj;
            } else {
                success = false;
                Slog.w(TAG, "Failed setting oom adj of " + app + " to " + app.curAdj);
            }
        }
     
    private boolean isInCmccWhiteList(String packageName) {
        if("shcmcc".equals(SystemProperties.get("ro.product.target"))) {
            for (int i = 0; i < mCmccWhiteList.length; i++) {
                if (packageName.equals(mCmccWhiteList[i])) {
                    return true;
                }
            }
        }

        return false;
    }
//add for shcmcc apk white list
    private String []mCmccWhiteList = new String[]{"net.sunniwell.inputmethod.swpinyin2",
                                      "net.sunniwell.service.swupgrade.chinamobile",
                                      "com.shcmcc.swdevicemanger",
                                      "com.cmcc.dlna",
                                      "net.sunniwell.service.controlserver",
                                      "com.chinamobile.sh.playservice",
                                      "com.shcmcc.setting",
                                      "com.android.chinamobile.migu.ott.ad",
                                      "com.android.multivideotest",
                                      "net.sunniwell.app.localplayer.sh"};
```



## 方法2：

```java
private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed) {
        boolean success = true;
        int changes = 0;
 
        if (app.curAdj != app.setAdj) {
        	String[] packages = app.getPackageList();
        	if(packages != null){
        		for(String p : packages){
        			if(p.equals("你的包名")){
        				//android.util.Log.d(TAG_OOM_ADJ, "set usettings adj -1");
        				app.curAdj = -1;
        				break;
        			}
 
        		}
        	}
            ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(TAG_OOM_ADJ,
                    "Set " + app.pid + " " + app.processName + " adj " + app.curAdj + ": "
                    + app.adjType);
            app.setAdj = app.curAdj;
            app.verifiedAdj = ProcessList.INVALID_ADJ;
        }
｝
```



## 方法3

```java
+++ b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@@ -18056,6 +18056,24 @@ public final class ActivityManagerService extends ActivityManagerNative
         }
     }
 
+    //银商特权应用，防止被lowmemkill掉 
+    final List<String> mUmsPrivilegedApplication = new ArrayList<String>(){{
+        add("com.ums.assitance.updater");//智能桌面升级助手
+        add("com.ums.tss.mastercontrol.daemon");//智能桌面守护进程
+        add("com.ums.tss.mastercontrol");//智能桌面
+        add("com.ums.upos.uapi");//银商U架构
+    }};  
+    
+    //List是有序的
+    final List<Integer>mUmsPrivilegedApplicationAdj = new ArrayList<Integer>(){
+        {
+          add(ProcessList.PREVIOUS_APP_ADJ); 
+          add(ProcessList.PREVIOUS_APP_ADJ); 
+          add(ProcessList.PREVIOUS_APP_ADJ); 
+          add(ProcessList.PREVIOUS_APP_ADJ); 
+        }
+    };
+
     private final boolean applyOomAdjLocked(ProcessRecord app,
             ProcessRecord TOP_APP, boolean doingAll, long now) {
         boolean success = true;
@@ -18067,11 +18085,37 @@ public final class ActivityManagerService extends ActivityManagerNative
         int changes = 0;
 
         if (app.curAdj != app.setAdj) {
-            ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
-            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(
-                TAG, "Set " + app.pid + " " + app.processName +
-                " adj " + app.curAdj + ": " + app.adjType);
-            app.setAdj = app.curAdj;
+           boolean isUmsAppWhiteProcess = false;
+           int mThirdPartyAdj = ProcessList.CACHED_APP_MIN_ADJ;
+           //paxsz 2018.12.17 add
+           if(mUmsPrivilegedApplication.size() == mUmsPrivilegedApplicationAdj.size()){
+               for(int i = 0; i < mUmsPrivilegedApplication.size(); i++){
+                   if((mUmsPrivilegedApplication.get(i).equals(app.processName)) && (app.curAdj > mUmsPrivilegedApplicationAdj.get(i)))
+                    {
+                          isUmsAppWhiteProcess = true;
+                          mThirdPartyAdj = mUmsPrivilegedApplicationAdj.get(i);
+                          break;
+                    }
+               }
+           }
+           if(isUmsAppWhiteProcess){
+               ProcessList.setOomAdj(app.pid, app.info.uid, mThirdPartyAdj);
+               if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(
+                   TAG, "Set " + app.pid + " " + app.processName +
+                   " adj " + mThirdPartyAdj + ": " + app.adjType);
+               app.setAdj = mThirdPartyAdj;
+           }else{
+               ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
+               if (DEBUG_SWITCH || DEBUG_OOM_ADJ) Slog.v(
+                   TAG, "Set " + app.pid + " " + app.processName +
+                   " adj " + app.curAdj + ": " + app.adjType);
+               app.setAdj = app.curAdj;
+           }
         }
```



## persistent应用启动

1 启动persistent应用
    在Android系统中，有一种永久性应用。它们对应的AndroidManifest.xml文件里，会将persistent属性设为true，比如：

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