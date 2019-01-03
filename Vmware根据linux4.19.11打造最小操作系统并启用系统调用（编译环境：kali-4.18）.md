# Vmware根据linux4.19.11打造最小操作系统并启用系统调用（编译环境：kali-4.18


这段时间，学校要做操作系统课程设计，任务如下：

1、编写一个新系统调用的响应函数，函数的名称和功能由实验者自行定义。把新的系统调用函数嵌入到Linux内核中

2、编写应用程序以测试新的系统调用并输出测试结果

要实现以上设计任务，就必须要独立编译内核进行测试，然而要编译内核需要一些配置，让我们来看看有哪些配置：


## 第一步，下载linux内核源码并解压：

```
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.19.11.tar.xz
xz -d linux-4.19.11.tar.xz
tar -xvf  linux-4.19.11.tar
cd linux-4.19.11
```


## 第二步，安装依赖并运行图形化配置菜单

```
sudo apt install libncurses5-dev libssl-dev 
sudo apt install build-essential openssl 
sudo apt install zlibc minizip 
sudo apt install libidn11-dev libidn11
sudo apt install bison flex
make menuconfig
```
中途需要编译一段时间，如果出现问题，可能是因为缺少相关依赖，使用apt安装即可。
## 第三步，让我们看看图形化配置
![配置界面](https://img-blog.csdnimg.cn/20181231110638491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZqaDE5OTc=,size_16,color_FFFFFF,t_70)

&#8195;  注意，第一次进入的界面的配置是根据当前kali系统的配置文件.config自动生成的。这个配置介于命令make allyesconfig和make allnoconfig之间，如果按照这个配置编译，系统内核大约有300M~400M之间，且编译的时间大概需要6个小时，不利于我们的编译和调试，如果按照make allnoconfig配置，1~2分钟之间可以编译好，系统内核也只有500k,但很多功能用不了，显然是不切实际的，所以我们要根据自己的需求，不断去除和添加自己想要的功能。最简便的方法是根据当前配置不断地删减功能，直到不能再减小为止（这样的目的是保留系统的verbose报错功能，输出启动日志以便后续调试，如果一开始就make allnoconfig 虽然系统最小，但很多配置不全，不如使用原系统默认配置，然后一项项删减。）。比如启动日志遇到/bin/sh exists but can't execute,意味着可执行文件执行遇到问题，就需要把excutable file format里面的kernel support for elf binaries启用以及64bit kernel启动，遇到permission denied 就说明可能把自制内核按照在发行版内核同一个分区里导致冲突，因此需要把自制内核安装在另一个分区里。如果遇到can't udev的问题，说明守护程序udev没有打开或者硬盘驱动没有安装，就需要我们安装相应的硬盘驱动，步骤如下:
键入modinfo mptbase，提示如下信息：

```
root@kali:/home/kali/linux-4.19.11# modinfo mptbase
filename:       /lib/modules/4.18.0-kali2-amd64/kernel/drivers/message/fusion/mptbase.ko
version:        3.04.20
license:        GPL
description:    Fusion MPT base driver
author:         LSI Corporation
srcversion:     AE14687F5D92F8703B5805F
depends:        
retpoline:      Y
intree:         Y
name:           mptbase
vermagic:       4.18.0-kali2-amd64 SMP mod_unload modversions 
parm:           mpt_msi_enable_spi: Enable MSI Support for SPI controllers (default=0) (int)
parm:           mpt_msi_enable_fc: Enable MSI Support for FC controllers (default=0) (int)
parm:           mpt_msi_enable_sas: Enable MSI Support for SAS controllers (default=0) (int)
parm:           mpt_channel_mapping: Mapping id's to channels (default=0) (int)
parm:           mpt_debug_level: debug level - refer to mptdebug.h - (default=0)
parm:           mpt_fwfault_debug:Enable detection of Firmware fault and halt Firmware on fault - (default=0) (int)
```
可见硬盘驱动为 Fusion MPT base driver

配置PCI总线驱动,依次选"Bus Option"-"Pci support":


前面看到我的磁盘是Scsi类型，因此需要配置Scsi，依次选"Device Drivers"-"Scsi device support"--"Scsi device support","Scsi disk support"


Scsi磁盘需要Sata控制器，这一步比较关键，要从不同厂商的控制器驱动中挑选出合适的驱动。根据前面udevadm的输出我的虚拟机，使用的是Fusion LSI磁盘驱动：依次选取
1)"Device Driver"-"Serial ATA and Parallel ATA driver"-"AHCI SATA support";
2)"Device Driver"-"Fusion MPT device support"-"Fusion MPT ScsiHost driver for SPI";

3)"Device Driver"-"Scsi device support"-"SCSI low-level drivers"-"LSI MPT Fusion SAS 2.0 Device Driver"


设备驱动配置完毕，需要再配置内核支持的文件系统和ELF文件格式。为了方便起见，我们不设置内核支持initramfs，直接用busybox启动。

选取"File Systems"-"The Extend 4(ext4) filesystem"


选取"Executable file formats"-"kernel support for ELF binaries"

"Kernel support for scripts starting with #!"

其他设置：
可见硬盘驱动为 Fusion MPT base driver

配置PCI总线驱动,依次选"Bus Option"-"Pci support":


前面看到我的磁盘是Scsi类型，因此需要配置Scsi，依次选"Device Drivers"-"Scsi device support"--"Scsi device support","Scsi disk support"


Scsi磁盘需要Sata控制器，这一步比较关键，要从不同厂商的控制器驱动中挑选出合适的驱动。根据前面udevadm的输出我的虚拟机，使用的是Fusion LSI磁盘驱动：依次选取
1)"Device Driver"-"Serial ATA and Parallel ATA driver"-"AHCI SATA support";
2)"Device Driver"-"Fusion MPT device support"-"Fusion MPT ScsiHost driver for SPI";

3)"Device Driver"-"Scsi device support"-"SCSI low-level drivers"-"LSI MPT Fusion SAS 2.0 Device Driver"


设备驱动配置完毕，需要再配置内核支持的文件系统和ELF文件格式。为了方便起见，我们不设置内核支持initramfs，直接用busybox启动。

选取"File Systems"-"The Extend 4(ext4) filesystem"


选取"Executable file formats"-"kernel support for ELF binaries"

"Kernel support for scripts starting with #!"

其他设置：
```
[*]64 bit kernel

Processor type and features --->

Processor family () --->

[*] Generuc x86 support

Bus options (PCI etc.) --->

[*] PCI support

PCI access mode (Any) --->

Executable file formats / Emulations --->

[*] Kernel support for ELF binaries

[*] Write ELF core dumps with partial segments

[*] Enable the block layer #这步很重要，不然无法启用SCSI

Device Drivers  --->

[*] Block devices --->

<*> Loopback device support

SCSI device support --->

<*> SCSI device support

[*] legacy /proc/scsi/ support (NEW)

<*> SCSI disk support

input device support--->

[*]Generic input interface

[*]event interface

[*]event debugging

***** Input Device Drivers****

[*]keyboards----->

[*]AT keyboard

[*]……keyboard   #键盘驱动可以都选上

Hardware I/O Ports--->   

[*]Serial I/O support

……             #这里可以全选

[*]character devices --->

[*]enable tty

[*] Fusion MPT device support --->

<*> Fusion MPT ScsiHost drivers for SPI

<*> Fusion MPT ScsiHost drivers for FC

<*> Fusion MPT ScsiHost drivers for SAS

<*> Fusion MPT misc device (ioctl) driver

File systems  --->

[*]   The extended 4(ext4) filesystem

[*]   Use ext4 for ext2 filesystem

[*]     Ext4 POSIX Access Control Lists 

[*]     Ext4 Security Labels

[*]     Ext4 Encryption 

 

Processor type and features --->

Processor family () --->

[*] Generuc x86 support

Bus options (PCI etc.) --->

[*] PCI support

PCI access mode (Any) --->

Executable file formats / Emulations --->

[*] Kernel support for ELF binaries

[*] Write ELF core dumps with partial segments

[*] Enable the block layer #这步很重要，不然无法启用SCSI

Device Drivers  --->

[*] Block devices --->

<*> Loopback device support

SCSI device support --->

<*> SCSI device support

[*] legacy /proc/scsi/ support (NEW)

<*> SCSI disk support

input device support--->

[*]Generic input interface

[*]event interface

[*]event debugging

*****Input Device Drivers****

[*]keyboards----->

[*]AT keyboard

[*]……keyboard   #键盘驱动可以都选上

Hardware I/O Ports--->   

[*]Serial I/O support

……             #这里可以全选

[*]character devices --->

[*]enable tty

[*] Fusion MPT device support --->

<*> Fusion MPT ScsiHost drivers for SPI

<*> Fusion MPT ScsiHost drivers for FC

<*> Fusion MPT ScsiHost drivers for SAS

<*> Fusion MPT misc device (ioctl) driver

File systems  --->

[*]   The extended 4(ext4) filesystem

[*]   Use ext4 for ext2 filesystem

[*]     Ext4 POSIX Access Control Lists 

[*]     Ext4 Security Labels

[*]     Ext4 Encryption 

 ```

## 第四步、在原系统编译内核

时间不长，大概五六分钟（前提是你精简过），系统编译完大概有2M大小

```
make clean
make bzImage -j4
```


## 第六步、添加硬盘并分区

在vmware里面添加一块大概500MB的SCSI硬盘，重启后执行

	fdisk -l
可以看到/dev/sdb装置
进一步分区

	fdisk /dev/sdb
输入n后分区形成/dev/sdb1
格式化

	mkfs -t ext4 /dev/sdb1
挂载分区	

	mount /dev/sdb1 /mnt
编译busybox
```
wget https://busybox.net/downloads/busybox-1.29.3.tar.bz2
tar xvf busybox-1.29.3.tar.bz2
cd busybox-1.29.3
make menuconfig
```

下面是需要编译进busybox的功能选项。

General Configuration应该选的选项

Don't use /usr

&#8195;  这个选项一定要选,否则make install 后busybox将安装在原系统的/usr下，这将覆盖掉系统原有的命令。

Build Options

Build BusyBox as a static binary (no shared libs)

&#8195;  这个选项也是一定要选择的,这样才能把busybox编译成静态链接的可执行文件，运行时才独立于其他函数库.否则必需要其他库文件才能运行，在单一个linux内核不能使它正常工作。

其它选项都是一些linux基本命令选项，自己需要哪些命令就编译进去，一般用默认的就可以了，配置好后退出并保存。

编译并安装busybox

```
make
make install 

```
make install后会在busybox目录下生成一个叫_install的目录，里面有busybox和指向它的链接。

```
cp _install/* /mnt/           # 将_install下的文件全复制到sdb1  
rm -f /mnt/linuxrc  
cp -r ./examples/bootfloppy/etc /mnt       #将etc下的配置文件拷到sdb1下  
cd /mnt/  
mkdir proc mnt usr var tmp dev sys         #创建目录  
cp -a /dev/{console,tty,tty2} dev/    #将原系统的设备文件复制到自制系统
mkdir dev/input
cp -r /dev/input/* dev/input/
cp /root/linux-4.19.11/arch/x86/boot/bzImage /mnt/bzImage #复制内核
```


## 第七步、更新grub

打开grub配置文件

	vi /etc/grub.d/40_custom #不同的系统前面的序号可能不一样
在末尾添加如下信息
```
menuentry "My Linux-4.19.11" {
 
insmod ext2 #ext2包括了ext3、ext4
 
set root='(hd1,1)'
 
linux /bzImage ro root=/dev/sdb1
 
}
```
更新grub

	update-grub
重启系统，在grub菜单选择My Linux-4.19.11如果看到类似以下界面说明编译成功
![成功界面](https://img-blog.csdnimg.cn/20181231212319443.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZqaDE5OTc=,size_16,color_FFFFFF,t_70)

## 第八步、添加系统调用
回到原系统，更改linux源码以下文件：

	vi /root/linux-4.19.11/arch/x86/entry/syscalls/syscall_64.tbl
	
添加如下两个系统调用，注意这两种系统调用的形式不太一样，主要是为了传参，后面会讲到。
![系统调用表](https://img-blog.csdnimg.cn/20181231214600413.png)
修改头文件，添加如下两个函数原型

	vi /root/linux-4.19.11/include/linux/syscall.h
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181231214906557.png)
修改sys.c源文件，添加如下函数定义

	vi /root/linux-4.19.11/kernel/sys.c

```
long sys_hello(void)
{
printk("hello world");
return 1;
}
SYSCALL_DEFINE2(add,int,a,int,b)
{
printk("a:%d \n",a);
printk("b:%d \n",b);
printk("a+b:%d \n",a+b);
return a+b;
}
```

注意以上函数定义是不一样的，前者是直接定义，后者是通过宏来定义，之后sys_add会被编译器自动替换为__x64_sys_add目的是为了让系统调用实现传参，相当于一个接口。SYSCALL_DEFINE2意思是有两个参数，int a，int b。
之后重新编译内核并复制到新硬盘。
## 第九步、编写测试程序
目的是为了手动实现系统调用，使用gcc编译好并输出为test1 文件

```
#include<stdio.h>
#include<linux/kernel.h>
#include<sys/syscall.h>
#include<unistd.h>
#include<stdlib.h>
 
int main(int argc,char *argv[])
{
	long int amma;
	if(argc==4)amma=syscall(atoi(argv[1]),atoi(argv[2]),atoi(argv[3]));
	else amma=syscall(atoi(argv[1]));
	printf("System call %d return %ld\n",atoi(argv[1]),amma);
	return 0;
}
```
复制test1到新硬盘

	cp test1 /mnt/bin/test1
目前在新系统里执行test1是不行的，所以需要分析test1的文件类型
	
	string test1
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181231220104790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZqaDE5OTc=,size_16,color_FFFFFF,t_70)

由输出信息可见，执行该文件需要The GNU C Library的部分库，所以要从原系统复制过去
```
mkdir /mnt/lib/x86_64-linux-gnu
mkdir /mnt/lib64
cp -a /lib/x86_64-linux-gnu/{libc.so.6,ld-2.28.so,libc-2.28.so} /mnt/lib/x86_64-linux-gnu/ 
cp /lib64/ld-linux-x86-64.so.2 /lib64/
```

注意除了两个.so结尾的文件，其余复制过去的都是软连接，软连接指向那两个文件，如果软连接发生故障可以用ln -s 修复。

重启进入自制系统，如果测试成功，会有如下结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181231221213599.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZqaDE5OTc=,size_16,color_FFFFFF,t_70)

至此，系统调用完成。

###  配置文件：

https://github.com/fjh1997/vmware-linux-minimize-kernel-student/blob/master/.config

### 参考：

VMware中打造最小Linux系统（一）——构建内核&文件系统 - yejingx的专栏 - CSDN博客

https://blog.csdn.net/yejingx/article/details/6525405

制作最小linux内核(1) - lixiangminghate的专栏 - CSDN博客

https://blog.csdn.net/lixiangminghate/article/details/55224412

linux添加系统调用总结(内核版本4.4.4) - sinat_28750977的博客 - CSDN博客

https://blog.csdn.net/sinat_28750977/article/details/50837996

Linux系统调用之SYSCALL_DEFINE - 柠檬C的专栏 - CSDN博客

https://blog.csdn.net/hxmhyp/article/details/22699669

ubuntu18.04 编译内核 学习记录 - Hynn01的博客 - CSDN博客

https://blog.csdn.net/weixin_38180645/article/details/82856407

