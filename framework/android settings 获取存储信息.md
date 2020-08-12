应用对应的fragment为：

```xml
<span style="font-size:14px;"> <!-- Application Settings -->
    <header
        android:fragment="com.android.settings.applications.ManageApplications"
        android:icon="@drawable/ic_settings_applications"
        android:title="@string/applications_settings"
        android:id="@+id/application_settings" /></span>
```


因此需要查看ManageApplications如何实现。

ManageApplications所在路径为：packages\apps\Settings\src\com\android\settings\applications

从Application UI可以看出，fragment主要是一个tab，以及每一个tab下都会显示和存储相关的信息，比如RAM，SDCARD和内部存储空间的大小。接下来分析tab如何实现以及这些存储信息如何获取。

查看ManageApplications的onCreateView函数，可以看到：

```java
 View rootView = mInflater.inflate(R.layout.manage_applications_content,
                container, false);
```

这里会使用manage_applications_content.xml，我们查看xml的内容：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v4.view.ViewPager
		android:id="@+id/pager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_weight="1">

        <android.support.v4.view.PagerTabStrip
            android:id="@+id/tabs"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_gravity="top"
            android:textAppearance="@style/TextAppearance.PagerTabs"
            android:padding="0dp">
        </android.support.v4.view.PagerTabStrip>
    </android.support.v4.view.ViewPager>

</LinearLayout>
```


application fragment的布局本质就是一个ViewPager，因此可以支持左右滑动，每一页对应一个tab的显示。

```java
MyPagerAdapter adapter = new MyPagerAdapter();
mViewPager.setAdapter(adapter);
mViewPager.setOnPageChangeListener(adapter);
```


这里会设置一个adapter，用来填充ViewPager里的内容。

 

```java
class MyPagerAdapter extends PagerAdapter implements ViewPager.OnPageChangeListener {
	int mCurPos = 0;
	@Override
    public int getCount() {
        return mNumTabs;
    }
    
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        TabInfo tab = mTabs.get(position);
        View root = tab.build(mInflater, mContentContainer, mRootView);
        container.addView(root);
        root.setTag(R.id.name, tab);
        return root;
    }
 
    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView((View)object);
    }
 
    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }
 
    @Override
    public int getItemPosition(Object object) {
        return super.getItemPosition(object);
        //return ((TabInfo)((View)object).getTag(R.id.name)).mListType;
    }
 
    @Override
    public CharSequence getPageTitle(int position) {
        return mTabs.get(position).mLabel;
    }
 
    @Override
    public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
    }
 
    @Override
    public void onPageSelected(int position) {
        mCurPos = position;
    }
 
    @Override
    public void onPageScrollStateChanged(int state) {
        if (state == ViewPager.SCROLL_STATE_IDLE) {
            updateCurrentTab(mCurPos);
        }
    }
}
```
此adapter中instantiateItem函数会初始化每一个子项，即每一页：

```java
TabInfo tab = mTabs.get(position);
View root = tab.build(mInflater, mContentContainer, mRootView);
```


因此需要查看TabInfo中的build函数。



```java
public View build(LayoutInflater inflater, ViewGroup contentParent, View contentChild) {
	if (mRootView != null) {
		return mRootView;
	}
	mInflater = inflater;
	mRootView = inflater.inflate(mListType == LIST_TYPE_RUNNING
                ? R.layout.manage_applications_running
                : R.layout.manage_applications_apps, null);
        mLoadingContainer = mRootView.findViewById(R.id.loading_container);
        mLoadingContainer.setVisibility(View.VISIBLE);
        mListContainer = mRootView.findViewById(R.id.list_container);
        if (mListContainer != null) {
            // Create adapter and list view here
            View emptyView = mListContainer.findViewById(com.android.internal.R.id.empty);
            ListView lv = (ListView) mListContainer.findViewById(android.R.id.list);
            if (emptyView != null) {
                lv.setEmptyView(emptyView);
            }
            lv.setOnItemClickListener(this);
            lv.setSaveEnabled(true);
            lv.setItemsCanFocus(true);
            lv.setTextFilterEnabled(true);
            mListView = lv;
            mApplications = new ApplicationsAdapter(mApplicationsState, this, mFilter);
            mListView.setAdapter(mApplications);
            mListView.setRecyclerListener(mApplications);
            mColorBar = (LinearColorBar)mListContainer.findViewById(R.id.storage_color_bar);
            mStorageChartLabel = (TextView)mListContainer.findViewById(R.id.storageChartLabel);
            mUsedStorageText = (TextView)mListContainer.findViewById(R.id.usedStorageText);
            mFreeStorageText = (TextView)mListContainer.findViewById(R.id.freeStorageText);
            Utils.prepareCustomPreferencesList(contentParent, contentChild, mListView, false);
            if (mFilter == FILTER_APPS_SDCARD) {
                mStorageChartLabel.setText(mOwner.getActivity().getText(
                        R.string.sd_card_storage));
            } else {
                mStorageChartLabel.setText(mOwner.getActivity().getText(
                        R.string.internal_storage));
            }
            applyCurrentStorage();
        }
        mRunningProcessesView = (RunningProcessesView)mRootView.findViewById(
                R.id.running_processes);
        if (mRunningProcessesView != null) {
            mRunningProcessesView.doCreate(mSavedInstanceState);
        }
 
        return mRootView;
    }
```
此函数中用来显示tab中listview和tab下对应的storage显示信息。

对于tab中listView的填充：

```java
public View getView(int position, View convertView, ViewGroup parent) {
	// A ViewHolder keeps references to children views to avoid unnecessary calls
 	// to findViewById() on each row.
 	AppViewHolder holder = AppViewHolder.createOrRecycle(mTab.mInflater, convertView);
 	convertView = holder.rootView;
  // Bind the data efficiently with the holder
        ApplicationsState.AppEntry entry = mEntries.get(position);
        synchronized (entry) {
            holder.entry = entry;
            if (entry.label != null) {
                holder.appName.setText(entry.label);
            }
            mState.ensureIcon(entry);
            if (entry.icon != null) {
                holder.appIcon.setImageDrawable(entry.icon);
            }
            holder.updateSizeText(mTab.mInvalidSizeStr, mWhichSize);
            if ((entry.info.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
                holder.disabled.setVisibility(View.VISIBLE);
                holder.disabled.setText(R.string.not_installed);
            } else if (!entry.info.enabled) {
                holder.disabled.setVisibility(View.VISIBLE);
                holder.disabled.setText(R.string.disabled);
            } else {
                holder.disabled.setVisibility(View.GONE);
            }
            if (mFilterMode == FILTER_APPS_SDCARD) {
                holder.checkBox.setVisibility(View.VISIBLE);
                holder.checkBox.setChecked((entry.info.flags
                        & ApplicationInfo.FLAG_EXTERNAL_STORAGE) != 0);
            } else {
                holder.checkBox.setVisibility(View.GONE);
            }
        }
        mActive.remove(convertView);
        mActive.add(convertView);
        return convertView;
    }
```


对于storage的获取，需要使用到IMediaContainerService，绑定此service的地方：

getActivity().bindService(containerIntent, mContainerConnection, Context.BIND_AUTO_CREATE);


连接此service，每一个tab都会得到此service实例，通过此实例获取storage相关信息

 private volatile IMediaContainerService mContainerService;

    private final ServiceConnection mContainerConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mContainerService = IMediaContainerService.Stub.asInterface(service);
            for (int i=0; i<mTabs.size(); i++) {
                mTabs.get(i).setContainerService(mContainerService);
            }
        }
     
        @Override
        public void onServiceDisconnected(ComponentName name) {
            mContainerService = null;
        }
    };

## 获取storage信息代码如下：

获取SD卡：

```java
if (mFilter == FILTER_APPS_SDCARD) {
 	if (mContainerService != null) {
        try {
            final long[] stats = mContainerService.getFileSystemStats(
            Environment.getExternalStorageDirectory().getPath());
            mTotalStorage = stats[0];
            mFreeStorage = stats[1];
        } catch (RemoteException e) {
            Log.w(TAG, "Problem in container service", e);
        }
	}
	if (mApplications != null) {
 		final int N = mApplications.getCount();
 		for (int i=0; i<N; i++) {
 			ApplicationsState.AppEntry ae = mApplications.getAppEntry(i);
 			mAppStorage += ae.externalCodeSize + ae.externalDataSize
 						+ ae.externalCacheSize;
 		}
 	}
 }
```

## 获取本地存储空间信息：

```java
 if (mContainerService != null) {
 	try {
 		final long[] stats = mContainerService.getFileSystemStats(
 		Environment.getDataDirectory().getPath());
 		mTotalStorage = stats[0];
 		mFreeStorage = stats[1];
 	} catch (RemoteException e) {
 		Log.w(TAG, "Problem in container service", e);
 	}
}
```

mContainerService.getFileSystemStats实现：

frameworks/base/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java

```java
 @Override
public long[] getFileSystemStats(String path) {
	Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

 	try {
 		final StructStatVfs stat = Libcore.os.statvfs(path);
 		final long totalSize = stat.f_blocks * stat.f_bsize;
 		final long availSize = stat.f_bavail * stat.f_bsize;
 		return new long[] { totalSize, availSize };
 	} catch (ErrnoException e) {
 		throw new IllegalStateException(e);
 	}
}
```

最后显示到UI上去：

```java
void applyCurrentStorage() {
 	// If view hierarchy is not yet created, no views to update.
 	if (mRootView == null) {
 		return;
 	}
 	if (mTotalStorage > 0) {
 		BidiFormatter bidiFormatter = BidiFormatter.getInstance();
 		mColorBar.setRatios((mTotalStorage-mFreeStorage-mAppStorage)/(float)mTotalStorage,
 		mAppStorage/(float)mTotalStorage, mFreeStorage/(float)mTotalStorage);
 		long usedStorage = mTotalStorage - mFreeStorage;
 		if (mLastUsedStorage != usedStorage) {
 			mLastUsedStorage = usedStorage;
 			String sizeStr = bidiFormatter.unicodeWrap(
 			Formatter.formatShortFileSize(mOwner.getActivity(), usedStorage));
 			mUsedStorageText.setText(mOwner.getActivity().getResources().getString(
 				R.string.service_foreground_processes, sizeStr));
 		}
 		if (mLastFreeStorage != mFreeStorage) {
 			mLastFreeStorage = mFreeStorage;
 			String sizeStr = bidiFormatter.unicodeWrap(
 			Formatter.formatShortFileSize(mOwner.getActivity(), mFreeStorage));
 			mFreeStorageText.setText(mOwner.getActivity().getResources().getString(
 				R.string.service_background_processes, sizeStr));
 		}
 	} else {
 	mColorBar.setRatios(0, 0, 0);
            if (mLastUsedStorage != -1) {
                mLastUsedStorage = -1;
                mUsedStorageText.setText("");
            }
            if (mLastFreeStorage != -1) {
                mLastFreeStorage = -1;
                mFreeStorageText.setText("");
            }
        }
    }
```

当点击listView子项时，会跳转到详细列表页面：

```java
public void onItemClick(TabInfo tab, AdapterView<?> parent, View view, int position,
            long id) {
        if (tab.mApplications != null && tab.mApplications.getCount() > position) {
            ApplicationsState.AppEntry entry = tab.mApplications.getAppEntry(position);
            mCurrentPkgName = entry.info.packageName;
            startApplicationDetailsActivity();
        }
    }
  // utility method used to start sub activity
    private void startApplicationDetailsActivity() {
        // start new fragment to display extended information
        Bundle args = new Bundle();
        args.putString(InstalledAppDetails.ARG_PACKAGE_NAME, mCurrentPkgName);

        PreferenceActivity pa = (PreferenceActivity)getActivity();
        pa.startPreferencePanel(InstalledAppDetails.class.getName(), args,
                R.string.application_info_label, null, this, INSTALLED_APP_DETAILS);
    }


```

查看详细列表对应的类：

```
public class InstalledAppDetails extends Fragment
        implements View.OnClickListener, CompoundButton.OnCheckedChangeListener,
        ApplicationsState.Callbacks
```

即也是一个fragment。

 详细列表界面相关button操作代码如下：

```java
public void onClick(View v) {
        String packageName = mAppEntry.info.packageName;
        if(v == mUninstallButton) {
            if (mUpdatedSysApp) {
                showDialogInner(DLG_FACTORY_RESET, 0);
            } else {
                if ((mAppEntry.info.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                    if (mAppEntry.info.enabled) {
                        showDialogInner(DLG_DISABLE, 0);
                    } else {
                        new DisableChanger(this, mAppEntry.info,
                                PackageManager.COMPONENT_ENABLED_STATE_DEFAULT)
                        .execute((Object)null);
                    }
                } else if ((mAppEntry.info.flags & ApplicationInfo.FLAG_INSTALLED) == 0) {
                    uninstallPkg(packageName, true, false);
                } else {
                    uninstallPkg(packageName, false, false);
                }
            }
        } else if(v == mSpecialDisableButton) {
            showDialogInner(DLG_SPECIAL_DISABLE, 0);
        } else if(v == mActivitiesButton) {
            mPm.clearPackagePreferredActivities(packageName);
            try {
                mUsbManager.clearDefaults(packageName, UserHandle.myUserId());
            } catch (RemoteException e) {
                Log.e(TAG, "mUsbManager.clearDefaults", e);
            }
            mAppWidgetManager.setBindAppWidgetPermission(packageName, false);
            TextView autoLaunchTitleView =
                    (TextView) mRootView.findViewById(R.id.auto_launch_title);
            TextView autoLaunchView = (TextView) mRootView.findViewById(R.id.auto_launch);
            resetLaunchDefaultsUi(autoLaunchTitleView, autoLaunchView);
        } else if(v == mClearDataButton) {
            if (mAppEntry.info.manageSpaceActivityName != null) {
                if (!Utils.isMonkeyRunning()) {
                    Intent intent = new Intent(Intent.ACTION_DEFAULT);
                    intent.setClassName(mAppEntry.info.packageName,
                            mAppEntry.info.manageSpaceActivityName);
                    startActivityForResult(intent, REQUEST_MANAGE_SPACE);
                }
            } else {
                showDialogInner(DLG_CLEAR_DATA, 0);
            }
        } else if (v == mClearCacheButton) {
            // Lazy initialization of observer
            if (mClearCacheObserver == null) {
                mClearCacheObserver = new ClearCacheObserver();
            }
            mPm.deleteApplicationCacheFiles(packageName, mClearCacheObserver);
        } else if (v == mForceStopButton) {
            showDialogInner(DLG_FORCE_STOP, 0);
            //forceStopPackage(mAppInfo.packageName);
        } else if (v == mMoveAppButton) {
            if (mPackageMoveObserver == null) {
                mPackageMoveObserver = new PackageMoveObserver();
            }
            int moveFlags = (mAppEntry.info.flags & ApplicationInfo.FLAG_EXTERNAL_STORAGE) != 0 ?
                    PackageManager.MOVE_INTERNAL : PackageManager.MOVE_EXTERNAL_MEDIA;
            mMoveInProgress = true;
            refreshButtons();
            mPm.movePackage(mAppEntry.info.packageName, mPackageMoveObserver, moveFlags);
        }
    }
```

## mContainerService使用方法：

```java
private volatile IMediaContainerService mContainerService;

    private final ServiceConnection mContainerConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mContainerService = IMediaContainerService.Stub.asInterface(service);
            for (int i=0; i<mTabs.size(); i++) {
                mTabs.get(i).setContainerService(mContainerService);
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mContainerService = null;
        }
    };

final Intent containerIntent = new Intent().setComponent(
 	StorageMeasurement.DEFAULT_CONTAINER_COMPONENT);
	getActivity().bindService(containerIntent, mContainerConnection, Context.BIND_AUTO_CREATE);
```

AIDL接口：

frameworks/base/core/java/com/android/internal/app/IMediaContainerService.aidl

实现类：

frameworks/base/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java





————————————————
原文链接：https://blog.csdn.net/zhudaozhuan/article/details/40619371