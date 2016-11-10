# 缩小 kvm 磁盘分区

磁盘分区比较大，随着kvm系统的使用，虽然可用空间没有减少，但是文件读写后会留下痕迹而不断占用硬盘空间。

所以将磁盘分区调整到一个较小的范围，可以很大的减少硬盘占用，同时也能减少维护、备份等操作的时间。

**系统环境**

```
archlinux, libvirt
```

**安装 kpartx**

```
su user
git clone https://aur.archlinux.org/multipath-tools.git
cd multipath-tools

makepkg -si
```

**将磁盘文件转为 raw 格式**

```
virsh destroy builder
qemu-img convert -O raw builder.img builder.raw
```

## 调整 lvm 格式的逻辑分区

**查看分区信息**

```
fdisk -l builder.raw

Disk builder.raw: 80 GiB, 85899345920 bytes, 167772160 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x69d905f3

Device       Boot   Start       End   Sectors  Size Id Type
builder.raw1 *       2048    999423    997376  487M 83 Linux
builder.raw2      1001470 167770111 166768642 79.5G  5 Extended
builder.raw5      1001472 167770111 166768640 79.5G 8e Linux LVM
```

```
parted builder.raw

GNU Parted 3.2
Using /home/libvirt/images/builder.raw
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/builder.raw: 85.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext2         boot
 2      513MB   85.9GB  85.4GB  extended
 5      513MB   85.9GB  85.4GB  logical                lvm
```

可以看出 builder.raw 硬盘镜像是总共有 80GB(85.9GB) 的空间，分为三个分区， 487M(512M) 的 boot 主分区1，79.5G(85.4GB) 扩展分区2, 79.5G(85.4GB) 逻辑分区5。

我们的目标是调整 builder.raw 硬盘镜像到 10GB 空间。由于含有 boot 分区占用487M空间，所以最后的扩展分区2约为 9.5G。

lvm 文件系统直接作用在扩展分区2上，所以后续的 lvresize 等调整都会使用约 10G-487M 的大小而不是 10G。

**挂载分区**

```
kpartx -av builder.raw

#dmsetup remove builder--vg-root
#dmsetup remove builder--vg-swap_1
```

```
dmsetup ls

builder--vg-root	(254:3)
builder--vg-swap_1	(254:4)
loop1p5	(254:2)
loop1p2	(254:1)
loop1p1	(254:0)
```

可以看出 lvm 中有两个分区 `builder--vg-root` 和 `builder--vg-swap_1`。 整理后我们不需要 `swap` 分区，可以在完成后使用 `swap` 文件替代。

检查分区错误

```
e2fsck -fy /dev/mapper/loop1p1
e2fsck -fy /dev/mapper/builder--vg-root
```

**调整 lvm 分区**

```
lvdisplay 
  --- Logical volume ---
  LV Path                /dev/builder-vg/root
  LV Name                root
  VG Name                builder-vg
  LV UUID                NHlDej-hJQr-AT3T-q7fY-RqdO-Fig1-DGmsdn
  LV Write Access        read/write
  LV Creation host, time builder, 2016-10-09 11:25:55 +0200
  LV Status              available
  # open                 0
  LV Size                71.52 GiB
  Current LE             18309
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:3
   
  --- Logical volume ---
  LV Path                /dev/builder-vg/swap_1
  LV Name                swap_1
  VG Name                builder-vg
  LV UUID                ptA94e-ms1T-5Q8s-qdcn-9GKy-DX4S-5Lx23T
  LV Write Access        read/write
  LV Creation host, time builder, 2016-10-09 11:25:55 +0200
  LV Status              available
  # open                 0
  LV Size                8.00 GiB
  Current LE             2048
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:4
   
```
删除 `swap` 分区

```
lvremove /dev/builder-vg/swap_1
```

查看硬盘内容实际占用的空间

```
qemu-img info builder.raw

image: builder.raw
file format: raw
virtual size: 80G (85899345920 bytes)
disk size: 7.6G
```

可以看到只占用了 7.6G


缩小 `/dev/mapper/builder--vg-root`，注意 `resize2fs` 时不要小于实际占用的空间。

```
resize2fs /dev/mapper/builder--vg-root 8G
```

缩小 lv 分区到 `10G-487M=9.5G`

```
lvreduce -L 9.5G /dev/builder-vg/root
e2fsck -fy /dev/mapper/builder--vg-root
resize2fs /dev/mapper/builder--vg-root
```

```
lvdisplay
vgdisplay
pvdisplay
```

注意删除 swap 分区后，可以挂载文件系统并修改 fstab 文件

```
mount /dev/builder-vg/root /mnt
vi /mnt/etc/fstab
```

```
#/dev/mapper/builder--vg-swap_1 none
```

```
umount /mnt
```

**调整扩展分区**

```
dmsetup remove /dev/mapper/builder--vg-root
kpartx -d builder.raw

parted builder.raw
```

目标镜像文件为 `10GB` 即 `10*1024*1024*1024`， parted 是以 `10^3` 为单位的，所以目标为 `10*1024*1024*1024/1000/1000/1000 = 10.73741824 GB = 10.73GB`。

注意这里取小一些，留几M空闲空间保证安全。

```
GNU Parted 3.2
Using /home/libvirt/images/builder.raw
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/builder.raw: 85.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext2         boot
 2      513MB   85.9GB  85.4GB  extended
 5      513MB   85.9GB  85.4GB  logical                lvm

(parted) resizepart 5
End?  [85.9GB]? 10.73GB                                                   
Warning: Shrinking a partition can cause data loss, are you sure you want to
continue?
Yes/No? yes                                                               
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/builder.raw: 85.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext2         boot
 2      513MB   85.9GB  85.4GB  extended
 5      513MB   10.7GB  10.2GB  logical                lvm

(parted) resizepart 2 10.73GB                                             
Warning: Shrinking a partition can cause data loss, are you sure you want to
continue?
Yes/No? yes                                                               
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/builder.raw: 85.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system  Flags
 1      1049kB  512MB   511MB   primary   ext2         boot
 2      513MB   10.7GB  10.2GB  extended
 5      513MB   10.7GB  10.2GB  logical                lvm

(parted) quit
```

**调整物理卷大小**

首先确认需要分配的空间大小

```
kpartx -av builder.raw
pvs -v --segments /dev/mapper/loop1p5
```

```
    Wiping internal VG cache
    Wiping cache of LVM-capable devices
  WARNING: Device /dev/mapper/loop1p5 has size of 19955560 sectors which is smaller than corresponding PV size of 166768640 sectors. Was device resized?
  One or more devices used as PVs in VG builder-vg have changed sizes.
  PV                  VG         Fmt  Attr PSize  PFree  Start SSize LV   Start Type   PE Ranges                 
  /dev/mapper/loop1p5 builder-vg lvm2 a--  79.52g 70.02g     0  2432 root     0 linear /dev/mapper/loop1p5:0-2431
  /dev/mapper/loop1p5 builder-vg lvm2 a--  79.52g 70.02g  2432 17925          0 free
```

可以看出需要 `2432*4M = 9728M` 

```
#pvresize --setphysicalvolumesize 9728M /dev/mapper/loop1p5
pvresize /dev/mapper/loop1p5
```

```
e2fsck -fy /dev/mapper/builder--vg-root

dmsetup remove /dev/mapper/builder--vg-root
kpartx -d builder.raw
```

**调整镜像文件大小**

```
qemu-img resize builder.raw 10G
```

这样缩小 kvm 镜像文件就完成了。


**转换镜像格式**

```
qemu-img convert -O qcow2 builder.raw builder.qcow2
```

之后备份 `builder.img`， 将 `builder.qcow2` 替换为原来的 `builder.img` 文件， 启动 kvm 虚拟机，登陆虚拟机后使用 `df -h` `blkid` 等命令检查分区状态。

```
virsh start builder
```

## 调整普通分区

```
fdisk -l ubuntu.raw
Disk ubuntu.raw: 30 GiB, 32212254720 bytes, 62914560 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4708a4ce

Device      Boot    Start      End  Sectors  Size Id Type
ubuntu.raw1 *        2048 60817407 60815360   29G 83 Linux
ubuntu.raw2      60819454 62912511  2093058 1022M  5 Extended
ubuntu.raw5      60819456 62912511  2093056 1022M 82 Linux swap / Solaris
```

下面操作将移除 swap 分区和扩展分区，调整主分区为合适的大小

**挂载文件系统并修改 fstab**

```
mount /dev/mapper/loop0p1 /mnt
```

注释掉 swap 分区，可以在完成后以 swap 文件的形式挂载

```
vi /mnt/etc/fstab

#UUID=95a9913e-06d2-4bd1-90eb-d1d63a00a853 none            swap    sw 
```

对文件系统做清零的操作

```
dd if=/dev/zero of=/mnt/test.file bs=1M
rm /mnt/test.file
sync
```

检查需要分配的空间，可以看出只需要分配 10G 就可以满足需求了

```
df -h
/dev/mapper/loop0p1   29G  6.4G   21G  24% /mnt
```

卸载分区

```
umount /mnt
```

**调整分区**

首先检查文件系统，再通过 `resize2fs` 将文件系统内容转移。

```
e2fsck -f /dev/mapper/loop0p1
resize2fs /dev/mapper/loop0p1 9G
```

使用 `parted` 命令调整分区结构, 注意 `parted` 是以 `10^3` 为进位单位，所以调整分区时 10GB 输入为 10.73GB (10*1024*1024*1024/1000/1000/1000)。注意可以取稍微小一些。

```
kpartx -d ubuntu.raw
parted ubuntu.raw
```

```
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/ubuntu.raw: 32.2GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type      File system     Flags
 1      1049kB  31.1GB  31.1GB  primary   ext4            boot
 2      31.1GB  32.2GB  1072MB  extended
 5      31.1GB  32.2GB  1072MB  logical   linux-swap(v1)
 
(parted) rm 2                                                             
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/ubuntu.raw: 32.2GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  31.1GB  31.1GB  primary  ext4         boot
 
(parted) resizepart 1 10.73GB                                          
Warning: Shrinking a partition can cause data loss, are you sure you want to
continue?
Yes/No? yes                                                               
(parted) print                                                            
Model:  (file)
Disk /home/libvirt/images/ubuntu.raw: 32.2GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  10.7GB  10.7GB  primary  ext4         boot
```

查看分区状态

```
fdisk -l ubuntu.raw

kpartx -av ubuntu.raw

e2fsck -f /dev/mapper/loop0p1
```

**调整镜像文件大小**

```
qemu-img resize ubuntu.raw 10G

kpartx -d ubuntu.raw
kpartx -av ubuntu.raw
e2fsck -f /dev/mapper/loop0p1
```

**转换镜像格式**

```
kpartx -d ubuntu.raw
qemu-img convert -O qcow2 ubuntu.raw ubuntu.qcow2
```

最后的 `qcow2` 文件就是缩小分区后的镜像文件，备份原始镜像后，重命名并启动 kvm 虚拟机测试是否一切正常。

```
mkdir backup

mv ubuntu.img backup
mv ubuntu.raw backup
mv ubuntu.qcow2 ubuntu.img

virsh start ubuntu
```
