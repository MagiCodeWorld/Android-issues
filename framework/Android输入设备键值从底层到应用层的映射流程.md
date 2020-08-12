## [android : 输入设备键值从底层到应用层的映射流程](https://www.cnblogs.com/blogs-of-lxl/p/9490205.html) 		

## 一、Android输入子系统简介：

　　Android输入事件的源头是位于/dev/input/下的设备节点，而输入系统的终点是由WMS管理的某个窗口。最初的输入事件为内核生成的原始事件，而最终交付给窗口的则是KeyEvent或MotionEvent对象。因此Android输入系统的主要工作是读取设备节点中的原始事件，将其加工封装，然后派发给一个特定的窗口以及窗口中的控件。这个过程由InputManagerService（以下简称IMS）系统服务为核心的多个参与者共同完成。

![821933-20180816203042379-1797695639](assets/821933-20180816203042379-1797695639.png)

　　　　　　　　　　　　　　　　　图1：输入系统的总体流程与参与者



## 二、kernel键值定义

（1）键扫描码

ScanCode是由linux的Input驱动框架定义的整数类型，可参考input.h头文件，即getevent得到的键值。

```c
#define KEY_Q 16
#define KEY_W 17
#define KEY_E 18
#define KEY_R 19
#define KEY_T 20
```

(2) 键盘布局文件(\*.kl)

 将input event报的键值转换成具体键盘对应的按键供android上层使用，时通过键盘布局文件(*.kl)完成转换的。放在/system/usr/keylayout/下面

而qwert.kl中定义如下：

```
 ScanCode + 字符串值 ：
key 16   Q
key 17   W
key 18   E
key 19   R
key 20   T
```

 其中ScanCode 是驱动报的值（即驱动input.h中定义的键值 ）

(3) 添加kl文件：

abcxxxx.kl（文件名须与input 的device设备的name一致）

  `Key 199  CAMERA` [199为 驱动定义的scanCode ，CAMERA 为Android中 KEYCODES[]定义按键对应的keylabel字符]

> 注：
>
> 1）kl文件须与键盘输入的input的devic的名称一致，否则EventHub在加载设备时因找不到对应的kl而加载默认的qwert.kl，导致键值转换错误
>
> 2）kl中的scanCode和android中定义的keylabel字符必须对应，否则会转换错误。keyMapper在转换时是根据scanCode，来确定对应的按键字符，再根据此字符在KEYCODES中的位置来确定对应android中的键值

(4) kl文件添加到system

将kl文件（通常）放在/device/mtk/XXX/(XXX为项目名称)

添加AndroidBoard.mk ：

```makefile
include $(CLERA_VARS)
LOCAL_MODULE         := abcxxxx.kl
LOCAL_MODULE_TARGS   := optional  eng
LOCAL_MODULE_CLASS   := ETC
LOCAL_SRC_FILES      := $(LOCAL_MODULE)
LOCAL_MODULE_PATH    := $(TARGET_OUT_KEYLAYOUT)
include $(BUILD_PREBUILT)
```

在/device/mtk/common/base.mk添加

```makefile
KEYPAD += abcxxxx.kl    注：不加会导致kl文件不被打包进/system/usr/keylayout/
```

 

## 三、键值映射关系：

  ①IR硬件扫描码在驱动里面被映射为 include/uapi/linux/input.h 里面定义的某个键值，但这个键值只在linux系统(内核)中使用。
 　②Android通过源码目录下的 device/xxx/xxx.kl(keylayout) 文件完成linux键值到Android系统要使用的键值映射。

​		以HID设备为例，首先内核中的键值转换在drivers/hid/[hid-input.c](https://android.googlesource.com/kernel/msm/+/android-4.4/drivers/hid/hid-input.c) 中进行映射，键值通道也有多种类型，例如：keyboard、consumer 等等；

　　//keyboard通道键值则是在如下数组添加修改：

```c
static const unsigned char hid_keyboard[256] = {
      0,  0,  0,  0, 30, 48, 46, 32, 18, 33, 34, 35, 23, 36, 37, 38,
     50, 49, 24, 25, 16, 19, 31, 20, 22, 47, 17, 45, 21, 44,  2,  3,
      4,  5,  6,  7,  8,  9, 10, 11, 28,  1, 14, 15, 57, 12, 13, 26,
     27, 43, 43, 39, 40, 41, 51, 52, 53, 58, 59, 60, 61, 62, 63, 64,
     65, 66, 67, 68, 87, 88, 99, 70,119,110,102,104,111,107,109,106,
    105,108,103, 69, 98, 55, 74, 78, 96, 79, 80, 81, 75, 76, 77, 71,
     72, 73, 82, 83, 86,127,116,117,183,184,185,186,187,188,189,190,
    191,192,193,194,134,138,130,132,128,129,131,137,133,135,136,113,
    115,114,unk,unk,unk,121,unk, 89, 93,124, 92, 94, 95,unk,unk,unk,
    122,123, 90, 91, 85,unk,unk,unk,unk,unk,unk,unk,111,unk,unk,unk,
    unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,
    unk,unk,unk,unk,unk,unk,179,180,unk,unk,unk,unk,unk,unk,unk,unk,
    unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,unk,
    unk,unk,unk,unk,unk,unk,unk,unk,111,unk,unk,unk,unk,unk,unk,unk,
     29, 42, 56,125, 97, 54,100,126,164,166,165,163,161,115,114,113,
    150,158,159,128,136,177,178,176,142,152,173,140,unk,unk,unk,unk
};

......

//然后在以下代码部分使用：
    case HID_UP_KEYBOARD:
        set_bit(EV_REP, input->evbit);

        if ((usage->hid & HID_USAGE) < 256) {
            if (!hid_keyboard[usage->hid & HID_USAGE]) goto ignore;
            map_key_clear(hid_keyboard[usage->hid & HID_USAGE]);
        } else
            map_key(KEY_UNKNOWN);

        break;
```

　//consumer通道键值则是在如下添加修改：

```c
case HID_UP_CONSUMER:    /* USB HUT v1.12, pages 75-84 */
        switch (usage->hid & HID_USAGE) {
        case 0x000: goto ignore;
        case 0x030: map_key_clear(KEY_POWER);        break;
        case 0x031: map_key_clear(KEY_RESTART);        break;
        case 0x032: map_key_clear(KEY_SLEEP);        break;
        case 0x034: map_key_clear(KEY_SLEEP);        break;
        case 0x035: map_key_clear(KEY_KBDILLUMTOGGLE);    break;
        case 0x036: map_key_clear(BTN_MISC);        break;
        case 0x040: map_key_clear(KEY_MENU);        break; /* Menu */
        case 0x041: map_key_clear(KEY_SELECT);        break; /* Menu Pick */
        case 0x042: map_key_clear(KEY_UP);        break; /* Menu Up */
        case 0x043: map_key_clear(KEY_DOWN);        break; /* Menu Down */
        case 0x044: map_key_clear(KEY_LEFT);        break; /* Menu Left */
        case 0x045: map_key_clear(KEY_RIGHT);        break; /* Menu Right */
        case 0x046: map_key_clear(KEY_ESC);        break; /* Menu Escape */
        case 0x047: map_key_clear(KEY_KPPLUS);        break; /* Menu Value Increase */
        case 0x048: map_key_clear(KEY_KPMINUS);        break; /* Menu Value Decrease */
        case 0x060: map_key_clear(KEY_INFO);        break; /* Data On Screen */
        case 0x061: map_key_clear(KEY_SUBTITLE);    break; /* Closed Caption */
        case 0x063: map_key_clear(KEY_VCR);        break; /* VCR/TV */
        case 0x065: map_key_clear(KEY_CAMERA);        break; /* Snapshot */
        case 0x069: map_key_clear(KEY_RED);        break;
        case 0x06a: map_key_clear(KEY_GREEN);        break;
        case 0x06b: map_key_clear(KEY_BLUE);        break;
        case 0x06c: map_key_clear(KEY_YELLOW);        break;
        case 0x06d: map_key_clear(KEY_ZOOM);        break;
        case 0x082: map_key_clear(KEY_VIDEO_NEXT);    break;
        case 0x083: map_key_clear(KEY_LAST);        break;
        case 0x084: map_key_clear(KEY_ENTER);        break;
        case 0x088: map_key_clear(KEY_PC);        break;
        case 0x089: map_key_clear(KEY_TV);        break;
        case 0x08a: map_key_clear(KEY_WWW);        break;
        case 0x08b: map_key_clear(KEY_DVD);        break;
        case 0x08c: map_key_clear(KEY_PHONE);        break;
        case 0x08d: map_key_clear(KEY_PROGRAM);        break;
        case 0x08e: map_key_clear(KEY_VIDEOPHONE);    break;
        case 0x08f: map_key_clear(KEY_GAMES);        break;
        case 0x090: map_key_clear(KEY_MEMO);        break;
        case 0x091: map_key_clear(KEY_CD);        break;
        case 0x092: map_key_clear(KEY_VCR);        break;
        case 0x093: map_key_clear(KEY_TUNER);        break;
        case 0x094: map_key_clear(KEY_EXIT);        break;
        case 0x095: map_key_clear(KEY_HELP);        break;
        case 0x096: map_key_clear(KEY_TAPE);        break;
        case 0x097: map_key_clear(KEY_TV2);        break;
        case 0x098: map_key_clear(KEY_SAT);        break;
        case 0x09a: map_key_clear(KEY_PVR);        break;
        case 0x09c: map_key_clear(KEY_CHANNELUP);    break;
        case 0x09d: map_key_clear(KEY_CHANNELDOWN);    break;
        case 0x0a0: map_key_clear(KEY_VCR2);        break;
        case 0x0b0: map_key_clear(KEY_PLAY);        break;
        case 0x0b1: map_key_clear(KEY_PAUSE);        break;
        case 0x0b2: map_key_clear(KEY_RECORD);        break;
        case 0x0b3: map_key_clear(KEY_FASTFORWARD);    break;
        case 0x0b4: map_key_clear(KEY_REWIND);        break;
        case 0x0b5: map_key_clear(KEY_NEXTSONG);    break;
        case 0x0b6: map_key_clear(KEY_PREVIOUSSONG);    break;
        case 0x0b7: map_key_clear(KEY_STOPCD);        break;
        case 0x0b8: map_key_clear(KEY_EJECTCD);        break;
        case 0x0bc: map_key_clear(KEY_MEDIA_REPEAT);    break;
        case 0x0b9: map_key_clear(KEY_SHUFFLE);        break;
        case 0x0bf: map_key_clear(KEY_SLOW);        break;
        case 0x0cd: map_key_clear(KEY_PLAYPAUSE);    break;
        case 0x0e0: map_abs_clear(ABS_VOLUME);        break;
        case 0x0e2: map_key_clear(KEY_MUTE);        break;
        case 0x0e5: map_key_clear(KEY_BASSBOOST);    break;
        case 0x0e9: map_key_clear(KEY_VOLUMEUP);    break;
        case 0x0ea: map_key_clear(KEY_VOLUMEDOWN);    break;
        case 0x0f5: map_key_clear(KEY_SLOW);        break;
        case 0x182: map_key_clear(KEY_BOOKMARKS);    break;
        case 0x183: map_key_clear(KEY_CONFIG);        break;
        case 0x184: map_key_clear(KEY_WORDPROCESSOR);    break;
        case 0x185: map_key_clear(KEY_EDITOR);        break;
        case 0x186: map_key_clear(KEY_SPREADSHEET);    break;
        case 0x187: map_key_clear(KEY_GRAPHICSEDITOR);    break;
        case 0x188: map_key_clear(KEY_PRESENTATION);    break;
        case 0x189: map_key_clear(KEY_DATABASE);    break;
        case 0x18a: map_key_clear(KEY_MAIL);        break;
        case 0x18b: map_key_clear(KEY_NEWS);        break;
        case 0x18c: map_key_clear(KEY_VOICEMAIL);    break;
        case 0x18d: map_key_clear(KEY_ADDRESSBOOK);    break;
        case 0x18e: map_key_clear(KEY_CALENDAR);    break;
        case 0x191: map_key_clear(KEY_FINANCE);        break;
        case 0x192: map_key_clear(KEY_CALC);        break;
        case 0x193: map_key_clear(KEY_PLAYER);        break;
        case 0x194: map_key_clear(KEY_FILE);        break;
        case 0x196: map_key_clear(KEY_WWW);        break;
        case 0x199: map_key_clear(KEY_CHAT);        break;
        case 0x19c: map_key_clear(KEY_LOGOFF);        break;
        case 0x19e: map_key_clear(KEY_COFFEE);        break;
        case 0x1a6: map_key_clear(KEY_HELP);        break;
        case 0x1a7: map_key_clear(KEY_DOCUMENTS);    break;
        case 0x1ab: map_key_clear(KEY_SPELLCHECK);    break;
        case 0x1ae: map_key_clear(KEY_KEYBOARD);    break;
        case 0x1b6: map_key_clear(KEY_IMAGES);        break;
        case 0x1b7: map_key_clear(KEY_AUDIO);        break;
        case 0x1b8: map_key_clear(KEY_VIDEO);        break;
        case 0x1bc: map_key_clear(KEY_MESSENGER);    break;
        case 0x1bd: map_key_clear(KEY_INFO);        break;
        case 0x201: map_key_clear(KEY_NEW);        break;
        case 0x202: map_key_clear(KEY_OPEN);        break;
        case 0x203: map_key_clear(KEY_CLOSE);        break;
        case 0x204: map_key_clear(KEY_EXIT);        break;
        case 0x207: map_key_clear(KEY_SAVE);        break;
        case 0x208: map_key_clear(KEY_PRINT);        break;
        case 0x209: map_key_clear(KEY_PROPS);        break;
        case 0x21a: map_key_clear(KEY_UNDO);        break;
        case 0x21b: map_key_clear(KEY_COPY);        break;
        case 0x21c: map_key_clear(KEY_CUT);        break;
        case 0x21d: map_key_clear(KEY_PASTE);        break;
        case 0x21f: map_key_clear(KEY_FIND);        break;
        case 0x221: map_key_clear(KEY_SEARCH);        break;
        case 0x222: map_key_clear(KEY_GOTO);        break;
        case 0x223: map_key_clear(KEY_HOMEPAGE);    break;
        case 0x224: map_key_clear(KEY_BACK);        break;
        case 0x225: map_key_clear(KEY_FORWARD);        break;
        case 0x226: map_key_clear(KEY_STOP);        break;
        case 0x227: map_key_clear(KEY_REFRESH);        break;
        case 0x22a: map_key_clear(KEY_BOOKMARKS);    break;
        case 0x22d: map_key_clear(KEY_ZOOMIN);        break;
        case 0x22e: map_key_clear(KEY_ZOOMOUT);        break;
        case 0x22f: map_key_clear(KEY_ZOOMRESET);    break;
        case 0x233: map_key_clear(KEY_SCROLLUP);    break;
        case 0x234: map_key_clear(KEY_SCROLLDOWN);    break;
        case 0x238: map_rel(REL_HWHEEL);        break;
        case 0x23d: map_key_clear(KEY_EDIT);        break;
        case 0x25f: map_key_clear(KEY_CANCEL);        break;
        case 0x269: map_key_clear(KEY_INSERT);        break;
        case 0x26a: map_key_clear(KEY_DELETE);        break;
        case 0x279: map_key_clear(KEY_REDO);        break;
        case 0x289: map_key_clear(KEY_REPLY);        break;
        case 0x28b: map_key_clear(KEY_FORWARDMAIL);    break;
        case 0x28c: map_key_clear(KEY_SEND);        break;
        default:    goto ignore;
        }
        break;
```

​		通过 **map_key_clear** 宏将原始键值（usage->hid & HID_USAGE）转换成内核的定义，映射函数的具体实现可看内核源码，

​		以上键值的定义在 include/uapi/linux/[input-event-codes.h](https://android.googlesource.com/kernel/msm/+/android-4.4/include/uapi/linux/input-event-codes.h) （内核代码,较新版本定义整合进了[input.h](https://android.googlesource.com/kernel/msm/+/android-msm-angler-3.10-marshmallow-mr1/include/uapi/linux/input.h)），对应到Android系统层的头文件则是 bionic/libc/kernel/uapi/linux/input-event-codes.h：

```c
#define KEY_RESERVED  0
#define KEY_ESC   1
#define KEY_1   2
#define KEY_2   3
#define KEY_3   4
#define KEY_4   5
#define KEY_5   6
#define KEY_6   7
#define KEY_7   8
#define KEY_8   9
#define KEY_9   10
#define KEY_0   11
#define KEY_MINUS  12
#define KEY_EQUAL  13
#define KEY_BACKSPACE  14
#define KEY_TAB   15
#define KEY_Q   16
#define KEY_W   17
#define KEY_E   18
#define KEY_R   19
#define KEY_T   20
#define KEY_Y   21
#define KEY_U   22
#define KEY_I   23
#define KEY_O   24
#define KEY_P   25
#define KEY_LEFTBRACE  26
#define KEY_RIGHTBRACE  27
#define KEY_ENTER  28
#define KEY_LEFTCTRL  29
#define KEY_A   30
#define KEY_S   31
#define KEY_D   32
#define KEY_F   33
#define KEY_G   34
#define KEY_H   35
#define KEY_J   36
#define KEY_K   37
#define KEY_L   38
#define KEY_SEMICOLON  39
#define KEY_APOSTROPHE  40
#define KEY_GRAVE  41
#define KEY_LEFTSHIFT  42
#define KEY_BACKSLASH  43
#define KEY_Z   44
#define KEY_X   45
#define KEY_C   46
#define KEY_V   47
#define KEY_B   48
#define KEY_N   49
#define KEY_M   50
#define KEY_COMMA  51
#define KEY_DOT   52
#define KEY_SLASH  53
#define KEY_RIGHTSHIFT  54
#define KEY_KPASTERISK  55
#define KEY_LEFTALT  56
#define KEY_SPACE  57
#define KEY_CAPSLOCK  58
#define KEY_F1   59
#define KEY_F2   60
#define KEY_F3   61
#define KEY_F4   62
#define KEY_F5   63
#define KEY_F6   64
#define KEY_F7   65
#define KEY_F8   66
#define KEY_F9   67
#define KEY_F10   68
#define KEY_NUMLOCK  69
#define KEY_SCROLLLOCK  70
#define KEY_KP7   71
#define KEY_KP8   72
#define KEY_KP9   73
#define KEY_KPMINUS  74
#define KEY_KP4   75
#define KEY_KP5   76
#define KEY_KP6   77
#define KEY_KPPLUS  78
#define KEY_KP1   79
#define KEY_KP2   80
#define KEY_KP3   81
#define KEY_KP0   82
#define KEY_KPDOT  83

#define KEY_ZENKAKUHANKAKU 85
#define KEY_102ND  86
#define KEY_F11   87
#define KEY_F12   88
#define KEY_RO   89
#define KEY_KATAKANA  90
#define KEY_HIRAGANA  91
#define KEY_HENKAN  92
#define KEY_KATAKANAHIRAGANA 93
#define KEY_MUHENKAN  94
#define KEY_KPJPCOMMA  95
#define KEY_KPENTER  96
#define KEY_RIGHTCTRL  97
#define KEY_KPSLASH  98
#define KEY_SYSRQ  99
#define KEY_RIGHTALT  100
#define KEY_LINEFEED  101
#define KEY_HOME  102
#define KEY_UP   103
#define KEY_PAGEUP  104
#define KEY_LEFT  105
#define KEY_RIGHT  106
#define KEY_END   107
#define KEY_DOWN  108
#define KEY_PAGEDOWN  109
#define KEY_INSERT  110
#define KEY_DELETE  111
#define KEY_MACRO  112
#define KEY_MUTE  113
#define KEY_VOLUMEDOWN  114
#define KEY_VOLUMEUP  115
#define KEY_POWER  116 /* SC System Power Down */
#define KEY_KPEQUAL  117
#define KEY_KPPLUSMINUS  118
#define KEY_PAUSE  119
#define KEY_SCALE  120 /* AL Compiz Scale (Expose) */

#define KEY_KPCOMMA  121
#define KEY_HANGEUL  122
#define KEY_HANGUEL  KEY_HANGEUL
#define KEY_HANJA  123
#define KEY_YEN   124
#define KEY_LEFTMETA  125
#define KEY_RIGHTMETA  126
#define KEY_COMPOSE  127

#define KEY_STOP  128 /* AC Stop */
#define KEY_AGAIN  129
#define KEY_PROPS  130 /* AC Properties */
#define KEY_UNDO  131 /* AC Undo */
#define KEY_FRONT  132
#define KEY_COPY  133 /* AC Copy */
#define KEY_OPEN  134 /* AC Open */
#define KEY_PASTE  135 /* AC Paste */
#define KEY_FIND  136 /* AC Search */
#define KEY_CUT   137 /* AC Cut */
#define KEY_HELP  138 /* AL Integrated Help Center */
#define KEY_MENU  139 /* Menu (show menu) */
#define KEY_CALC  140 /* AL Calculator */
#define KEY_SETUP  141
#define KEY_SLEEP  142 /* SC System Sleep */
#define KEY_WAKEUP  143 /* System Wake Up */
#define KEY_FILE  144 /* AL Local Machine Browser */
#define KEY_SENDFILE  145
#define KEY_DELETEFILE  146
#define KEY_XFER  147
#define KEY_PROG1  148
#define KEY_PROG2  149
#define KEY_WWW   150 /* AL Internet Browser */
#define KEY_MSDOS  151
#define KEY_COFFEE  152 /* AL Terminal Lock/Screensaver */
#define KEY_SCREENLOCK  KEY_COFFEE
#define KEY_DIRECTION  153
#define KEY_CYCLEWINDOWS 154
#define KEY_MAIL  155
#define KEY_BOOKMARKS  156 /* AC Bookmarks */
#define KEY_COMPUTER  157
#define KEY_BACK  158 /* AC Back */
#define KEY_FORWARD  159 /* AC Forward */
#define KEY_CLOSECD  160
#define KEY_EJECTCD  161
#define KEY_EJECTCLOSECD 162
#define KEY_NEXTSONG  163
#define KEY_PLAYPAUSE  164
#define KEY_PREVIOUSSONG 165
#define KEY_STOPCD  166
#define KEY_RECORD  167
#define KEY_REWIND  168
#define KEY_PHONE  169 /* Media Select Telephone */
#define KEY_ISO   170
#define KEY_CONFIG  171 /* AL Consumer Control Configuration */
#define KEY_HOMEPAGE  172 /* AC Home */
#define KEY_REFRESH  173 /* AC Refresh */
#define KEY_EXIT  174 /* AC Exit */
#define KEY_MOVE  175
#define KEY_EDIT  176
#define KEY_SCROLLUP  177
#define KEY_SCROLLDOWN  178
#define KEY_KPLEFTPAREN  179
#define KEY_KPRIGHTPAREN 180
#define KEY_NEW   181 /* AC New */
#define KEY_REDO  182 /* AC Redo/Repeat */

#define KEY_F13   183
#define KEY_F14   184
#define KEY_F15   185
#define KEY_F16   186
#define KEY_F17   187
#define KEY_F18   188
#define KEY_F19   189
#define KEY_F20   190
#define KEY_F21   191
#define KEY_F22   192
#define KEY_F23   193
#define KEY_F24   194

#define KEY_PLAYCD  200
#define KEY_PAUSECD  201
#define KEY_PROG3  202
#define KEY_PROG4  203
#define KEY_DASHBOARD  204 /* AL Dashboard */
#define KEY_SUSPEND  205
#define KEY_CLOSE  206 /* AC Close */
#define KEY_PLAY  207
#define KEY_FASTFORWARD  208
#define KEY_BASSBOOST  209
#define KEY_PRINT  210 /* AC Print */
#define KEY_HP   211
#define KEY_CAMERA  212
#define KEY_SOUND  213
#define KEY_QUESTION  214
#define KEY_EMAIL  215
#define KEY_CHAT  216
#define KEY_SEARCH  217
#define KEY_CONNECT  218
#define KEY_FINANCE  219 /* AL Checkbook/Finance */
#define KEY_SPORT  220
#define KEY_SHOP  221
#define KEY_ALTERASE  222
#define KEY_CANCEL  223 /* AC Cancel */
#define KEY_BRIGHTNESSDOWN 224
#define KEY_BRIGHTNESSUP 225
#define KEY_MEDIA  226

#define KEY_SWITCHVIDEOMODE 227 /* Cycle between available video
        outputs (Monitor/LCD/TV-out/etc) */
#define KEY_KBDILLUMTOGGLE 228
#define KEY_KBDILLUMDOWN 229
#define KEY_KBDILLUMUP  230

#define KEY_SEND  231 /* AC Send */
#define KEY_REPLY  232 /* AC Reply */
#define KEY_FORWARDMAIL  233 /* AC Forward Msg */
#define KEY_SAVE  234 /* AC Save */
#define KEY_DOCUMENTS  235

#define KEY_BATTERY  236

#define KEY_BLUETOOTH  237
#define KEY_WLAN  238
#define KEY_UWB   239

#define KEY_UNKNOWN  240

#define KEY_VIDEO_NEXT  241 /* drive next video source */
#define KEY_VIDEO_PREV  242 /* drive previous video source */
#define KEY_BRIGHTNESS_CYCLE 243 /* brightness up, after max is min */
#define KEY_BRIGHTNESS_AUTO 244 /* Set Auto Brightness: manual
       brightness control is off,
       rely on ambient */
#define KEY_BRIGHTNESS_ZERO KEY_BRIGHTNESS_AUTO
#define KEY_DISPLAY_OFF  245 /* display device to off state */

#define KEY_WWAN  246 /* Wireless WAN (LTE, UMTS, GSM, etc.) */
#define KEY_WIMAX  KEY_WWAN
#define KEY_RFKILL  247 /* Key that controls all radios */

#define KEY_MICMUTE  248 /* Mute / unmute the microphone */

/* Code 255 is reserved for special needs of AT keyboard driver */
```

可通过 getevent 指令查看上报的键值，键值为十六进制显示：

![821933-20180816205242760-1243866432](assets/821933-20180816205242760-1243866432.png)

 		首先要确定按键输入设备是对应/dev/input目录下哪个event，根据VID PID匹配对应的kl文件，可通过如下命令 **cat /proc/bus/input/devices** 查看设备信息：

```shell
shell@flo:/ $ cat /proc/bus/input/devices
I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="elan-touchscreen"
P: Phys=
S: Sysfs=/devices/virtual/input/input0
U: Uniq=
H: Handlers=event0 cpufreq
B: PROP=2
B: EV=b
B: KEY=0
B: ABS=6618000 0

I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="lid_input"
P: Phys=/dev/input/lid_indev
S: Sysfs=/devices/virtual/input/input1
U: Uniq=
H: Handlers=event1 cpufreq
B: PROP=0
B: EV=21
B: SW=1

I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="apq8064-tabla-snd-card Button Jack"
P: Phys=ALSA
S: Sysfs=/devices/platform/soc-audio.0/sound/card0/input2
U: Uniq=
H: Handlers=event2 cpufreq
B: PROP=0
B: EV=3
B: KEY=ff 0 0 0 0 0 0 0 0

I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="apq8064-tabla-snd-card Headset Jack"
P: Phys=ALSA
S: Sysfs=/devices/platform/soc-audio.0/sound/card0/input3
U: Uniq=
H: Handlers=event3 cpufreq
B: PROP=0
B: EV=21
B: SW=1c054

I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="h2w button"
P: Phys=
S: Sysfs=/devices/virtual/input/input4
U: Uniq=
H: Handlers=kbd event4 cpufreq
B: PROP=0
B: EV=3
B: KEY=4 0 0 0 0 0 0 0

I: Bus=0019 Vendor=0001 Product=0001 Version=0100
N: Name="gpio-keys"
P: Phys=gpio-keys/input0
S: Sysfs=/devices/platform/gpio-keys.0/input/input5
U: Uniq=
H: Handlers=kbd event5 keychord cpufreq
B: PROP=0
B: EV=3
B: KEY=1c0000 0 0 0

I: Bus=0005 Vendor=0030 Product=001D Version=0101
N: Name="Smart Remote"
P: Phys=
S: Sysfs=/devices/virtual/misc/uhid/input7
U: Uniq=
H: Handlers=sysrq kbd event6 keychord cpufreq
B: PROP=0
B: EV=10001f
B: KEY=4837fff 72ff32d bf544446 0 0 1 30f90 8b17c007 ffff7bfa d9415fff febeffdf ffefffff ffffffff fffffffe
B: REL=40
B: ABS=1 0
B: MSC=10
```

​		如上我的设备名是："Smart Remote" ， VID PID信息是：Vendor=0030 Product=001D ,则对应 /system/usr/keylayout/Vendor_0030_Product_001D.kl,如果该目录下没有对应VID  PID的.kl则默认使用 Generic.kl，根据系统差异可能另外有  /data/system/devices/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  ，此外还有键值对应字符的转换表：/system/usr/keychars/Generic.kcm 。

​		所以上面通过getevent获得的原始键值是0x44(十进制：68)，然后由 **hid-input.c** 可知 hid_keyboard[68]=87 而 **input.h** 中定义是 #define KEY_F11 87，所以Android系统层获取到内核转换后的键值为<87>，然后通过加载Generic.kl解析<87>的含义是"F11"（一般都是和内核头文件定义相匹配，也可以自己修改使其映射成其他含义）:

```c
#
# Generic key layout file for full alphabetic US English PC style external keyboards.
#
# This file is intentionally very generic and is intended to support a broad rang of keyboards.
# Do not edit the generic key layout to support a specific keyboard; instead, create
# a new key layout file with the required keyboard configuration.
#

key 1     ESCAPE
key 2     1
key 3     2
key 4     3
key 5     4
key 6     5
key 7     6
key 8     7
key 9     8
key 10    9
key 11    0
key 12    MINUS
key 13    EQUALS
key 14    DEL
key 15    TAB
key 16    Q
key 17    W
key 18    E
key 19    R
key 20    T
key 21    Y
key 22    U
key 23    I
key 24    O
key 25    P
key 26    LEFT_BRACKET
key 27    RIGHT_BRACKET
key 28    ENTER
key 29    CTRL_LEFT
key 30    A
key 31    S
key 32    D
key 33    F
key 34    G
key 35    H
key 36    J
key 37    K
key 38    L
key 39    SEMICOLON
key 40    APOSTROPHE
key 41    GRAVE
key 42    SHIFT_LEFT
key 43    BACKSLASH
key 44    Z
key 45    X
key 46    C
key 47    V
key 48    B
key 49    N
key 50    M
key 51    COMMA
key 52    PERIOD
key 53    SLASH
key 54    SHIFT_RIGHT
key 55    NUMPAD_MULTIPLY
key 56    ALT_LEFT
key 57    SPACE
key 58    CAPS_LOCK
key 59    F1
key 60    F2
key 61    F3
key 62    F4
key 63    F5
key 64    F6
key 65    F7
key 66    F8
key 67    F9
key 68    F10
key 69    NUM_LOCK
key 70    SCROLL_LOCK
key 71    NUMPAD_7
key 72    NUMPAD_8
key 73    NUMPAD_9
key 74    NUMPAD_SUBTRACT
key 75    NUMPAD_4
key 76    NUMPAD_5
key 77    NUMPAD_6
key 78    NUMPAD_ADD
key 79    NUMPAD_1
key 80    NUMPAD_2
key 81    NUMPAD_3
key 82    NUMPAD_0
key 83    NUMPAD_DOT
# key 84 (undefined)
key 85    ZENKAKU_HANKAKU
key 86    BACKSLASH
key 87    F11
key 88    F12
key 89    RO
# key 90 "KEY_KATAKANA"
# key 91 "KEY_HIRAGANA"
key 92    HENKAN
key 93    KATAKANA_HIRAGANA
key 94    MUHENKAN
key 95    NUMPAD_COMMA
key 96    NUMPAD_ENTER
key 97    CTRL_RIGHT
key 98    NUMPAD_DIVIDE
key 99    SYSRQ
key 100   ALT_RIGHT
# key 101 "KEY_LINEFEED"
key 102   MOVE_HOME
key 103   DPAD_UP
key 104   PAGE_UP
key 105   DPAD_LEFT
key 106   DPAD_RIGHT
key 107   MOVE_END
key 108   DPAD_DOWN
key 109   PAGE_DOWN
key 110   INSERT
key 111   FORWARD_DEL
# key 112 "KEY_MACRO"
key 113   VOLUME_MUTE
key 114   VOLUME_DOWN
key 115   VOLUME_UP
key 116   POWER             WAKE
key 117   NUMPAD_EQUALS
# key 118 "KEY_KPPLUSMINUS"
key 119   BREAK
# key 120 (undefined)
key 121   NUMPAD_COMMA
key 122   KANA
key 123   EISU
key 124   YEN
key 125   META_LEFT
key 126   META_RIGHT
key 127   MENU              WAKE_DROPPED
key 128   MEDIA_STOP
# key 129 "KEY_AGAIN"
# key 130 "KEY_PROPS"
# key 131 "KEY_UNDO"
# key 132 "KEY_FRONT"
# key 133 "KEY_COPY"
# key 134 "KEY_OPEN"
# key 135 "KEY_PASTE"
# key 136 "KEY_FIND"
# key 137 "KEY_CUT"
# key 138 "KEY_HELP"
key 139   MENU              WAKE_DROPPED
key 140   CALCULATOR
# key 141 "KEY_SETUP"
key 142   POWER             WAKE
key 143   POWER             WAKE
# key 144 "KEY_FILE"
# key 145 "KEY_SENDFILE"
# key 146 "KEY_DELETEFILE"
# key 147 "KEY_XFER"
# key 148 "KEY_PROG1"
# key 149 "KEY_PROG2"
key 150   EXPLORER
# key 151 "KEY_MSDOS"
key 152   POWER             WAKE
# key 153 "KEY_DIRECTION"
# key 154 "KEY_CYCLEWINDOWS"
key 155   ENVELOPE
key 156   BOOKMARK
# key 157 "KEY_COMPUTER"
key 158   BACK              WAKE_DROPPED
key 159   FORWARD
key 160   MEDIA_CLOSE
key 161   MEDIA_EJECT
key 162   MEDIA_EJECT
key 163   MEDIA_NEXT
key 164   MEDIA_PLAY_PAUSE
key 165   MEDIA_PREVIOUS
key 166   MEDIA_STOP
key 167   MEDIA_RECORD
key 168   MEDIA_REWIND
key 169   CALL
# key 170 "KEY_ISO"
key 171   MUSIC
key 172   HOME
# key 173 "KEY_REFRESH"
# key 174 "KEY_EXIT"
# key 175 "KEY_MOVE"
# key 176 "KEY_EDIT"
key 177   PAGE_UP
key 178   PAGE_DOWN
key 179   NUMPAD_LEFT_PAREN
key 180   NUMPAD_RIGHT_PAREN
# key 181 "KEY_NEW"
# key 182 "KEY_REDO"
# key 183   F13
# key 184   F14
# key 185   F15
# key 186   F16
# key 187   F17
# key 188   F18
# key 189   F19
# key 190   F20
# key 191   F21
# key 192   F22
# key 193   F23
# key 194   F24
# key 195 (undefined)
# key 196 (undefined)
# key 197 (undefined)
# key 198 (undefined)
# key 199 (undefined)
key 200   MEDIA_PLAY
key 201   MEDIA_PAUSE
# key 202 "KEY_PROG3"
# key 203 "KEY_PROG4"
# key 204 (undefined)
# key 205 "KEY_SUSPEND"
# key 206 "KEY_CLOSE"
key 207   MEDIA_PLAY
key 208   MEDIA_FAST_FORWARD
# key 209 "KEY_BASSBOOST"
# key 210 "KEY_PRINT"
# key 211 "KEY_HP"
key 212   CAMERA
key 213   MUSIC
# key 214 "KEY_QUESTION"
key 215   ENVELOPE
# key 216 "KEY_CHAT"
key 217   SEARCH
# key 218 "KEY_CONNECT"
# key 219 "KEY_FINANCE"
# key 220 "KEY_SPORT"
# key 221 "KEY_SHOP"
# key 222 "KEY_ALTERASE"
# key 223 "KEY_CANCEL"
key 224   BRIGHTNESS_DOWN
key 225   BRIGHTNESS_UP
key 226   HEADSETHOOK

key 256   BUTTON_1
key 257   BUTTON_2
key 258   BUTTON_3
key 259   BUTTON_4
key 260   BUTTON_5
key 261   BUTTON_6
key 262   BUTTON_7
key 263   BUTTON_8
key 264   BUTTON_9
key 265   BUTTON_10
key 266   BUTTON_11
key 267   BUTTON_12
key 268   BUTTON_13
key 269   BUTTON_14
key 270   BUTTON_15
key 271   BUTTON_16

key 288   BUTTON_1
key 289   BUTTON_2
key 290   BUTTON_3
key 291   BUTTON_4
key 292   BUTTON_5
key 293   BUTTON_6
key 294   BUTTON_7
key 295   BUTTON_8
key 296   BUTTON_9
key 297   BUTTON_10
key 298   BUTTON_11
key 299   BUTTON_12
key 300   BUTTON_13
key 301   BUTTON_14
key 302   BUTTON_15
key 303   BUTTON_16


key 304   BUTTON_A
key 305   BUTTON_B
key 306   BUTTON_C
key 307   BUTTON_X
key 308   BUTTON_Y
key 309   BUTTON_Z
key 310   BUTTON_L1
key 311   BUTTON_R1
key 312   BUTTON_L2
key 313   BUTTON_R2
key 314   BUTTON_SELECT
key 315   BUTTON_START
key 316   BUTTON_MODE
key 317   BUTTON_THUMBL
key 318   BUTTON_THUMBR


# key 352 "KEY_OK"
key 353   DPAD_CENTER
# key 354 "KEY_GOTO"
# key 355 "KEY_CLEAR"
# key 356 "KEY_POWER2"
# key 357 "KEY_OPTION"
# key 358 "KEY_INFO"
# key 359 "KEY_TIME"
# key 360 "KEY_VENDOR"
# key 361 "KEY_ARCHIVE"
key 362   GUIDE
# key 363 "KEY_CHANNEL"
# key 364 "KEY_FAVORITES"
# key 365 "KEY_EPG"
key 366   DVR
# key 367 "KEY_MHP"
# key 368 "KEY_LANGUAGE"
# key 369 "KEY_TITLE"
# key 370 "KEY_SUBTITLE"
# key 371 "KEY_ANGLE"
# key 372 "KEY_ZOOM"
# key 373 "KEY_MODE"
# key 374 "KEY_KEYBOARD"
# key 375 "KEY_SCREEN"
# key 376 "KEY_PC"
key 377   TV
# key 378 "KEY_TV2"
# key 379 "KEY_VCR"
# key 380 "KEY_VCR2"
# key 381 "KEY_SAT"
# key 382 "KEY_SAT2"
# key 383 "KEY_CD"
# key 384 "KEY_TAPE"
# key 385 "KEY_RADIO"
# key 386 "KEY_TUNER"
# key 387 "KEY_PLAYER"
# key 388 "KEY_TEXT"
# key 389 "KEY_DVD"
# key 390 "KEY_AUX"
# key 391 "KEY_MP3"
# key 392 "KEY_AUDIO"
# key 393 "KEY_VIDEO"
# key 394 "KEY_DIRECTORY"
# key 395 "KEY_LIST"
# key 396 "KEY_MEMO"
key 397   CALENDAR
# key 398 "KEY_RED"
# key 399 "KEY_GREEN"
# key 400 "KEY_YELLOW"
# key 401 "KEY_BLUE"
key 402   CHANNEL_UP
key 403   CHANNEL_DOWN
# key 404 "KEY_FIRST"
# key 405 "KEY_LAST"
# key 406 "KEY_AB"
# key 407 "KEY_NEXT"
# key 408 "KEY_RESTART"
# key 409 "KEY_SLOW"
# key 410 "KEY_SHUFFLE"
# key 411 "KEY_BREAK"
# key 412 "KEY_PREVIOUS"
# key 413 "KEY_DIGITS"
# key 414 "KEY_TEEN"
# key 415 "KEY_TWEN"

key 429   CONTACTS

# key 448 "KEY_DEL_EOL"
# key 449 "KEY_DEL_EOS"
# key 450 "KEY_INS_LINE"
# key 451 "KEY_DEL_LINE"


key 464   FUNCTION
key 465   ESCAPE            FUNCTION
key 466   F1                FUNCTION
key 467   F2                FUNCTION
key 468   F3                FUNCTION
key 469   F4                FUNCTION
key 470   F5                FUNCTION
key 471   F6                FUNCTION
key 472   F7                FUNCTION
key 473   F8                FUNCTION
key 474   F9                FUNCTION
key 475   F10               FUNCTION
key 476   F11               FUNCTION
key 477   F12               FUNCTION
key 478   1                 FUNCTION
key 479   2                 FUNCTION
key 480   D                 FUNCTION
key 481   E                 FUNCTION
key 482   F                 FUNCTION
key 483   S                 FUNCTION
key 484   B                 FUNCTION


# key 497 KEY_BRL_DOT1
# key 498 KEY_BRL_DOT2
# key 499 KEY_BRL_DOT3
# key 500 KEY_BRL_DOT4
# key 501 KEY_BRL_DOT5
# key 502 KEY_BRL_DOT6
# key 503 KEY_BRL_DOT7
# key 504 KEY_BRL_DOT8

# Keys defined by HID usages
key usage 0x0c006F BRIGHTNESS_UP
key usage 0x0c0070 BRIGHTNESS_DOWN

# Joystick and game controller axes.
# Axes that are not mapped will be assigned generic axis numbers by the input subsystem.
axis 0x00 X
axis 0x01 Y
axis 0x02 Z
axis 0x03 RX
axis 0x04 RY
axis 0x05 RZ
axis 0x06 THROTTLE
axis 0x07 RUDDER
axis 0x08 WHEEL
axis 0x09 GAS
axis 0x0a BRAKE
axis 0x10 HAT_X
axis 0x11 HAT_Y
```

键值从底层上报到上层的流程简图如下：

![821933-20180816202424670-516556732](assets/821933-20180816202424670-516556732.png)

　　　　　　                    图2：键值上报流程

　　从上图可以看到，framework层通过.kl文件将获取的键值转换成实际按键含义后，又会通过KeycodeLabel转换成相应的keycode,具体文件在：frameworks\native\include\input\KeycodeLabels.h(android 4.4.4源码)：

```c
/*
 * Copyright (C) 2008 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef _LIBINPUT_KEYCODE_LABELS_H
#define _LIBINPUT_KEYCODE_LABELS_H

#include <android/keycodes.h>

struct KeycodeLabel {
    const char *literal;
    int value;
};

static const KeycodeLabel KEYCODES[] = {
    { "SOFT_LEFT", 1 },
    { "SOFT_RIGHT", 2 },
    { "HOME", 3 },
    { "BACK", 4 },
    { "CALL", 5 },
    { "ENDCALL", 6 },
    { "0", 7 },
    { "1", 8 },
    { "2", 9 },
    { "3", 10 },
    { "4", 11 },
    { "5", 12 },
    { "6", 13 },
    { "7", 14 },
    { "8", 15 },
    { "9", 16 },
    { "STAR", 17 },
    { "POUND", 18 },
    { "DPAD_UP", 19 },
    { "DPAD_DOWN", 20 },
    { "DPAD_LEFT", 21 },
    { "DPAD_RIGHT", 22 },
    { "DPAD_CENTER", 23 },
    { "VOLUME_UP", 24 },
    { "VOLUME_DOWN", 25 },
    { "POWER", 26 },
    { "CAMERA", 27 },
    { "CLEAR", 28 },
    { "A", 29 },
    { "B", 30 },
    { "C", 31 },
    { "D", 32 },
    { "E", 33 },
    { "F", 34 },
    { "G", 35 },
    { "H", 36 },
    { "I", 37 },
    { "J", 38 },
    { "K", 39 },
    { "L", 40 },
    { "M", 41 },
    { "N", 42 },
    { "O", 43 },
    { "P", 44 },
    { "Q", 45 },
    { "R", 46 },
    { "S", 47 },
    { "T", 48 },
    { "U", 49 },
    { "V", 50 },
    { "W", 51 },
    { "X", 52 },
    { "Y", 53 },
    { "Z", 54 },
    { "COMMA", 55 },
    { "PERIOD", 56 },
    { "ALT_LEFT", 57 },
    { "ALT_RIGHT", 58 },
    { "SHIFT_LEFT", 59 },
    { "SHIFT_RIGHT", 60 },
    { "TAB", 61 },
    { "SPACE", 62 },
    { "SYM", 63 },
    { "EXPLORER", 64 },
    { "ENVELOPE", 65 },
    { "ENTER", 66 },
    { "DEL", 67 },
    { "GRAVE", 68 },
    { "MINUS", 69 },
    { "EQUALS", 70 },
    { "LEFT_BRACKET", 71 },
    { "RIGHT_BRACKET", 72 },
    { "BACKSLASH", 73 },
    { "SEMICOLON", 74 },
    { "APOSTROPHE", 75 },
    { "SLASH", 76 },
    { "AT", 77 },
    { "NUM", 78 },
    { "HEADSETHOOK", 79 },
    { "FOCUS", 80 },
    { "PLUS", 81 },
    { "MENU", 82 },
    { "NOTIFICATION", 83 },
    { "SEARCH", 84 },
    { "MEDIA_PLAY_PAUSE", 85 },
    { "MEDIA_STOP", 86 },
    { "MEDIA_NEXT", 87 },
    { "MEDIA_PREVIOUS", 88 },
    { "MEDIA_REWIND", 89 },
    { "MEDIA_FAST_FORWARD", 90 },
    { "MUTE", 91 },
    { "PAGE_UP", 92 },
    { "PAGE_DOWN", 93 },
    { "PICTSYMBOLS", 94 },
    { "SWITCH_CHARSET", 95 },
    { "BUTTON_A", 96 },
    { "BUTTON_B", 97 },
    { "BUTTON_C", 98 },
    { "BUTTON_X", 99 },
    { "BUTTON_Y", 100 },
    { "BUTTON_Z", 101 },
    { "BUTTON_L1", 102 },
    { "BUTTON_R1", 103 },
    { "BUTTON_L2", 104 },
    { "BUTTON_R2", 105 },
    { "BUTTON_THUMBL", 106 },
    { "BUTTON_THUMBR", 107 },
    { "BUTTON_START", 108 },
    { "BUTTON_SELECT", 109 },
    { "BUTTON_MODE", 110 },
    { "ESCAPE", 111 },
    { "FORWARD_DEL", 112 },
    { "CTRL_LEFT", 113 },
    { "CTRL_RIGHT", 114 },
    { "CAPS_LOCK", 115 },
    { "SCROLL_LOCK", 116 },
    { "META_LEFT", 117 },
    { "META_RIGHT", 118 },
    { "FUNCTION", 119 },
    { "SYSRQ", 120 },
    { "BREAK", 121 },
    { "MOVE_HOME", 122 },
    { "MOVE_END", 123 },
    { "INSERT", 124 },
    { "FORWARD", 125 },
    { "MEDIA_PLAY", 126 },
    { "MEDIA_PAUSE", 127 },
    { "MEDIA_CLOSE", 128 },
    { "MEDIA_EJECT", 129 },
    { "MEDIA_RECORD", 130 },
    { "F1", 131 },
    { "F2", 132 },
    { "F3", 133 },
    { "F4", 134 },
    { "F5", 135 },
    { "F6", 136 },
    { "F7", 137 },
    { "F8", 138 },
    { "F9", 139 },
    { "F10", 140 },
    //{ "F11", 141 },
    { "F11", 546 },
    { "F12", 142 },
    { "NUM_LOCK", 143 },
    { "NUMPAD_0", 144 },
    { "NUMPAD_1", 145 },
    { "NUMPAD_2", 146 },
    { "NUMPAD_3", 147 },
    { "NUMPAD_4", 148 },
    { "NUMPAD_5", 149 },
    { "NUMPAD_6", 150 },
    { "NUMPAD_7", 151 },
    { "NUMPAD_8", 152 },
    { "NUMPAD_9", 153 },
    { "NUMPAD_DIVIDE", 154 },
    { "NUMPAD_MULTIPLY", 155 },
    { "NUMPAD_SUBTRACT", 156 },
    { "NUMPAD_ADD", 157 },
    { "NUMPAD_DOT", 158 },
    { "NUMPAD_COMMA", 159 },
    { "NUMPAD_ENTER", 160 },
    { "NUMPAD_EQUALS", 161 },
    { "NUMPAD_LEFT_PAREN", 162 },
    { "NUMPAD_RIGHT_PAREN", 163 },
    { "VOLUME_MUTE", 164 },
    { "INFO", 165 },
    { "CHANNEL_UP", 166 },
    { "CHANNEL_DOWN", 167 },
    { "ZOOM_IN", 168 },
    { "ZOOM_OUT", 169 },
    { "TV", 170 },
    { "WINDOW", 171 },
    { "GUIDE", 172 },
    { "DVR", 173 },
    { "BOOKMARK", 174 },
    { "CAPTIONS", 175 },
    { "SETTINGS", 176 },
    { "TV_POWER", 177 },
    { "TV_INPUT", 178 },
    { "STB_POWER", 179 },
    { "STB_INPUT", 180 },
    { "AVR_POWER", 181 },
    { "AVR_INPUT", 182 },
    { "PROG_RED", 183 },
    { "PROG_GREEN", 184 },
    { "PROG_YELLOW", 185 },
    { "PROG_BLUE", 186 },
    { "APP_SWITCH", 187 },
    { "BUTTON_1", 188 },
    { "BUTTON_2", 189 },
    { "BUTTON_3", 190 },
    { "BUTTON_4", 191 },
    { "BUTTON_5", 192 },
    { "BUTTON_6", 193 },
    { "BUTTON_7", 194 },
    { "BUTTON_8", 195 },
    { "BUTTON_9", 196 },
    { "BUTTON_10", 197 },
    { "BUTTON_11", 198 },
    { "BUTTON_12", 199 },
    { "BUTTON_13", 200 },
    { "BUTTON_14", 201 },
    { "BUTTON_15", 202 },
    { "BUTTON_16", 203 },
    { "LANGUAGE_SWITCH", 204 },
    { "MANNER_MODE", 205 },
    { "3D_MODE", 206 },
    { "CONTACTS", 207 },
    { "CALENDAR", 208 },
    { "MUSIC", 209 },
    { "CALCULATOR", 210 },
    { "ZENKAKU_HANKAKU", 211 },
    { "EISU", 212 },
    { "MUHENKAN", 213 },
    { "HENKAN", 214 },
    { "KATAKANA_HIRAGANA", 215 },
    { "YEN", 216 },
    { "RO", 217 },
    { "KANA", 218 },
    { "ASSIST", 219 },
    { "BRIGHTNESS_DOWN", 220 },
    { "BRIGHTNESS_UP", 221 },
    { "MEDIA_AUDIO_TRACK", 222 },

    // NOTE: If you add a new keycode here you must also add it to several other files.
    //       Refer to frameworks/base/core/java/android/view/KeyEvent.java for the full list.

    { NULL, 0 }
};

// NOTE: If you edit these flags, also edit policy flags in Input.h.
static const KeycodeLabel FLAGS[] = {
    { "WAKE", 0x00000001 },
    { "WAKE_DROPPED", 0x00000002 },
    { "SHIFT", 0x00000004 },
    { "CAPS_LOCK", 0x00000008 },
    { "ALT", 0x00000010 },
    { "ALT_GR", 0x00000020 },
    { "MENU", 0x00000040 },
    { "LAUNCHER", 0x00000080 },
    { "VIRTUAL", 0x00000100 },
    { "FUNCTION", 0x00000200 },
    { NULL, 0 }
};

static const KeycodeLabel AXES[] = {
    { "X", 0 },
    { "Y", 1 },
    { "PRESSURE", 2 },
    { "SIZE", 3 },
    { "TOUCH_MAJOR", 4 },
    { "TOUCH_MINOR", 5 },
    { "TOOL_MAJOR", 6 },
    { "TOOL_MINOR", 7 },
    { "ORIENTATION", 8 },
    { "VSCROLL", 9 },
    { "HSCROLL", 10 },
    { "Z", 11 },
    { "RX", 12 },
    { "RY", 13 },
    { "RZ", 14 },
    { "HAT_X", 15 },
    { "HAT_Y", 16 },
    { "LTRIGGER", 17 },
    { "RTRIGGER", 18 },
    { "THROTTLE", 19 },
    { "RUDDER", 20 },
    { "WHEEL", 21 },
    { "GAS", 22 },
    { "BRAKE", 23 },
    { "DISTANCE", 24 },
    { "TILT", 25 },
    { "GENERIC_1", 32 },
    { "GENERIC_2", 33 },
    { "GENERIC_3", 34 },
    { "GENERIC_4", 35 },
    { "GENERIC_5", 36 },
    { "GENERIC_6", 37 },
    { "GENERIC_7", 38 },
    { "GENERIC_8", 39 },
    { "GENERIC_9", 40 },
    { "GENERIC_10", 41 },
    { "GENERIC_11", 42 },
    { "GENERIC_12", 43 },
    { "GENERIC_13", 44 },
    { "GENERIC_14", 45 },
    { "GENERIC_15", 46 },
    { "GENERIC_16", 47 },

    // NOTE: If you add a new axis here you must also add it to several other files.
    //       Refer to frameworks/base/core/java/android/view/MotionEvent.java for the full list.

    { NULL, -1 }
};

#endif // _LIBINPUT_KEYCODE_LABELS_H
```

​		然后app可以通过如下方法获得对应键按下时的keyCode值，即“F11”对应获得的keyCode即为上面自定义的<546>：

```java
    public boolean onKeyDown(int keyCode, KeyEvent event) { //重写的键盘按下监听
        alert("获取系统keyCode : "+ keyCode);
        return super.onKeyDown(keyCode, event);
    }
```

##  四、添加自定义的键值：

1、Kernel层：

　　　　**①** include/uapi/linux/input.h 中添加: #define KEY_LXL        123
　　　　**②** drivers/hid/hid-input.c 中添加:        case **0x188**: map_key_clear(KEY_LXL);   break;  //其中**0x188**是HID设备上报的原始键值

2、Android系统层：

（1）定义按键对应的key label**

在KEYCODES[]数组的最后添加按键的key label，即:

```c
static const KeycodeLabel KEYCODES[] = {
    …
DEFINE_KEYCODE(HELP)
DEFINE_KEYCODE(CAMERA )
};
```

*位置*:

> **Android 4.4** **以前版本 frameworks/base/include/ui/KeycodeLabels.h**  的KEYCODES[]数组中添加： **{ "LXL", 666 },**
>
> **Android 4.4** **在framework/native/include/input/KeyCodelabels.h**
>
> **Android5.0** **以后在framework/native/include/input/InputEventLabels.h**

(2)定义keyCode

A: native 定义（keycodes.h）

```c
enum {
   ………
   AKEYCODE_HELP   = 259,
   AKEYCODE_CAMERA = 260
};
```

> 注：
>
> 1）位置：frameworks/base/include/android/keycodes.h
>
> 2）此处keycode的定义的值即是 上面key label定义在KEYCODES数组中的位置（index），否则会映射错误

B JAVA定义（KeyEvent.java定义键值）

```java
public static final int KEYCODE_HELP        = 259;
public static final int KEYCODE_LXL         = 666;
```

修改LAST_KEYCODE

```java
private static final int LAST_KEYCODE      =  KEYCODE_LXL;
```

 >注：
 >
 >1) 位置：frameworks/base/core/java/android/view/KeyEvent.java
 >
 >2) 此处的key code必须与native定义的一致

C ：资源文件（attrs.xml）添加keycode

>注：
>
>1) 位置：frameworks\base\core\res\res\values\attrs.xml
>
>影响到API则需要调用  make update-api  然后就可以使用了



列表：

> **①** bionic/libc/kernel/uapi/linux/input-event-codes.h 中添加 ： **#define KEY_LXL        123**   //与kernel中头文件定义一致**
> ② Generic.kl或Vendor_xxxx_Product_xxxx.kl文件中添加   ： **key 123 LXL;
> ③** /frameworks/native/include/android/keycodes.h 中添加 ： **AKEYCODE_LXL      = 666，
> ④** /frameworks/native/include/input/KeycodeLabels.h 的KEYCODES[]数组中添加： **{ "LXL", 666 },**
> **⑤** 在frameworks/base/core/res/res/values/attrs.xml 中添加 ： ****
> **⑥** 在frameworks/base/core/java/android/view/KeyEvent.java添加： **public static final int KEYCODE_LXL= 666;**

​		经过如上的步骤就将Linux驱动向上层抛出的"123"键值和Android系统中的KEYCODE_LXL  <666>对应起来了，然后可以在Android的framework层的键值处理函数中，捕获按键事件，并进行相应自定义处理，具体在**frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java** 的**interceptKeyBeforeQueueing()**函数中实现。

## 五：相关命令

> dumpsys input

这个命令可以查看到输入设备映射到了哪一个kl文件 kl文件映射

>  getevent   getevent -ltr

这个命令可以查看到输入的具体键值时多少,输出的是16进制。

> input keyevent

模拟键值输入，input keyevent android键值 例如：

```
input keyevent 20
```

注意这里的android 键值就是前面KeyEvent.java 里面对应的键值

>  cat /proc/bus/input/devices

查看设备信息，详见上文

## 其它参考文章：

```
https://blog.csdn.net/iteye_12837/article/details/82327773
https://blog.csdn.net/dec_sun/article/details/89282857
https://blog.csdn.net/anfeng3664/article/details/101180412
https://blog.csdn.net/xct841990555/article/details/81067727
```

# [遥控器]Android红外及蓝牙遥控器适配流程

https://blog.csdn.net/shmily_jing/article/details/100881886