Домашнее задание: Работа с LVM

Задание:
1. уменьшить том под / до 8G
2. выделить том под /home
3. выделить том под /var (/var - сделать в mirror)
4. для /home - сделать том для снэпшотов
5. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
6. Работа со снапшотами:
• сгенерировать файлы в /home/
• снять снэпшот
• удалить часть файлов
• восстановиться со снэпшота
(залоггировать работу можно утилитой script, скриншотами и т.п.)



1) Создаём виртальную машину:
vagrant up

2) Подключаемся к ВМ:
vagrant ssh

3) Посмотрим какие блочные устройства у нас есть:
a) [vagrant@ubuntu2 ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
+-sda1                    8:1    0    1M  0 part
+-sda2                    8:2    0    1G  0 part /boot
L-sda3                    8:3    0   39G  0 part
  +-VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  L-VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

b) [vagrant@ubuntu2 ~]$ sudo lvmdiskscan
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

4) Разметим диск для будущего использования LVM - создадим PV:
[vagrant@ubuntu2 ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

5) Создадим первый уровень абстракции - VG:
[vagrant@ubuntu2 ~]$ sudo vgcreate otus /dev/sdb
  Volume group "otus" successfully created

6) Создадим LV:
[vagrant@ubuntu2 ~]$ sudo lvcreate -l+80%FREE -n test otus
  Logical volume "test" created.

7) Информация о VG:
[vagrant@ubuntu2 ~]$ sudo vgdisplay otus
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
  VG UUID               FoFjE1-i1Eu-3RNI-ArU7-8wAy-1FNg-URUnxS

8) Какие диски входят в VG:
[vagrant@ubuntu2 ~]$ sudo vgdisplay -v otus | grep 'PV NAME'
[vagrant@ubuntu2 ~]$ sudo vgdisplay -v otus | grep 'PV Name'
  PV Name               /dev/sdb

9) Информация о LV:

a) [vagrant@ubuntu2 ~]$ sudo lvdisplay /dev/otus/test
  --- Logical volume ---
  LV Path                /dev/otus/test
  LV Name                test
  VG Name                otus
  LV UUID                Xj2hi9-wVsF-1kwF-azbG-1cio-nGsf-eaavhE
  LV Write Access        read/write
  LV Creation host, time ubuntu2, 2024-01-28 11:29:20 +0000
  LV Status              available
  # open                 0
  LV Size                <8.00 GiB
  Current LE             2047
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

b) [root@ubuntu2 ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
  otus         1   1   0 wz--n- <10.00g 2.00g
c) [root@ubuntu2 ~]# lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  test     otus       -wi-a-----  <8.00g

10) запуск от root:
[vagrant@ubuntu2 ~]$ sudo -i

11) Создадим LV = 100 МБ:
[root@ubuntu2 ~]# lvcreate -L100M -n small otus
  Logical volume "small" created.

12) Информация о LV:
[root@ubuntu2 ~]# lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  small    otus       -wi-a----- 100.00m
  test     otus       -wi-a-----  <8.00g

13) Создадим на LV файловую систему:
[root@ubuntu2 ~]# mkfs.ext4 /dev/otus/test
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

14) Монтирвание:
[root@ubuntu2 ~]# mkdir /data
[root@ubuntu2 ~]# mount /dev/otus/test /data/
[root@ubuntu2 ~]# mount | grep /data
/dev/mapper/otus-test on /data type ext4 (rw,relatime,seclabel,data=ordered)

РАСШИРЕНИЕ LVM

15) Создадим PV на sdc:
[root@ubuntu2 ~]# pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.

16) Расширим VG otus, добавив диск sdc:
[root@ubuntu2 ~]# vgextend otus /dev/sdc
  Volume group "otus" successfully extended

17) Проверим, что диск добавился в VG:
[root@ubuntu2 ~]# vgdisplay -v otus | grep 'PV Name'
  PV Name               /dev/sdb
  PV Name               /dev/sdc

18) Проверим, что место в VG прибавилось:
[root@ubuntu2 ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g     0
  otus         2   2   0 wz--n-  11.99g <3.90g

19) Займем всё месчто на диске:
[root@ubuntu2 ~]# dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress
8257536000 bytes (8.3 GB) copied, 188.748600 s, 43.7 MB/s
dd: error writing ‘/data/test.log’: No space left on device
7880+0 records in
7879+0 records out
8262189056 bytes (8.3 GB) copied, 189.102 s, 43.7 MB/s

20) Занято всё место на диске:
[root@ubuntu2 ~]# df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data

21) Увеличим LVM:
[root@ubuntu2 ~]# lvextend -l+80%FREE /dev/otus/test
  Size of logical volume otus/test changed from <8.00 GiB (2047 extents) to <11.12 GiB (2846 extents).
  Logical volume otus/test successfully resized.

22) LVM расширен до 11.12:
[root@ubuntu2 ~]# lvs /dev/otus/test
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  test otus -wi-ao---- <11.12g

23) Файловая система осталось без изменений:
[root@ubuntu2 ~]# df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  7.8G  7.8G     0 100% /data

24) Произведем resize файловой системy:
[root@ubuntu2 ~]# resize2fs /dev/otus/test
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/otus/test is mounted on /data; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/otus/test is now 2914304 blocks long.

25) Информация о VLM:
[root@ubuntu2 ~]# df -Th /data
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4   11G  7.8G  2.6G  76% /data

26) Отмонтируем ФС:
[root@ubuntu2 ~]# umount /data/

27) Проверим ФС:
[root@ubuntu2 ~]# e2fsck -fy /dev/otus/test
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/otus/test: 12/729088 files (0.0% non-contiguous), 2105907/2914304 blocks

28) Уменьшим размер:
[root@ubuntu2 ~]# resize2fs /dev/otus/test 10G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/otus/test to 2621440 (4k) blocks.
The filesystem on /dev/otus/test is now 2621440 blocks long.

[root@ubuntu2 ~]# lvreduce /dev/otus/test -L 10G
  WARNING: Reducing active logical volume to 10.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce otus/test? [y/n]: н
  WARNING: Invalid input ''.
Do you really want to reduce otus/test? [y/n]: y
  Size of logical volume otus/test changed from <11.12 GiB (2846 extents) to 10.00 GiB (2560 extents).
  Logical volume otus/test successfully resized.

29) Примонтируем ФС:
[root@ubuntu2 ~]# mount /dev/otus/test /data/

30) Проверим размер:
[root@ubuntu2 ~]# df -Th /data/
Filesystem            Type  Size  Used Avail Use% Mounted on
/dev/mapper/otus-test ext4  9.8G  7.8G  1.6G  84% /data

31) Создадим снапшот LVM:
[root@ubuntu2 ~]# lvcreate -L 500M -s -n test-snap /dev/otus/test
 Logical volume "test-snap" created.

32) Проверим командой vgs:
[root@ubuntu2 ~]# sudo vgs -o +lv_size,lv_name | grep test
  otus         2   3   1 wz--n-  11.99g <1.41g  10.00g test
  otus         2   3   1 wz--n-  11.99g <1.41g 500.00m test-snap

33) Проверим командой lsblk:
[root@ubuntu2 ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
+-sda1                    8:1    0    1M  0 part
+-sda2                    8:2    0    1G  0 part /boot
L-sda3                    8:3    0   39G  0 part
  +-VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  L-VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
+-otus-small            253:3    0  100M  0 lvm
L-otus-test-real        253:4    0   10G  0 lvm
  +-otus-test           253:2    0   10G  0 lvm  /data
  L-otus-test--snap     253:6    0   10G  0 lvm
sdc                       8:32   0    2G  0 disk
+-otus-test-real        253:4    0   10G  0 lvm
¦ +-otus-test           253:2    0   10G  0 lvm  /data
¦ L-otus-test--snap     253:6    0   10G  0 lvm
L-otus-test--snap-cow   253:5    0  500M  0 lvm
  L-otus-test--snap     253:6    0   10G  0 lvm
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

34) Смонтируем снапшот
[root@ubuntu2 ~]# mkdir /data-snap

[root@ubuntu2 ~]# mount /dev/otus/test-snap /data-snap/

[root@ubuntu2 ~]# ll /data-snap/
total 8068564
drwx------. 2 root root      16384 Jan 28 11:44 lost+found
-rw-r--r--. 1 root root 8262189056 Jan 28 14:33 test.log

35) Отмонтируем
[root@ubuntu2 ~]# unmount /data-snap
-bash: unmount: command not found
[root@ubuntu2 ~]# umount /data-snap

36) Удалим файл:
[root@ubuntu2 ~]# ll /data
total 8068564
drwx------. 2 root root      16384 Jan 28 11:44 lost+found
-rw-r--r--. 1 root root 8262189056 Jan 28 14:33 test.log

[root@ubuntu2 ~]# rm /data/test.log
rm: remove regular file ‘/data/test.log’? y

37) Проверим
[root@ubuntu2 ~]# ll /data
total 16
drwx------. 2 root root 16384 Jan 28 11:44 lost+found

38) Откатимся из снапшота
[root@ubuntu2 ~]# umount /data
[root@ubuntu2 ~]# lvconvert --merge /dev/otus/test-snap
  Merging of volume otus/test-snap started.
  otus/test: Merged: 100.00%
[root@ubuntu2 ~]# mount /dev/otus/test /data
[root@ubuntu2 ~]# ll /data
total 8068564
drwx------. 2 root root      16384 Jan 28 11:44 lost+found
-rw-r--r--. 1 root root 8262189056 Jan 28 14:33 test.log

39) LVM зеркалирование
[root@ubuntu2 ~]# pvcreate /dev/sd{d,e}
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
[root@ubuntu2 ~]# vgcreate vg0 /dev/sd{d,e}
  Volume group "vg0" successfully created
[root@ubuntu2 ~]# lvcreate -l+80%FREE -m1 -n mirror vg0
  Logical volume "mirror" created.
[root@ubuntu2 ~]# lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  small    otus       -wi-a----- 100.00m
  test     otus       -wi-ao----  10.00g
  mirror   vg0        rwi-a-r--- 816.00m                                    100.00

40) Подготовим временный том для / раздела:
[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
WARNING: ext4 signature detected on /dev/vg_root/lv_root at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/vg_root/lv_root.
  Logical volume "lv_root" created.

41) Создадим на нем файловую систему и смонтируем его
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# mount /dev/vg_root/lv_root /mnt

42) Скопируем все данные с / раздела в /mnt:
[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsrestore: Restore Status: SUCCESS

43) Убедимся, что скопировалось:
[root@lvm ~]# ls /mnt
bin   data       dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
boot  data-snap  etc  lib   media  opt  root  sbin  sys  usr  var

44) Симитируем текущий root -> сделаем в него chroot и обновим grub:
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt/

[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

45) Обновим образ initrd
[root@lvm boot]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

46)Для того, чтобы при загрузке был смонтирован нужный root нужно в файле
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

47)После перезагрузки убедимся, что загружаемся с нового рут тома:
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
??sda1                    8:1    0    1M  0 part
??sda2                    8:2    0    1G  0 part /boot
??sda3                    8:3    0   39G  0 part
  ??VolGroup00-LogVol00 253:1    0 37.5G  0 lvm
  ??VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
??vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

48) Удаляем старый LV размером в 40G и создаем новый на 8G:
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed

[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.

49) Копируем данные:
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt

[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
xfsrestore: Restore Status: SUCCESS

50) Переконфигурируем grub
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

[root@lvm ~]# chroot /mnt/

[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done

51) На свободных дисках создаем зеркало:
[root@lvm ~]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.

[root@lvm ~]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created

[root@lvm ~]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  No input from event server.
  Logical volume "lv_var" created.

52) Создаем на нем ФС и перемещаем туда /var:
[root@lvm ~]# mkfs.ext4 /dev/vg_var/lv_var
Writing superblocks and filesystem accounting information: done

[root@lvm ~]# mount /dev/vg_var/lv_var /mnt

[root@lvm ~]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/

54) сохран?ем содержимое старого var
[root@lvm ~]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

55) монтируем нов?й var в каталог /var:
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/vg_var/lv_var /var

56) Правим fstab дл? автоматического монтировани? /var:
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

57) Выделяем том под /home по тому же принципу что делали для /var:
[root@lvm /]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.

[root@lvm /]# mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm /]# mount /dev/VolGroup00/LogVol_Home /mnt/

[root@lvm /]# cp -aR /home/* /mnt/

[root@lvm /]# rm -rf /home/*

[root@lvm /]# umount /mnt

[root@lvm /]# mount /dev/VolGroup00/LogVol_Home /home/

58)Правим fstab дл? автоматического монтировани? /home
[root@lvm /]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

59) Сгенерируем файл? в /home/:
[root@lvm /]# touch /home/file{1..20}

60) Сн?т? снапшот:
[root@lvm /]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.

[root@lvm /]# dir /home
file1   file12  file15  file18  file20  file5  file8
file10  file13  file16  file19  file3   file6  file9
file11  file14  file17  file2   file4   file7  vagrant

61) Удалит? част? файлов:
[root@lvm /]# rm -f /home/file{11..20}

[root@lvm /]# dir /home
file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant

62) Процесс восстановлени? со снапшота:
[root@lvm /]# umount /home

[root@lvm /]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%

[root@lvm /]# mount /home

[root@lvm /]# dir /home
file1   file12  file15  file18  file20  file5  file8
file10  file13  file16  file19  file3   file6  file9
file11  file14  file17  file2   file4   file7  vagrant
