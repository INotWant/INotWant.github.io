---
title: "OS U启动版的“HelloWorld”" 
layout: post
date: 2021-03-15 10:24
tag:
- OS U-boot
category: OS
author: inotwant
description: U启动版的"HelloWorld"
---

### 出发点

使用 **U启动**、**大白菜** 多次制作过U盘启动盘，终于有一天，**好奇U启动的引导过程**，所以不如尝试自己动手实现U启动版的“HelloWorld”。

### 目标

制作一个U盘，使其支持以下场景：

1. 把U盘插入电脑；
2. 开机，选择 `USB-HDD` 模式启动；
3. U盘启动，在屏幕打印 **Hello, OS world!**

### 实现方法

注：本节只介绍怎么实现，具体原理请参考下一节。另，每个扇区 512 字节。

#### 1. 一些准备

- U盘、U盘分区工具(如，Win10自带的 **磁盘管理** 、**DiskGenius**)；
- `nasm` `dd` 等命令行工具

#### 2. 具体操作

注：下面操作会把U盘数据清空，注意数据备份

1. 把U盘分区类型转换为 **MBR** 格式

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/1_U盘分区类型转化为MBR格式.png)

2. 转换为 HDD 模式

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/2_转换HDD模式.png)

   转换成功后会新建分区（ 会**重建主引导记录** 文件系统选择**FAT32** 并作为**主分区**）

3. 编写汇编代码实现屏幕打印 **Hello, OS world!**

   ```assembly
   ; 文件名 boot.asm
   [org 0x7c5A]            ; 定义起始地址
       mov ax, cs
       mov ds, ax
       mov es, ax
       call DispStr				; 调用屏幕打印
       jmp $								; 死循环
   DispStr:								; 使用 int 0x10 中断实现屏幕打印
       mov ax, BootMessage
       mov bp, ax
       mov cx, 16
       mov ax, 0x1301
       mov bx, 0x000c
       mov dl, 0
       int 0x10
       ret
   BootMessage:        db "Hello, OS world!"
   times 510-($-$$)    db 0
   dw 0xaa55
   ```

   使用 `nasm boot.asm -o boot.bin` 命令把它编译成机械码

4. 把步骤3生成的机器码拷贝至U盘

   步骤2完成后，已经能从U盘启动。不过从U盘启动后得到的是 **DiskGenius** 界面，而不是想要的打印  **Hello, OS world!**。

   步骤3实现的是屏幕打印的逻辑，所以用步骤3的机器码替换运行 DiskGenius 的机器码便可达成预期目标。

   拷贝之前还需确定：

   - 具体拷贝到U盘的哪个位置——**刚刚新建分区的第一个扇区**
   - 需要把第一个扇区的所有数据都替换？并不是，第一个扇区中还维护着一些扇区属性，这些数据需要保留

   为了便于操作，拷贝流程设计如下：

   - 先把第一个扇区的原始数据读取出来
   - 接着从中提取出扇区属性相关的数据
   - 然后把这些数据与步骤3得到的代码拼接
   - 最后把拼接的数据拷贝到第一个扇区

   以下代码直接实现前三个步骤：

   ```c
   #include <fcntl.h>
   #include <sys/types.h>
   #include <sys/uio.h>
   #include <unistd.h>
   #include <stdio.h>
   
   #define FIRST_END 0x5A
   #define SECOND_END 0x1FE
   #define TOTAL_NUM 512
   
   #define PARTITION_DEVICE_PATH "/dev/disk3s1"	// U盘新建分区对应的设备文件
   #define BOOT_PATH "./boot.bin"								// 步骤3得到的机器码
   #define COMPOSITE_PATH "./composite"					// 合成后的文件
   
   int main(int argc, char const *argv[])
   {
       int fd = open(PARTITION_DEVICE_PATH, O_RDWR);
       if (fd == -1)
       {
           perror("open 1st");
           return -1;
       }
       char bytes[TOTAL_NUM];
       size_t read_num = 0, curr_read_num;
       /* read 1st */
       while ((curr_read_num = read(fd, bytes + read_num, FIRST_END - read_num)) > 0 &&
              (read_num = curr_read_num + read_num) < FIRST_END)
           ;
       if (curr_read_num == -1)
       {
           perror("read 1st");
           close(fd);
           return -2;
       }
       else if (read_num != FIRST_END)
       {
           printf("1st data not enough\n");
           close(fd);
           return -3;
       }
       close(fd);
       /* read 2ed */
       fd = open(BOOT_PATH, O_RDWR);
       if (fd == -1)
       {
           perror("open 2st");
           return -4;
       }
       while ((curr_read_num = read(fd, bytes + read_num, SECOND_END - read_num)) > 0 &&
              (read_num += curr_read_num) < SECOND_END)
           ;
       if (curr_read_num == -1)
       {
           perror("read 2nd");
           close(fd);
           return -5;
       }
       else if (read_num != SECOND_END)
       {
           printf("2nd data not enough\n");
           close(fd);
           return -6;
       }
       close(fd);
       /* end flag */
       bytes[TOTAL_NUM - 2] = 0x55;
       bytes[TOTAL_NUM - 1] = 0xaa;
       /* write */
       fd = open(COMPOSITE_PATH, O_RDWR | O_CREAT);
       if (fd == -1)
       {
           perror("open");
           return -7;
       }
       size_t write_num = 0, curr_write_num;
       while ((curr_write_num = write(fd, bytes + write_num, TOTAL_NUM - write_num)) > 0 &&
              (write_num += curr_write_num) < TOTAL_NUM)
           ;
       if (curr_write_num == -1)
       {
           perror("write");
           close(fd);
           return -8;
       }
       close(fd);
       return 0;
   }
   ```

   通过以下命令把 `composite` 写到第一扇区：

   `dd if=composite of=/dev/disk3s1` （注意使用 `sudo` 提升权限）

#### 效果展示

1. 把U盘插入电脑；

2. 开机，选择 `USB-HDD` 模式启动；

3. 屏幕打印如下：

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/效果.jpeg)

### 探索原理

1. MBR 格式

   下图显示了 MBR 分区类型下U盘 **第一个扇区** 的结构，注意不要与上述的第一个分区的第一个扇区混淆。

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/MBR结构.jpg)

   由图可得，它主要由三部分组成：启动引导程序（注：不一定是 `GRUB`）、分区表以及结束标记。

   - 后面将详细分析启动引导程序部分

   - 分区表记录了U盘（或硬盘）的分区状况。通过图中可看到，总共只有四个条目，**这就是MBR格式下只能有四个主分区的原因**。上述操作中，只给U盘划分了一个分区，所以这里只使用了第一个条目
   - 结束标记永为 `0x55AA`，用作验证

   > MBR的内容是在硬盘分区时由分区软件写入该扇区的，MBR不属于任何一个操作系统，不随操作系统的不同而不同，即使不同，MBR也不会夹带操作系统的性质，具有公共引导的特性。但安装某些多重引导功能的软件或LINUX的LILO时有可能改写它，它先于所有的操作系统被调入内存并发挥作用，然后才将控制权交给活动主分区内的操作系统。

2. 启动引导程序部分

   启动引导程序部分的功能：

   - 扫描分区表查找 **活动分区**（被标记为 **活动** 的分区才能用于启动系统）
   - 寻找活动分区的起始扇区（起始扇区，即上述U盘的唯一分区的第一个扇区）
   - 将活动分区的起始扇区读到内存
   - 执行起始扇区的运行代码（启动操作系统）

   在上一节的步骤2（转换为 HDD 模式）中，**DiskGenius** 修改了 **启动引导程序部分**。可通过反汇编查看其中的具体操作：

   ```shell
   # 提取启动引导程序部分
   sudo dd if=/dev/disk3 of=mbr_boot ibs=446 obs=446 count=1
   # 反汇编
   ndisasm mbr_boot > mbr_boot.disasm
   ```

   反汇编结果在 `mbr_boot.disasm` 文件中，为了读懂它耗费了很多时间。。。

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/反汇编结果.png)

   然而，深究发现 **DiskGenius** “借用” 的 `Window7, 8/8.1 or 10` 的 MBR。[这里](http://thestarman.narod.ru/asm/mbr/W7MBR.htm) 和 [这里](https://github.com/egormkn/mbr-boot-manager/blob/master/windows.asm) 有详细的分析。流程概括为：

   1）把U盘（或硬盘）第一扇区的内容从内存的 `0x7c00` （开始时由 BIOS 把U盘第一扇区加载到内存的 `0x7c00`）移至 `0x600`，为活动分区的起始扇区留空间

   2）按序检查分区表，查看哪些分区是活动分区

   3）使用 **LBA模式** or **CHS模式** 把活动分区的起始扇区加载至内存的 `0x7c00`（内部含失败时多次尝试的逻辑）

   4）读取成功后，执行 激活 **A20** 地址位、**TPC** 测试等动作

   5）最后跳转至 `0x7c00`，开始执行活动分区的启动逻辑（比如，这里在屏幕上打印 **Hello, OS world!**）

3. 活动分区的起始扇区

   下图显示了起始扇区的每个字节的意义。由图可看到，**0x03~0x59** 是扇区属性，**0x00~0x02** 加上 **0x5A~0x1FD** 才是启动逻辑。这边解释了上一节的步骤4是怎样合成的。

   ![](https://raw.githubusercontent.com/INotWant/INotWant.github.io/master/assets/images/2021-03-15/dbr.png)

### 参考

[1] [主引导目录（MBR）结构及作用详解](http://c.biancheng.net/view/1015.html)

[2] [FAT32文件系统结构详解](https://blog.csdn.net/li_wen01/article/details/79929730/)

[3] [U盘操作系统引导和内核加载器设计](http://www.buzhibushi.com/info/+IwNZYidByE=)

[4] [硬盘主引导扇区、分区表和分区引导扇区(MBR、DPT、DBR、BPB)详解](https://sites.google.com/site/fenghuangsite/dian-nao/ruan-jian/ying-pan-zhu-yin-dao-shan-qu-fen-qu-biao-he-fen-qu-yin-dao-shan-qu-mbr-dpt-dbr-bpb-xiang-jie?tmpl=%2Fsystem%2Fapp%2Ftemplates%2Fprint%2F&showPrintDialog=1)

[5] [x86指令格式](https://www.cnblogs.com/QKSword/p/8735119.html)

[6] [Windows启动过程(MBR引导过程分析)](https://www.cnblogs.com/LittleHann/p/6974928.html)

[7] [Windows™ 7, 8/8.1 or 10 MBR ( Master Boot Record )](http://thestarman.narod.ru/asm/mbr/W7MBR.htm)

[8] [inb $0x64, %al的原理](https://blog.csdn.net/Great_Enterprise/article/details/104063004)

[9] [清华大学教学内核ucore学习系列（1） bootloader](https://www.cnblogs.com/maruixin/p/3175894.html)

[10] [硬盘的int13中断](https://bbs.csdn.net/topics/138412)

[11] [TCG BIOS DOS Test Tool](https://docs.microsoft.com/en-us/previous-versions/windows/hardware/wlk/ff567624(v%3dvs.85))