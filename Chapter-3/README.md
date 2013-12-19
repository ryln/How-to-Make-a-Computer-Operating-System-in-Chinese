## 第3章: 使用GRUB进行初次启动

#### 启动要干些什么?

从一台x86计算机开机开始, 它就走在了一条复杂的路上, 直到控制权交到内核"主"程序("kmain()")手上. 在这一章, 我们只考虑BIOS启动, 而不是新出的UEFI.

BIOS的启动顺序是: 识别RAM -> 识别硬件/初始化 -> 引导序列

对我们而言, 最重要的一步是"引导序列". 这时BIOS刚完成初始化, 尝试把控制权移交给下一步的bootloader.

在"引导序列"中, BIOS首先选择"启动设备"(软盘, 硬盘, CD, usb闪存设备或是网络). 我们的第一次启动会使用硬盘(但它也可以从CD或usb闪存设备启动).

BIOS会寻找第一个有效的引导扇区, 从中读取512字节(最后两个字节会是0x55 0xAA). 如果没找到, 则查找错误信息. 然后是把这512字节, 从0x7c00地址开始写入到物理内存. 代码就从0x7c00这开始运行.

BIOS把控制权移交引导扇区后, CPU在16位实模式(Real Mode)下, 装载该区代码, 从物理地址0x7c00开始运行. 但我们的内核只能是是32位的, 因此需要bootloader来读取内核, 切换到保护模式(Protexted Mode)再运行.

#### 什么是GRUB?

> GNU GRUB(GNU GRand Unified Bootloader的缩写), 是GNU计划的一个bootloader包. GRUB是自由软件基金会关于多重引导规范的参考实现. 它让用户可以在一台计算机安装的多个操作系统中选择一个, 或是指定一个操作系统下可用的内核配置.

简而言之, GRUB是机器(bootloader)第一个启动的, 它简化载入硬盘中内核的过程.

#### 为什么使用GRUB?

* GRUB非常易用
* 不需要16位代码就能轻易地载入32位内核
* 提供Linux, Windows及其他系统的多重引导
* 容易载入内存中的额外模块

#### 怎么使用GRUB?

GRUB遵循多重引导规范, 其可执行的二进制(程序)是32位的, 而且必须在前8192字节内包含一个特定header(多重引导的header). 我们的内核会使用ELF可执行文件("Executable and Linkable Format", "可执行可链接格式", 是大部分UNIX系统中的可执行, 通用标准文件格式).

这个内核的首次启动顺序是用汇编来写的: [start.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm). 还会用一个链接器文件来定义可执行结构: [linker.ld](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld).

这个启动过程同时会初始化一些C++的运行环境. 下一章会详细介绍.

多重引导的header结构:

```
struct multiboot_info {
	u32 flags;
	u32 low_mem;
	u32 high_mem;
	u32 boot_device;
	u32 cmdline;
	u32 mods_count;
	u32 mods_addr;
	struct {
		u32 num;
		u32 size;
		u32 addr;
		u32 shndx;
	} elf_sec;
	unsigned long mmap_length;
	unsigned long mmap_addr;
	unsigned long drives_length;
	unsigned long drives_addr;
	unsigned long config_table;
	unsigned long boot_loader_name;
	unsigned long apm_table;
	unsigned long vbe_control_info;
	unsigned long vbe_mode_info;
	unsigned long vbe_mode;
	unsigned long vbe_interface_seg;
	unsigned long vbe_interface_off;
	unsigned long vbe_interface_len;
};
```

可以用```mbchk kernel.elf```命令来检查kernel.elf文件是否符合多重引导标准. 同样可以用```nm -n kernel.elf```来检查在ELF二进制中的对象的偏移.

#### 为内核与grub创建一个磁盘镜像

这个[diskimage.sh](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/sdk/diskimage.sh)脚本可以生成一个QEMU可用的硬盘镜像.

第一步, 使用qemu-img创建一个硬盘镜像(c.img):

```
qemu-img create c.img 2M
```

现在用fdisk进行磁盘分区

```
fdisk ./c.img

# Switch to Expert commands
> x

# Change number of cylinders (1-1048576)
> c
> 4

# Change number of heads (1-256, default 16):
> h
> 16

# Change number of sectors/track (1-63, default 63)
> s
> 63

# Return to main menu
> r

# Add a new partition
> n

# Choose primary partition
> p

# Choose partition number
> 1

# Choose first cylinder (1-4, default 1)
> 1

# Choose last cylinder, +cylinders or +size{K,M,G} (1-4, default 4)
> 4

# Toggle bootable flag
> a

# Choose first partition for bootable flag
> 1

# Write table to disk and exit
> w
```

现在需要使用losetup, 把刚创建的分区绑定到回环设备(loop-device, 可以让文件通过块设备的方式访问). 通过**偏移量 = 开始扇区 * 每扇区字节数** 计算分区偏移量, 作为参数传递.

```fdisk -l -u c.img``` 得到 63 * 512 = 32256.

(译注: 以下命令请使用sudo执行)

```
losetup -o 32256 /dev/loop1 ./c.img
```

在新设备上创建EXT2文件系统

```
mke2fs /dev/loop1
```

复制文件到已挂载磁盘上

```
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```

在磁盘上安装GRUB

```
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

最后脱离回环设备

```
losetup -d /dev/loop1
```

####参见

* [GNU GRUB on Wikipedia](http://en.wikipedia.org/wiki/GNU_GRUB)
* [Multiboot specification](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)

<table><tr><td><a href="../Chapter-2/README.md" >&larr; 上一篇</a></td><td><a href="../Chapter-4/README.md" >下一篇 &rarr;</a></td></tr></table>