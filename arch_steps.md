# IN USB MEDIA

- check if UEFI mode
```sh
$ cat /sys/firmware/efi/fw_platform_size # 64
```

- connect to the wifi
```sh
$ iwctl
[iwctl] device list
[iwctl] station WLAN0 scan &&
[iwctl] station WLAN0 get-networks
[iwctl] station WLAN0 connect NETWORK
$ ping gnu.org
```

- update clock
```sh
$ timedatectl set-ntp true
```

- find devices and list partitions
```sh
$ lsblk
$ fdisk -l
```

- create a partition
```sh
$ cfdisk /dev/sdX
#    label  GPT
#     1 Gb  FAT # efi_partition
#   XXX Gb ext4 # root_partition
```

- format partitions and mount
```sh
$ mkfs.ext4 /dev/root_partition
$ mkfs.fat -F 32 /dev/efi_partition

$ mount /dev/root_partition /mnt
$ mount --mkdir /dev/efi_partition /mnt/boot
```

- install kernel stuff
```sh
$ pacstrap -K /mnt base base-devel linux linux-firmware linux-headers intel-ucode neovim
```

- generate an fstab file
```sh
$ genfstab -U /mnt >> /mnt/etc/fstab
```

- chroot into new system
```sh
$ arch-chroot /mnt
```

# IN SYSTEM

- set timezone and clock
```sh
$ timedatectl set-ntp true
$ ln -sf /usr/share/zoneinfo/Americas/Sao_Paulo /etc/localtime
$ hwclock --systohc
$ locale-gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

- set network stuff
```sh
$ echo HOSTNAME > /etc/hostname
$ nvim /etc/hosts
# 127.0.0.1        localhost
# ::1              localhost
# 127.0.1.1        HOSTNAME
```

- test
```sh
mkinitcpio -P
```

- set users
```sh
$ passwd

$ useradd -m -g users -G wheel -s /bin/bash USERNAME
$ passwd USERNAME

$ EDITOR=nvim visudo
# %wheel ALL=(ALL) ALL
```

- enable services and download basic stuff
```sh
$ pacaman -S man man-db texinfo git

$ pacman -S networkmanager
$ systemctl enable NetworkManager.service

$ pacman -S reflector
$ systemctl enable reflector.service
$ reflector -c 'Brazil' -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist

$ pacman pipewire-audio pipewire-docs pipewire-alsa pipewire-jack pipewire-pulse

$ pacman -S wayland xorg gnome hyprland
$ systemctl enable gdm.service
```

- create grub configuration
```sh
$ lsblk # check if everything is mounted
$ pacman -S grub efibootmgr
$ grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/boot
$ grub-mkconfig -o /boot/grub/grub.cfg
```

- reboot the system

```sh
exit
umount -R /mnt
reboot
```

