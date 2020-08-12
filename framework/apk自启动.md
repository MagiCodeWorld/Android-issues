## Android 程序禁止自启动方法

​		Android 3.1后引入了一种机制，系统中的包管理服务跟踪应用的停止状态，然后用于控制是否启动这些应用。即Android Intent中定义了两种新的FLAG，FLAG_INCLUDE_STOPPED_PACKAGES和FLAG_EXCLUDE_STOPPED_PACKAGES,顾名思义，前者是允许已经停止的应用的Intent filter接收这个intent，而后者不可以。并且系统对于所有的broadcast intent都加了FLAG_EXCLUDE_STOPPED_PACKAGES这个标志。

​		也就是说，要想获得开机广播，你必须保证两点，1）你的应用程序在安装后必须运行一次；2）你的应用程序没有被强行停止。

对于一个apk，注册ACTION_BOOT_COMPLETED广播，可以实现开机自启动。

早期，有安全软件使用 pm命令禁止package或component，使得apk自启动失败。

最近做了几个实验，用Kingroot和LBE的自启动管理攻击关闭软件自启动后，发现apk中接受开机广播的component还是enable状态，但是已经无法自启动，所以，想了解一下这些软件是怎么管理自启动的。

## android 禁用和开启四大组件的方法

- setComponentEnabledSetting 

为什么要关闭组件？ 
在用到组件时，有时候我们可能暂时性的不使用组件，但又不想把组件kill掉，比如创建了一个broadcastReceiver广播监听器，用来想监听第一次开机启动后获得系统的许多相关信息，并保存在文件中，这样以后每次开机启动就不需要再去启动该服务了，也就是说如果没有把receiver关闭掉，就算是不做数据处理，但程序却还一直在后台运行会消耗电量和内存，这时候就需要把这个receiver给关闭掉。 


如何关闭组件？ 
关闭组件其实并不难，只要创建packageManager对象和ComponentName对象，并调用packageManager对象的setComponentEnabledSetting方法。


public void setComponentEnabledSetting (ComponentName componentName, int newState, int flags)
componentName：组件名称 
newState：组件新的状态，可以设置三个值，分别是如下： 
不可用状态：COMPONENT_ENABLED_STATE_DISABLED 
可用状态：COMPONENT_ENABLED_STATE_ENABLED 
默认状态：COMPONENT_ENABLED_STATE_DEFAULT 
flags:行为标签，值可以是DONT_KILL_APP或者0。 0说明杀死包含该组件的app
public int getComponentEnabledSetting(ComponentName componentName)

获取组件的状态



实例：

实例一：禁止开机启动的Receiver（可以是第三方的receiver）

final ComponentName receiver = new ComponentName(context,需要禁止的receiver); 
 final PackageManager pm = context.getPackageManager(); 
 pm.setComponentEnabledSetting(receiver,PackageManager.COMPONENT_ENABLED_STATE_DISABLED,PackageManager.DONT_KILL_APP); 　}



实例二：隐藏应用图标


如果设置一个app的mainActivity为COMPONENT_ENABLED_STATE_DISABLED状态

则不会再launcher的程序图标中发现该app

PackageManager packageManager = getPackageManager();
        ComponentName componentName = new ComponentName(this, StartActivity.class);
        int res = packageManager.getComponentEnabledSetting(componentName);
        if (res == PackageManager.COMPONENT_ENABLED_STATE_DEFAULT
                || res == PackageManager.COMPONENT_ENABLED_STATE_ENABLED) {
            // 隐藏应用图标
            packageManager.setComponentEnabledSetting(componentName, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                    PackageManager.DONT_KILL_APP);
        } else {
            // 显示应用图标
            packageManager.setComponentEnabledSetting(componentName, PackageManager.COMPONENT_ENABLED_STATE_DEFAULT,
                    PackageManager.DONT_KILL_APP);
        }

### Android系统应用隐藏和应用禁止卸载

 1、应用隐藏与禁用

Android设置中的应用管理器提供了一个功能，就是【应用停用】功能，这是针对某些系统应用的。当应用停用之后，应用的图标会被隐藏，但apk还是存在，不会删除，核心接口就是PackageManager的setComponentEnabledSetting(ComponentName, int, int)方法

具体代码可以查看设置模块：com.android.settings.applications.InstalledAppDetails.java

 

2、应用禁止卸载

需要禁止卸载指定应用，除了将应用放置system/app下成为系统级应用之外，还有其他方法，这个方法适用于第三方应用。

修改Android的PackageInstaller模块，源码位于pakages/apps目录下，具体代码位于：com.android.packageinstaller.UninstallerActivity.java

当Android弹出是否卸载窗口时，进入的就是这个类，在这个类可以根据包名，阻止应用的卸载。



## 开机拉起apk

```java
try {
	List<String> resultList = getTopwayWhiteList("[autostartlist]");
	if(resultList != null){
		for (int i = 0; i < resultList.size(); i++) {
			Slog.i(TAG, "package: " + resultList.get(i));
			//ApplicationInfo appInfo = AppGlobals.getPackageManager().getPackageInfo(resultList.get(i), 0).applicationInfo;
			ApplicationInfo appInfo = AppGlobals.getPackageManager().getApplicationInfo(resultList.get(i), 0, 0);
			Slog.i(TAG, "package appInfo: " +appInfo);
			if (appInfo != null) {
				Slog.i(TAG, "000 start addAppLocked");
				addAppLocked(appInfo, false);
			}
		}
	}
} catch (RemoteException ex) {
	// pm is in same process, this will never happen.
}
```



## [Android 一键加速原理](https://www.cnblogs.com/jamboo/articles/6016332.html)

# 说明

 

在上一篇中介绍了“垃圾清理”，在系统优化中有一个功能往往是与垃圾清理分不开的，那就是“手机加速”。目前流行的管理软件中以及网络上并没有明确的定义什么叫“垃圾清理”什么叫“手机加速”。结合上一篇的“垃圾清理”这里统一做一个在本系列文章中的定义：

n 垃圾清理：在本系列文章中认为扫描和清理的是**静态内容**，包括应用的文件缓存、缩略图、日志等系统或应用创建的文件，这些文件不具有“运行时”特征。

n 手机加速：在本系列文章中认为清理的是**动态内容**，包括杀死运行时进程、限制开机自启动、限制后台自启动，以及应用运行时所占用的内存等，这些内容都与进程相关，具有“运行时”特征。

在对垃圾清理和手机加速做了简单的区分以后，本篇接下来将研究一下手机加速的相关内容。

# 清理运行时进程

 

清理运行时进程也就是清理后台进程，有些手机管理软件中也叫“一键加速”或者“一键清理”等，其实指的都是这个功能。

在正式介绍介绍进程清理之前，先简单介绍一些[Android](http://lib.csdn.net/base/android)中进程的内存管理策略。

## Android内存管理策略

 

Android中，进程的生命周期都是由系统控制的，即使用户关掉了程序，进程依然是存在于内存之中。这样设计的目的是为了下次能快速启动。当然，随着系统运行时间的增长，内存会越来越少。Android Kernel 会定时执行一次检查，杀死一些进程，释放掉内存。

在Android对内存管理引入了[Linux](http://lib.csdn.net/base/linux)中使用的一种名称为OOM(Out Of Memory，内存不足)的机制来完成这个任务，该机制会在系统内存不足的情况下，选择一个进程并将其Kill掉。在Android中对Linux原生的OOM机制根据嵌入式设备的特点进行了一些改造，于是就有了Low Memory Killer。

 内存管理不是本文的重点，想了解Low Memory Killer的朋友可以自己查阅相关资料或者查看源码，这里不做详细介绍。Low Memory Killer在源码中的位置为：@/kernel/goldfish/drivers/staging/android/lowmemorykiller.c。这里要明确的问题由两点：

A.    Low Memory Killer在用户空间中指定了一组内存临界值，当其中的某个值与进程描述中的oom_adj值在同一范围时，该进程将被Kill掉。

B.    Android中的oom相关参数在init.rc中进行初始化配置，在系统运行时由ActivityManagerService进行动态调整。

## Android进程优先级

 

根据oom_adj，Android将进程分为6个等级,它们按优先级顺序由高到低依次是:

1)    前台进程( FOREGROUND_APP)

2)    可视进程(VISIBLE_APP )

3)    次要服务进程(SECONDARY_SERVER )

4)    后台进程 (HIDDEN_APP)

5)    内容供应节点(CONTENT_PROVIDER)

6)    空进程(EMPTY_APP)

这六类进程所对应的oom_adj的值在不同的手机中可能会有所不同。下面是在我的小米2S手机上的参数如下：

![img](http://img.blog.csdn.net/20140624133723078?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

注意值越大说明进程重要程度越低。

在我的小米2S上这几种进程会被杀死的时机如下：

![img](http://img.blog.csdn.net/20140624133750718?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

注意这些数字的单位是page. 1 page = 4 kB。

对于单个进程的oom_adj的值可以查看/proc/pid/oom_adj的值，如下：

![img](http://img.blog.csdn.net/20140624133816765?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

## 清理运行时进程

 

在简单的了解了Android对进程和内存的管理策略以后，我们会发现这样一个问题：Android中的Low Memory Killer策略会定时扫描，并根据其策略选择杀死进程的。那能给我们清理运行时进程带来什么参考呢？

我们在杀死运行时进程的一种可行性方案：可以根据oom_adj的值制定一个阀值，在用户触发（当然也可以定时触发）时，判断如果进程的oom_adj的值大于阀值，则杀死。杀死进程有两种方式，下面分别介绍：

### 选择在Linux层面杀死进程

 

可以选择根据1.2中的方式读取/proc/pid/oom_adj的值进行判断，然后通过Linux中的kill函数杀死进程。

在Android层面，系统是通过如下属性定义进程的重要程度的：

@/frameworks/base/core/[Java](http://lib.csdn.net/base/javaee)/android/app/ActivityManager$RunningAppProcessInfo

![img](http://img.blog.csdn.net/20140624133938531?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

其中，importance的取值如下，关于各个值的含义在下面的注释中：

![img](http://img.blog.csdn.net/20140624134026296?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![img](http://img.blog.csdn.net/20140624134048562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

采用这种方式杀死进程的部分示例代码如下：

![img](http://img.blog.csdn.net/20140624134118046?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

在上面的示例代码中我们是通过ActivityManager类中的killBackgroundProcesses接口来杀死进程的，下面来看一下killBackgroundProcesses：

![img](http://img.blog.csdn.net/20140624134140953?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

这里需要注意如下几点：

A.   但是通过该方法只能杀死进程优先级低于SERVICE_ADJ的进程。

B.   使用该方法需要声明KILL_BACKGROUND_PROCESSES权限

C.   该接口为ActivityManager中的公开接口，不需要任何权限。

对于大部分情况下，通过调用改接口杀死后台进程就够了，但有时候我们要执行更加严格的进程清理策略，杀死优先级更高的进程，该如何处理呢？下面一节将做介绍。

### 一种更加严格的进程杀死方案

 

接着上一节最后的问题继续分析，看一下系统是如何选择杀死指定优先级以下的进程的。虽然在Android中并没有提供接口给我们使用，但是可以确信Android自身肯定也会有我们类似的需求。最后我在ActivityManagerService（后面简称“Ams”）中找到了这个功能的系统实现，如下：

@/frameworks/base/services/java/com/android/server/am/ActivityManagerService.java

![img](http://img.blog.csdn.net/20140624134425515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![img](http://img.blog.csdn.net/20140624134431609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

@/frameworks/base/core/java/android/os/Process.java

![img](http://img.blog.csdn.net/20140624134456296?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

通过上面的过程可以看出，Ams在杀死进程时最后实际调用的是android.os.Process中的killProcessQuiet方法。到此就十分清晰了，我们在实现杀死进程时只有自己实现上述过程就可以了。

通过killProcessQuiet的实现我们可以看出该方法是一个隐藏的方法，在应用中时不能直接调用的。仔细观察Process类，我们又发现了一个类似的实现，即killProcess方法，如下：

![img](http://img.blog.csdn.net/20140624134519671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

这里的killProcess和killProcessQuiet是一样的，唯一的区别是通过killProcess杀死进程时会有“Sendingsignal. PID: %d SIG: %d"”字样的log打印出来。所以，我们在实现自己的杀进程逻辑时直接使用killProcess方法就可以了。

### 一种防止被杀死进程重启的方案

 

本节上面的三个小节中介绍了三种杀死进程的可行性方案，通过验证它们也是可行有效的。但是，细心的朋友在验证上面的方案时会发现直接杀死进程时可能会遇见如下一些问题：

A.    有些进程无法被杀死或者在被杀死后会重启（主要是系统应用和Service）。

B.    在杀死当前正在运行的进程时可能会出现异常状况。

那么需要如何解决这种情况呢？通过查看ActivityManager的代码，我发现了这样一个方法，如下：

![img](http://img.blog.csdn.net/20140624134616484?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

仔细看一下forceStopPackage方法的说明，哎呀，简直超出我们的预期。通过使用forceStopPackage方法，不仅可以强制停止应用进程，甚至可以强行停止掉与当前进程相关的其他一些相关的内容，比如：

 

- 与当前应用共享uid的其他进程。
- 所有证照运行的Service
- 所有的Activity
- 通知
- 定时器

 

进一步研究forceStopPackage的实现后发现，在调用forceStopPackage时系统会将当前的package状态设置为stopped状态。在Android2.3版本之后，被标记为stopped状态的应用是不能接受广播的，除非在发送广播时指定FLAG_EXCLUDE_STOPPED_PACKAGES标志，但是系统广播是无法认为指定改标志位的。

下面是forceStopPackage方法的内部实现：

@/frameworks/base/services/java/com/android/server/am/ActivityManagerService.java

![img](http://img.blog.csdn.net/20140624134702609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

通过对这个方法的研究我们可以得出如下几点结论：

A.    在使用forceStopPackage接口时需要声明FORCE_STOP_PACKAGES权限。

B.    forceStopPackage是一个隐藏接口，需要通过反射等手段实现调用。

C.    使用forceStopPackage接口需要系统签名.

D.    forceStopPackage接口除了可以用来杀死进程外，还可以达到禁止开机自启动和后台自启动的目的。

E.    在使用forceStopPackage接口时可能会有不想要发生的副作用（清空定时器等），慎重使用。

F.    Package的stopped状态是在Android4.0以后增加的，也就是说在2.3之前的版本forceStopPackage并不能禁用开机广播等。

### 一件加速快捷方式动画效果实现

 

关于UI相关的内容不是本文的终端，感兴趣的朋友可以参考网上的一篇文章：

http://blog.csdn.net/ruils/article/details/16922557

# 开机自启动管理

 

我们无论做什么事情总是要知其然，然后才能之前所以然。因此，在了解如何禁止开启自启动之前需要先来来了解一下开机自启动的原理，以及系统发出BOOT_COMPLETED广播的时机。

## 开机自启动的原理

 

Android在开机时，在系统启动完成后会发送一个Standard Broadcast，名为"android.intent.action.BOOT_COMPLETED”。所谓“开机自启”就是注册接收BOOT_COMPLETED的静态广播，在收到广播时将自己唤起。

下面是一小段示例程序：

![img](http://img.blog.csdn.net/20140624134853625?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

在AndroidManifest中注册静态BOOT_COMPLETED广播：

![img](http://img.blog.csdn.net/20140624134914984?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

要使BOOT_COMPLETED广播生效，需要声明RECEIVE_BOOT_COMPLETED权限。

![img](http://img.blog.csdn.net/20140624134942250?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

这里很简单，大家基本上都用过，没有什么可讲的。这里要说的是几点要注意的问题：

A.    在使用BOOT_COMPLETED广播时，要记得声明RECEIVE_BOOT_COMPLETED权限。

B.    在Android4.0开始，Android增加了Package stopped状态，处于该状态的应用是接收不到开机广播的。处于stopped状态的应用包括：

n 调用1.3.4中介绍的forceStopPackage接口停止的应用。

n 在“设置->应用->应用详情页”中点“停用”按钮，被停用的应用。

n 新安装完成，从未打开和运行过的应用。

C.    在Android2.3之前的版本中，不存在上一条所说的限制，可以接收的开机广播。

D.    在Android4.0以后的版本中，如果想使处于stopped状态的应用也接收到广播，需要在intent中增加FLAG_EXCLUDE_STOPPED_PACKAGES这个Flag。但是这里要注意的是，系统广播使用户无法自定义的。

E.    BOOT_COMPLETED是一个Standard广播，在开发应用时有时会希望自己的应用在接收到开机广播后比其他同样接收开机广播的应用更早的启动就需要指定”android:priority”属性的值为最大整数（priority为整形，取值范围为-1000到1000）即可。

## 开机自启动的时机

 

在2.1节中我们了解到开机自启动是通过接受系统广播“BOOT_COMPLETED”实现的，那“BOOT_COMPLETED”是系统什么时候发出的呢？对于Android系统启动的过程，不是本文研究的内容，不做介绍，这里只说一下与开机广播“BOOT_COMPLETED”相关的部分，如下图所示：

 

<img src="http://img.blog.csdn.net/20140624135056968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="img" style="zoom:50%;" />

 

可以看出开机广播是在系统启动完成后，在启动第一个应用进程（桌面）时发出的。

## 实现禁止开机自启动

 

了解了开机自启动的原理，接下来介绍一下在开发手机管理软件时，如何实现禁止应用开机自启动。通过前面的分析，我们想一下如果想禁用开机广播有哪些可能的实现？从应用接收处理广播的整个过程来说，其实无外乎下面这几种可能：

A.    阻止系统发出BOOT_COMPLETED开机完成广播。

B.    不阻止系统发出广播，但是阻止应用接收到BOOT_COMPLETED开机完成广播。

C.    应用可以接收到BOOT_COMPLETED广播，但是阻止应用进行响应。

D.    运行应用接收BOOT_COMPLETED广播并且进行响应，但是在应用响应后阻止其运行。

接下来将依次分析这几种情况的可行性：

A.    第一种方案在应用的层次难以实现，而且也不太合理，是不可行的。

B.    第二种方案理论上是可行的，至少到目前为止前面介绍过的forceStopPackage就可以做到。

C.    第三种方案，理论上也是可行的，但可能会有些难度(进程注入)。

D.    第四种方案，也是可行的，而且根据之前的分析也是比较容易做到的。

下面就依次分析一下具体的实现方案：

### 通过停止应用实现禁止开机自启动

 

对于上面的第二种方案中“不阻止系统发出广播，但是阻止应用接收到BOOT_COMPLETED开机完成广播“的想法，相信大家很快都会想到在1.3.4中介绍过的forceStopPackage这个接口，这个接口除了可以杀死正在运行的进程以外，还有一些“副作用”，其中将应用状态置为“stopped”和清空定时器这些正是我们在实现禁止开机自启动时所需要的。

由于前面一对forceStopPackage的使用和需要注意的问题等都做了比较详细的说明，这里不再赘述，不清楚的朋友可以参考1.3.4节。

### 通过停用广播接收器实现禁止开机启动

 

forceStopPackage接口完全可以满足我们的需求，甚至大大超出了我们的需求，不仅禁用了开机自启动，连后台自启动等等也都禁用了。这些副作用很多时候并不是我们想要的，那有没有一种”副作用”相对较小的实现方式呢？于是引出了本小节这种实现方式。

本方案也是基于第二种方案中“不阻止系统发出广播，但是阻止应用接收到BOOT_COMPLETED开机完成广播“的想法实现的。

刚开始有这个想法的时候可能会有些无从下手的感觉，没关系既然本方案的核心是“阻止广播接收”，那我们就先来看一下Android中静态的广播接收器是如何定义的，下面是Android官方文档中关于receiver的截图：

![img](http://img.blog.csdn.net/20140624135357609?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

引自：http://developer.android.com/intl/zh-cn/guide/topics/manifest/receiver-element.html



不知道各位读者在看到“android:enable”这个属性的时候有没有像我一样眼前一亮的感觉。就我而言，在平时开发普通的应用程序时几乎没有关注过这个属性，因为很少有这方面的需求。好了，让我们看一下“android:enable”这个属性，如下：

 

看完了这段描述，是不是感觉不仅满足了我们的需求，而且又有一个小惊喜呢？是的，就是这句“**The element has its own enabled attribute that applies to allapplication components, including broadcast receivers.**”。这说明除receiver以外的其他应用组件也都拥有“android:enable”属性。呵呵，到这里是不是为我们禁止后台自启动应用也找到了一种解决方案？

在了解了“android:enable”这个属性以后，下面看一下它的使用：

@/frameworks/base/core/java/android/content/pm/PackageManager.java

![img](http://img.blog.csdn.net/20140624135509296?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

在PackageManager中除了setComponentEnabledSetting以外，还提供了另外一个接口setApplicationEnabledSetting，通过setApplicationEnabledSetting可以停用应用中所有的组件，代码如下：

![img](http://img.blog.csdn.net/20140624135527671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

这里要注意的问题由这么几点：

A.    通过调用setApplicationEnabledSetting可以将应用中的所有组件置为disable状态。

B.    与forceStopPackage接口相比，该接口不会清空定时器等，因此如果应用通过定时器定时发送自定义广播，并且在广播中指定FLAG_EXCLUDE_STOPPED_PACKAGES是可以唤醒的。

C.    使用setComponentEnabledSetting接口必须是system程序并具有system签名。

D.    使用该接口需要声明CHANGE_COMPONENT_ENABLED_STATE。

E.    该接口在应用于在AndroidManifest中的入口Activity（intent-filter指定action为“android.intent.category.LAUNCHER”）时，还可以达到隐藏应用快捷方式图标的效果。

下面是使用该接口的一段代码片段：

![img](http://img.blog.csdn.net/20140624135549031?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

### 停用广播接收器方案实例研究

 

开机自启动管理有一款小而美的软件AutoStart相信很多朋友都听说过，截图如下：

 

<img src="http://img.blog.csdn.net/20140624135639375?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="img" style="zoom:30%;" />

 

这款软件就是利用了2.3.2中介绍的原理实现的，下面是一小段反编译后的代码片段：

 

![img](http://img.blog.csdn.net/20140624135704453?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

不少朋友会有到这里会有这样一个疑问：在上一节中不是说使用setComponentEnabledSetting接口需要system权限（是system程序并具有system签名）吗？AutoStart并不是system，它是怎么做到的呢？一个同事告诉我AutoStart是通过“pm”命令来做的提醒了我，下面看一下pm命令：

<img src="http://img.blog.csdn.net/20140624135726156?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="img" style="zoom:70%;" />

 

执行pm需要的shell权限，在取得root权限的情况下就可以直接执行使用了。这也提醒我们在分析一个问题遇到阻碍的时候，不防稍微发散一下和听取一下别人的建议。

### 停用广播接收器方案继续探讨

 

用过Windows系统的朋友都知道在Windows中有一个称为注册表的东西，维护着Windows系统中已安装应用的相关信息。在Android中严格意义上是不存在注册表这样的东西的，但是在维护安装包信息方面，Android中存在着一个类似的文件“/data/system/packages.xml”，如下图：

<img src="http://img.blog.csdn.net/20140624135815234?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="img" style="zoom:70%;" />

 

在通过setComponentEnabledSetting设置后，最终会在packages.xml中产生如下数据：

![img](http://img.blog.csdn.net/20140624135840453?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

那在取得root权限的情况下，直接修改packages.xml能不能达到停止自启动的效果呢？大家不妨可以自己验证一下，这里应该是不可以的。因为packages.xml是在PackageManagerService服务初始化的时候解析完成的，以后使用的都是保存在内存[数据结构](http://lib.csdn.net/base/datastructure)中的数据，直接修改packages.xml文件无法达到修改内存中的数据的目的。这里在重启手机下次进入时应该是会生效的（没有验证）。

### 通过杀死进程实现禁止开机自启动

 

本方案是对第四种方案“运行应用接收BOOT_COMPLETED广播并且进行响应，但是在应用响应后阻止其运行”的分析和讨论。本方案的想法是这样的：

如果我们在系统启动完成后能够尽早的接收到“BOOT_COMPLETED”，然后监控其他应用进程，如果发现接下来有其他应用被启动，则强行杀死。这样也就达到了禁止应用开机自启动的目的。下面依次分析该方案中的几个关键点：

A.    如何能够尽早的启动？

结合2.1和2.2中介绍的原理，这里我们可以采取两方面的措施：

1)    指定receiver的优先级为最大整数

2)    指定receiver的category为“android.intent.category.HOME”

![img](http://img.blog.csdn.net/20140624140009234?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

B    如何监控当前进程？

自身启动一个定时器，定时轮询正在运行的服务，执行清理工作。

查询开机启动的应用可以通过PackageManager中提供的下面这个接口实现：

![img](http://img.blog.csdn.net/20140624140028875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

除了杀死开机启动的应用进程外，在定时唤醒时也可以杀死后台启动的进程。在ActivityManager中提供了获取正在运行的Services和Application的方法，如下：

![img](http://img.blog.csdn.net/20140624140055812?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

![img](http://img.blog.csdn.net/20140624140107906?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

C    如何杀死进程？

这里可以采用前面介绍过的android.os.killProcess即可；也可以采用ActivityManager中的killBackgroundProcesses接口。ActivityManager中的restartPackage接口也能达到同样的目的，但已不推荐使用。

该方案存在的问题：

A.    采用定时轮询的方式实现，自身比较耗电。

B.    除了可以用来限制开机自启动，也可以用来限制后台自启动。

C.    在获取开启自启动列表时，可以先判断是否具有“android.permission.RECEIVE_BOOT_COMPLETED”权限。

### 杀死进程方案实例研究

 

采用杀死进程方案的应用中也存在一个非常有名的应用”AutorunManager”，截图如下：

 

<img src="http://img.blog.csdn.net/20140624140203718?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="img" style="zoom:40%;" />

 

AutorunStartupIntentReceiver接收到开启广播后，启动AutorunService服务：

![img](http://img.blog.csdn.net/20140624140225484?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

AutorunService服务启动后，启动一个Timer执行h任务：

![img](http://img.blog.csdn.net/20140624140245796?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

h任务读取程序开机自启动设置，如果程序不允许开机启动，则杀死该程序进程：

![img](http://img.blog.csdn.net/20140624140307250?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhneGh1YWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

注：本小节内容部分引用自http://blog.csdn.net/androidsecurity/article/details/8623264

# 后台自启动管理

 

所谓名不正则言不顺，在分析后台自启动之前，这里先对开机自启动和后台自启动做一个概念上的区分。网上没有找到相关的定义，这里主要是根据自己的理解来给出一下本系列文章中的定义。

 

- 开机自启动：

 

> 1)    从启动时机来说：发生在刚启动完成时。
>
> 2)    从启动原理来说：由开机启动广播“BOOT_COMPLETED”触发。

 

- 后台自启动

 

> 1)    从启动时机来说：发生在启动完成后正常运行时。
>
> 2)    从启动原理来说：由“BOOT_COMPLETED”广播以外的其他场景触发，常见的常见有：锁屏解锁、网络状态变化、定时器等。
>
> 开机自启动与后台自启动实际上并没有本质上的区别，在第2章中介绍过的方法也同样适用于后台自启动。这里只说几点需要注意的问题：

> > A.    在使用forceStopPackage接口时有一种情况比较难以避免，就是应用之前相互唤起的问题。
>
> > B.    在使用setComponentEnabledSetting接口时需要注意对用户自定义广播的处理。
>
> > C.    在使用killPrcocess方式时需要注意误杀情况的处理。

 

# 总结

 

垃圾清理和手机加速的功能到现在就差不多介绍完了，这里想说一些个人的想法，希望能够引起本系列文章读者的一些思考。

Android中的”垃圾清理“和”手机加速”本身就是一个悖论。你认为是垃圾的东西，可能正是产生垃圾的软件所想保留的；你认为通过手机加速为用户清理了内存空间，殊不知充分利用内存正是Linux内存管理的机制，不提供应用直接杀死进程，将进程统一由系统创建和回收，也是Android为快速创建进程的一种优化。我从不认为Linux内核的开发者或者Android框架的开发者会比我们更加白痴（无知），在系统的设计之初和系统的演化过程中，这里肯定经历过了无数次的讨论与权衡验证，证明了目前所采用的方案是一种可以平衡各方需求的相对优良的做法。

看到目前市面上形形色色的各类手机管理软件、电子市场各显神通地打着“用户体验”的幌子，为了满足用户心理上那种“爽”的需求，提供甚至自动去清理垃圾和手机加速。应用们为了搜集用户信息和推送等苦心积虑的监听各种广播和定时器来保证自身应用的驻留。这两者的相互博弈，导致应用的整体情况更加恶劣。看到目前Android应用的各种乱象，作为一名Android开发者，真的很痛心疾首。

并不一定所有的朋友都能认同我的观点，甚至很多人会觉得正是这些不规范造就了Android生态的繁荣，也同样造就了一批类似于360这样的手机安全公司。但是，我还是所有开发者希望在开发应用的过程中能够指定一个“**合适的策略**”，因为作为用户的我们已经被你们搞的**够乱了**。

 

到此，手机加速就介绍完了，欢迎大家交流讨论。下一篇将介绍《应用杂篇》，即不属于前几篇的一些应用管理方面的功能，如应用锁、山寨应用识别、应用安装位置判断、应用智能推荐等。