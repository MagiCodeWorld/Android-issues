                                                     Android 将后台应用切换到前台

## 方法1： moveTaskToFront

1、布局文件 activity_main.xml 内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
    <Button
        android:id="@+id/btnStart"
        android:layout_width="match_parent"
        android:layout_height="64dp"
        android:text="开 始" />
</android.support.constraint.ConstraintLayout>
```

2、自定义的系统帮助类 SystemHelper.java 内容如下：

```java
package com.lct.www.xiong.helper;
 
import android.app.ActivityManager;
import android.content.Context;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.util.Log;
 
import java.util.List;
 
import static android.content.Context.ACTIVITY_SERVICE;
 
/**
 * 系统帮助类
 */
public class SystemHelper {
    /**
     * 判断本地是否已经安装好了指定的应用程序包
     *
     * @param packageNameTarget ：待判断的 App 包名，如 微博 com.sina.weibo
     * @return 已安装时返回 true,不存在时返回 false
     */
    public static boolean appIsExist(Context context, String packageNameTarget) {
        if (!"".equals(packageNameTarget.trim())) {
            PackageManager packageManager = context.getPackageManager();
            List<PackageInfo> packageInfoList = packageManager.getInstalledPackages(PackageManager.MATCH_UNINSTALLED_PACKAGES);
            for (PackageInfo packageInfo : packageInfoList) {
                String packageNameSource = packageInfo.packageName;
                if (packageNameSource.equals(packageNameTarget)) {
                    return true;
                }
            }
        }
        return false;
    }
 
    /**
     * 将本应用置顶到最前端
     * 当本应用位于后台时，则将它切换到最前端
     *
     * @param context
     */
    public static void setTopApp(Context context) {
        if (!isRunningForeground(context)) {
            /**获取ActivityManager*/
            ActivityManager activityManager = (ActivityManager) context.getSystemService(ACTIVITY_SERVICE);
 
            /**获得当前运行的task(任务)*/
            List<ActivityManager.RunningTaskInfo> taskInfoList = activityManager.getRunningTasks(100);
            for (ActivityManager.RunningTaskInfo taskInfo : taskInfoList) {
                /**找到本应用的 task，并将它切换到前台*/
                if (taskInfo.topActivity.getPackageName().equals(context.getPackageName())) {
                    activityManager.moveTaskToFront(taskInfo.id, 0);
                    //activityManager.moveTaskToFront(getTaskId(), ActivityManager.MOVE_TASK_WITH_HOME);
                    break;
                }
            }
        }
    }
 
    /**
     * 判断本应用是否已经位于最前端
     *
     * @param context
     * @return 本应用已经位于最前端时，返回 true；否则返回 false
     */
    public static boolean isRunningForeground(Context context) {
        ActivityManager activityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        List<ActivityManager.RunningAppProcessInfo> appProcessInfoList = activityManager.getRunningAppProcesses();
        /**枚举进程*/
        for (ActivityManager.RunningAppProcessInfo appProcessInfo : appProcessInfoList) {
            if (appProcessInfo.importance == ActivityManager.RunningAppProcessInfo.IMPORTANCE_FOREGROUND) {
                if (appProcessInfo.processName.equals(context.getApplicationInfo().processName)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

3、主活动 MainActivity.java 的内容如下：

```java
package com.lct.www.xiong;
 
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;
 
import com.lct.www.xiong.helper.SystemHelper;
 
public class MainActivity extends AppCompatActivity {
 
    /**
     * buttonStart：开始按钮
     */
    private Button buttonStart;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
 
        bindView();
    }
 
    private void bindView() {
        /**
         * 为开始按钮绑定但即使事件
         */
        buttonStart = findViewById(R.id.btnStart);
        buttonStart.setOnClickListener(new Button.OnClickListener() {
 
            @Override
            public void onClick(View v) {
                try {
                    Log.i("Wmx Logs::", "开始按钮被点击了 id = " + v.getId() + "线程 = " + Thread.currentThread().getName());
 
                    /**
                     * 启动手机上 微博 APP (包名 com.sina.weibo)
                     * 休眠 10 秒
                     */
                    startLocalApp("com.sina.weibo");
                    Thread.sleep(10000);
 
                    /**最后将被挤压到后台的本应用重新置顶到最前端
                     * 当自己的应用在后台时，将它切换到前台来*/
                    SystemHelper.setTopApp(MainActivity.this);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
 
    /**
     * 启动本地安装好的第三方 APP
     * 注意：此种当时启动第三方 APP 时，如果第三方 APP 当时没有运行，则会启动它
     * 如果被启动的 APP 本身已经在运行，则直接将它从后台切换到最前端
     *
     * @param packageNameTarget :App 包名、如
     *                          微博 com.sina.weibo、
     *                          飞猪 com.taobao.trip、
     *                          QQ com.tencent.mobileqq、
     *                          腾讯新闻 com.tencent.news
     */
    private void startLocalApp(String packageNameTarget) {
 
        Log.i("Wmx logs::", "-----------------------开始启动第三方 APP=" + packageNameTarget);
 
        if (SystemHelper.appIsExist(MainActivity.this, packageNameTarget)) {
            PackageManager packageManager = getPackageManager();
            Intent intent = packageManager.getLaunchIntentForPackage(packageNameTarget);
            intent.addCategory(Intent.CATEGORY_LAUNCHER);
            intent.setFlags(Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED | Intent.FLAG_ACTIVITY_NEW_TASK);
 
            /**android.intent.action.MAIN：打开另一程序
             */
            intent.setAction("android.intent.action.MAIN");
            /**
             * FLAG_ACTIVITY_SINGLE_TOP:
             * 如果当前栈顶的activity就是要启动的activity,则不会再启动一个新的activity
             */
            intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
            startActivity(intent);
        } else {
            Toast.makeText(getApplicationContext(), "被启动的 APP 未安装", Toast.LENGTH_SHORT).show();
        }
    }
}
```

4、Android 系统中如果想要切换系统中的任务，是需要获取系统权限的，在全局配置文件中添加：

```xml
 <uses-permission android:name="android.permission.REORDER_TASKS" />

<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.lct.www.xiong">
 
    <!--排序系统任务权限	重新排序系统Z轴运行中的任务-->
    <uses-permission android:name="android.permission.REORDER_TASKS" />
 
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
 
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

## 方法2：Intent.FLAG_ACTIVITY_REORDER_TO_FRONT

```java
ActivityManager manager = (ActivityManager) context.getSystemService( Context.ACTIVITY_SERVICE);
List<RunningTaskInfo> task_info = manager.getRunningTasks(20);

String className = "";
for (int i = 0; i < task_info.size(); i++)
{
	if ("包名".equals(task_info.get(i).topActivity.getPackageName())){
		System.out.println("后台  " + task_info.get(i).topActivity.getClassName());
        
		className = task_info.get(i).topActivity.getClassName();

		//这里是指从后台返回到前台  前两个的是关键
		intent.setAction(Intent.ACTION_MAIN);
        intent.addCategory(Intent.CATEGORY_LAUNCHER);
        intent.setComponent(new ComponentName(context, Class.forName(className)));
        intent.addFlags(Intent.FLAG_ACTIVITY_REORDER_TO_FRONT
                  | Intent.FLAG_ACTIVITY_NEW_TASK
                  | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED);
        context.startActivity(intent);
	}
} 
```

