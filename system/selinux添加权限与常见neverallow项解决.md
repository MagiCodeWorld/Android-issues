

## 1. dac_override权限问题一种解法

```shell
egbin: type=1400 audit(0.0:879): avc: denied { dac_override } for capability=1 s:egbin:s0 tclass=capability permissive=1
```

​		需要给egbin dac_override权限，但是该权限是Android P的neverallow规则中的，不能被添加。dac_override权限意思是容许进程旁路的所有DAC权限：uid，gid，ACL 等等，即有这个权限可以无视linux的权限中的用户、用户组。谷歌这样做的原因可能是这个dac_override权限能力范围太大，不能给普通程序这个权限，只有少数程序有这个权限。

```shell
ueventd.te(7):allow ueventd self:capability { chown mknod net_admin setgid fsetid sys_rawio dac_override fowner };
zygote.te(7):allow zygote self:capability { dac_override setgid setuid fowner chown };
netd.te(44):allow netd self:capability { dac_override chown fowner };
runas.te(14):dontaudit runas self:capability dac_override;
vold.te(20):allow vold self:capability { net_admin dac_override mknod sys_admin chown fowner fsetid };
installd.te(6):allow installd self:capability { chown dac_override fowner fsetid setgid setuid };
tee.te(9):allow tee self:capability { dac_override };
```

dac_override权限问题一种解法
出现这种问题可能原因是进程的组与需要访问的文件的组不同，进程没有权限访问改组的文件，需要申请dac_override权限来旁路的所有DAC权限：uid，gid使进程可以访问该文件。以egbin为例，下面为开机时egbin的rc文件，及data的user和group。

```shell
drwxrwx--x  43 system system u:object_r:system_data_file:s0  4096 1970-01-29 01:46 data

service egbin /system/bin/egbin start
    class core
    user root
    group root
    writepid /dev/stune/top-app/tasks
    seclabel u:r:egbin:s0
```

​		可以看到 egbin 执行时候user是root，而data是属于system，因此导致需要申请dac_override才能访问data分区。如果不加dac_override的权限，可以将egbin的组切换为system组即可访问，rc修改如下:

```shell
service egbin /system/bin/egbin start
    class core
    user root
    group root system
    writepid /dev/stune/top-app/tasks
    seclabel u:r:egbin:s0
```

参考文献：https://blog.csdn.net/pen_cil/article/details/89434349

2 

```
allow update_engine self:capability { dac_override fowner sys_admin };
```

​		the problem was that /data/misc/update_engine was mkdir'ed by an update_engine.rc file with system:system as the owner but we're running as root. Fixing this to be owned by root:root let's us remove dac_override, I just verified. We'll remove the mkdir in update_engine and defer the mkdir'ing to https://android-review.googlesource.com/#/c/174930/ where it's root:root already.

```
# rootdir/init.rc
mkdir /data/misc/update_engine 0700 root root
```

init.rc: mkdir /data/misc/update_engine 0700 root root

Ensure that /data/misc/update_engine exists since it will be referenced
by selinux policy.

https://android-review.googlesource.com/c/platform/external/sepolicy/+/174530/5/update_engine.te#11

https://danwalsh.livejournal.com/79643.html

1.3 查看命令

man capabilities

## 2. entrypoint的一种解法

```shell
type=1400 audit(0.0:214): avc: denied { entrypoint } for path="/system/bin/csity_dhcplog.sh" dev="dm-2" ino=471 scontext=u:r:start-ssh:s0 tcontext=u:object_r:csity_dhcplog_system_exec:s0 tclass=file permissive=0
```

- 1、首先，让init域的进程要能够执行shell_exec类型的文件

  ```
  allow init shell_exec:file execute;
  ```

- 2、然后，你需要告诉SELinux，允许init域的进程切换到init_shell域

  ```
   allow init init_shell:process transition;
  ```

- 3、最后，还得告诉SELinux，域切换的入口（entrypoint）是执行shell_exec类型的文件

  ```
   allow init_shell shell_exec:file entrypoint;
  ```

看起来挺麻烦，不过SELinux支持宏定义，我们可以定义一个宏，把上面4个步骤全部包含进来。在SEAndroid中，所有的宏都定义在te_macros文件中，其中和DT相关的宏定义如下：

 \# 定义domain_trans宏，$1,$2,$3代表宏的第一个，第二个…参数

```
define(`domain_trans', 
     allow $1 $2:file { getattr open read execute };
     allow $1 $3:process transition;
     allow $3 $2:file { entrypoint open read execute getattr };
	...
 ')
```

 \# 定义domain_auto_trans宏，这个宏才是我们在te中直接使用的:

```
 define(`domain_auto_trans', `
     \# Allow the necessary permissions.
     domain_trans($1,$2,$3)
     \# Make the transition occur by default.
     type_transition $1 $2:process $3;
 ')
```

  呵呵，是不是简单多了，domain_trnas宏在DT需要的最小权限的基础上还增加了一些权限，在te文件中可以直接用domain_auto_trans宏来显示声明域转换，如下：

```
 domain_auto_trans(init, shell_exec, init_shell)
```

1)  init_daemon_domain(study)

程执行一个type为zygote_exec的文件时，将该子进程的domain设置为zygote，而不是继承父进程的domain。并且给与zygote这个domain，所有定义在tmpfs_domain宏中的权限。



```
# init_daemon_domain(domain)
# Set up a transition from init to thedaemon domain
# upon executing its binary.
define(`init_daemon_domain', `
domain_auto_trans(init, $1_exec, $1)
tmpfs_domain($1)
')
```

init_daemon_domain(study)是一个宏，声明当一个domain为init的进程创建一个子进程执行一个type为study_exec的文件时，将该子进程的domain设置为study，而不是继承父进程的domain。并且给与study这个domain，所有定义在tmpfs_domain宏中的权限。



To exec or transition that is the question：https://danwalsh.livejournal.com/72287.html

## 3 **DAC_READ_SEARCH** 

当我们在执行sh命令（如ls）时，可能会遇到selinux报有关 `DAC_READ_SEARCH` 的权限为问题。

```
07-05 18:38:00.268 I/ls  ( 4276): type=1400 audit(0.0:598): avc: denied { dac_read_search } for capability=2 scontext=u:r:hello_t:s0 tcontext=u:r:hello_t:s0 tclass=capability permissive=1
```

首先我们来了解一下 `capability` 中的`DAC_READ_SEARCH`是什么？

在Linux中，`root`被分为64中不同的capabilities。比如加载内核模块的能力、绑定到1024以下端口的能力，我们这里的`DAC_READ_SEARCH`就是其中的一个。

DAC代表自由访问控制，大多数人将其理解为 standard Linux permissions，每个进程都有所有者/组。 为所有文件系统对象分配了所有者，组和权限标志。 `DAC_READ_SEARCH`允许特权进程忽略DAC的某些部分以进行读取和搜索。

在linux系统上，输入*man capabilities*可以看到有关`DAC_READ_SEARCH`的说明：

> *CAP_DAC_READ_SEARCH*
>
> - Bypass file read permission checks and directory read and execute permission checks;*
>
> There is another CAPABILITY called DAC_OVERRIDE
>
> *CAP_DAC_OVERRIDE*
>
> - *Bypass file read, write, and execute permission checks.*

*man capabilities*

从上面的说明我们可以看到，`DAC_OVERRIDE`比  `DAC_READ_SEARCH`的权限更大，因为它可以忽略DAC规则来编写和执行内容，而不仅仅是读取内容。

据资料显示，`DAC_READ_SEARCH`的出现源于内核的改变，这种改变使内核变得更加安全了。

为了理解这句话，我们结合一个例子来进行讲解：

> 假设有一个叫 hello（hello_t）的小程序，登录到系统后最终会执行。 该程序需要读取/etc/shadow。 在Linux系统上（比如Fedora / RHEL / CentOS以及其他 ），/etc /shadow 具有0000模式。 这意味着，即使系统没有以根用户身份运行（UID = 0），也不允许它们读取/写入/etc/shadow，除非它们具有DAC功能。

随着策略的发展，由于hello需要读取 /etc/shadow，它生成了DAC_OVERIDE AVC，加入我们在编写sepolicy时，将它的sepolicy文件命名为hello_t， 多年以来，一切工作正常，直到内核发生了变化……

如果某个进程尝试读取/etc/shadow，则允许该进程具有`DAC_OVERRIDE`或`DAC_READ_SEARCH`。

 在老版本的kernel里面有类似这样一段代码：

```c
if DAC_OVERRIDE or DAC_READ_SEARCH:
	Read a file with 0000 mode.
```

但是，在新的kernel中，已经改成下面的形式：

```c
if DAC_READ_SEARCH or DAC_OVERRIDE
	Read a file with 0000 mode.
```

假如在老的版本，你已经为 hello_t 添加了 `DAC_OVERRIDE`的权限，但是它从来不检查DAC_READ_SEARCH。

但是在新版本中，它首先检查`DAC_READ_SEARCH`，因此即使允许最终访问，我们也看到正在生成AVC。但是我们可以看到，虽然会生成avc打印，但是事实上，仍然允许访问。

在前面讲到，由于DAC_OVERRIDE权限比较大，在不需要的情况下，如果需要添加DAC_READ_SEARCH的时候，建议可以去掉DAC_OVERRIDE。因为其实hello_t并不需要更改/etc/shadow，而只是读取而已，这样可以更加安全。

最后，消除`DAC_READ_SEARCH`最简单的方法，可以在我们的te文件里加上下面这一句：

```
dontaudit hello_t self:capability { dac_read_search };
```

https://danwalsh.livejournal.com/77140.html

## 4 SYS_PTRACE 

当我们运行ps 、top等命令时，经常会碰到selinux报 `SYS_PTRACE` 的权限问题。

通常，当我们的应用运行 `ps` 或者读取 `/proc`的内容时，会提示 `SYS_PTRACE` 的问题。

https://bugzilla.redhat.com/show_bug.cgi?id=1202043

比如selinux的打印可能像下面这样：

```
type=AVC msg=audit(1426354432.990:29008): avc:  denied  { sys_ptrace } for  pid=14391 comm="ps" capability=19  scontext=unconfined_u:unconfined_r:mozilla_plugin_t:s0-s0:c0.c1023 tcontext=unconfined_u:unconfined_r:mozilla_plugin_t:s0-s0:c0.c1023 tclass=capability permissive=0
```

`sys_ptrace`通常指示一个进程正在尝试查看具有不同UID的另一个进程的内存。

通过`man` 命令 *man capabilities* 查看`sys_ptrace`的说明：

*CAP_SYS_PTRACE*

- Trace arbitrary processes using ptrace(2); 
- apply get_robust_list(2) to arbitrary processes; 
- transfer data to or from the memory  of  arbitrary  processes
  using process_vm_readv(2) and process_vm_writev(2).
- inspect processes using kcmp(2).

这些访问类型可能应该处理为：  dontaudited

运行ps命令是特权进程，可能会导致sys_ptrace发生。

/proc下有一些特殊数据，特权进程可以通过运行ps命令来访问这些数据，运行ps的进程几乎从未真正需要此数据，调试工具使用了该数据，来看看在哪里设置了进程的一些随机内存。

因此最容易的修改方法是处理为：dontaudit 

>  **dontaudit**：表示不记录违反规则的决策信息，且违反规则不影响运行(允许操作且不记录)

```
dontaudit mozilla_plugin_t self:capability { sys_ptrace };
```

https://danwalsh.livejournal.com/71981.html

## 5 execute和execute_no_trans

当我们通过sh脚本或者bin文件运行系统命令（比如ls、ps等 ）时，可能会遇到下面的selinux avc提示：

```
avc: denied { execute } for comm="hello.sh" name="toolbox" dev="dm-1" ino=223 scontext=u:r:hello:s0 tcontext=u:object_r:vendor_toolbox_exec:s0 tclass=file permissive=0

avc: denied { execute_no_trans } for comm="hello.sh" path="/vendor/bin/toolbox" dev="dm-1" ino=223 scontext=u:r:hello:s0 tcontext=u:object_r:vendor_toolbox_exec:s0 tclass=file permissive=0
```

通常我们会按照sepolicy的规则，添加规则下面的规则到te文件：

```
allow hello vendor_toolbox_exec:file { execute execute_no_trans };
```

这时，我们编译工程时，可能会遇到编译失败的问题，提示neverallow：

```
libsepol.report_failure: neverallow on line 952 of system/sepolicy/public/domain.te (or line 12401 of policy.conf) violated by allow hello vendor_toolbox_exec:file { execute execute_no_trans };
```

我遇到的情况有一个`/system/bin/hello`的可执行文件，执行`ntpdate`命令时，出现这个avc。而/system分区下没有`ntpdate`命令，因此会自动去执行`/vendor/bin/ntpdate`命令。

当我们添加sepolicy规则时，之所以出现neverallow，就在于我们是通过/system分区去执行/vendor分区命令。由于Google启动的Treble计划，为实现分区可独立升级，为防止分区升级造成分区调用出现问题。因此，是禁止分区之间调用的。

经过一段时间的尝试之后，通过两种方法解决了这个问题：

- 将`/vendor/bin/ntpdate`移植到`/system/bin/`下，当然因为在`/system/bin/`下的这个bin文件只有我们自己使用，为安全起见，可以给文件名改名处理，比如`/system/bin/ntpdate_my`。

- 通过hidl机制，做一个服务去执行`/vendor/bin/ntpdate`，然后将结果返回给/system分区，不过这种做法比较适合只取一次结果的情况，不适合像top、ping之类的命令。



## 6 udp_socket ioctl

```
ifconfig: type=1400 audit(0.0:485): avc: denied { ioctl } for path="socket:[34115]" dev="sockfs" ino=34115 ioctlcmd=0x8927 scontext=u:r:start-ssh:s0 tcontext=u:r:start-ssh:s0 tclass=udp_socket permissive=1
tcpdump : type=1400 audit(0.0:766): avc: denied { ioctl } for path="socket:[35389]" dev="sockfs" ino=35389 ioctlcmd=0x8994 scontext=u:r:start-ssh:s0 tcontext=u:r:start-ssh:s0 tclass=udp_socket permissive=1
```

在`setenfoce 0`和配置`allow system_app self:udp_socket ioctl;` 后操作依然被denied，这是由于ioctl的控制在底层划分的更细，需要允许对应ioctlcmd操作。

具体方法为：

1、查找`android/system/sepolicy/public/ioctl_defines`中对应的`ioctlcmd`在`ioctl_defines`中的定义，如上文中的`ioctlcmd=0x8927`，对应的是`SIOCGIFHWADDR`，`ioctlcmd=0x8994`对应的是`SIOCBONDINFOQUERY`。

2、在对应的te文件加入如下的配置：

 ```
allowxperm start-ssh self:udp_socket ioctl { SIOCGIFHWADDR SIOCBONDINFOQUERY };
 ```

这样，在`ioctl`操作时，对应的`ioctlcmd`就会被允许了。



## 域转换

 SEAndroid中，init进程的SContext为u:r:init:s0（在init.rc中使用” setcon u:r:init:s0”命令设置），而init创建的子进程显然不会也不可能拥有和init进程一样的SContext（否则根据TE，这些子进程也就在MAC层面上有了和init进程一样的权限）。那么这些子进程是怎么被打上和父进程不一样的SContex呢？

 在SELinux中，上述问题被称为DomainTransition，即某个进程的Domain切换到一个更合适的Domain中去。DomainTransition也是在安全策略文件中配置的，而且有相关的关键字。

 这个关键字就是type_transition，表示类型转换，其完整格式如下：

 type_transition source_type target_type : class default_type

 表示source_type域的进程在对target_type类型的文件进行class定义的操作时，进程会切换到default_type域中，下面我们看个域转换的例子：

 type_transition init shell_exec:process init_shell

 这个例子表示：当init域的进程执行（process）shell_exec类型的可执行文件时，进程会从init域切换到init_shell域。那么，哪个文件是shell_exec类型呢？从file_contexts文件能看到，/system/bin/sh的安全属性是u:object_r:shell_exec:s0，也就是说init域的进程如果运行shell脚本的话，进程所在的域就会切换到init_shell域，这就是DomainTransition（简称DT）。

 请注意，DT也是SELinux安全策略的一部分，type_transition不过只是开了一个头而已，要真正实施成功这个DT，还至少需要下面三条权限设置语句：

 \# 首先，你得让init域的进程要能够执行shell_exec类型的文件

 allow init shell_exec:file execute;

 \# 然后，你需要告诉SELinux，允许init域的进程切换到init_shell域

 allow init init_shell:process transition;

 \# 最后，你还得告诉SELinux，域切换的入口（entrypoint）是执行shell_exec类型的文件

 allow init_shell shell_exec:file entrypoint;

 看起来挺麻烦，不过SELinux支持宏定义，我们可以定义一个宏，把上面4个步骤全部包含进来。在SEAndroid中，所有的宏都定义在te_macros文件中，其中和DT相关的宏定义如下：

 \# 定义domain_trans宏，$1,$2,$3代表宏的第一个，第二个…参数

 define(`domain_trans', `

 allow $1 $2:file { getattr open read execute };

 allow $1 $3:process transition;

 allow $3 $2:file { entrypoint open read execute getattr };

 … …

 ')

 \# 定义domain_auto_trans宏，这个宏才是我们在te中直接使用的

 define(`domain_auto_trans', `

 \# Allow the necessary permissions.

 domain_trans($1,$2,$3)

 \# Make the transition occur by default.

 type_transition $1 $2:process $3;

 ')

  呵呵，是不是简单多了，domain_trnas宏在DT需要的最小权限的基础上还增加了一些权限，在te文件中可以直接用domain_auto_trans宏来显示声明域转换，如下：

 domain_auto_trans(init, shell_exec, init_shell)



下面是domain迁移指示的例子：

domain_auto_trans(fu_t,azureus_exec_t,azureus_t)

意思就是，当在 fu_t domain里，实行了 被标为 azureus_exec_t的文件时，domain 从fu_t迁移到azureus_t。



## 类型转换

 除了DT外，还有针对Type的Transition（简称TT）。举个例子，假设目录A的SContext为u:r:dir_a，那么默认情况下，在该目录下创建的文件的SContext就是u:r:dir_a，如果想让它的SContext发生变化，那么就需要TypeTransition。

 和DT类似，TT的关键字也是type_transition，而且要顺利完成Transition，也需要申请相关权限，废话不多说了，直接看te_macros是怎么定义TT所需的宏：

 \# 定义file_type_trans宏，为Type Transition申请相关权限

 define(`file_type_trans', `

 allow $1 $2:dir ra_dir_perms;

 allow $1 $3:notdevfile_class_set create_file_perms;

 allow $1 $3:dir create_dir_perms;

 ')

 \# 定义file_type_auto_trans(domain, dir_type, file_type)宏

 \# 该宏的意思就是当domain域的进程在dir_type类型的目录创建文件时，该文件的SContext

 \# 应该是file_type类型

 define(`file_type_auto_trans', `

 \# Allow the necessary permissions.

 file_type_trans($1, $2, $3)

 \# Make the transition occur by default.

 type_transition $1 $2:dir $3;

 type_transition $1 $2:notdevfile_class_set $3;

 ')



## 模式切换

 SELinux支持Disabled，Permissive，Enforce三种模式；

 Disabled就不用说了，此时SELinux的权限检查机制处于关闭状态；

 Permissive模式就是SELinux有效，但是即使你违反了它的安全策略，它让你继续运行，但是会把你违反的内容记录下来。在策略开发的时候非常有用，相当于Debug模式；

 Enforce模式就是你违反了安全策略的话，就无法继续操作下去。

 在Eng版本使用setenforce命令，可以在Permissive模式和Enforce模式之间切换。



## 相关命令

SELinux是经过安全强化的Linux操作系统，一些原有的命令都进行了扩展，另外还增加了一些新的命令，下面让我们看看经常用到的几个命令：



**1）ls -Z命令**

查看文件，目录的安全属性：

root@xxx:/ # ls –Z

drwxr-x--x  root   sdcard_r  u:object_r:rootfs:s0 storage

dr-xr-xr-x   root   root       u:object_r:sysfs:s0 sys

drwxr-xr-x  root   root       u:object_r:system_file:s0 system

… …



**2）ps -Z命令**

查看进程的安全属性：

root@xxx:/ # ls -Z

u:r:rild:s0            radio    272  1   /system/bin/rild

u:r:drmserver:s0     drm      273  1   /system/bin/drmserver

u:r:mediaserver:s0   media    274  1   /system/bin/mediaserver

u:r:installd:s0        install    283  1   /system/bin/installd

**
3）chcon命令**

更改文件的安全属性

root@xxx:/ # ls -Z /dev/tfa9890                     crw-rw---- media  media    u:object_r:audio_device:s0 tfa9890root@xxx:/ # chcon u:object_r:device:s0 /dev/tfa9890root@xxx:/ # ls -Z /dev/tfa9890                      crw-rw---- media  media    u:object_r: device:s0      tfa9890

**
**

**4）restorecon命令**

当文件的安全属性在安全策略配置文件里面有定义时，使用restorecon命令，可以恢复原来的安全属性

root@xxx:/ # restorecon /dev/tfa9890                   SELinux: Loaded file_contexts from /file_contextsroot@xxx:/ # ls -Z /dev/tfa9890                      crw-rw---- media  media    u:object_r:audio_device:s0 tfa9890

**
**

**5）id命令**

使用id命令，能确认自己的SecurityContext

root@xxx:/ # iduid=0(root) gid=0(root) groups=1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats) context=u:r:su:s0

**
**

**6）getenforce命令**

得到当前SELinux的模式值

root@xxx:/ # getenforce                         Enforcing

**
**

**7）setenforce命令**

更改当前的SELINUX的模式值，后面可以跟 enforcing,permissive 或者1,0。