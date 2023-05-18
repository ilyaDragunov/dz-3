		 Домашнее задание 3

		   Работа с LVM

1. Уменьшить том под / до 8G
2. Выделить том под /home
3. Выделить том под /var (/var - сделать в mirror)
4. Для /home - сделать том для снэпшотов
5. Прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)

		 Работа со снапшотами

1. Сгенерировать файлы в /home/
2. Снять снэпшот
3. Удалить часть файлов
4. Восстановиться со снэпшота (залоггировать работу можно утилитой script, скриншотами и т.п.)

	        Задание со звездочкой*

1. На нашей куче дисков попробовать поставить btrfs/zfs: с кешем и снэпшотами
2. Разметить здесь каталог /opt


		Практика с введением в LVM

1. Опеределяем устройства, которые будут использоваться под Physical Volumes (далее - PV) для наших будущих Volume Groups (далее - VG)

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$  lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

-------------------------------------------------------------------------------------------------------------------------------------

2. На дисках sdd,sde создадим lvm mirror.

[vagrant@lvm ~]$ sudo lvmdiskscan
  /dev/VolGroup00/LogVol00 [     <37.47 GiB]
  /dev/VolGroup00/LogVol01 [       1.50 GiB]
  /dev/sda2                [       1.00 GiB]
  /dev/sda3                [     <39.00 GiB] LVM physical volume
  /dev/sdb                 [      10.00 GiB]
  /dev/sdc                 [       2.00 GiB]
  /dev/sdd                 [       1.00 GiB]
  /dev/sde                 [       1.00 GiB]
  4 disks
  3 partitions
  0 LVM physical volume whole disks
  1 LVM physical volume
-------------------------------------------------------------------------------------------------------------------------------------

3. Размечаем диск для будущего использования LVM, создаем - PV (Physical Volumes)

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
-------------------------------------------------------------------------------------------------------------------------------------

4. Создаем первый уровень абстракции - VG (Volume Groups)

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgcreate otus /dev/sdb
  Volume group "otus" successfully created
-------------------------------------------------------------------------------------------------------------------------------------

5. Создаем логический том - LG (Logical Volume)

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvcreate -l+80%FREE -n test otus
  Logical volume "test" created.
-------------------------------------------------------------------------------------------------------------------------------------

5. Смотрим информацию об созданной VG

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgdisplay otus
  --- Volume group ---
  VG Name               otus
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <10.00 GiB
  PE Size               4.00 MiB
  Total PE              2559
  Alloc PE / Size       2047 / <8.00 GiB
  Free  PE / Size       512 / 2.00 GiB
  VG UUID               rdJW0l-T9Je-2jZy-112K-Mt6M-Zsiy-gSHkO3
-------------------------------------------------------------------------------------------------------------------------------------

6. Можем просмотреть информацию, о том какие диски входят в VG

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgdisplay -v otus | grep 'PV Name'
  PV Name /dev/sdb
-------------------------------------------------------------------------------------------------------------------------------------

7. Получим детальную информацию о LV

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvdisplay /dev/otus/test
  --- Logical volume ---
  LV Path                /dev/otus/test
  LV Name                test
  VG Name                otus
  LV UUID                cFyC8L-csfA-3L40-JZ46-ucEz-t3wP-ZxeGwm
  LV Write Access        read/write
  LV Creation host, time lvm, 2023-05-12 14:47:20 +0000
  LV Status              available
  # open                 0
  LV Size                <8.00 GiB
  Current LE             2047
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2
-------------------------------------------------------------------------------------------------------------------------------------

8. Получим в сжатом виде инфомацию командами vgs и lvs

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
  otus         1   1   0 wz--n- <10.00g 2.00g

[vagrant@lvm ~]$ sudo lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  test     otus       -wi-a-----  <8.00g
-------------------------------------------------------------------------------------------------------------------------------------

9. Создаем еще один LV из свободного места не экстентами, а абсолютным значением в мегабайтах

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvcreate -L100M -n small otus
  Logical volume "small" created.

[vagrant@lvm ~]$ sudo lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  small    otus       -wi-a----- 100.00m
  test     otus       -wi-a-----  <8.00g

[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
├─otus-test             253:2    0    8G  0 lvm
└─otus-small            253:3    0  100M  0 lvm
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

-------------------------------------------------------------------------------------------------------------------------------------

10. Создаем на LV файловую систему и монтируем ее

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo mkfs.ext4 /dev/otus/test
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
524288 inodes, 2096128 blocks
104806 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2147483648
64 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

[vagrant@lvm ~]$ mkdir /data
[vagrant@lvm ~]$ sudo mkdir /data
[vagrant@lvm ~]$ sudo mount /dev/otus/test /data/
[vagrant@lvm ~]$ mount | grep /data
/dev/mapper/otus-test on /data type ext4 (rw,relatime,seclabel,data=ordered)
[vagrant@lvm ~]$ /dev/mapper/otus-test on /data type ext4 (rw,relatime,seclabel,data=ordered

-------------------------------------------------------------------------------------------------------------------------------------

11. Встала проблема нехватки свободного места в директории /data. Мы расширим файловую систему на LV /dev/otus/test за счет нового
   блочного устройства  /dev/sdc. Сначала так же создаем PV

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.

-------------------------------------------------------------------------------------------------------------------------------------

12. Далее расширяем VG добавив в него новый диск 

[vagrant@lvm ~]$ sudo vgextend otus /dev/sdc
  Volume group "otus" successfully extended

-------------------------------------------------------------------------------------------------------------------------------------

13. Убедимся что новый диск присутствует в новой VG
 
-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgdisplay -v otus | grep 'PV Name'
  PV Name               /dev/sdb
  PV Name               /dev/sdc

-------------------------------------------------------------------------------------------------------------------------------------

14. Убеждаемся что место в VG увеличелось

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g     0
  otus         2   2   0 wz--n-  11.99g <3.90g

-------------------------------------------------------------------------------------------------------------------------------------

15. Суммируем занятое место с помощью команды "dd"

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress
8205107200 bytes (8.2 GB) copied, 19.256557 s, 426 MB/s
dd: error writing ‘/data/test.log’: No space left on device
7880+0 records in
7879+0 records out
8262189056 bytes (8.3 GB) copied, 19.3665 s, 427 MB/s

[vagrant@lvm ~]$ df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data

-------------------------------------------------------------------------------------------------------------------------------------

16. Увеличиваем LV за счет появившегося свободного места. Возьмем не все место - для того, чтобы осталосы место 
   для демонстрации снапшотов

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvextend -l+80%FREE /dev/otus/test
  Size of logical volume otus/test changed from <8.00 GiB (2047 extents) to <11.12 GiB (2846 extents).
  Logical volume otus/test successfully resized.

-------------------------------------------------------------------------------------------------------------------------------------

17. Проверим, что LV расширен до 11.12g

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvs /dev/otus/test
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test otus -wi-ao---- <11.12g

-------------------------------------------------------------------------------------------------------------------------------------

18. Но файловая система при этом осталась прежнего размера, произведем resize файловой системы

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data

[vagrant@lvm ~]$ sudo resize2fs /dev/otus/test
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/otus/test is mounted on /data; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/otus/test is now 2914304 blocks long.

[vagrant@lvm ~]$ sudo df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4   11G  7.8G  2.6G  76% /data

-------------------------------------------------------------------------------------------------------------------------------------

19. Допустим мы забыли оставить место для снапшота. Можно уменьшить существующий LV с помощью команды "lvreduce",
   но перед этим необходимо отмонтировать файловую систему, проверить её на ошибки и уменьшить ее размер

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo umount /data/

[vagrant@lvm ~]$ sudo e2fsck -fy /dev/otus/test
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/otus/test: 12/729088 files (0.0% non-contiguous), 2105907/2914304 blocks

[vagrant@lvm ~]$ sudo resize2fs /dev/otus/test 10G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/otus/test to 2621440 (4k) blocks.
The filesystem on /dev/otus/test is now 2621440 blocks long.

[vagrant@lvm ~]$ sudo  lvreduce /dev/otus/test -L 10G
  WARNING: Reducing active logical volume to 10.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce otus/test? [y/n]: y
  Size of logical volume otus/test changed from <11.12 GiB (2846 extents) to 10.00 GiB (2560 extents).
  Logical volume otus/test successfully resized.

[vagrant@lvm ~]$ sudo mount /dev/otus/test /data/

[vagrant@lvm ~]$ sudo df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  9.8G  7.8G  1.6G  84% /data

[vagrant@lvm ~]$ sudo lvs /dev/otus/test
  LV   VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test otus -wi-ao---- 10.00g

-------------------------------------------------------------------------------------------------------------------------------------

20. Создаем снапшот командой "lvcreate", только с флагом -s, который указывает на то, что это снимок

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo  lvcreate -L 500M -s -n test-snap /dev/otus/test
  Logical volume "test-snap" created.

-------------------------------------------------------------------------------------------------------------------------------------

21. С помощью команд "vgs" и "lsblk" покажем что у нас получилось

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo vgs -o +lv_size,lv_name | grep test
  otus         2   3   1 wz--n-  11.99g <1.41g  10.00g test
  otus         2   3   1 wz--n-  11.99g <1.41g 500.00m test-snap

[vagrant@lvm ~]$ sudo lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
├─otus-small            253:3    0  100M  0 lvm
└─otus-test-real        253:4    0   10G  0 lvm
  ├─otus-test           253:2    0   10G  0 lvm  /data
  └─otus-test--snap     253:6    0   10G  0 lvm
sdc                       8:32   0    2G  0 disk
├─otus-test-real        253:4    0   10G  0 lvm
│ ├─otus-test           253:2    0   10G  0 lvm  /data
│ └─otus-test--snap     253:6    0   10G  0 lvm
└─otus-test--snap-cow   253:5    0  500M  0 lvm
  └─otus-test--snap     253:6    0   10G  0 lvm
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

-------------------------------------------------------------------------------------------------------------------------------------

22. Снапшот монтируем как и любой другой LV

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo mkdir /data-snap
[vagrant@lvm ~]$ sudo mount /dev/otus/test-snap /data-snap/
[vagrant@lvm ~]$  ll /data-snap/
total 8068564
drwx------. 2 root root      16384 May 16 19:04 lost+found
-rw-r--r--. 1 root root 8262189056 May 16 19:28 test.log

[vagrant@lvm ~]$ sudo lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
├─otus-small            253:3    0  100M  0 lvm
└─otus-test-real        253:4    0   10G  0 lvm
  ├─otus-test           253:2    0   10G  0 lvm  /data
  └─otus-test--snap     253:6    0   10G  0 lvm  /data-snap
sdc                       8:32   0    2G  0 disk
├─otus-test-real        253:4    0   10G  0 lvm
│ ├─otus-test           253:2    0   10G  0 lvm  /data
│ └─otus-test--snap     253:6    0   10G  0 lvm  /data-snap
└─otus-test--snap-cow   253:5    0  500M  0 lvm
  └─otus-test--snap     253:6    0   10G  0 lvm  /data-snap
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

[vagrant@lvm ~]$ umount /data-snap

[vagrant@lvm ~]$ sudo lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
├─otus-small            253:3    0  100M  0 lvm
└─otus-test-real        253:4    0   10G  0 lvm
  ├─otus-test           253:2    0   10G  0 lvm  /data
  └─otus-test--snap     253:6    0   10G  0 lvm
sdc                       8:32   0    2G  0 disk
├─otus-test-real        253:4    0   10G  0 lvm
│ ├─otus-test           253:2    0   10G  0 lvm  /data
│ └─otus-test--snap     253:6    0   10G  0 lvm
└─otus-test--snap-cow   253:5    0  500M  0 lvm
  └─otus-test--snap     253:6    0   10G  0 lvm
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

-------------------------------------------------------------------------------------------------------------------------------------

23. Восстановим предыдущее состояние. “Откатимся” на снапшот. Для этого сначала для большей наглядности удалим наш log файл

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo rm test.log 
rm: remove regular file 'test.log'? y 

[vagrant@lvm ~]$  ll /data-snap/
total 16
drwx------. 2 root root      16384 May 16 19:04 lost+found

[vagrant@lvm ~]$ umount /data

[vagrant@lvm ~]$ sudo lvconvert --merge /dev/otus/test-snap 
 Merging of volume otus/test-snap started. 
 otus/test: Merged: 100.00% 

[vagrant@lvm ~]$ sudo mount /dev/otus/test /data 

[vagrant@lvm ~]$  ll /data-snap/
total 8068564
drwx------. 2 root root      16384 May 16 19:04 lost+found
-rw-r--r--. 1 root root 8262189056 May 16 19:28 test.log

-------------------------------------------------------------------------------------------------------------------------------------

24. Работа с LVM

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo pvcreate /dev/sd{d,e}
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
[vagrant@lvm ~]$ vgcreate vg0 /dev/sd{d,e}

[vagrant@lvm ~]$ sudo vgcreate vg0 /dev/sd{d,e}
  Volume group "vg0" successfully created
[vagrant@lvm ~]$ sudo lvcreate -l+80%FREE -m1 -n mirror vg0
  Logical volume "mirror" created.
[vagrant@lvm ~]$ sudo  lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  mirror   vg0        rwi-a-r--- 816.00m                                    100.00

-------------------------------------------------------------------------------------------------------------------------------------



				Домашнее задание 	



1. Перед началом работы установим пакет xfsdump, он будет необходим для снятия копии/тома

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo yum install xfsdump
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: ftp.halifax.rwth-aachen.de
 * extras: ftp.halifax.rwth-aachen.de
 * updates: ftp.halifax.rwth-aachen.de
Resolving Dependencies
--> Running transaction check
---> Package xfsdump.x86_64 0:3.1.7-3.el7_9 will be installed
--> Processing Dependency: attr >= 2.0.0 for package: xfsdump-3.1.7-3.el7_9.x86_                64
--> Running transaction check
---> Package attr.x86_64 0:2.4.46-13.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch            Version                Repository        Size
================================================================================
Installing:
 xfsdump          x86_64          3.1.7-3.el7_9          updates          309 k
Installing for dependencies:
 attr             x86_64          2.4.46-13.el7          base              66 k

Transaction Summary
================================================================================
Install  1 Package (+1 Dependent package)

Total download size: 374 k
Installed size: 1.1 M
Is this ok [y/d/N]: y
Downloading packages:
(1/2): attr-2.4.46-13.el7.x86_64.rpm                                     |  66 kB  00:00:00
(2/2): xfsdump-3.1.7-3.el7_9.x86_64.rpm                                  | 309 kB  00:00:00
------------------------------------------------------------------------------------------------
Total                                                           475 kB/s | 374 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : attr-2.4.46-13.el7.x86_64                                                    1/2
  Installing : xfsdump-3.1.7-3.el7_9.x86_64                                                 2/2
  Verifying  : attr-2.4.46-13.el7.x86_64                                                    1/2
  Verifying  : xfsdump-3.1.7-3.el7_9.x86_64                                                 2/2

Installed:
  xfsdump.x86_64 0:3.1.7-3.el7_9

Dependency Installed:
  attr.x86_64 0:2.4.46-13.el7

Complete!

-------------------------------------------------------------------------------------------------------------------------------------

2. Подготовим внутренний том для раздела

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

[vagrant@lvm ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[vagrant@lvm ~]$ sudo vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

[vagrant@lvm ~]$ sudo lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

-------------------------------------------------------------------------------------------------------------------------------------

3. Создаем на нем файловую систему и смонтируем его, чтобы перенести туда данные

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[vagrant@lvm ~]$ sudo mount /dev/vg_root/lv_root /mnt


-------------------------------------------------------------------------------------------------------------------------------------

4. Командой "xfsdump" скопируем все даннsе с раздела /dev/VolGroup00/LogVol00 в /mnt 

[vagrant@lvm ~]$ sudo xfsdump -J - /dev/VolGroup00/LogVol00 | sudo xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed May 17 09:54:38 2023
xfsdump: session id: 1b4d963e-1e86-45af-b488-1bc5a0ea2fc4
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 913478272 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Wed May 17 09:54:38 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 1b4d963e-1e86-45af-b488-1bc5a0ea2fc4
xfsrestore: media id: 989f6fa5-638b-421c-bdfe-bacf4f66cce6
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 3136 directories and 26917 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 885637192 bytes
xfsdump: dump size (non-dir files) : 870467848 bytes
xfsdump: dump complete: 24 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 25 seconds elapsed
xfsrestore: Restore Status: SUCCESS
[vagrant@lvm ~]$

-------------------------------------------------------------------------------------------------------------------------------------

5. Переконфигурируем grub для того, чтобы при старте перейти в новый католог.
   Сымитируем текущий root -> сделаем в него chroot и обновим grub 

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done

[vagrant@lvm ~]$ sudo chroot /mnt/

[root@lvm /]# sudo grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

-------------------------------------------------------------------------------------------------------------------------------------

6. Обновим образ initrd

-------------------------------------------------------------------------------------------------------------------------------------

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;  s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

-------------------------------------------------------------------------------------------------------------------------------------

7. Для того, чтобы при загрузке был смонтирован нужный root нужно в файле  /boot/grub2/grub.cfg
   заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

-------------------------------------------------------------------------------------------------------------------------------------

Оригинальная строка

linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/vg_root-lv_root ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=VolGroup00/LogVol00 rd.lvm.lv$
        initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img

Замененнная строка

linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/vg_root-lv_root ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=rd.lvm.lv=vg_root/lv_root rd.lvm.lv$
        initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img

-------------------------------------------------------------------------------------------------------------------------------------

8. Перегружаемся с новым рутом.

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

-------------------------------------------------------------------------------------------------------------------------------------

9. Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удалāем старый LV размером в 40G и создаем новый на 8G

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed

[vagrant@lvm ~]$ sudo lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.

[vagrant@lvm ~]$ sudo mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol00 /mnt

[vagrant@lvm ~]$ sudo xfsdump -J - /dev/vg_root/lv_root | sudo xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed May 17 13:35:33 2023
xfsdump: session id: c844cbfe-7c78-46f5-93fd-3999efe40cce
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 914157568 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_root-lv_root
xfsrestore: session time: Wed May 17 13:35:33 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: 88653911-ff56-4406-adf8-8fe60c253684
xfsrestore: session id: c844cbfe-7c78-46f5-93fd-3999efe40cce
xfsrestore: media id: 1e90c8ad-d5bc-43bd-9f5b-5c1686f6d775
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 3156 directories and 27043 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 886156960 bytes
xfsdump: dump size (non-dir files) : 870905200 bytes
xfsdump: dump complete: 23 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 24 seconds elapsed
xfsrestore: Restore Status: SUCCESS

-------------------------------------------------------------------------------------------------------------------------------------

10. Переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done

[vagrant@lvm ~]$ sudo chroot /mnt/

[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;  s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

-------------------------------------------------------------------------------------------------------------------------------------

11. Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var. На свободных дисках создаем зеркало

-------------------------------------------------------------------------------------------------------------------------------------

[root@lvm boot]# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.

[root@lvm boot]# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created

[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.

[root@lvm boot]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /
sdb                        8:16   0   10G  0 disk
└─vg_root-lv_root        253:0    0   10G  0 lvm
sdc                        8:32   0    2G  0 disk
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm
sde                        8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm

-------------------------------------------------------------------------------------------------------------------------------------

12. Создаем на нем ФС и перемещаем туда /var:

-------------------------------------------------------------------------------------------------------------------------------------

[root@lvm boot]# sudo mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

[root@lvm boot]# sudo mount /dev/vg_var/lv_var /mnt

[root@lvm boot]# sudo cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/

[root@lvm boot]# sudo mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

[root@lvm boot]# sudo umount /mnt

[root@lvm boot]# sudo mount /dev/vg_var/lv_var /var

-------------------------------------------------------------------------------------------------------------------------------------

13. Правим fstab для автоматического монтирования /var 

-------------------------------------------------------------------------------------------------------------------------------------

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

[vagrant@lvm ~]$ lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /mnt/boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /mnt
sdb                        8:16   0   10G  0 disk
└─vg_root-lv_root        253:0    0   10G  0 lvm  /
sdc                        8:32   0    2G  0 disk
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt/var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt/var
sde                        8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt/var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt/var

-------------------------------------------------------------------------------------------------------------------------------------

14. После чего можно успешно перезагружаемся в новый (уменьшеный root) и удаляем  временную Volume Group

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[vagrant@lvm ~]$ sudo vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[vagrant@lvm ~]$ sudo  pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
[vagrant@lvm ~]$
[vagrant@lvm ~]$ lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk
sdc                        8:32   0    2G  0 disk
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var

-------------------------------------------------------------------------------------------------------------------------------------

15. Выделяем том под /home по тому же принципу что делали для /var

-------------------------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.

[vagrant@lvm ~]$ sudo mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /mnt/

[vagrant@lvm ~]$ sudo cp -aR /home/* /mnt/

[vagrant@lvm ~]$ sudo rm -rf /home/*

[vagrant@lvm ~]$ sudo umount /mnt

[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /home/

[vagrant@lvm etc]$ sudo lvs
  LV          VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00    VolGroup00 -wi-ao----   8.00g
  LogVol01    VolGroup00 -wi-ao----   1.50g
  LogVol_Home VolGroup00 -wi-ao----   2.00g
  lv_var      vg_var     rwi-aor--- 952.00m                                    100.00
[vagrant@lvm etc]$
[vagrant@lvm etc]$
[vagrant@lvm etc]$ sudo pvs
  PV         VG         Fmt  Attr PSize    PFree
  /dev/sda3  VolGroup00 lvm2 a--   <38.97g <27.47g
  /dev/sdd   vg_var     lvm2 a--  1020.00m  64.00m
  /dev/sde   vg_var     lvm2 a--  1020.00m  64.00m

[vagrant@lvm etc]$ su
Password:

[root@lvm etc]# lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk
├─sda1                       8:1    0    1M  0 part
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk
sdc                          8:32   0    2G  0 disk
sdd                          8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var

[root@lvm etc]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

Позже найдена команда что бы бороться с правами файлика fstab От sudo

[vagrant@lvm ~]$ sudo sh -c "echo "`blkid | grep home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab"

--------------------------------------------------------------------------------------------------------------------

16. Сгенерируем файлs в /home/

--------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo touch /home/file{1..20}

--------------------------------------------------------------------------------------------------------------------

17. Снимаем снапшот

--------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.

--------------------------------------------------------------------------------------------------------------------

18. Удаляем часть файлов

--------------------------------------------------------------------------------------------------------------------

[vagrant@lvm ~]$ sudo rm -f /home/file{11..20}

--------------------------------------------------------------------------------------------------------------------

19. Процесс восстановлениā со снапшота

--------------------------------------------------------------------------------------------------------------------

[root@lvm /]# touch /home/file{1..20}

[root@lvm /]# ls /home/
file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
file10  file12  file14  file16  file18  file2   file3   file5  file7  file9

[root@lvm /]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.

[root@lvm /]# rm -f /home/file{5..20}

[root@lvm /]# ls /home/
file1  file2  file3  file4  vagrant

[root@lvm /]# umount /home

[root@lvm /]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%

[root@lvm /]# mount /home

[root@lvm /]# ls /home/
file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
file10  file12  file14  file16  file18  file2   file3   file5  file7  file9














































































