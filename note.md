# record my common notes

## dos2unix
```shell
find ./ -name "*.py" | xargs dos2unix
```

## some useful docker commands
```shell
#list images with format like 'repository:tag'
docker images --format '{{.Repository}}:{{.Tag}}'
#search images of a certain registry
docker images --format '{{.Repository}}:{{.Tag}}' | grep registryName | sed 's#/# #g' | awk '{print $2}'

```

## ip forward
```shell
#!/bin/bash
pro='tcp'
NAT_Host='192.168.56.105'
NAT_Port=10680
Dst_Host='192.168.56.106'
Dst_Port=80
iptables -t nat -A PREROUTING -p $pro --dport $NAT_Port -j DNAT --to-destination $Dst_Host:$Dst_Port
iptables -t nat -A POSTROUTING -p $pro --dport $Dst_Port -d $Dst_Host -j SNAT --to $NAT_Host
#echo 1 > /proc/sys/net/ipv4/ip_forward
```
## raspberry pi simple gpio example using shell
```shell
#!/bin/bash

echo "5" > /sys/class/gpio/export
echo "out" > /sys/class/gpio/gpio5/direction
echo "1" > /sys/class/gpio/gpio5/value

sleep 1

echo "5" > /sys/class/gpio/unexport
```

## some raspberry pi docker repository
- armbuild: https://hub.docker.com/u/armbuild/
- resin: https://hub.docker.com/u/resin/
- hypriot: https://hub.docker.com/u/hypriot/


-----------------------------------------------------------
# 记录一些异常状况的解决。
## virtualbox 恢复到最近状态。
情况：virtualbox中虚拟机出错，重启后恢复到上次的snapshot状态（snapshot很久之前）。    

解决：关机。查找`VirtualBox VMs\<vmname>\Snapshots]`目录下的.vdi文件，如果有出错之前最新的，在virtualbox界面，点击设置->存储，将当前的.vdi移除，再点击添加虚拟硬盘，选中最新的.vdi文件，添加，没问题的话话启动虚拟机，应该就可以恢复到出错之前的状态，前提是可以找到出错前的.vdi文件。    

![](raw/vbox1.png)
![](raw/vbox2.png)
![](raw/vbox3.png)


## virtualbox扩展磁盘
名称为centos7的centos VM    
1. 宿主机cmd,切换到`User\<name>VirtualBox VMs\centos7\`目录：`\path\to\VBoxManage.exe modifyhd centos7.vdi --resize 25600`
2. 进入linux VM, `sudo su`切换到root用户。
3. `fdisk -l /dev/sda` 查看磁盘情况。可以看到已经扩展，但还不能使用。
```shell
[root@centos7mao ~]# fdisk -l /dev/sda

Disk /dev/sda: 26.8 GB, 26843545600 bytes, 52428800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a0117

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    16777215     7875584   8e  Linux LVM

[mao@centos7mao ~]$ df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  6.7G  6.1G  649M  91% /
devtmpfs                 906M     0  906M   0% /dev
tmpfs                    921M  100K  920M   1% /dev/shm
tmpfs                    921M   17M  904M   2% /run
tmpfs                    921M     0  921M   0% /sys/fs/cgroup
/dev/sda1                497M  249M  249M  50% /boot
tmpfs                    185M   16K  184M   1% /run/user/1000

```
4. `fdisk /dev/sda` 将虚拟磁盘的空闲空间创建为一个新的分区。使用代表 Linux LVM 的分区号 8e 来作为 ID。
```shell
[root@centos7mao ~]# fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
Partition number (3,4, default 3): 3
First sector (16777216-52428799, default 16777216): 
Using default value 16777216
Last sector, +sectors or +size{K,M,G} (16777216-52428799, default 52428799): 
Using default value 52428799
Partition 3 of type Linux and of size 17 GiB is set

Command (m for help): t
Partition number (1-3, default 3): 3
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.

```
5. `reboot` 重启。
6. `vgdisplay` 查看VG Name，此处为centos。
```shell
[root@centos7mao ~]# vgdisplay 
  --- Volume group ---
  VG Name               centos
  System ID             
... ... ...
```
7. `pvcreate /dev/sda3` 把新分配的空间创建一个新的物理卷, `vgextend centos /dev/sda3` 使用新的物理卷来扩展 LVM 的 VolGroup,  `lvextend /dev/centos/root /dev/sda3`, 然后扩展 LVM 的逻辑卷centos/root 。
```shell
[root@centos7mao ~]# pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created
[root@centos7mao ~]# vgextend centos /dev/sda3
  Volume group "centos" successfully extended
[root@centos7mao ~]# lvextend /dev/centos/root /dev/sda3
  Size of logical volume centos/root changed from 6.67 GiB (1707 extents) to 23.66 GiB (6058 extents).
  Logical volume root successfully resized.
 ```
8. `xfs_growfs /dev/centos/root` 调整逻辑卷的大小，完成。
```shell
[root@centos7mao ~]# xfs_growfs /dev/centos/root 
meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=436992 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=1747968, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 1747968 to 6203392
[root@centos7mao ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   24G  6.1G   18G  26% /
devtmpfs                 906M     0  906M   0% /dev
tmpfs                    921M   84K  920M   1% /dev/shm
tmpfs                    921M  8.8M  912M   1% /run
tmpfs                    921M     0  921M   0% /sys/fs/cgroup
/dev/sda1                497M  249M  249M  50% /boot
tmpfs                    185M   16K  184M   1% /run/user/42
tmpfs                    185M     0  185M   0% /run/user/1000
```
