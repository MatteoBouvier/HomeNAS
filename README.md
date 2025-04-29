# HomeNAS

Mini homelab NAS setup based on Raspberry Pi 5.

## Material

### Raspberry Pi
- [x] Raspberry Pi 5
- [x] power supply
- [x] heat sink : Geekworm H505
- [x] micro SD card 64 GB

### SSD PCIe hat
- [x] GeeekPi N16 Quad M-key (52PI EP-0180)
- [x] heat sink 14x14mm

### SSD drives
- [ ] 4x SSD : 
    - [x] 2x Samsung 990 EVO [73€] (https://www.cdiscount.com/informatique/ssd/samsung-ssd-interne-990-evo-1-to-mz-v9e1t0bw/f-10703-sam8806095300276.html#mpos=0|cd)
    - [x] WD Black SN770 [73€] (https://www.cdiscount.com/informatique/ssd/disque-ssd-interne-sn770-nvme-wd_black-1-to/f-10703-wds100t3x0e.html?idOffre=3874422962#mpos=0|mp)

Host Memory Buffer : to avoid including a DRAM chip on the SSD but to keep high performance, some host RAM is used as cache
- https://unix.stackexchange.com/questions/681131/how-to-check-change-nvme-hmb-on-linux

## Wiki
- Geekworm X1011 : https://wiki.geekworm.com/X1011 
- 52PI EP-0180 : https://wiki.52pi.com/index.php?title=EP-0180#Specifications
- SD speed guide : https://www.kingston.com/en/blog/personal-storage/memory-card-speed-classes

## Operating System
Raspberry Pi OS Lite


## File system
File system : ZFS https://wiki.archlinux.org/title/ZFS
doc : https://www.socallinuxexpo.org/scale11x-supporting/default/files/presentations/zfs.pdf

Root on ZFS : https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Bookworm%20Root%20on%20ZFS.html

### Layout
--> ZFS RAIDz striped mirrors 2x2 (pool of 2 mirror vdevs of 2 disks (1 Samsung, 1 WD in each vdev))

/
    boot/              0.5GB
    home/              -> /data/home
    ...
    data/              2To striped mirrors
        home/
        store/

#### Root
https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Bookworm%20Root%20on%20ZFS.html

```shell
apt install --yes gdisk zfsutils-linux
DISK=/dev/disk/by-id/mmc-SR64G_0x4d30da99
swapoff --all
rm /var/swap
```

Partition the disk
```shell
sgdisk --zap-all $DISK
sgdisk -n1:0:512M -t1:EF00 $DISK -c1:BOOT         <-- boot partition
sgdisk -n2:0:0 -t2:8305 $DISK -c2:ROOT            <-- root partition

mkfs -t vfat $DISK-part1                          <-- boot on RPi requires vFAT partition
```

Create the root ZFS pool
```shell
zpool create \
    -o ashift=12 \
    -o autotrim=on \
    -O encryption=on -O keylocation=prompt -O keyformat=passphrase \
    -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
    -O compression=zstd \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=/ -R /mnt \
    rpool ${DISK}-part2
```

Create filesystems (mounted at /media)
```shell
zfs list -o name,mountpoint,canmount,com.sun:auto-snapshot
NAME                MOUNTPOINT          CANMOUNT    COM.SUN:AUTO-SNAPSHOT
rpool               /mnt                off         -
rpool/ROOT          none                off         -
rpool/ROOT/raspi    /mnt                noauto      -
rpool/boot          /mnt/boot           on          -
rpool/etc           /mnt/etc            on          -
rpool/home          /mnt/home           on          -
rpool/home/root     /mnt/root           on          -
rpool/opt           /mnt/opt            on          -
rpool/usr           /mnt/usr            off         -
rpool/usr/local     /mnt/usr/local      on          -
rpool/var           /mnt/var            off         -
rpool/var/cache     /mnt/var/cache      on          false
rpool/var/lib       /mnt/var/lib        off         -
rpool/var/lib/nfs   /mnt/var/lib/nfs    on          false
rpool/var/log       /mnt/var/log        off         -
rpool/var/spool     /mnt/var/spool      off         -
rpool/var/tmp       /mnt/var/tmp        on          false

# zpool import -a -R /mnt
zfs mount rpool/ROOT/raspi
zfs mount -al
chmod 700 /mnt/root
chmod 1777 /mnt/var/tmp

mount -m -t tmpfs tmpfs /mnt/run
mkdir /mnt/run/lock

mount -m -t tmpfs tmpfs /mnt/tmp

mount -m $DISK-part1 /mnt/boot/firmware
```

Copy files to new filesystem
```shell
(cd /; sudo tar cf - --one-file-system . ) | pv -p -bs $( sudo du -sxm --apparent-size / | cut -f1 )m | (sudo tar -xp -C /mnt)

cp /etc/zfs/zpool.cache /mnt/etc/zfs/
```

Configuration
```shell
hostname NAS
hostname > /mnt/etc/hostname
vi /mnt/etc/hosts
127.0.1.1   NAS

# uncomment lines in /mnt/etc/apt/sources.list

# vi /mnt/etc/network/interfaces.d/eth0
# auto eth0
# iface eth0 inet dhcp

# remove / partition in /mnt/etc/fstab

echo "zfs_enable=YES" > /etc/rc.conf

mount --make-private --rbind /dev /mnt/dev
mount --make-private --rbind /proc /mnt/proc
mount --make-private --rbind /sys /mnt/sys
chroot /mnt /usr/bin/env bash --login
```

```shell
apt update
# apt install --yes console-setup locales
# fr_FR and en_US
# dpkg-reconfigure locales tzdata keyboard-configuration console-setup

# apt install --yes dpkg-dev linux-headers-generic linux-image-generic
apt install --yes zfs-initramfs zfsutils-linux
echo REMAKE_INITRD=yes > /etc/dkms/zfs.conf

# check zfs is in initramfs
lsinitramfs /boot/initrd.img-6.6.31+rpt-rpi-v8 | grep zfs

# apt install systemd-timesyncd

passwd
```

Boot config
```shell
vi /boot/firmware/cmdline.txt
console=serial0,115200 console=tty1 zfs=rpool/ROOT/raspi rootfstype=zfs rw rootwait plymouth.enable=0 cfg80211.ieee80211_regdom=FR
```

Mount tmpfs to /tmp
```shell
cp /usr/share/systemd/tmp.mount /etc/systemd/system/
systemctl enable tmp.mount
```

Create swap partition
https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-use-a-zvol-as-a-swap-device
```shell
zfs create 
    -V 10G \
    -b $(getconf PAGESIZE) \
    -o compression=zle \
    -o logbias=throughput \
    -o sync=standard \
    -o primarycache=metadata \
    -o secondarycache=none \
    -o com.sun:auto-snapshot=false \
    rpool/swap

mkswap -f /dev/zvol/rpool/swap

echo /dev/zvol/rpool/swap none swap defaults 0 0 >> /etc/fstab

swapon -av
```

Update fstab
```shell
```

First boot
```shell
zfs snapshot rpool/ROOT/raspi@install

exit

umount /mnt/run
umount /mnt/tmp
umount /mnt/boot/firmware

zfs unmount -a
zpool export -a
```


#### Data tank

```shell
DISK1=/dev/disk/by-id/nvme-Samsung_SSD_990_EVO_1TB_S7GCNS0X135336E
DISK2=/dev/disk/by-id/nvme-WD_BLACK_SN770_1TB_23523W804827
DISK3=/dev/disk/by-id/nvme-Samsung_SSD_990_EVO_1TB_S7GCNS0X135322D
DISK4=/dev/disk/by-id/nvme-WD_BLACK_SN770_1TB_2343GV403336

zpool create \
    -o ashift=12 \
    -O encryption=on -O keylocation=prompt -O keyformat=passphrase \
    -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
    -O compression=zstd \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=none \
    dpool mirror $DISK1 $DISK2 mirror $DISK3 $DISK4

zfs list -o name,mountpoint,canmount,com.sun:auto-snapshot
NAME            MOUNTPOINT   CANMOUNT  COM.SUN:AUTO-SNAPSHOT
data            none         off       -
data/ROOT       /data        on        -
data/home       /home        on        -
data/home/root  /root        on        -
data/store      /data/store  on        -
```

### Software
- neovim
- bat
- btop


### Misc
- second SD card https://ralimtek.com/posts/2016/2016-12-10-raspberry_pi_secondary_sd_card/
                 https://forums.raspberrypi.com/viewtopic.php?t=316829
- SATA hat https://shop.allnetchina.cn/products/dual-sata-hat-open-frame-for-raspberry-pi-4
- SAS expander


zpool mount trouble
```shell
mkdir /etc/zfs/zfs-list.cache
systemctl disable zfs-mount.service
systemctl enable zfs-zed.service
touch /etc/zfs/zfs-list.cache/rpool
zfs set canmount=off rpool/boot
zfs set canmount=on rpool/boot
```
