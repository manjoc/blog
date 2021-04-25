---
title: "Centos6_startup_process"
date: 2021-04-25T10:48:13+08:00
lastmod: 2021-04-25T10:48:13+08:00
draft: false
keywords: []
description: ""
tags: ["linux","sre","ops"]
categories: ["linux"]
author: "王清"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

文中实验基于`CentOS6.4`版本（`CentOS6.x`版本都可以，但是`CentOS7.x`版本会有所区别）

[from 袁平](https://husteryp.github.io/)

## 一. 启动流程

1. 开关电源（`SMPS`）在开机之后将`AC`信号转换为`DC`信号，然后`SMPS`会进行电压的检测，如果正常，`SMPS`将会发送`POWER GOOD` 信号给主板定时器，主板定时器接收到`POWER GOOD`信号之后，将会停止发送`reset`指令给`CPU`，这意味着电脑可以正常启动
2. 在`CPU`的设置中有一些值是固定的，这样`CPU`才知道从哪里开始读取指令，在`X86`计算机中，第一条指令是` FFFF:0000h `，指向`ROM`的最后一个字节，该指令因为只有一个字节，所以只是简单的包含一个跳转指令，该跳转指令指向的是`BIOS`所在的地址（`EPROM`或者`ROM`中）
3. `BIOS`就是基本输入输出系统，其功能主要有两个：

> 1. POST：`Power On Self Test`；硬件自检；即在`BIOS`中有一个硬件设备的列表，为了检测某一个硬件设备是否可用，就发送一个电脉冲给每一个设备，如果该脉冲被设备返回，说明设备可用，否则，不可用；如果有新的硬件接入，那么会使用同样的检测过程，同时会更新`BIOS`中的列表，用于下次启动过程
> 2. 选择第一个启动设备：` BIOS`会根据`CMOS`里面的记录的启动顺序以及上一步硬件自检过程中，设备的可用装态，确定最终的可启动设备顺序，具体的过程是`BIOS`将磁盘的第一扇区（磁盘最开始的`512`字节）载入内存，放在 `0X0000:0X7C00`处，然后检查这个扇区的最后两个字节是不是`55AA`，如果是则认为这是一个有效的启动扇区，如果不是就会尝试下一个启动介 质，如果找到可以启动的程序就会从这一介质启动，如果所有的启动介质都判断过后仍然没有找到可启动的程序那么`BIOS`会给出错误提示

**注**： 我们平时所说的更改`BIOS`设置，实际上是错的，我们更改的只是`CMOS`中存储的设置，`BIOS`中信息不能被用户改变，需要厂商提供闪存程序

1. **MBR**：`BIOS`确定好第一个启动设备之后，去该设备磁盘的第一个扇区读取最开始的`512`字节到内存（通过硬件的`INT 13`中断功能来读取的），该`512`字节就是`MBR`，即主引导记录；这`512`字节中有`446`字节才是正真的主引导加载代码（`Boot Loader`，即可以安装引导加载程序的地方），还有`64`字节用于存放磁盘分区表信息，以及最后两个字节用于校验（`55AA`)是否为有效的启动扇区；每个分区需要`16`个字节用于存储分区的开始，结束位置，分区类型等（由于每个分区信息需要`16`个字节，所以对于采用`MBR`型分区结构的硬盘(其磁盘卷标类型为`MS-DOS`)，最多只能识别`4`个主要分区。所以对于一个采用此种分区结构的硬盘来说，想要得到`4`个以上的主要分区是不可能的。这里就需要引出了扩展分区。扩展分区也是主分区（`Primary partition`）的一种，但它与主分区的不同在于理论上可以划分为无数个逻辑分区，每一个逻辑分区都有一个和`MBR`结构类似的扩展引导记录(`EBR`) ）（` MBR`分区方案无法支持超过`2TB`容量的磁盘。因为这一方案用`4`个字节存储分区的总扇区数，最大能表示`2`的`32`次方的扇区个数，按每扇区`512`字节计算，每个分区最大不能超过`2TB`，如果磁盘容量超过`2TB`，那分区的起始位置 就无法表示了）
2. 继续之前，我们先看一下`MBR`的信息，这里以`CentOS6.4`为例（后文不再声明），使用命令`dd if=/dev/sda of=mbr bs=512 count=1`，将第一个扇区的`512`字节内容转存到`mbr`文件中，其中`bs`表示大小；然后可以通过`file mbr`来查看分区信息等，输出如下；其中`partition x`表示第`x`个分区的信息，上面说了，使用`MBR`时最多只能有四个分区，而且这四个分区中最多只能有一个活动分区，也就是下面被标记为`active`的`partition 1`

```
 x86 boot sector; partition 1: ID=0x83, active, starthead 32, startsector 2048, 19451904 sectors; partition 2: ID=0x5, starthead 254, startsector 19455998, 2093058 sectors, code offset 0x63
```

1. **Grub–Stage1**：`MBR`中的前`446`字节存储着引导加载程序，即`Boot Loader`，这里以`Grub`为例；`Grub`有两个阶段（实际上应该是三个阶段，后面会解释），实际上这`446`字节中存储的只是`Grub`的第一阶段，因为`446`字节的空间实在有限，做不了太多事情；`Grub`第一阶段的作用主要是加载第二阶段

**注**：`GRUB`实际上分为三个阶段：`stage1 -- stage1.5 -- stage2`）`stage1`是存储在`MBR`前`446`字节中的引导加载程序；`stage1.5`位于`MBR GAP`（`MBR GAP`：扇区编号从0开始，但是真正的磁盘分区一般是从第63个扇区开始的，也就是说从1-63扇区是空闲的（第0个扇区存储`MBR`），这段空间叫做`MBR GAP`），因为`stage2`的配置文件（`/boot/grub/grub.conf`）存储于磁盘上，但是读取文件系统需要驱动，`MBR GAP`中主要存储的就是读取文件系统的`driver`，`stage1`读取该部分内容之后，就可以去读取`grub`配置文件，即`stage2`的内容

1. **Grub–Stage2**：`stage1`加载`stage2`的配置文件，主要是`/boot/grub/grub.conf`，这里我们先看一下`grub.conf`的内容，如下

> 1. `default=0`：这个必须和下面的`title`对应，在配置文件中有几个`title`，启动的时候就会有几个菜单可以选择；`default=0`表示使用第一个`title`选项来启动，`default`表示在读秒时间结束前都没有按键，则使用默认选项启动
> 2. `timeout=5`：读秒时间，如果`5`秒钟内没有按键，则使用默认选项启动；当然可以自行调整时间
> 3. `splashimage=(hd0,0)/grub/splash.xpm.gz`：`CentOS`启动的时候，后台不是黑白而是彩色变化的，就是这个文件提供的后台图示
> 4. `hidemenu`：是否显示提示菜单，默认是`hide`的，但是笔者这里将这个选项注释掉了，在选择开机启动项的时候还会显示提示菜单
> 5. `title`：这部分指定的是菜单选项，开机时有多少个`title`就会有多少个菜单选项；每一个`title`下的内容，一个是指定`kernel`文件及设置，另一个是指定`initrd`位置

```shell
cat /boot/grub/grub.conf
# grub.conf generated by anaconda
#
# Note that you do not have to rerun grub after making changes to this file
# NOTICE:  You do not have a /boot partition.  This means that
#          all kernel and initrd paths are relative to /, eg.
#          root (hd0,0)
#          kernel /boot/vmlinuz-version ro root=/dev/vda1 console=tty0 console=ttyS0,115200n8 net.ifnames=0
#          initrd /boot/initrd-[generic-]version.img
#boot=/dev/vda
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu
title CentOS (2.6.32-754.15.3.el6.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-2.6.32-754.15.3.el6.x86_64 ro root=UUID=3d083579-f5d9-4df5-9347-8d27925805d4 rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto  KEYBOARDTY
PE=pc KEYTABLE=us rd_NO_DM rhgb quiet console=tty0 console=ttyS0,115200n8 net.ifnames=0
        initrd /boot/initramfs-2.6.32-754.15.3.el6.x86_64.img
```

1. **Kernel**加载：`Boot Loade`根据用户的选项，去加载相应的磁盘映像到内存，内核是一个压缩文件，解压缩后，内核会再一次进行测试与驱动周边设备；之后，内核需要去加载各个模块，`Linux`内核可以动态加载内核模块，这些内核模块放置在`/lib/modules/`目录下，由于模块放置到磁盘根目录下，所以内核要去读取这些模块的话，需要挂载到根目录，这样才能读取内核模块加载驱动程序的功能，但是内核根本不认识`SATA`磁盘，所以需要加载`SATA`磁盘的驱动程序，否则根本无法挂载根目录，但是`SATA`的驱动程序是在`/lib/modules/`内，这就出现了两难的问题，所以为了去挂载根目录，就出现了`initrd`这个东西
2. **initrd**：虚拟文件系统，一般位于`/boot/initrd`，`Boot Loader`在将内核加载到内存的同时，也会将`initrd`加载到内存中，并仿真成为一个根目录；`initrd`的作用就是去加载必要的驱动，以便内核可以访问真正的根文件系统（如：磁盘控制器驱动，文件系统驱动（`ext3`，`ext4`等）），然后去挂载真正的根文件系统，进行根切换操作，用真正的根文件系统进行启动；`initrd`完成上述任务之后，会清除掉自己在内存中的痕迹，让出空间
3. 内核及其必要模块加载完毕之后，会主动执行第一个程序，即`/sbin/init`，这也是第一个启动的进程；`/sbin/init`的主要功能是根据其配置文件（`/etc/inittab`）去准备软件的执行环境（系统服务等）；这里我们看一下其配置文件`/etc/inittab`的内容；如下；前面的注释是对该配置文件的描述，在笔者的该系统中，`inittab`中只有一项配置，即运行级别（`runlevel`）的设置，所谓的`run level`就是`Linux`会根据`run level`的设置来启动不同的服务，让`Linux`的运行环境不同；至于`run level`，分为七个等级：如下；

> 1. `0 -- halt`：系统直接关机
> 2. `1 -- single user mode`：单用户维护模式，用在系统出问题时的维护
> 3. `2 -- Multi-user， without NFS`：类似下面的`level 3`，只是没有`NFS`服务
> 4. `3 -- Full multi-user mode`：完整含有网络功能的纯文本模式
> 5. `4 -- unused`：系统保留功能
> 6. `5 -- X11`：与`level 3`类似，但是加载使用`X Window`
> 7. `6 -- reboot`：重新启动

这里笔者将运行级别设置为`3`，所以是以纯文本形式启动，即纯命令行模式

```shell
cat /etc/inittab
# inittab is only used by upstart for the default runlevel.
#
# ADDING OTHER CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.
#
# System initialization is started by /etc/init/rcS.conf
#
# Individual runlevels are started by /etc/init/rc.conf
#
# Ctrl-Alt-Delete is handled by /etc/init/control-alt-delete.conf
#
# Terminal gettys are handled by /etc/init/tty.conf and /etc/init/serial.conf,
# with configuration in /etc/sysconfig/init.
#
# For information on how to write upstart event handlers, or how
# upstart works, see init(5), init(8), and initctl(8).
#
# Default runlevel. The runlevels used are:
#   0 - halt (Do NOT set initdefault to this)
#   1 - Single user mode
#   2 - Multiuser, without NFS (The same as 3, if you do not have networking)
#   3 - Full multiuser mode
#   4 - unused
#   5 - X11
#   6 - reboot (Do NOT set initdefault to this)
#
id:3:initdefault:
```

1. 到这里系统就启动起来啦 ~ ，这也是系统启动的大致流程 ~