# securely erase the disk
cryptsetup open --type plain /dev/sda container
dd if=/dev/zero of=/dev/mapper/container

# create partitions
sgdisk -Z /dev/sda
sgdisk -n 1:0:+200M -t 1:EF00 -n 2:0:0 -t 2:8300 -p /dev/sda

# encrypt /dev/sda2
cryptsetup luksFormat /dev/sda2
# cryptsetup luksDump /dev/sda2
cryptsetup luksOpen /dev/sda2 opnd

# create logical volumes; 1st PE located at 417792s (assuming sgdisk makes 2048s alignment and cryptsetup makes 4096s alignment)
pvcreate /dev/mapper/opnd
# pvs -o +pe_start 
vgcreate vg /dev/mapper/opnd
lvcreate -L 4G vg -n swapv
lvcreate -L 30G vg -n rootv
lvcreate -L 880G vg -n homev
# ~ 20G left for snapshots

# format filesystems
mkfs.vfat -F32 /dev/sda1
mkswap /dev/mapper/vg-swapv
mkfs.ext4 /dev/mapper/vg-rootv
mkfs.ext4 /dev/mapper/vg-homev

# mount filesystems
swapon /dev/mapper/vg-swapv
mount /dev/mapper/vg-rootv /mnt
mkdir /mnt/{boot,home}
mount /dev/sda1 /mnt/boot
mount /dev/mapper/vg-homev /mnt/home

# install the base system
pacstrap -i /mnt base base-devel

# generate an fstab, set relatime
genfstab -U -p /mnt >> /mnt/etc/fstab

# chroot
arch-chroot /mnt /bin/bash

pacman -S efibootmgr intel-ucode ifplugd wpa_actiond

# edit /etc/locale.gen
locale-gen
locale > /etc/locale.conf

# configure mkinitcpio
vi /etc/mkinitcpio.conf
# MODULES="i915"
# HOOKS="base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck"
mkinitcpio -p linux

# configure the boot loader
efibootmgr -d /dev/sda -p 1 -c -L "Arch Linux" -l /vmlinuz-linux -u "cryptdevice=/dev/sda2:vg root=/dev/mapper/vg-rootv rw resume=/dev/mapper/vg-swapv acpi_osi= initrd=/intel-ucode.img initrd=/initramfs-linux.img"

passwd

# add a new user & edit sudoers
useradd -m -G wheel archie
passwd archie
visudo

exit
umount -R /mnt
