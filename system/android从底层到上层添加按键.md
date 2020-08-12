Android设备已经定义不少物理按键，如Home、Assist、Search、VOLUME_UP等按键。有时我们需要去自定义添加键值，提供全局按键或者给APP使用使用。

## 1 内核修改

​		Android使用标准的linux输入事件设备(/dev/input目录下)和驱动，按键定义在内核./include/uapi/linux/input-event-codes.h文件中，按键定义形式如下：

```
#define KEY_SEARCH              217
```

## 2 从硬件GPIO口到内核标准按键的映射

​	内核中(我的平台是arch/arm/mach-msm/board-msmxx.c文件),将MFP_PIN_GPIO2这个GPIO口的按键映射到Linux的KEY按键

## 3 Linux到Android映射

​		第2步实现了从硬件GPIO口到内核标准按键的映射,但是android并没有直接使用映射后的键值，而且对其再进行了一次映射，从内核标准键值到android所用键值的映射表定义在android文件系统的/system/usr/keylayout目录下。标准的映射文件为qwerty.kl，定义如下：
key 399   GRAVE
key 2     1
key 3     2
key 4     3

key 158   BACK              WAKE_DROPPED
key 229   MENU              WAKE_DROPPED
key 139   MENU              WAKE_DROPPED
key 59    MENU              WAKE_DROPPED
key 127   SEARCH            WAKE_DROPPED
key 217   SEARCH            WAKE_DROPPED

key 231   CALL              WAKE_DROPPED
key 61    CALL              WAKE_DROPPED

key 102   HOME              WAKE

key 115   VOLUME_UP
key 114   VOLUME_DOWN
key 116   POWER             WAKE
key 212   CAMERA


(4)android对底层按键的处理方法(高通平台示例)
android按键的处理是Window Manager负责，主要的映射转换实现在android源代码frameworks/base/services/input/EventHub.cpp
此文件处理来自底层的所有输入事件，并根据来源对事件进行分类处理，对于按键事件，处理过程如下：
(a)记录驱动名称为
(b)获取环境变量ANDROID_ROOT为系统路径(默认是/system，定义在android源代码/system/core/rootdir/init.rc文件中)
(c)查找路径为"系统路径/usr/keylayout/驱动名称.kl"的按键映射文件，如果不存在则默认用路径为"系统路径/usr/keylayout/qwerty.kl"
这个默认的按键映射文件，映射完成后再把经映射得到的android按键码值发给上层应用程序。
所以我们可以在内核中定义多个按键设备，然后为每个设备设定不同的按键映射文件，不定义则会默认用qwerty.kl

 注：qwerty.kl 文件中：：

BEGIN
这一段统一是 "key"，应该是按键映射的标识。 

SCANCODE
这一段直接将字符串转化为数字，也就是说 kl 文件里的这一段存储的是数字。这里值就是从低层硬件读取出来值。也就是我们在 OM 项目中， vfb 应该发送给 android 的值。这个值不是 android sdk 中描述的值（这个值是上层应用使用的）。不同的硬件会不一样。 

KEYCODE
这个值就是 android sdk 里描述的啦。不过这一段有很多都是字符串，不是直接保存数值的，这个有个转化关系，具体的后面再说。 

FLAG
这里目前来说就2个值 WAKE 和 WAKE_DROPPED 。好像 WAKE 代表可以在锁机状态下唤醒屏幕。

(5)举例
上面(2)步我们在内核中声明了一个名为"gpio-keys"的按键设备，此设备定义在内核drivers/input/keyboard/gpio_keys.c文件中
然后我们在内核启动过程中注册此设备：  platform_device_register(&gpio_keys);
然后我们可以自己定义一个名为gpio-keys.kl的android按键映射文件，此文件的定义可以参考querty.kl的内容，比如说我们想将MPF_PIN_GPIO3
对应的按键作android中的MENU键用，首先我们在内核中将MPF_PIN_GPIO3映射到KEY_F2，在内核include/linux/input.h中查找KEY_F2发现
#define KEY_F2            60
参照KEY_F2的值我们在gpio-keys.kl中加入如下映射即可
key 60    MENU              WAKE
其它按键也照此添加，完成后将按键表放置到/system/usr/keylayout目录下即可。

补充：
(1)android按键设备的映射关系可以在logcat开机日志中找的到(查找EventHub即可)
(2)android按键设备由Window Manager负责，Window Manager从按键驱动读取内核按键码，然后将内核按键码转换成android按键码，转换完成
后Window Manager会将内核按键码和android按键码一起发给应用程序来使用，这一点一定要注意。

#### 前提：

framework层添加前，要确定按键驱动是否调好:

> adb shell getevent

/dev/input/event3: 0001 02fe 00000001

/dev/input/event3: 0000 0000 00000000

/dev/input/event3: 0001 02fe 00000000

/dev/input/event3: 0000 0000 00000000

其中02fe就是驱动上报的值，两次的1，0是指按下和弹起的动作。

#### 1. gpio-keys.kl

gpio-keys.kl 这个文件对应的是定制机的kl 文件，不一定是这个名字，定义在 AndroidBoard.mk 中的： LOCAL_MODULE       := gpio-keys.kl ，在手机中可以在/system/usr/keylayout 找到这个文件

将驱动上报的02fe转为十进制的766, 并且定义：

key 766   F14

这样就完成了对物理按键kl文件的映射到“F14”



#### 2.framework native 中定义：

/frameworks/native/include/android/keycodes.h

> AKEYCODE_F14 = 901

/frameworks/native/include/input/InputEventLabels.h

> DEFINE_KEYCODE(F14)

#### 3. framework base下的定义:

frameworks/base/core/java/android/view/KeyEvent.java:

定义按键的keyCode 也就是 APP 在onKeyDown() 获取的keyCode

> public static final int KEYCODE_F14 = 901; // 注意APP获取的keyCode是901  不是766 

frameworks/base/core/res/res/values/attrs.xml:

> <enum name="KEYCODE_F14" value="901"/>

按照以往的经验来看，配置到这里编译运行，添加的按键应该可以生效了。但是在8.0上并非如此

------

#### 4.android O 需要添加：

frameworks/base/data/keyboards/Generic.kl

frameworks/base/data/keyboards/qwerty.kl

> key 766   F14

完成这一步才会生效。

键值最大767

总结：android 8.0 在代码的编译上有很多变化，在6.0/7.0 的可以生效的设置往往会失效，这时候需要我们多研究了，因为很多版本上的差异官网也没有说清楚。