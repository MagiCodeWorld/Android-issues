对于linux和Android开发者，有时可能需要查看或者修改.so文件，下面来讲述如何查看或者修改so文件。

## 1、本文使用的工具

IDA Pro： https://www.52pojie.cn/thread-675251-1-1.html

010 Editor：http://www.pc6.com/softview/SoftView_55129.html

## 2、反编译.so文件

​	本文使用的反编译软件是IDA Pro，它是一个优秀的静态反编译软件。由于IDA功能较为复杂，本文只给出查看和修改so直接相关的功能。

> 注意安装完IDA后通常会生成两个快捷图标32位和64位，这个是对应于so的，如果so是32位的，则打开32位的32位的快捷图标

打开之后，加载so，通常可以直接识别到so的信息，如果不能识别或者识别有误，可以手动选择，如图所示：

![20200601145758](D:\myFile\Project\MarkDown\assets\20200601145758.png)

​														图1：加载so文件界面

进入之后，可以看到的界面如图2所示，在软件菜单栏下面，软件用不同的颜色标注除了so的信息，并且在信息条的下方有对应的说明。

![20200601145653](D:\myFile\Project\MarkDown\assets\20200601145653.png)

​																			图2：主界面

## 3、阅读so代码

​	在左侧的函数窗口（function  Window）的函数名称（function name）中找到.text段，然后找到需要查看的函数，双击即可打开。打开后反编译出来的汇编语言（IDA View-A）看起来比较费劲，可以按F5转成伪代码（Pseudocade-A窗口）。我们这里找到需要修改的地方rtsp_open(v3, &src, v65, 0)，这里我们需要的修改rtsp_open的第三个参数。

![QQ截图20200530190319](D:\myFile\Project\MarkDown\assets\QQ截图20200530190319-1590995050608.png)

## 4、定位so代码修改位置

​	打开并找到需要修改的代码之后，然后在对应的IDA View-A视图中找到对应的汇编代码，注意如果找不到或者关闭了IDA View-A视图，可以重新打开软件。找到rtsp_open的第三个参数 MOVS R3, #0

![QQ截图20200530190929](D:\myFile\Project\MarkDown\assets\QQ截图20200530190929.png)

找到对应的代码之后，鼠标右键选择 文本视图，可以看到该条指令的具体位置，记住这个位置 0x0003F8EC

![QQ截图20200530191208](D:\myFile\Project\MarkDown\assets\QQ截图20200530191208.png)

## 5、修改so

使用010 Editor修改.so文件打开so文件，然后找到刚才记录的需要修改的数组的位置，直接修改其数值。

这里我们需要修改的是0x0003F8EC，对应的十六进制是00，直接修改它。

如果需要修改汇编指令，不知道怎么改的话，可以使用汇编转16进制码的工具arm_asm。

![QQ截图20200530191224](D:\myFile\Project\MarkDown\assets\QQ截图20200530191224.png)

