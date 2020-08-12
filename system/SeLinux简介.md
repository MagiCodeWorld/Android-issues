# Android SeLinux简介

## 1 文件位置

android 系统原生的sepolicy定义在android/system/sepolicy，而厂商自定义的通常位于android/device/manufacturer 或者 android/vendor/manufacturer下。其中android原生的sepolicy目录如下图所示

![selinux1](D:\myFile\Project\MarkDown\assets\selinux1-1592966569538.png)



## 2 配置文件与语法简介

### 2.1 基本配置文件定义

- attributes   属性定义
- access_vectors 对应了每一个class可以被允许执行的命令 
- roles Android中只定义了一个role，名字就是r，将r和attribute domain关联起来 
- users  其实是将user与roles进行了关联，设置了user的安全级别，s0为最低级是默认的级别，mls_systemHigh是最高的级别
- security_classes 指的是上文命令中的class，个人认为这个class的内容是指在android运行过程中，程序或者系统可能用到的操作的模块 
- te_macros 系统定义的宏全在te_macros文件 
- *.te 一些配置的文件，包含了各种运行的规则

### 2.2 selinux有两种工作模式

“permissive”：所有操作都被允许（即没有MAC），但是如果有违反权限的话，会记录日志
“enforcing”：所有操作都会进行权限检查

## 3  *.te TE：Type Enforcement）策略， 基本定义

3.1 定义类型
		在TE中，所有的东西都被抽象成类型。进程，抽象成类型；资源，抽象成类型。属性，是类型的集合。所以，TE规则中的最小单位就是类型。

```
定义type  type 类型名称 [alias 别名集] [,属性集];
type XXXXX;
tyep  alias{ aa , bb , cc }     XX ,  YY ;      属性用逗号分开
```

(2) 使用typealias申明别名

```
# 这两条语句等同于   
type hello_t, domain;   
typealias hello_t alias kitty_t;   
#下面这一条语句   
type hello_t alias kitty_t, domain; 
```

(3) 声明属性

```
## sepolicy/public/attributes
 ######################################
  2 # Attribute declarations
  3 #
  4 
  5 # All types used for devices.
  6 # On change, update CHECK_FC_ASSERT_ATTRS
  7 # in tools/checkfc.c
  8 attribute dev_type;
  9 
 10 # All types used for processes.
 11 attribute domain;
 12 
 13 # All types used for filesystems.
 14 # On change, update CHECK_FC_ASSERT_ATTRS
 15 # definition in tools/checkfc.c.
 16 attribute fs_type;
 17 
 18 # All types used for context= mounts.
 19 attribute contextmount_type;
 20 
 21 # All types used for files that can exist on a labeled fs.
 22 # Do not use for pseudo file types.
 23 # On change, update CHECK_FC_ASSERT_ATTRS
 24 # definition in tools/checkfc.c.
 25 attribute file_type;
 26 
 27 # All types used for domain entry points.
 28 attribute exec_type;
 29 
 30 # All types used for /data files.
 31 attribute data_file_type;
 32 expandattribute data_file_type false;
 33 # All types in /data, not in /data/vendor
 34 attribute core_data_file_type;
 35 expandattribute core_data_file_type false;
```

(4) 类型 与 属性 关联

```
# sepolicy/public/init.te
# init is its own domain.
type init, domain, mlstrustedsubject;
type init_exec, system_file_type, exec_type, file_type;
type init_tmpfs, file_type;

# /dev/__null__ node created by init.
allow init tmpfs:chr_file { create setattr unlink rw_file_perms };

# init direct restorecon calls.
#
# /dev/kmsg
```

也可以用 typeattribute 进行“属性”关联

```
  type keystore, domain;
  type keystore_exec, exec_type, file_type;
  init_daemon_domain(keystore)
  typeattribute keystore mlstrustedsubject;
```

```
  type httpd_user_content_t; 
  typeattribute httpd_user_content_t file_type, httpdcontent;
```

## 4  类型 与 属性 关联

访问向量（AV）规则 AV用来描述主体对客体的访问许可。通常有四类AV规则 ：

- allow：表示允许主体对客体执行许可的操作。

- neverallow：表示不允许主体对客体执行制定的操作。

- auditallow： 表示允许操作并记录访问决策信息。

- dontaudit：表示不记录违反规则的决策信息，切违反规则不影响运行。

### 4.1 allow

```
allow user_t bin_t : file execute;
allow user_t { bin_t sbin_t } : file execute;
allow {user_t domain} {bin_t file_type sbin_t} : file {execute  read };
allow user_t bin_t : { file dir } { read getattr }; 
```

  为通配符时，表示所有操作

```
 allow user_t bin_t : { file dir } *; 
```

求补算操作符（~），即除了列出的许可外，其它的许可都包括 ：
下面这个规则允许任何安全上下文中类型具有user_t的进程对任何安全上下文中具有类型为 bin_t 的普通文件所有write setattr ioctl访问权。

```
allow user_t bin_t : file ~{ write setattr ioctl };
```

### 4.2 neverallow

```
allow domain { exec_type -sbin_t } : file execute;  
#等同于  
allow domain { -sbin_t exec_type } : file execute; 
neverallow * domain : dir ~{ read getattr }; 
```

​		这条规则指出没有哪条allow可以授予任何类型对具有domain属性的类型的目录有任何访问权，除了read和getattr访问权外（即读访问权），这条规则的中通配符意味着所有的类型，在真实的策略中，类似这样的规则很常见，它们用来阻止对/proc/目录适当的访问。
​		我们从前面这个例子中看出，在源类型列表中需要使用通配符，因为我们想要指出任何类型或所有类型，包括那些还没有创建的类型，使用通配符可以预防我们未来犯错。
另一个常见的neverallow规则是：
​		这条neverallow规则增强了domain属性，它指出了进程不能转换到无domain属性的类型，这就使得要为一个类型无doamin属性的进程创建一个有效的策略是不可能的。

```
neverallow domain ~domain : process transition;
```

### 4.3 auditallow

​		SELinux有大量的工具记录日志信息，或审核、访问尝试被策略允许或拒绝的信息。审核消息通常叫做”AVC消息”，它提供了详细了关于访问尝试的信息，包括是允许还是拒绝，源和目标的安全上下文，以及其它一些访问尝试涉及到资源信息。AVC消息与其它内核消息类似，都是存储在/var/log目录下的日志文件中，它是策略开发、系统管理和系统监视不可缺少的工具。在此，我们检查是哪一个访问尝试产生了审核消息。

​		默认情况下，SELinux不会记录任何允许的访问检查，只会记录被拒绝的访问检查。这并没什么奇怪的，在大多数系统上，每秒会允许成千上万的访问，只有很少的一部分会被拒绝，允许的访问通常是在预料之中的，通常不需要审核，被拒绝的访问通常是（但不总是）非预期的访问，对它们进行审核便于管理员发现策略的bug和可能的入侵尝试。策略语言允许我们取消这些默认的预料之中的拒绝审核消息，改为记录允许的访问尝试审核消息。

```
dontaudit httpd_t etc_t : dir search; 
```

​		SELinux提供两个AV规则允许我们控制审核哪一种访问尝试：dontaudit和auditallow。使用这两条规则我们就可以改变默认的审核类型了，最常用的是dontaudit规则，它指出哪一个访问尝试被拒绝时不审核，这样就覆盖了SELinux默认的审核所有拒绝的访问尝试的行为。

记住，审核（audit）规则让我们覆盖了默认的审核设置，allow规则指出了什么访问是允许的，auditallow规则不允许访问，它只审核允许的许可。

注意：在许可模式和强制模式下审核是不一样的。运行在强制模式下时，每次允许或拒绝时都会进行审核，应该在策略中对审核频率进行限制（可以使用auditctl实现）。在许可模式下时，只会记录第一次访问尝试，直到下一次策略载入，或固定为强制模式，在开发时通常使用的就是许可模式，这种模式可以减少日志文件的大小。

### 4.4 dontaudit

## 5 客体类别许可(file和process)

  在2.3中，描述了每种客户类别的许可，为了更好地理解许可是如何控制对系统资源的访问，下面我们进一步讨论一下两个客体类别许可：file和process。

### 5.1 与文件相关的客体类别

​         此类客体类别是那些与文件及其他存储在文件系统中的资源有关的，这是大多数用户最熟悉的客体类别了，它包括了所有的与持续不变的，在磁盘上的文件系统和在内存中的文件系统，如proc和sysfs结合在一起的客体类别。与文件相关的客体类别如下表所示：

| 客体类别   | 描述                         |
| ---------- | ---------------------------- |
| filesystem | 文件系统（如一个真实的分区） |
| file       | 普通文件                     |
| dir        | 目录                         |
| fd         | 文件描述符                   |
| lnk_file   | 符号链接                     |
| chr_file   | 字符文件                     |
| blk_file   | 块文件                       |
| sock_file  | UNIX域套接字                 |
| fifo_file  | 命名管道                     |

### 5.2 文件客体类别许可

​		下表列出了file客体类别许可，大多数对所有与文件有关的客体类别的许可都是公用的，只有execute_no_trans，entrypoint和execmod是特定给file客体类别的（这些许可都加了星号*）。

| file许可         | 描述                                                 | 分类            |
| ---------------- | ---------------------------------------------------- | --------------- |
| ioctl            | ioctl（2）系统调用请求                               |                 |
| read             | 读取文件内容，对应标准Linux下的r访问权               | 标准Linux许可   |
| write            | 写入文件内容，对应标准Linux下的w访问权               | 标准Linux许可   |
| create           | 创建一个新文件                                       |                 |
| getattr          | 获取文件的属性，如访问模式（例如：stat，部分ioctl）  |                 |
| setattr          | 改变文件的属性，如访问模式（例如：chmod，部分ioctl） |                 |
| lock             | 设置和清除文件锁                                     |                 |
| relabelfrom      | 从现有类型改变安全上下文                             | SELinux特定许可 |
| relabelto        | 改变新类型的安全上下文                               | SELinux特定许可 |
| append           | 附加到文件内容（即用o_append标记打开）               | 文件扩展许可    |
| unlink           | 移除硬链接（删除）                                   |                 |
| link             | 创建一个硬链接                                       |                 |
| rename           | 重命名一个硬链接                                     |                 |
| execute          | 执行，与标准Linux下的x访问权一致                     | 标准Linux许可   |
| swapon           | 不赞成使用。它用于将文件当做换页/交换空间            | 文件扩展许可    |
| quotaon          | 允许文件用作一个限额数据库                           | 文件扩展许可    |
| mounton          | 用作挂载点                                           | 文件扩展许可    |
| audit_access     |                                                      |                 |
| open             | 打开文件                                             |                 |
| execmod          | 使被修改过的文件可执行（含有写时复制的意思）         | SELinux特定许可 |
| execute_no_trans | 在访问者域转换的执行文件（即没有域转换）             | SELinux特定许可 |
| entrypoint       | 通过域转换，可以用作新域的入口点的文件               | SELinux特定许可 |

### 5.3 SELinux特定许可

​      对于文件而言有五种SELinux特定许可：relabelfrom，relabelto，execute_no_trans，entrypoint和execmod。

​		relabelfrom和relabelto许可控制域类型将文件从一个类型改为另一个类型的能力，为了使重新标记文件成功，域类型必须要有该文件客体当前类型的relabelfrom许可，并且还要有新类型的relabelto许可，注意这些许可不允许控制确切的许可对，域可以将它有relabelfrom许可的任何类型改为它有relabelto许可的任何类型，在重新标记时可以增加约束。

  		execute_no_trans许可允许域执行一个无域转换的文件，这个许可还不够执行一个文件，还需要execute许可，没有execute_no_trans许可，进程可能只能在一个域内执行，如果我们想要确保一个执行过程总是会引发一个域转换（或失败）时，此时就会想要排除execute_no_trans许可，例如：当登陆进程为一个用户登陆执行一个shell时，我们总是想要shell进程从有特权的登陆域类型转移出来。

​		entrypoint许可控制使用可执行文件允许域转换的能力，execute，execute_no_trans和entrypoint许可允许精确控制什么代码可以执行什么域类型，SELinux控制各个程序域类型的能力是它能够提供强壮灵活的安全的主要原因。

​		execmod许可控制执行在进程内存中已经被修改了的内存映像文件的能力，这在防止共享库被另一个进程修改时非常有用，没有这个许可时，如果一个内存映像文件在内存中已经被修改了，进程就不能再执行这个文件了。



## 6 类型规则

​		类型规则在创建客体或在运行过程中重新标记时指定其默认类型，它仅提供一个新的默认类型标记。在策略语言中定义了两个类型规则：

```
type_transition 
```

在域转换过程中标记行为发生时以及创建客体时，指定其默认的类型。

```
type_change 
```

​		使用SELinux的应用程序执行标记时指定其默认类型。我们叫这些规则为”类型规则”，因为它们与AV规则类似，除了规则的末尾是一个类型名而不是许可集外。

### 6.1 通用类型规则语法

​		与AV规则一样，每条类型规则有不同的用途和语义，但它们的语法都是通用的，每条类型规则都具有下列五项元素：

(1) 规则名称：type_transition或type_change

- 源类型：创建或拥有进程的类型
- 目标类型：包含新的或重新标记的客体的客体类型
- 客体类别：新创建的或重新标记的客体的类别
- 默认类型：新创建的或重新标记的客体的单个默认类型

(2) 类型规则语法： 规则名称 类型集 类型集:类别集 单个默认类型;

- 规则名称：类型规则的名称，有效的规则名称有type_transition，type_change和type_member。

- 类型集：一个或多个类型或属性。在规则中源和目标类型有其独立的类型集，多个类型和属性使用空格进行分隔，并用大括号将它们括起来，如{bin_t sbin_t}，可以在类型名前放一个（-）符合将其排除，如{exec_type –sbin_t}。

- 类别集：一个或多个客体类别，多个客体类别必须使用大括号括起来，并用空格分开，如{file lnk_file}。

- 默认类型：为新创建的或重新标记的客体类别指定的单个默认类型，这里不能使用属性和多个类型。

- 所有类型规则在单个策略，基础载入模块，非基础载入模块和条件语句中都有效。

  ​		类型规则语法大部分都和AV规则类似，但也有一些不同的地方，首先就是类型规则中没有许可，不像AV规则那样，类型规则不指定访问权或审核，因此就需要许可了；

  第二个不同点是客体类别没有关联目标类型，相反，客体类别指的是将要被默认类型标记的客体。

最简单的类型规则包括一个源默认类型，一个目标默认类型和一个客体类别，如下:

```
 type_transition user_t passwd_exec_t : process passwd_t; 
```

​		它指出了当一个类型为user_t的进程执行一个类型为passwd_exec_t的文件时，进程类型将会尝试转换，除非有其它请求，默认是换到passwd_t，当声明的客体类别是进程（process）时，
隐含着目标类型要与file客体类别关联，声明的客体类别（process）与源和默认类型关联，这个隐藏着的关联很容易被忽略，即使你成为一个策略编写专家也容易犯这个错。

### 6.2 类型转换规则type_transition

我们使用type_transition规则指定默认类型，目前有两种格式的type_transiton规则：

1. 支持默认域转换事件
2. 支持客体转换，它允许我们指定默认的客体标记

​         这两种形式的type_transition规则帮助增强了SELinux透明转换到Linux用户的安全性，默认情况下，在SELinux中，新创建的客体继承包括它们的客体的类型（如目录），
​        进程会继承父进程的类型，type_transition规则允许我们覆盖这些默认类型，这非常有用，例如：为了确保密码程序在/tmp/目录下创建一个文件时要给一个不同与普通用户的类型。

​		type_transition规则没有allow访问权，它仅提供一个新的默认类型标记，要成功进行类型转换，也必须要一套相关联的allow规则，以允许进程类型可以创建客体和标记客体。
此外，默认的标记指定在type_transition规则中了，只有创建进程没有明确地覆盖默认标记行为它才有效

#### 6.2.1 默认域转换（进程类型转换process）

让我们一起来详细地看一下这条规则中的域转换格式，执行一个文件时，域转换改变了进程的类型，如下面这条规则：

```
type_transition init_t apache_exec_t : process apache_t; 
```

​		这条规则指出类型为init_t的进程执行一个类型为apache_exec_t的文件时，进程类型将会转换到apache_t。客体类别process只表示这是一个域转换规则的格式。
下图显示了一个域转换，实际上，域转换只是改变了进程现有的类型，而不是新创建了一个进程，这是因为在Linux转换创建一个新的进程首先是要调用fork()系统调用复制一份现有的进程，如果进程类型在fork上被改变了，它就会允许域在新的域中执行任意的代码了，通过execve()系统调用执行一个新的程序时，发生域转换时就更安全些。

 正如前面谈到的，只有当策略允许了有关的访问权时才会发生类型转换，域转换要成功，策略必须允许下面三个访问权：

```
execute：源类型（init_t）对目标类型（apache_exec_t）文件有execute许可
transition：源域（init_t）对默认类型（apache_t）必须要有transition许可
entrypoint： 新的（默认）类型（apache_t）对目标类型（apache_exec_t）文件必须要有entryponit许可
```

同时，上面的域转换规则要想成功，还必须要有下面的allow规则：
这条域转换规则.

    type_transition init_t apache_exec_t : process apache_t;   
    至少需要下面三条allow规则才能成功   
    allow init_t apache_exec_t : file execute;   
    allow init_t apache_t : process transition;   
    allow apache_t apache_exec_t : file entrypoint;  

​		在实际中，除了上面这几个最小allow规则外，我们可能还想增加一些额外的规则，例如：常见的有默认类型向源类型发送exit信号（即sigchld许可），继承文件描述符，使用管道进行通信。
域转换最关键的概念是清楚地定义了入口点，即类型为apache_exec_t的文件对新的默认类型apache_t有entrypoint许可，入口点文件允许我们严格控制哪个程序可以在哪个域中执行（可以认为这就是类型强制的安全特性）， 我们知道只有程序的可执行文件的类型对域有entrypoint许可时，这个程序才可以进入一个给定的域，因此我们可以知道并控制哪个程序有哪个特权了。

#### 6.2.2 默认客体转换(file

​		客体转换规则为新创建的客体指定一个默认的类型，实际上，我们通常是在与文件系统有关的客体（如file，dir，lnk_file等）上使用这种type_transition规则，和域转换一样，这些规则只会引发一个默认客体标记尝试，
也只有策略允许了有关的访问权时，尝试才会成功。
客体转换规则由客体类别进行标记，如：

```
type_transition passwd_t tmp_t : file passwd_tmp_t;
```

​		这条type_transition规则指出当一个类型为passwd_t的进程在一个类型为tmp_t的目录下创建一个普通文件（file客体类别）时，默认情况下，如果策略允许的话，新创建的文件类型应该为passwd_tmp_t，注意客体类别目标类型不是tmp_t而是默认类型passwd_tmp_t，在这个例子中，tmp_t隐含关联了dir客体类别，因为它是唯一能够容纳文件的客体类别，同样，和前面一样，策略必须允许对默认标记的访问，对于前面的例子，对类型为tmp_t的目录的访问权需要包括add_name，write和search，对类型为passwd_tmp_t的文件要有read和write访问权。
​		这个例子很典型，它显示了一个解决在同一个目录下多个应用程序共享和继承的安全问题，如在一个临时目录下，客体转换规则对于那些在运行时创建的客体非常有用。

​		某些情况下不能使用客体转换规则，当进程需要在同一个客体容器中创建有多个不同类型的客体时，一条type_transition规则还不够，例如：假设一个进程在/tmp/目录下创建两个UNIX域套接字，这些套接字将用于和其它域通信，如果我们想给每个sock文件不同的类型，客体转换规则将不能满足了，这时需要两条规则，它们有相同的源类型，目标类型和客体类别，只是默认类型不同，但这样会在编译时产生错误，解决这个问题的办法是在安装时创建sock文件，并明确地标记它们，将sock文件分别放在不同目录类型的目录下，或让进程在创建时明确地请求类型。

### 6.3 类型改变规则：type_change

​		我们使用type_change规则为使用SELinux特性的应用程序执行重新标记指定默认类型，和type_transition规则类似，type_change规则指定默认标记，但不允许访问。
与type_transition规则不同点如下：
type_change规则的影响不会在内核中生效，而是依赖于用户空间应用程序，如login或sshd。
为了在策略基础上重新标记客体，如下面的规则：

```
type_change sysadm_t tty_device_t : chr_file sysadm_tty_device_t;
```

​		这条type_change规则指出以sysadm_t名义重新标记一个类型为tty_device_t的字符文件时，应该使用sysadm_tty_device_t类型。
​		这条规则是最常见的使用type_change规则的示例，它在用户登陆时重新标记终端设备，login程序会通过一个内核接口查询SELinux模块中的策略，传递类型sysadm_t和tty_device_t，
接收sysadm_tty_device_t类型作为重新标记的类型，这个机制允许在一个新的登陆会话过程中，登陆进程以用户的名义标记tty设备，将特殊的类型封装到策略中，而不用硬编码到应用程序中。
我们可能很少使用type_change规则，因为它们通常只由核心操作系统服务使用。

## 6 总结

​		类型是SELinux中访问控制的主要基础。它们起着所有客体（进程，文件，目录，套接字等）访问控制属性的作用，类型使用Types语句声明。

1. 属性是类型组。在大多数策略中，能够使用类型的地方就可以使用属性。在使用属性前，我们必须先声明，在Types声明语句中，我们可以将类型添加到属性中，或使用typeattribute语句也行。

2. 别名是类型的另一个名字，主要用于重新命名类型时保持向后的兼容性，可以在声明类型时就声明一个别名，或单独使用typealias语句声明。

3. 有四个AV规则，它们的语法都一样：allow，neverallow，auditallow和dontaudit。

4. 我们使用allow规则指出访问一个域类型时需要一个什么客体类型，我们根据客体类别和许可指定访问权。

​        默认情况下，访问被允许时不产生审核消息，而是访问被拒绝时产生。我们使用dontaudit规则指出被拒绝的访问不产生审核消息，我们使用auditallow规则指出允许的访问要产生审核消息。

1. AV规则（如allow）是累加的，对于一个给定的源类型，目标类型和客体类别的密钥，在运行时，被允许和被审核的访问权是所有引用了该密钥的规则的并集。
2. 我们使用neverallow规则指出了永远都不会被allow规则允许的固定属性，如果某个allow规则违背了这个原则，checkpolicy编译器在编译时就会产生一个错误。
3. 有两个类型规则，它们的语法是一样的：type_transition和type_change。类型规则没有allow访问权，相反，它们指定了客体创建和重新标记事件想要的默认标记策略。
4. 我们使用type_transition规则在创建新的客体时标记它（客体转换），或在执行一个新的应用程序时改变进程的类型（域转换）。
5. 我们使用type_change规则为重新标记客体指定默认的类型，它们用于SELinux敏感的程序如login和sshd。
6. 策略分析工具apol在理解和分析复杂的SELinux策略时是一个非常有价值工具。