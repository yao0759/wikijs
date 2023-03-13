---
title: BeeGFS分区对齐指南
description: 
published: true
date: 2023-03-13T13:58:59.338Z
tags: beegfs, storage, 性能优化
editor: markdown
dateCreated: 2023-03-13T13:58:59.338Z
---



最近在理解分区对齐，看了些文档，觉得beegfs的官方文档写的步骤最简单易操作，很适合去辅助理解，所以这里翻译了一下[官方文档](https://doc.beegfs.io/latest/advanced_topics/partition_alignment.html#partitionalignmentguide)

## 最简单的方法

存储设备上创建文件系统，而没有任何分区。要做到这一点，你只需使用没有任何分区号的`mkfs`，例如，对于`XFS`。

```
mkfs.xfs /dev/sdb
```

因此，你会很容易地避免由分区表带来的错位。然而，即使在这种情况下，你仍然应该看看本页底部的 "创建RAID优化的文件系统 "部分。

## 分区排列 - 例子

默认情况下，Linux对设备上的第一个主分区使用512字节对齐（更具体地说：63*512字节）。这对单个磁盘来说是很好的，至少是传统的磁盘，它使用512字节的块。对于RAID阵列或SSD，你要设置一个不同的对齐方式，以避免读-改-写操作的开销，并启用内部XFS或ext4 RAID优化。注意，如果你在RAID上使用其他软件层，如LVM，这些也会引入另一个偏移，因此需要考虑到正确的对齐方式。

在我们的例子中，我们在一个600GB的卷上使用。- 11个磁盘的阵列 - 在RAID-6模式下 - 条带大小为64KB - 作为设备`/dev/sdc`

通过RAID-6，我们实际上有9个磁盘用于用户数据，所以我们要对准`9*64KB`。(对于SSD，你可能想对准擦除块的大小，通常是512KB或其倍数。)

## 分区对准 - 检查当前

> 注意：下面的例子是基于fdisk的。并非所有版本的fdisk都与GPT分区表兼容。对于GPT分区，你可以用parted或gdisk代替。

如果你已经在你的硬盘上创建了一个分区，你可以用下面的命令检查它的排列。

```
$ fdisk -lu /dev/sdc

Disk /dev/sdc: 599.9 GB, 599999905792 bytes
255 heads, 63 sectors/track, 72945 cylinders, total 1171874816 sectors
Units = sectors of 1 * 512 = 512 bytes

   Device Boot      Start         End      Blocks   Id  System
/dev/sdc1              63  1171861424   585930681   83  Linux
```

正如你在上面看到的，分区的起始位置目前在63*512字节的位置，这对我们的用例是不合适的。

## 分区对齐 - 创建对齐的

> 注意 下面的例子是基于fdisk的，它与GPT分区表不兼容。要创建对齐的 GPT 分区，请使用 parted，例如。
>
> ```
> $ parted /dev/sdc mklabel gpt
> $ parted --align=opt /dev/sdc unit KiB mkpart pri $((9*64)) 100%
> ```

如果你的设备上已经有一个分区，请确保先删除它。

```
$ fdisk /dev/sdc

Command (m for help): d
Selected partition 1

Command (m for help): w
The partition table has been altered!
```

如上所述，我们要对齐到9*64KB。我们将使用`fdisk`的参数"`-H 8 -S 16` "来手动指定磁头和扇区的（逻辑）数量。这些参数允许我们创建一个对齐到64KB或64KB的任何倍数的分区。我们的设置是基于64KB的圆柱体，我们希望我们的分区在第一个条纹集（9个圆柱体）之后就开始，所以我们将第一个圆柱体设置为10。

```shell
$ fdisk -H 8 -S 16 /dev/sdc

The number of cylinders for this disk is set to 9155272.
There is nothing wrong with that, but this is larger than 1024,
and could in certain setups cause problems with:
1) software that runs at boot time (e.g., old versions of LILO)
2) booting and partitioning software from other OSs
   (e.g., DOS FDISK, OS/2 FDISK)

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-9155272, default 1): 10
Last cylinder or +size or +sizeM or +sizeK (10-9155272, default 9155272):
Using default value 9155272

Command (m for help): w
The partition table has been altered!
```

如果你想验证新的对齐方式。

```
$ fdisk -l /dev/sdc

Disk /dev/sdc: 599.9 GB, 599999905792 bytes
8 heads, 16 sectors/track, 9155272 cylinders
Units = cylinders of 128 * 512 = 65536 bytes

Device       Boot   Start         End      Blocks   Id  Type
/dev/sdc1              10     9155272   585936832   83  Linux
```

我们可以看到，该分区从第10扇区开始，扇区的大小为64KB。现在，是时候在新的分区上创建一个文件系统了。

## 创建RAID优化的文件系统

XFS和ext4允许你指定RAID设置。这使得文件系统能够为RAID调整优化其读写访问，例如，通过将数据作为完整的条带集提交以获得最大的吞吐量。这些设置可以在文件系统创建时进行，或者在XFS的情况下，也可以在安装时进行。

注意，这些RAID优化可以显著提高性能，但前提是你的分区是正确对齐的，或者你通过在一个没有分区的设备上创建XFS来避免不对齐。

要为9个磁盘（其中数字9不包括RAID-5或RAID-6奇偶校验磁盘的数量）和64KB的块大小创建一个新的XFS文件系统，使用。

```
$ mkfs.xfs -d su=64k,sw=9 -l version=2,su=64k /dev/sdc
```

> 注意：如果你的数据存储在一个RAID-5或RAID-6卷上，你可能要考虑把文件系统日志放在一个单独的RAID-1卷上，因为RAID-5和RAID-6的读-改-写开销很大。

