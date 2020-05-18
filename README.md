## **LVM**

При выполнении задания использовалось следующее ПО:

####**Хост**
** ОС - Linux Mint 19.3 **
** Гипервизор - VirtualBox 6.1.6**
** Средство для создания и конфигурирования виртуальной среды - Vagrant 2.2.7 **
** Создание образа виртуальной машины - Packer 1.5.5 **
** Система контроля версий -  Git 2.26.2 **

####**Виртуальная машина**
** ОС - CentOS Linux release 7.5.1804**

#### **Установка ПО на виртуальной машине**

При разворачивании конфигурации виртуальной машины  в секции `box.vm.provision` выполняется установка необходимых для выполнения задания пакетов

```
box.vm.provision "shell", inline: <<-SHELL
          yum install -y mdadm smartmontools hdparm gdisk lvm2 xfsdump
        SHELL
```

#### **Создание LVM**

Для начала при помощи утилиты `lsblk`определяем имеющиеся в конфиграции машины диски

```
[vagrant@lvm ~]$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk 
└─sda1   8:1    0  40G  0 part /
sdb      8:16   0  10G  0 disk 
sdc      8:32   0   2G  0 disk 
sdd      8:48   0   1G  0 disk 
sde      8:64   0   1G  0 disk 
```

Создаём `physical volume` на диске `/dev/sdb` для дальнейшей работы LVM с диском

```
[vagrant@lvm ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

Затем, включим созданное устройство в `volume group`c именем qwerty, которая объединяет 

```
vagrant@lvm ~]$ sudo vgcreate qwerty /dev/sdb
  Volume group "qwerty" successfully created
```

И в итоге создадим `logical volume` логическое устройство которое уже может быть отформатировано и использовано для хранения данных

```
[vagrant@lvm ~]$ sudo lvcreate -l+80%FREE -n disk1 qwerty
  Logical volume "disk1" created.
```

Просмотреть информацию о созданном `volume group` можно с помощью команды `vgdisplay qwerty`

```
[vagrant@lvm ~]$ sudo vgdisplay qwerty
  --- Volume group ---
  VG Name               qwerty
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
  VG UUID               rX3uG8-YngE-jOMd-PtQD-QWCG-GtxE-12ggrc

```

А информацию о физических носителях включенных в `volume group` с помощью команды `vgdisplay -v`

```
[vagrant@lvm ~]$ sudo vgdisplay -v qwerty | grep 'PV Name'
  PV Name               /dev/sdb

```

Дополнительную информацию можно можно получить с помощью команды `lvdisplay`

```
[vagrant@lvm ~]$ sudo lvdisplay /dev/qwerty/disk1
  --- Logical volume ---
  LV Path                /dev/qwerty/disk1
  LV Name                disk1
  VG Name                qwerty
  LV UUID                38lul0-xh0X-bK9c-DbtF-X3vA-wtdq-mA7u7M
  LV Write Access        read/write
  LV Creation host, time lvm, 2020-05-13 13:57:10 +0000
  LV Status              available
  # open                 0
  LV Size                <8.00 GiB
  Current LE             2047
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

Создадим из нерамеченной области диска `dev/sdb` раздел в абсолютных значениях, размером 100 Мб с именем 100 в `volume group` qwerty

```
[vagrant@lvm ~]$ sudo lvcreate -L100M -n 100M qwerty
  Logical volume "100M" created.

```

Теперь отформатируем раздел созданный ранее `disk1` при помощи утилиты `mkfs.ext4`

```
[vagrant@lvm ~]$ sudo mkfs.ext4 /dev/qwerty/disk1 
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

```

Создадим в директории `mnt` поддиректорию `data` для монтирования для этого диска и смонтируем её

```
[vagrant@lvm ~]$ sudo mkdir /mnt/data
[vagrant@lvm ~]$ sudo mount /dev/qwerty/disk1 /mnt/data/
[vagrant@lvm ~]$ sudo mount |grep data 
/dev/mapper/qwerty-disk1 on /mnt/data type ext4 (rw,relatime,seclabel,data=ordered)
```



#### **Расширение LVM**

При нехватке свободного места в директории `/mnt/data` мы можем расширить объем в `logical volume` `/dev/qwerty/disk1` за счет нового диска `/dev/sdc`

Создаём на нем `phisycal volume `

```
[vagrant@lvm ~]$ sudo pvcreate /dev/sdc
  Physical volume "/dev/sdc" successfully created.
```

При помощи команды `vgextend` добавляем его в группу `qwerty`

```
[vagrant@lvm ~]$ sudo vgextend qwerty /dev/sdc
  Volume group "qwerty" successfully extended

```

Убеждаемся что в группе появился второй диск

```
[vagrant@lvm ~]$ sudo vgdisplay -v qwerty | grep 'PV Name'
  PV Name               /dev/sdb
  PV Name               /dev/sdc
```

Проверяем объем 

```
[vagrant@lvm ~]$ sudo vgs
  VG     #PV #LV #SN Attr   VSize  VFree 
  qwerty   2   2   0 wz--n- 11.99g <3.90g
```

Смоделируем нехватку свободного места в директории при помощи утилиты `dd`

```
[vagrant@lvm ~]$ sudo dd if=/dev/zero of=/mnt/data/test.log bs=1M count=8000 status=progress
8237613056 bytes (8.2 GB) copied, 118.312415 s, 69.6 MB/s
dd: error writing ‘/mnt/data/test.log’: No space left on device
7880+0 records in
7879+0 records out
8262189056 bytes (8.3 GB) copied, 118.651 s, 69.6 MB/s
```

Занято 100% пространства:

```
[vagrant@lvm ~]$ df -Th /mnt/data
Filesystem               Type  Size  Used Avail Use% Mounted on
/dev/mapper/qwerty-disk1 ext4  7.8G  7.8G     0 100% /mnt/data
```

Увеличим объем на 80% от имеющегося запаса

```
[vagrant@lvm ~]$ sudo lvextend -l+80%FREE /dev/qwerty/disk1
  Size of logical volume qwerty/disk1 changed from <8.00 GiB (2047 extents) to <11.12 GiB (2846 extents).
  Logical volume qwerty/disk1 successfully resized.
```

`logical volume` расширен до 11.12g:

```
[vagrant@lvm ~]$ sudo lvs /dev/qwerty/disk1
  LV    VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  disk1 qwerty -wi-ao---- <11.12g
```
 
  
Теперь необходимо произвести расширение раздела при помощи утилиты `resize2fs`


```
[vagrant@lvm ~]$ df -Th /mnt/data/
Filesystem               Type  Size  Used Avail Use% Mounted on
/dev/mapper/qwerty-disk1 ext4   11G  7.8G  2.6G  76% /mnt/data
```

В результате раздел расширен.



#### **Уменьшение LVM**


Для уменьшения раздела его необходимо отмонтировать

```
[vagrant@lvm ~]$ sudo umount /mnt/data/
```

И проверить на отсутствие ошибок

```
[vagrant@lvm ~]$ sudo e2fsck -fy /dev/qwerty/disk1 
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/qwerty/disk1: 12/729088 files (0.0% non-contiguous), 2105907/2914304 blocks
```

Уменьшаем размер размеченной области до 10 гигибайт

```
[vagrant@lvm ~]$ sudo resize2fs /dev/qwerty/disk1 10G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/qwerty/disk1 to 2621440 (4k) blocks.
The filesystem on /dev/qwerty/disk1 is now 2621440 blocks long.

```

После этого уменьшаем размер `logical volume` до 10 гигабайт

```
[vagrant@lvm ~]$ sudo lvreduce /dev/qwerty/disk1 -L 10G
  WARNING: Reducing active logical volume to 10.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce qwerty/disk1? [y/n]: y
  Size of logical volume qwerty/disk1 changed from <11.12 GiB (2846 extents) to 10.00 GiB (2560 extents).
  Logical volume qwerty/disk1 successfully resized.
```

Примонтируем раздел обратно

```
[vagrant@lvm ~]$ sudo mount /dev/qwerty/disk1 /mnt/data/
```

И проверим размер

```
[vagrant@lvm ~]$ df -Th /mnt/data/
Filesystem               Type  Size  Used Avail Use% Mounted on
/dev/mapper/qwerty-disk1 ext4  9.8G  7.8G  1.6G  84% /mnt/data
```

#### **Создание снапшотов на LVM**

Создадим снапшот при помощи утилиты  `lvcreate` с параметром `-s`

```
[vagrant@lvm ~]$ sudo lvcreate -L 500M -s -n qwerty-snap /dev/qwerty/disk1
  Logical volume "qwerty-snap" created.
```

Проверим результат

```
[vagrant@lvm ~]$ sudo vgs -o +lv_size,lv_name | grep qwerty
  qwerty   2   3   1 wz--n- 11.99g <1.41g  10.00g disk1
  qwerty   2   3   1 wz--n- 11.99g <1.41g 100.00m 100M
  qwerty   2   3   1 wz--n- 11.99g <1.41g 500.00m disk1-snap

```

Создадим точку монтирования для снапшота

```
sudo mkdir /mnt/data-snap
```

Смонтируем снапшот

```
[vagrant@lvm ~]$ sudo mount /dev/qwerty/disk1-snap /mnt/data-snap/

```

Проверим содержимое директорий в подключенном снапшоте


```
[vagrant@lvm ~]$ ll -l  /mnt/data-snap/
total 8068564
drwx------. 2 root root      16384 May 13 15:23 lost+found
-rw-r--r--. 1 root root 8262189056 May 13 15:53 test.log
```

Омонтируем снапшот

```
unmount /data-snap
```

Удалим из директории /mnt/data а затем восстановим файл из снапшота

```
[vagrant@lvm data]$ sudo rm test.log

[vagrant@lvm ~]$ cd /mnt/data/
[vagrant@lvm data]$ ll -l
total 0


```

Отмонтируем каталог /mnt/data

```
umount /mnt/data
```

Объединим снимок с оригиналом

```
[vagrant@lvm data]$ sudo lvconvert --merge /dev/qwerty/disk1-snap 
  Merging of volume qwerty/disk1-snap started.
  qwerty/disk1: Merged: 100.00%
```
Проверяем результат

```
[vagrant@lvm data]$ sudo touch test.log
[vagrant@lvm data]$ ll -l
total 0
-rw-r--r--. 1 root root 0 May 14 13:04 test.log
```

#### **Зеркалирование на LVM**

Создадим новый `physical volume` из имеющихся у нас двух дисков

```
[vagrant@lvm data]$ sudo  pvcreate /dev/sd{d,e}
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
```

Создадим `volume group` `vg1`

```
[vagrant@lvm data]$ sudo  pvcreate /dev/sd{d,e}
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
[vagrant@lvm data]$ sudo vgcreate vg1 /dev/sd{d,e}
  Volume group "vg1" successfully created
```

Создаем зеркалированный логический том 

```
[vagrant@lvm data]$ sudo  lvcreate -l+80%FREE -m1 -n mirror vg1
  Logical volume "mirror" created.
  
```


## **Домашнее задание**

#### **Уменьшение / до 8 Gb**

Уменьшим раздел `/` c 37,5 Gb до 8 Gb

Для этого создадим `physical volume` на диске `/dev/sdb`

```
[vagrant@lvm ~]$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

Создадим на нём `volume group` с названием `vg_root`

```
[vagrant@lvm ~]$ sudo vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
```

Создадим радел `lv_root`

```
[vagrant@lvm ~]$ sudo lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```

Отформатируем созданный раздел

```
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
```

Примонтируем раздел в директорию `mnt/root`

```
[vagrant@lvm ~]$ sudo mkdir -p /mnt/root
[vagrant@lvm ~]$ sudo mount /dev/vg_root/lv_root /mnt/root
```

Создадим дамп 

```
[vagrant@lvm ~]$ sudo xfsdump -J - /dev/VolGroup00/LogVol00 | sudo xfsrestore -J - /mnt/root
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsrestore: searching media for dump
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Thu May 14 15:43:37 2020
...
xfsdump: ending media file
xfsdump: media file size 1438951272 bytes
xfsdump: dump size (non-dir files) : 1410558360 bytes
xfsdump: dump complete: 68 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 70 seconds elapsed
xfsrestore: Restore Status: SUCCESS

```

Проверяем результат

```
[vagrant@lvm root]$ sudo ls -l  /mnt/root/
total 12
lrwxrwxrwx.  1 root root    7 May 14 15:43 bin -> usr/bin
drwxr-xr-x.  2 root root    6 May 12  2018 boot
drwxr-xr-x.  2 root root    6 May 12  2018 dev
drwxr-xr-x. 80 root root 8192 May 14 15:42 etc
drwxr-xr-x.  3 root root   21 May 12  2018 home
lrwxrwxrwx.  1 root root    7 May 14 15:43 lib -> usr/lib
lrwxrwxrwx.  1 root root    9 May 14 15:43 lib64 -> usr/lib64
drwxr-xr-x.  2 root root    6 Apr 11  2018 media
drwxr-xr-x. 17 root root  224 May 14 15:40 mnt
drwxr-xr-x.  2 root root    6 Apr 11  2018 opt
drwxr-xr-x.  2 root root    6 May 12  2018 proc
dr-xr-x---.  5 root root  180 May 14 15:42 root
drwxr-xr-x.  2 root root    6 May 12  2018 run
lrwxrwxrwx.  1 root root    8 May 14 15:43 sbin -> usr/sbin
drwxr-xr-x.  2 root root    6 Apr 11  2018 srv
drwxr-xr-x.  2 root root    6 May 12  2018 sys
drwxrwxrwt.  9 root root  271 May 14 15:42 tmp
drwxr-xr-x. 13 root root  155 May 12  2018 usr
drwxr-xr-x. 18 root root  254 May 14 15:30 var
```
Переконфигуриреум grub

```
[vagrant@lvm root]$ sudo -i
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/root/$i; done
[root@lvm ~]# chroot /mnt/root/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

Обновляем образ initd

```
[root@lvm ~]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```
Заменяем в файле `/boot/grub2/grub.cfg` старый раздел с расположенным на нем разделом `/` c `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=vg_root/lv_root`

После перезагрузки проверим изменения

```
[vagrant@lvm ~]$ lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:1    0 37.5G  0 lvm 
  └─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

Теперь можно удадить старый раздел

```
[vagrant@lvm ~]$ sudo lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed

```

Создаем новый раздел с размером 8Gb

```
[vagrant@lvm ~]$ sudo lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```
Повторяем все те же операции что и для переноса на временный раздел

Форматируем раздел

[vagrant@lvm ~]$ sudo mkfs.xfs /dev/VolGroup00/LogVol00

```
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

Монтируем раздел `/` в папку `/mnt/root`

```
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol00/ /mnt/root/
```

Создаем дамп раздела `/`

```
[vagrant@lvm ~]$ sudo xfsdump -J - /dev/vg_root/lv_root | sudo xfsrestore -J - /mnt/root/
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Thu May 14 17:14:26 2020
...
xfsdump: ending media file
xfsdump: media file size 694454568 bytes
xfsdump: dump size (non-dir files) : 681238856 bytes
xfsdump: dump complete: 16 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 17 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
Переконфигурируем grub

```
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/root/$i; done
[root@lvm ~]# chroot /mnt/root/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
```

После перезагрузки проверим результат. Раздел `/` с размером в 8 Gb

```
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

#### **Перенос /var на зеркалированный раздел**

Из двух свободных дисков в системе создадим два новых `physical volume`

```
[vagrant@lvm ~]$ sudo pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

Создадим новую `volume group` с названием `vg_var` и добавим в нее созданые на предыдущем щаге `physical volume`

```
[vagrant@lvm ~]$ sudo vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
```

Создадим новый `logical volume` размером 950 MB с зеркалирование и именем `lv_var` расположеном в `volume group ` `vg_var`

```
[vagrant@lvm ~]$ sudo lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```

Отформатируем созданный раздел

```
[vagrant@lvm ~]$ sudo mkfs.ext4 /dev/vg_var/lv_var 
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
```

Создадим директорию `/mnt/var` и смонтируем в неё новый раздел

```
[vagrant@lvm ~]$ sudo mkdir -p /mnt/var
[vagrant@lvm ~]$ sudo mount /dev/vg_var/lv_var /mnt/var/
```

Скопируем содержимое каталога в созданную директорию

```
[vagrant@lvm ~]$ sudo cp -aR /var/* /mnt/var/
```

Проверим результат

```
[varant@lvm ~]$ ls -l /mnt/var
total 80
drwxr-xr-x.  2 root root  4096 Apr 11  2018 adm
drwxr-xr-x.  5 root root  4096 May 12  2018 cache
drwxr-xr-x.  3 root root  4096 May 12  2018 db
drwxr-xr-x.  3 root root  4096 May 12  2018 empty
drwxr-xr-x.  2 root root  4096 Apr 11  2018 games
drwxr-xr-x.  2 root root  4096 Apr 11  2018 gopher
drwxr-xr-x.  3 root root  4096 May 12  2018 kerberos
drwxr-xr-x. 28 root root  4096 May 14 17:14 lib
drwxr-xr-x.  2 root root  4096 Apr 11  2018 local
lrwxrwxrwx.  1 root root    11 May 14 17:14 lock -> ../run/lock
drwxr-xr-x.  8 root root  4096 May 14 16:49 log
drwx------.  2 root root 16384 May 14 17:34 lost+found
lrwxrwxrwx.  1 root root    10 May 14 17:14 mail -> spool/mail
drwxr-xr-x.  2 root root  4096 Apr 11  2018 nis
drwxr-xr-x.  2 root root  4096 Apr 11  2018 opt
drwxr-xr-x.  2 root root  4096 Apr 11  2018 preserve
lrwxrwxrwx.  1 root root     6 May 14 17:14 run -> ../run
drwxr-xr-x.  8 root root  4096 May 12  2018 spool
drwxrwxrwt.  4 root root  4096 May 14 17:23 tmp
drwxr-xr-x.  2 root root  4096 Apr 11  2018 yp
```

Перемещаем содержимое старого каталога `/var`

```
[vagrant@lvm ~$ sudo mkdir /tmp/oldvar && sudo mv /var/* /tmp/oldvar
```

Отмонтируем старый раздел и монтируем новый

```
[vagrant@lvm ~]$ sudo umount /mnt/var/
[vagrant@lvm ~]$ sudo  mount /dev/vg_var/lv_var /var/
```

Добавляем новый раздел в `/etc/fstab`

```
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

```

Перезагружаемся


Удаляем `logical volume` `lv_root`

```
[vagrant@lvm ~]$ sudo lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed

```

Удаляем `volume group` `vg_root`

```
[vagrant@lvm ~]$ sudo vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
```

Удаляем `physical volume`

```
[vagrant@lvm ~]$ sudo pvremove /dev/sdb
 Labels on physical volume "/dev/sdb" successfully wiped.
```

Проверяем результат

```
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
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
```

#### **Перенос /home на другой раздел**


Создадим `logical volume`  размером в 2Gb с именем`LogVol_Home`

```
[vagrant@lvm ~]$ sudo lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.

```

Отформатируем его

```
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
```

Создадим директорию `mnt/home` и примонтируем к ней раздел

```
[vagrant@lvm ~]$ sudo mkdir -p /mnt/home
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /mnt/home/
```

Скопируем содержимое директории `home` в созданую директорию `/mnt/home/`

```
[vagrant@lvm ~]$ sudo cp -aR /home/* /mnt/home/
```

Проверим результат

```
[vagrant@lvm ~]$ ls -l /mnt/home/
total 0
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```

Удаляем содежимое  директории `home`

```
[vagrant@lvm ~]$ sudo rm -rf /home/*
```

Отмонтируем директорию

```
[vagrant@lvm ~]$ sudo umount /mnt/home/
```

Монтируем новый раздел к директории `home`

```
[vagrant@lvm ~]$ sudo mount /dev/VolGroup00/LogVol_Home /home/
```

Добавляем новый раздел в `/etc/fstab`

```
 [vagrant@lvm ~]$ sudo -i
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Проверяем результат

```
[vagrant@lvm ~]$ lsblk 
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
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sdd                          8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
```

#### **Создание тома для снапштов в /home**

Создадим файлы для проверки

```
[vagrant@lvm ~]$ touch /home/vagrant/file{1..20}
```

Создадим снапшот размером 100MB и именем `home_snap`

```
[vagrant@lvm home]$ sudo lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
```

Удалим половину из созданных файлов

```
[vagrant@lvm home]$ rm -f /home/vagrant/file{1..10}
[vagrant@lvm ~]$ ls -l /home/vagrant
total 0
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file11
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file12
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file13
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file14
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file15
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file16
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file17
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file18
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file19
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file20
```

Восстановим удаленные файлы при помощи снапшота Отмонтриуем директрию `home`


```
[vagrant@lvm /]$ sudo umount /home
```

Воссстановим состояние директории на момент создания снапшота

```
[vagrant@lvm /]$ sudo lvconvert --merge /dev/VolGroup00/home_snap 
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
```

Монтируем раздел `home`

```
[vagrant@lvm /]$ sudo mount /home/
```

Проверяем результат

```
[vagrant@lvm /]$ ls -l /home/vagrant
total 0
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file1
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file10
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file11
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file12
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file13
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file14
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file15
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file16
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file17
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file18
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file19
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file2
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file20
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file3
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file4
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file5
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file6
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file7
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file8
-rw-rw-r--. 1 vagrant vagrant 0 May 15 13:28 file9
```

