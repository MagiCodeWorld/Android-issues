# AppOpsManager 基本流程和使用方法

单个应用的权限管理需要使用到 AppOpsManager 的接口，接下来通过代码记录下：

AppOpsManager 是对外的管理接口，真正实现功能的是 AppOpsService。

##### 一、AppOpsManager的工作流程

###### 1、 AppOpsService 初始化：

```java
public ActivityManagerService(Context systemContext) {
        mContext = systemContext;
        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();

        ....
        // TODO: Move creation of battery stats service outside of activity manager service.
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();
        ....
        mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (mAppOpsService.checkOperation(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
        });
                
```

从以上代码可知应用信息的储存在“data/system/appops.xml”文件中，格式为：

```xml
<app-ops>
<pkg n="root">
<uid n="0" p="false">
<op n="34" t="1499140707282" pu="0" />
</uid>
</pkg>
<pkg n="com.mediatek.xxx">
<uid n="1000" p="false">
<op n="11" t="1499224813388" pu="0" />
<op n="15" m="0" />
<op n="59" t="1499224797843" pu="0" />
<op n="60" t="1499224797843" pu="0" />
<op n="71" t="1499224813389" pu="0" />
<op n="75" t="1499224813387" pu="0" />
<op n="76" t="1499224813387" pu="0" />
<op n="77" t="1499224813388" pu="0" />
</uid>
</pkg>
<pkg n="com.xxx.usercenter">
<uid n="1000" p="true">
<op n="0" t="1498721353479" pu="0" />
<op n="2" t="1498721359244" d="5787" />
<op n="10" t="1498721356900" pu="0" />
<op n="12" t="1498721353481" pu="0" />
<op n="41" t="1498721359246" d="5793" />
<op n="42" t="1498721359226" d="5769" />
<op n="59" t="1498721350806" pu="0" />
<op n="60" t="1498721350806" pu="0" />
<op n="72" m="0" t="1498721353877" pu="0" />
</uid>
</pkg>
```

初始化 AppOpsService 同时，解析appops.xml文件中的信息并储存在名为 UidState 的内部类中：

```tsx
    void readState() {
        synchronized (mFile) {
            synchronized (this) {
                FileInputStream stream;
                try {
                    stream = mFile.openRead();
                } catch (FileNotFoundException e) {
                    ....
                    return;
                }
                boolean success = false;
                mUidStates.clear();
                try {
                    XmlPullParser parser = Xml.newPullParser();
                    parser.setInput(stream, StandardCharsets.UTF_8.name());
                    int type;
                        ....
                        String tagName = parser.getName();
                        if (tagName.equals("pkg")) {
                            //读取每个应用权限信息，并保存至UidState.pkgOps变量
                            readPackage(parser);
                        } else if (tagName.equals("uid")) {
                            //读取每个应用对应的相关权限以及权限状态，并保存至UidState.opModes变量
                            readUidOps(parser);
                        } else {
                            Slog.w(TAG, "Unknown element under <app-ops>: "
                                    + parser.getName());
                            XmlUtils.skipCurrentTag(parser);
                        }
                    }
                    success = true;
                } catch (IllegalStateException e) {
                    ....
                }
            }
        }
    }

    void readPackage(XmlPullParser parser) throws NumberFormatException,
            XmlPullParserException, IOException {
        String pkgName = parser.getAttributeValue(null, "n");
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            ....
            String tagName = parser.getName();
            if (tagName.equals("uid")) {
                readUid(parser, pkgName);
            } else {
              ....
            }
        }
    }

    void readUid(XmlPullParser parser, String pkgName) throws NumberFormatException,
            XmlPullParserException, IOException {
        int uid = Integer.parseInt(parser.getAttributeValue(null, "n"));
        String isPrivilegedString = parser.getAttributeValue(null, "p");
        boolean isPrivileged = false;
        if (isPrivilegedString == null) {
            try {
                IPackageManager packageManager = ActivityThread.getPackageManager();
                if (packageManager != null) {
                    ApplicationInfo appInfo = ActivityThread.getPackageManager()
                            .getApplicationInfo(pkgName, 0, UserHandle.getUserId(uid));
                    if (appInfo != null) {
                        isPrivileged = (appInfo.privateFlags
                                & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
                    }
                } else {
                    // Could not load data, don't add to cache so it will be loaded later.
                    return;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Could not contact PackageManager", e);
            }
        } else {
            isPrivileged = Boolean.parseBoolean(isPrivilegedString);
        }
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("op")) {
                Op op = new Op(uid, pkgName, Integer.parseInt(parser.getAttributeValue(null, "n")));
                String mode = parser.getAttributeValue(null, "m");
                if (mode != null) {
                    op.mode = Integer.parseInt(mode);
                }
                String time = parser.getAttributeValue(null, "t");
                if (time != null) {
                    op.time = Long.parseLong(time);
                }
                time = parser.getAttributeValue(null, "r");
                if (time != null) {
                    op.rejectTime = Long.parseLong(time);
                }
                String dur = parser.getAttributeValue(null, "d");
                if (dur != null) {
                    op.duration = Integer.parseInt(dur);
                }
                String sum = parser.getAttributeValue(null, "sum");
                if (sum != null) {
                    op.mSum = Integer.parseInt(sum);
                }
                String proxyUid = parser.getAttributeValue(null, "pu");
                if (proxyUid != null) {
                    op.proxyUid = Integer.parseInt(proxyUid);
                }
                String proxyPackageName = parser.getAttributeValue(null, "pp");
                if (proxyPackageName != null) {
                    op.proxyPackageName = proxyPackageName;
                }

                UidState uidState = getUidStateLocked(uid, true);
                if (uidState.pkgOps == null) {
                    uidState.pkgOps = new ArrayMap<>();
                }

                Ops ops = uidState.pkgOps.get(pkgName);
                if (ops == null) {
                    ops = new Ops(pkgName, uidState, isPrivileged);
                    uidState.pkgOps.put(pkgName, ops);
                }
                ops.put(op.op, op);
            } else {
                Slog.w(TAG, "Unknown element under <pkg>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }


 void readUidOps(XmlPullParser parser) throws NumberFormatException,
            XmlPullParserException, IOException {
        final int uid = Integer.parseInt(parser.getAttributeValue(null, "n"));
        int outerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("op")) {
                //code对应AppopsManager中的OP值
                final int code = Integer.parseInt(parser.getAttributeValue(null, "n"));
                //mode 对应AppopsManager中权限状态，有0 1 2 3四种状态
                final int mode = Integer.parseInt(parser.getAttributeValue(null, "m"));
                UidState uidState = getUidStateLocked(uid, true);
                if (uidState.opModes == null) {
                    uidState.opModes = new SparseIntArray();
                }
                uidState.opModes.put(code, mode);
            } else {
                Slog.w(TAG, "Unknown element under <uid-ops>: "
                        + parser.getName());
                XmlUtils.skipCurrentTag(parser);
            }
        }
    }
```

###### 2、 AppOps 权限状态设置流程：

AppOpsManager：

```cpp
    /** @hide */
    public void setMode(int code, int uid, String packageName, int mode) {
        try {
            mService.setMode(code, uid, packageName, mode);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

AppOpsService：

```java
    @Override
    public void setMode(int code, int uid, String packageName, int mode) {
        /// M: Log enhancement @{
        String callingApp = mContext.getPackageManager().getNameForUid(Binder.getCallingUid());
      
        if (Binder.getCallingPid() != Process.myPid()) {
            mContext.enforcePermission(android.Manifest.permission.UPDATE_APP_OPS_STATS,
                    Binder.getCallingPid(), Binder.getCallingUid(), null);
        }
        //验证code值，不在AppOpsManager._NUM_OP范围内会直接抛异常
        verifyIncomingOp(code);
        ArrayList<Callback> repCbs = null;
        code = AppOpsManager.opToSwitch(code);
        synchronized (this) {
            //通过UID去查找UidState 、Op 对象，如果没有就创建一个新的。
            UidState uidState = getUidStateLocked(uid, false);
            Op op = getOpLocked(code, uid, packageName, true);
            if (op != null) {
                //判断是否需要更新appops.xml
                if (op.mode != mode) {
                    op.mode = mode;
                    ....
                    //将app权限信息写入 appops.xml 文件
                    scheduleFastWriteLocked();
                }
            }
        }
       .....
    }

    private void scheduleFastWriteLocked() {
        if (!mFastWriteScheduled) {
            mWriteScheduled = true;
            mFastWriteScheduled = true;
            mHandler.removeCallbacks(mWriteRunner);
            mHandler.postDelayed(mWriteRunner, 10*1000);
        }
    }
```

mWriteRunner 中通过异步执行了writeState()的写入操作，将更新后的UidState 对象信息重新写入appops.xml中。

###### 2、 AppOps 权限状态读取流程：

<br />
AppOpsManager 中常用的读取方法有四种：

[checkOp](file:///D:/AndroidStudioSDK/docs/reference/android/app/AppOpsManager.html#checkOp%28java.lang.String,%20int,%20java.lang.String%29)([String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) op, int uid, [String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) packageName)

只读取不做记录，权限不通过会抛异常

[noteOp](file:///D:/AndroidStudioSDK/docs/reference/android/app/AppOpsManager.html#noteOp%28java.lang.String,%20int,%20java.lang.String%29)([String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) op, int uid, [String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) packageName)

和checkOp基本相同，但是在检验后会做记录。

[checkOpNoThrow](file:///D:/AndroidStudioSDK/docs/reference/android/app/AppOpsManager.html#checkOpNoThrow%28java.lang.String,%20int,%20java.lang.String%29)([String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) op, int uid, [String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) packageName)

和checkOp类似，但是权限错误，不会抛出SecurityException，而是返回AppOpsManager.MODE_ERRORED.

[noteOpNoThrow](file:///D:/AndroidStudioSDK/docs/reference/android/app/AppOpsManager.html#noteOpNoThrow%28java.lang.String,%20int,%20java.lang.String%29)([String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) op, int uid, [String](file:///D:/AndroidStudioSDK/docs/reference/java/lang/String.html) packageName)

类似noteOp，但不会抛出SecurityException。

以 noteOp 来记录下读取的流程：

AppOpsManager：

```cpp
public int noteOp(int op, int uid, String packageName) {
        try {
            int mode = mService.noteOperation(op, uid, packageName);
            //如果权限不通过直接抛异常
            if (mode == MODE_ERRORED) {
                throw new SecurityException(buildSecurityExceptionMsg(op, uid, packageName));
            }
            return mode;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

AppOpsService：



```dart
 @Override
    public int noteOperation(int code, int uid, String packageName) {
        verifyIncomingUid(uid);
        verifyIncomingOp(code);
        String resolvedPackageName = resolvePackageName(uid, packageName);
        if (resolvedPackageName == null) {
            return AppOpsManager.MODE_IGNORED;
        }
        return noteOperationUnchecked(code, uid, resolvedPackageName, 0, null);
    }

    private int noteOperationUnchecked(int code, int uid, String packageName,
            int proxyUid, String proxyPackageName) {
        synchronized (this) {
            Ops ops = getOpsRawLocked(uid, packageName, true);
            if (ops == null) {
                ....
                return AppOpsManager.MODE_ERRORED;
            }
            //这里会对op 对象的变量重新设值，最终会打印在appops.xml中
            Op op = getOpLocked(ops, code, true);
            if (isOpRestricted(uid, code, packageName)) {
                return AppOpsManager.MODE_IGNORED;
            }
            if (op.duration == -1) {
                  ....
            }
            op.duration = 0;
            final int switchCode = AppOpsManager.opToSwitch(code);
            UidState uidState = ops.uidState;
            // If there is a non-default per UID policy (we set UID op mode only if
            // non-default) it takes over, otherwise use the per package policy.
            if (uidState.opModes != null && uidState.opModes.indexOfKey(switchCode) >= 0) {
                final int uidMode = uidState.opModes.get(switchCode);
                if (uidMode != AppOpsManager.MODE_ALLOWED) {
                    if (!operationFallBackCheck(uidState, switchCode, uid, packageName)) {
                        op.rejectTime = System.currentTimeMillis();
                        op.mSum ++;
                        return uidMode;
                    }
                }
            } else {
                final Op switchOp = switchCode != code ? getOpLocked(ops, switchCode, true) : op;
                if (switchOp.mode != AppOpsManager.MODE_ALLOWED) {
                    if (!operationFallBackCheck(ops, switchOp.op, uid, packageName)) {

                        op.rejectTime = System.currentTimeMillis();
                        op.mSum ++;
                        return switchOp.mode;
                    }
                }
            }
            op.time = System.currentTimeMillis();
            op.rejectTime = 0;
            op.proxyUid = proxyUid;
            op.proxyPackageName = proxyPackageName;
            return AppOpsManager.MODE_ALLOWED;
        }
    }
```

##### 二、AppOpsManager的使用方法

###### 1.添加静态变量

1.1 添加op：

```java
    /** @hide */
    public static final int OP_BOOT_COMPLETED = 69;
```

1.2 修改_NUM_OP（op总数+1）：

```java
    /** @hide */
    public static final int _NUM_OP = 70;
```

1.3 添加OP_STR：

```dart
    private static final String OPSTR_BOOT_COMPLETED =
            "android:boot_completed";
```

###### 2.在op静态数组中添加新增op

初始化时会对每个数组的数量进行check，如果与_NUM_OP 值不一致，就会抛异常，会导致开不了机。

```java
    private static final int[] RUNTIME_PERMISSIONS_OPS = {
        ....
        OP_BOOT_COMPLETED,
    }

    private static int[] sOpToSwitch = new int[] {
        ....
        OP_BOOT_COMPLETED,
    }

    private static int[] sOpToSwitchCta = new int[] {
        ....
        OP_BOOT_COMPLETED,
    }

    private static String[] sOpToString = new String[] {
        ....
        OPSTR_BOOT_COMPLETED,
    };

    private static String[] sOpNames = new String[] {
        ....
        "BOOT_COMPLETED",
    };

    private static String[] sOpPerms = new String[] {
        ....
        Manifest.permission.RECEIVE_BOOT_COMPLETED,
    };

    private static String[] sOpRestrictions = new String[] {
        ....
        null, //BOOT_COMPLETED
    };

    private static boolean[] sOpAllowSystemRestrictionBypass = new boolean[] {
        ....
        true, // BOOT_COMPLETED
    };

    private static int[] sOpDefaultMode = new int[] {
        ....
        AppOpsManager.MODE_IGNORED, // OP_BOOT_COMPLETED     
    };

    private static boolean[] sOpDisableReset = new boolean[] {
        ....
        false,     // OP_BOOT_COMPLETED
    };
```

以上已经将需要添加的op代码添加完毕，接下来需要在系统模块中使用。

###### 3.在系统模块中使用

3.1 设置权限

```css
mAppOpsService.setMode(AppOpsManager.OP_BOOT_COMPLETED, uid, packageName, AppOpsManager.MODE_ALLOWED);
```

3.2 判断权限状态

```undefined
 if (mAppOps.noteOpNoThrow(AppOpsManager.OP_POST_NOTIFICATION, uid, pkg)
                != AppOpsManager.MODE_ALLOWED) {
        ....
 }
```

https://www.jianshu.com/nb/13197250)