# Setup Guide

## WiFi

Enter iwctl

```
$ iwctl
```

When inside, check for the name of your wireless devices.

```
devices list
```

If your device name is wlan0, connect using the following command

```
station wlan0 connect <SSID>
```

Make sure to enter in your password

exit when complete

```
exit
```

## SSH

Enable sshd (should be done by default)

```
$ systemctl enable sshd
```

set a password for the current user

```
$ passwd
```

## Write random data

List blocks. In my case, my drives are nvme0n1 and nvme1n1. Your's might be the
same, or the might be an sdx drive, such as sda or sdb.

```
$ lsblk
```

Write random data into your drive. 

```
$ dd if=/dev/urandom of=/dev/nvme0n1 status=progress bs=4096
```

## Partitioning Data

Get the names of the blocks

```
$ lsblk
```

For both partition setups, you'll want to setup a table on your primary drive.

```
$ gdisk /dev/nvme0n1
```

Inside of gdisk, you can print the table using the `p` command.

To create a new partition use the `n` command. The below table shows 
the disk setup I have for my primary drive

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | +512M       | ef00 |
| 2         | default      | +4G         | ef02 |
| 3         | default      | default     | 8309 |

If you have a second drive for your home disk, then your table would be as 
follows.

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | default     | 8302 |

## Encryption

Load the encryption modules to be safe.

```
$ modprobe dm-crypt
$ modprobe dm-mod
```

Setting up encryption on our luks lvm partiton

```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p3
```

Enter in your password and **Keep it safe**. There is no "forgot password" here.


If you have a home partition, then initialize this as well

```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme1n1p1
```

Mount the drives:

```
$ cryptsetup open /dev/nvme0n1p3 luks_lvm
```

If you have a home parition:

```
$ cryptsetup open /dev/nvme1n1p1 arch-home
```

## Volume setup

Create the volume and volume group

```
$ pvcreate /dev/mapper/luks_lvm

$ vgcreate arch /dev/mapper/luks_lvm
```

Create a volume for your swap space. A good size for this is your disk space + 2GB.
In my case 66G.

```
$ lvcreate -n swap -L 66G arch
```

Next you have a few options depending on your setup

### Single Disk
If you have a single disk, you can either have a single volume for your root 
and home, or two separate volumes.

#### Single volume 

Single volume is the most straightforward. To do this, just use the entire
disk space for your root volume

```
$ lvcreate -n root -l +100%FREE arch
```

#### Two volumes

For two volumes, you'll need to estimate the max size you want for either your
root or home volumes. With a root volume of 200G, this looks like:

```
$ lvcreate -n root -L 200G arch
```

Then use remaining disk space for home

```
$ lvcreate -n home -l +100%FREE arch
```

### Dual Disk

If you have two disks, then create a single volume on your LVM disk.

```
$ lvcreate -n root -l +100%FREE arch
```


## Filesystems

FAT32 on EFI partiton

```
$ mkfs.fat -F32 /dev/nvme0n1p1 
```

EXT4 on Boot partiton

```
$ mkfs.ext4 /dev/nvme0n1p2
```

BTRFS on root

```
$ mkfs.btrfs -L root /dev/mapper/arch-root
```

BTRFS on home if exists

```
$ mkfs.btrfs -L home /dev/mapper/arch-home
```

Setup swap device

```
$ mkswap /dev/mapper/arch-swap
```

## Mounting

Mount swap

```
$ swapon /dev/mapper/arch-swap
$ swapon -a
```

Mount root 

```
$ mount /dev/mapper/arch-root /mnt
```

Create home and boot

```
$ mkdir -p /mnt/{home,boot}
```

Mount the boot partiton

```
$ mount /dev/nvme0n1p2 /mnt/boot
```

Mount the home partition if you have one, otherwise skip this

```
$ mount /dev/mapper/arch-home /mnt/home
```

Create the efi directory

```
$ mkdir /mnt/boot/efi
```

Mount the EFI directory

```
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Install arch

```
$ pacstrap -K /mnt base linux linux-firmware
```

With base-devel

```
$ pacstrap -K /mnt base base-devel linux linux-firmware
```

Load the file table

```
$ genfstab -U -p /mnt > /mnt/etc/fstab
```

chroot into your installation

```
$ arch-chroot /mnt /bin/bash
```

## Configuring

### Text Editor

Install a text editor

```
$ pacman -S neovim
```

```
$ pacman -S nano
```

### Decrypting volumes

Open up mkinitcpio.conf

```
$ nvim /etc/mkinitcpio.conf
```

add `encrypt` and `lvm2` into the hooks

```
HOOKS=(... block encrypt lvm2 filesystems fsck)
```

install lvm2

```
$ pacman -S lvm2
```

### Bootloader

Install grub and efibootmgr

```
$ pacman -S grub efibootmgr
```

Setup grub on efi partition

```
$ grub-install --efi-directory=/boot/efi
```

obtain your lvm partition device UUID

```
lsblk /dev/nvme0n1p3
```

Copy this to your clipboard

```
$ nvim /etc/default/grub
```

Add in the following kernel parameters

```
root=/dev/mapper/arch-root cryptdevice=UUID=<uuid>:luks_lvm
```

### Keyfile

```
$ mkdir /secure
```

Root keyfile
```
$ dd if=/dev/random of=/secure/root_keyfile.bin bs=512 count=8
```

Home keyfile if home partition exists

```
$ dd if=/dev/random of=/secure/home_keyfile.bin bs=512 count=8
```

Change permissions on these

```
$ chmod 000 /secure/*
```

Add to partitions

```
$ cryptsetup luksAddKey /dev/nvme0n1p3 /secure/root_keyfile.bin
# skip below if using single disk
$ cryptsetup luksAddKey /dev/nvme1n1p1 /secure/home_keyfile.bin
```

```
$ nvim /etc/mkinitcpio.conf
```

```
FILES=(/secure/root_keyfile.bin)
```

### Home Partition Crypttab (Skip if single disk)

Get uuid of home partition

```
$ blkid /dev/nvme1n1p1
```

Open up the crypt table.
```
$ nvim /etc/crypttab
```

Add in the following line at the bottom of the table
```
arch-home      UUID=<uuid>    /secure/home_keyfile.bin
```

Reload linux

```
$ mkinitcpio -p linux
```

## Grub

Create grub config

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

## System Configuration

### Timezone

```
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

### NTP

```
$ nvim /etc/systemd/timesyncd.conf
```

Add in the NTP servers

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org 
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

Enable timesyncd

```
# systemctl enable systemd-timesyncd.service
```

### Locale

```
$ nvim /etc/locale.gen
```

uncomment the UTF8 lang you want

```
en_US.UTF-8 UTF-8
```

```
$ locale-gen
```

```
$ nvim /etc/locale.conf
```

```
LANG=en_US.UTF-8
```


### hostname

enter it into your /etc/hostname file

```
$ nvim /etc/hostname
```

or 

```
$ echo "mymachine" > /etc/hostname
```

### Users

First secure the root user by setting a password

```
$ passwd
```

Then install the shell you want

```
$ pacman -S zsh
```

Add a new user as follows

```
$ useradd -m -G wheel -s /bin/zsh user
```

set the password on the user

```
$ passwd user
```

Add the wheel group to sudoers

```
$ EDITOR=nvim visudo
```

```
%wheel ALL=(ALL:ALL) ALL
```

### Network Connectivity

```
$ pacman -S networkmanager
$ systemctl enable NetworkManager
```

### Display Manager

```
$ pacman -S gnome
```

```
$ systemctl enable gdm
```


### Microcode

For AMD

```
$ pacman -S amd-ucode
```

For intel

```
$ pacman -S intel-ucode
```

```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```


## Reboot

```
$ exit
$ umount -R /mnt
$ reboot now
```
