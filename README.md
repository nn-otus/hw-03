## ДЗ-03 Работа с LVM

На виртуальной машине с Ubuntu 24.04 и LVM. 
* Уменьшить том под / до 8G.
* Выделить том под /home.
* Выделить том под /var - сделать в mirror.
* /home - сделать том для снапшотов.
* Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
* Работа со снапшотами:
* сгенерить файлы в /home/;
* снять снапшот;
* удалить часть файлов;
* восстановиться со снапшота.
** На дисках попробовать поставить btrfs/zfs — с кэшем, снапшотами и разметить там каталог /opt.

```
root@u24srv02:~# vgs
  VG        #PV #LV #SN Attr   VSize  VFree
  ubuntu-vg   1   1   0 wz--n- 18.22g 8.22g

root@u24srv02:~# echo "### Уменьшить том / до 8GB"
```
### Уменьшить том / до 8GB
```
root@u24srv02:~# pvcreate /dev/sdb
WARNING: dos signature detected on /dev/sdb at offset 510. Wipe it? [y/n]: y
  Wiping dos signature on /dev/sdb.
  Physical volume "/dev/sdb" successfully created.
root@u24srv02:~#
root@u24srv02:~# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
root@u24srv02:~#
root@u24srv02:~# lvcreate --name lv_root --size +8G /dev/vg_root
WARNING: LVM2_member signature detected on /dev/vg_root/lv_root at offset 536. Wipe it? [y/n]: Y
  Wiping LVM2_member signature on /dev/vg_root/lv_root.
  Logical volume "lv_root" created.
root@u24srv02:~#
root@u24srv02:~# vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n-  18.22g  8.22g
  vg_root     1   1   0 wz--n- <10.00g <2.00g
root@u24srv02:~# echo "### Создаем файловую систему на lv_root и монтируем, чтобы перенести данные с / "
```
### Создаем файловую систему на lv_root и монтируем, чтобы перенести данные с /
```
root@u24srv02:~# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: 10beefcb-bcf2-4763-a108-18f1709319bd
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

root@u24srv02:~# mount /dev/vg_root/lv_root /mnt
```
### Копируем все данные с раздела / на /mnt
```
root@u24srv02:~#
root@u24srv02:~# rsync -avxHAX --progress / /mnt/
...
sent 4,240,669,166 bytes  received 1,170,976 bytes  34,071,005.16 bytes/sec
total size is 4,239,810,489  speedup is 1.00
root@u24srv02:~#
nn@u24srv02:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk
+-sda1                      8:1    0    1M  0 part
+-sda2                      8:2    0  1.8G  0 part /boot
L-sda3                      8:3    0 18.2G  0 part
  L-ubuntu--vg-ubuntu--lv 252:2    0   10G  0 lvm  /
sdb                         8:16   0   10G  0 disk
L-vg_root-lv_root         252:0    0    8G  0 lvm  /mnt
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
nn@u24srv02:~$ ls /mnt
bin                boot   dev  home  lib.usr-is-merged  lost+found  mnt  proc  root  sbin                snap  swap.img  tmp  var
bin.usr-is-merged  cdrom  etc  lib   lib64              media       opt  raid  run   sbin.usr-is-merged  srv   sys       usr
nn@u24srv02:~$
```
### Сымитируем текущий root, сделаем в него chroot и обновим grub
```
nn@u24srv02:~$ su -
Password:
root@u24srv02:~# for i in /boot/ /dev/ /proc/ /sbin/ /etc/ /lib/ /lib64/ /opt/ /run/ /sys/ /usr/; do mount --bind $i /mnt/$i; done
root@u24srv02:~#
root@u24srv02:~# chroot /mnt/
root@u24srv02:/#
root@u24srv02:/# grub-mkconfig --output /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-63-generic
Found initrd image: /boot/initrd.img-6.8.0-63-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
```
### Обновим образ initrd
```
root@u24srv02:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-63-generic
root@u24srv02:/#exit
root@u24srv02:/#reboot
...
root@u24srv02:/#
nn@u24srv02:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0 18.2G  0 part
  └─ubuntu--vg-ubuntu--lv 252:1    0   10G  0 lvm
sdb                         8:16   0   10G  0 disk
└─vg_root-lv_root         252:0    0    8G  0 lvm  /
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
```
### Изменяем размер исходной volume group c 20GB на 8GB
```
nn@u24srv02:~$ su -
Password:
root@u24srv02:~# lvremove /dev/ubuntu-vg/ubuntu-lv
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.
root@u24srv02:~#
root@u24srv02:~# lvcreate --name ubuntu-vg/ubuntu-lv --size +8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.
root@u24srv02:~# echo "Повторяем те же команды, что и для vg_root/lv_root"
```
#### Повторяем те же команды, что и для vg_root/lv_root
```
root@u24srv02:~#
root@u24srv02:~#
root@u24srv02:~# mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: ffad6b7d-b547-4fa4-90ae-99bee9e8f0d3
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

root@u24srv02:~# mount /dev/ubuntu-vg/ubuntu-lv /mnt
root@u24srv02:~#
root@u24srv02:~# rsync -avxHAX --progress / /mnt/
...
sent 4,265,847,707 bytes  received 1,171,014 bytes  72,940,490.96 bytes/sec
total size is 4,264,981,113  speedup is 1.00
root@u24srv02:~#
root@u24srv02:~# for i in /boot/ /dev/ /proc/ /sbin/ /etc/ /lib/ /lib64/ /opt/ /run/ /sys/ /usr/; do mount --bind $i /mnt/$i; done
root@u24srv02:~#
root@u24srv02:~# chroot /mnt/
root@u24srv02:/#
root@u24srv02:/# grub-mkconfig --output /boot/grub/grub.cfg
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-63-generic
Found initrd image: /boot/initrd.img-6.8.0-63-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
root@u24srv02:/#
root@u24srv02:/# update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-63-generic
W: Couldn't identify type of root file system for fsck hook
root@u24srv02:/#
```
### Выделить том под /var в зеркало

```
root@u24srv02:/# pvcreate /dev/sd{c,d}
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
root@u24srv02:/#
root@u24srv02:/# vgcreate vg_var /dev/sd{c,d}
  Volume group "vg_var" successfully created
root@u24srv02:/#
root@u24srv02:/# lvcreate --size 950M --mirrors 1 --name lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
root@u24srv02:/#
```
#### Создаем на нем ФС и перемещаем туда /var:
```
root@u24srv02:/#
root@u24srv02:/#
root@u24srv02:/# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 243712 4k blocks and 60928 inodes
Filesystem UUID: 4827bd8f-acbe-4746-bcc7-3109f3ee4671
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@u24srv02:/# mount /dev/vg_var/lv_var /mnt
root@u24srv02:/# cp -aR /var/* /mnt/
root@u24srv02:/#
root@u24srv02:/# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
root@u24srv02:/# ls /tmp/oldvar/
backups  cache  crash  lib  local  lock  log  mail  opt  run  snap  spool  tmp
root@u24srv02:/#
```
#### монтируем новый var в каталог /var:
```
root@u24srv02:/# umount /mnt
root@u24srv02:/# mount /dev/vg_var/lv_var /var
```
#### Правим fstab для автоматического монтирования /var:~

_Узнать UUID можно командой blkid_
```
root@u24srv02:/# blkid | grep lv_var
/dev/mapper/vg_var-lv_var: UUID="4827bd8f-acbe-4746-bcc7-3109f3ee4671" BLOCK_SIZE="4096" TYPE="ext4"
/dev/mapper/vg_var-lv_var_rimage_1: UUID="4827bd8f-acbe-4746-bcc7-3109f3ee4671" BLOCK_SIZE="4096" TYPE="ext4"
/dev/mapper/vg_var-lv_var_rimage_0: UUID="4827bd8f-acbe-4746-bcc7-3109f3ee4671" BLOCK_SIZE="4096" TYPE="ext4"
root@u24srv02:/#
root@u24srv02:/# echo "UUID="4827bd8f-acbe-4746-bcc7-3109f3ee4671"  /var ext4 defaults 0 0"  >> /etc/fstab
root@u24srv02:/#
```
#### Удаляем временную lv_root
```
root@u24srv02:/# lvremove /dev/vg
vg_root/     vg_var/      vga_arbiter
root@u24srv02:/# lvremove /dev/vg_root/lv_root
  Logical volume vg_root/lv_root contains a filesystem in use.
root@u24srv02:/# exit
exit
root@u24srv02:~# reboot
```
...

#### Удаляем временную группу lv_root
```
root@u24srv02:~# lvremove /dev/vg_root/lv_root
Do you really want to remove and DISCARD active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed.
root@u24srv02:~#
root@u24srv02:~# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
root@u24srv02:~#
root@u24srv02:~# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
root@u24srv02:~#
```
### Выделить том под /home
```
root@u24srv02:~#
root@u24srv02:~# lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
  Logical volume "LogVol_Home" created.
root@u24srv02:~# mkfs.ext4 /dev/ubuntu-vg/LogVol_Home
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: 45c393b1-899a-4142-a8e3-5af955fb9ca3
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

root@u24srv02:~# mount /dev/ubuntu-vg/LogVol_Home /mnt/
root@u24srv02:~# cp -aR /home/* /mnt/
root@u24srv02:~# rm -rf /home/*
root@u24srv02:~# umount /mnt
root@u24srv02:~# mount /dev/ubuntu-vg/LogVol_Home /home/
```
#### Правим fstab для автоматического монтирования /home
```
root@u24srv02:~# echo "`blkid | grep Home | awk '{print $2}'` \
 /home xfs defaults 0 0" >> /etc/fstab
root@u24srv02:~# cat /etc/fstab | grep home
UUID="45c393b1-899a-4142-a8e3-5af955fb9ca3"  /home xfs defaults 0 0
root@u24srv02:~#
```
### Работа со снапшотами

Генерируем файлы в /home/
```
root@u24srv02:~#
root@u24srv02:~# touch /home/file{1..20}
```
#### Снять снапшот
```
root@u24srv02:~# lvcreate -L 100MB -s -n home_snap \
 /dev/ubuntu-vg/LogVol_Home
  Logical volume "home_snap" created.
```
#### Удалить часть файлов
```
root@u24srv02:~# rm -f /home/file{11..20}
root@u24srv02:~# echo "Процесс восстановления из снапшота"
```
#### Восстановление из снапшота
```
root@u24srv02:~# umount /home
root@u24srv02:~# lvconvert --merge /dev/ubuntu-vg/home_snap
  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/LogVol_Home: Merged: 100.00%

root@u24srv02:~#
root@u24srv02:~# mount /dev/mapper/ubuntu--vg-LogVol_Home /home
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
root@u24srv02:~# systemctl daemon-reload
root@u24srv02:~# ls  /home
file1  file10  file11  file12  file13  file14  file15  file16  file17  file18  file19  file2  file20  file3  file4  file5  file6  file7  file8  file9  lost+found  nn
root@u24srv02:~#
```
#### Проверяем отмонтированием
```
root@u24srv02:~# umount /home/
root@u24srv02:~# mount /dev/mapper/ubuntu--vg-LogVol_Home /home
rroot@u24srv02:~# ls  /home
file1  file10  file11  file12  file13  file14  file15  file16  file17  file18  file19  file2  file20  file3  file4  file5  file6  file7  file8  file9  lost+found  nn
root@u24srv02:~#
```
#### Файлы успешно восстановлены с помощью снапшота
## Домашнее задание выполнено













