​												android settings存储代码分析

存储模块所在的fragment为：


```xml
<!-- Storage -->
<header
    android:id="@+id/storage_settings"
    android:fragment="com.android.settings.deviceinfo.Memory"
    android:icon="@drawable/ic_settings_storage"
    android:title="@string/storage_settings" />
```

我们现在看Memory这个类:
```java
@Override
public void onCreate(Bundle icicle) {
	super.onCreate(icicle);
    final Context context = getActivity();

    mUsbManager = (UsbManager) getSystemService(Context.USB_SERVICE);
 
    mStorageManager = StorageManager.from(context);
    mStorageManager.registerListener(mStorageListener);
 
    addPreferencesFromResource(R.xml.device_info_memory);
 
    addCategory(StorageVolumePreferenceCategory.buildForInternal(context));
 
    final StorageVolume[] storageVolumes = mStorageManager.getVolumeList();
    for (StorageVolume volume : storageVolumes) {
        if (!volume.isEmulated()) {
            addCategory(StorageVolumePreferenceCategory.buildForPhysical(context, volume));
        }
    }
    setHasOptionsMenu(true);
}
```

在onCreate函数中，主要做了几件事情:

1.各种初始化

2.实例化布局，最主要是对Category的添加

3.获取当前挂载的volume，并且实例化为Category

 

Category最主要的是StorageVolumePreferenceCategory，构造函数如下：

 

```java
  /**
     * Build category to summarize specific physical {@link StorageVolume}.
     */
public static StorageVolumePreferenceCategory buildForPhysical(
	Context context, StorageVolume volume) {
	return new StorageVolumePreferenceCategory(context, volume);
}

private StorageVolumePreferenceCategory(Context context, StorageVolume volume) {
    super(context);
 
    mVolume = volume;
    mMeasure = StorageMeasurement.getInstance(context, volume);
 
    mResources = context.getResources();
    mStorageManager = StorageManager.from(context);
    mUserManager = (UserManager) context.getSystemService(Context.USER_SERVICE);
 
    setTitle(volume != null ? volume.getDescription(context)
            : context.getText(R.string.internal_storage));
}
```
构造函数会被如下函数调用：

```java
/**
 * Build category to summarize internal storage, including any emulated
 * {@link StorageVolume}.
 */
public static StorageVolumePreferenceCategory buildForInternal(Context context) {
    return new StorageVolumePreferenceCategory(context, null);
}
```
到这里，主要是在Category里面做一些初始化，对于存储fragment最上面的“内部存储设备”text显示就是在构造函数中完成：

```java
setTitle(volume != null ? volume.getDescription(context)
                : context.getText(R.string.internal_storage));
```

对于外置设备Category的创建，主要是在：

```java
/**
   * Build category to summarize specific physical {@link StorageVolume}.
   */
public static StorageVolumePreferenceCategory buildForPhysical(
     Context context, StorageVolume volume) {
     return new StorageVolumePreferenceCategory(context, volume);
}
```

唯一的区别就是Volume是否为NULL。

创建Category后，主要是对preference的创建，主要是在init函数：

```java
public void init() {
	final Context context = getContext();
	removeAll();
 
    final UserInfo currentUser;
    try {
        currentUser = ActivityManagerNative.getDefault().getCurrentUser();
    } catch (RemoteException e) {
        throw new RuntimeException("Failed to get current user");
    }
 
    final List<UserInfo> otherUsers = getUsersExcluding(currentUser);
    final boolean showUsers = mVolume == null && otherUsers.size() > 0;
 
    mUsageBarPreference = new UsageBarPreference(context);
    mUsageBarPreference.setOrder(ORDER_USAGE_BAR);
    addPreference(mUsageBarPreference);
 
    mItemTotal = buildItem(R.string.memory_size, 0);
    mItemAvailable = buildItem(R.string.memory_available, R.color.memory_avail);
    addPreference(mItemTotal);
    addPreference(mItemAvailable);
 
    mItemApps = buildItem(R.string.memory_apps_usage, R.color.memory_apps_usage);
    mItemDcim = buildItem(R.string.memory_dcim_usage, R.color.memory_dcim);
    mItemMusic = buildItem(R.string.memory_music_usage, R.color.memory_music);
    mItemDownloads = buildItem(R.string.memory_downloads_usage, R.color.memory_downloads);
    mItemCache = buildItem(R.string.memory_media_cache_usage, R.color.memory_cache);
    mItemMisc = buildItem(R.string.memory_media_misc_usage, R.color.memory_misc);
 
    mItemCache.setKey(KEY_CACHE);
 
    final boolean showDetails = mVolume == null || mVolume.isPrimary();
    if (showDetails) {
        if (showUsers) {
            addPreference(new PreferenceHeader(context, currentUser.name));
        }
 
        addPreference(mItemApps);
        addPreference(mItemDcim);
        addPreference(mItemMusic);
        addPreference(mItemDownloads);
        addPreference(mItemCache);
        addPreference(mItemMisc);
 
        if (showUsers) {
            addPreference(new PreferenceHeader(context, R.string.storage_other_users));
 
            int count = 0;
            for (UserInfo info : otherUsers) {
                final int colorRes = count++ % 2 == 0 ? R.color.memory_user_light
                        : R.color.memory_user_dark;
                final StorageItemPreference userPref = new StorageItemPreference(
                        getContext(), info.name, colorRes, info.id);
                mItemUsers.add(userPref);
                addPreference(userPref);
            }
        }
    }
 
    final boolean isRemovable = mVolume != null ? mVolume.isRemovable() : false;
    // Always create the preference since many code rely on it existing
    mMountTogglePreference = new Preference(context);
    if (isRemovable) {
        mMountTogglePreference.setTitle(R.string.sd_eject);
        mMountTogglePreference.setSummary(R.string.sd_eject_summary);
        addPreference(mMountTogglePreference);
    }
 
    final boolean allowFormat = mVolume != null;
    if (allowFormat) {
        mFormatPreference = new Preference(context);
        mFormatPreference.setTitle(R.string.sd_format);
        mFormatPreference.setSummary(R.string.sd_format_summary);
        addPreference(mFormatPreference);
    }
 
    final IPackageManager pm = ActivityThread.getPackageManager();
    try {
        if (pm.isStorageLow()) {
            mStorageLow = new Preference(context);
            mStorageLow.setOrder(ORDER_STORAGE_LOW);
            mStorageLow.setTitle(R.string.storage_low_title);
            mStorageLow.setSummary(R.string.storage_low_summary);
            addPreference(mStorageLow);
        } else if (mStorageLow != null) {
            removePreference(mStorageLow);
            mStorageLow = null;
        }
    } catch (RemoteException e) {
    }
}
```
对preference数据进行更新是在：

```java
public void updateApproximate(long totalSize, long availSize) {
    mItemTotal.setSummary(formatSize(totalSize));
    mItemAvailable.setSummary(formatSize(availSize));
 
    mTotalSize = totalSize;
 
    final long usedSize = totalSize - availSize;
 
    mUsageBarPreference.clear();
    mUsageBarPreference.addEntry(0, usedSize / (float) totalSize, android.graphics.Color.GRAY);
    mUsageBarPreference.commit();
 
    updatePreferencesFromState();
}
```

当点击cache prefrence时，会弹出dialog，主要是在Memory.java中响应：

```java
public boolean onPreferenceTreeClick(PreferenceScreen preferenceScreen, Preference preference) {
	if (StorageVolumePreferenceCategory.KEY_CACHE.equals(preference.getKey())) {
		ConfirmClearCacheFragment.show(this);
		return true;
    }
}
```

查看ConfirmClearCacheFragment的函数：

```java
 @Override
public Dialog onCreateDialog(Bundle savedInstanceState) {
	final Context context = getActivity();
 
	final AlertDialog.Builder builder = new AlertDialog.Builder(context);
	builder.setTitle(R.string.memory_clear_cache_title);
	builder.setMessage(getString(R.string.memory_clear_cache_message));
 
	builder.setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() 	{
		@Override
		public void onClick(DialogInterface dialog, int which) {
			final Memory target = (Memory) getTargetFragment();
            final PackageManager pm = context.getPackageManager();
            final List<PackageInfo> infos = pm.getInstalledPackages(0);
			final ClearCacheObserver observer = new ClearCacheObserver(target, infos.size());
			for (PackageInfo info : infos) {
				pm.deleteApplicationCacheFiles(info.packageName, observer);
			}
		}
	});
	builder.setNegativeButton(android.R.string.cancel, null);
 
	return builder.create();
}
```
会通过PackageManager获取所有安装apk，然后清除所有apk的缓存数据。

点击“卸载SD卡”，会弹出dialog，对应的代码为：

```java
private void unmount() {
	// Check if external media is in use.
	try {
		if (hasAppsAccessingStorage()) {
			// Present dialog to user
			showDialogInner(DLG_CONFIRM_UNMOUNT);
		} else {
			doUnmount();
		}
	} catch (RemoteException e) {
		// Very unlikely. But present an error dialog anyway
		Log.e(TAG, "Is MountService running?");
		showDialogInner(DLG_ERROR_UNMOUNT);
	}
}
    
@Override
public Dialog onCreateDialog(int id) {
	switch (id) {
		case DLG_CONFIRM_UNMOUNT:
			return new AlertDialog.Builder(getActivity())
				.setTitle(R.string.dlg_confirm_unmount_title)
              .setPositiveButton(R.string.dlg_ok, new DialogInterface.OnClickListener() {
				public void onClick(DialogInterface dialog, int which) {
					doUnmount();
				}})
                    .setNegativeButton(R.string.cancel, null)
                    .setMessage(R.string.dlg_confirm_unmount_text)
                    .create();
 private void doUnmount() {
        // Present a toast here
        Toast.makeText(getActivity(), R.string.unmount_inform_text, Toast.LENGTH_SHORT).show();
        IMountService mountService = getMountService();
        try {
            sLastClickedMountToggle.setEnabled(false);
            sLastClickedMountToggle.setTitle(getString(R.string.sd_ejecting_title));
            sLastClickedMountToggle.setSummary(getString(R.string.sd_ejecting_summary));
            mountService.unmountVolume(sClickedMountPoint, true, false);
        } catch (RemoteException e) {
            // Informative dialog to user that unmount failed.
            showDialogInner(DLG_ERROR_UNMOUNT);
        }
    }
```


原文链接：https://blog.csdn.net/zhudaozhuan/article/details/40621335/