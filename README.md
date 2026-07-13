# Ansible
personal ansible playbooks

### Full Bootstrap

```sh
$ sudo ansible-playbook site.yml -l <host>
```

### Server Setup

```sh
$ sudo ansible-playbook playbooks/server.yml
```

### Laptop Setup

```sh
$ sudo ansible-playbook playbooks/desktop.yml
```

### Maintenance Scripts

```sh
$ sudo ansible-playbook playbooks/maintenance.yml
```

#### Syncthing Pairing

```sh
$ ansible-playbook playbooks/syncthing-pair.yml --ask-become-pass
```

---

## Storage & Backup

### Where to put files

Everything goes in `~/Sync/` — Syncthing distributes it to all devices automatically.

```
~/Sync/
├── photos/
├── documents/
├── books/
└── ...
```

Drop files in `~/Sync/` on any device (desktop, phone, tablet, homeserver) and they appear everywhere.

### How backup works

```
~/Sync (desktop)  ──[restic]──►  ~/backup/restic/ (homeserver)
       ▲                                │
       └──────[Syncthing]───────────────┘
              (live sync, all devices)
```

- **Syncthing** — live sync across all devices. Every device has a current copy.
- **restic** — versioned snapshots from desktop → homeserver. Recover deleted or corrupted files from any past point.

Backup runs automatically as part of `ansible-maintenance`.

### First-time restic setup (one-time)

```sh
# 1. create password file
mkdir -p ~/.config/restic
echo "yourpassword" > ~/.config/restic/password
chmod 600 ~/.config/restic/password

# 2. init repo on server (SSH config handles port/key)
restic -r sftp:homeserver:/home/neumann/backup/restic/ \
  --password-file ~/.config/restic/password init
```

### Restore a file

```sh
# list snapshots
restic -r sftp:homeserver:/home/neumann/backup/restic/ \
  --password-file ~/.config/restic/password snapshots

# restore specific snapshot
restic -r sftp:homeserver:/home/neumann/backup/restic/ \
  --password-file ~/.config/restic/password \
  restore SNAPSHOT_ID --target ~/restore/
```

---

# FIRST INSTALL HOW TO

### IN USB MEDIA

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

### IN SYSTEM

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


- test inintramfs
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
$ pacman -S man man-db texinfo git

$ pacman -S networkmanager
$ systemctl enable NetworkManager.service

$ pacman -S reflector
$ systemctl enable reflector.service
$ reflector -c 'Brazil' -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist

$ pacman pipewire-audio pipewire-docs pipewire-alsa pipewire-jack pipewire-pulse

$ pacman -S wayland gnome hyprland # or: $ pacman -S wayland gdm hyprland
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


### AFTER REBOOT

- clone this repository and run one of the playbooks

```bash
git clone https://github.com/neumann-mlucas/ansible
cd ansible
# change hosts to only localhost
sudo ansible-playbook playbooks/desktop.yml
```

- set a static IP address

Use DHCP reservation on the router (bind server MAC to fixed IP). Avoids
broken static config when changing networks. The server playbook configures
static IP via nmcli but DHCP reservation is more resilient.

- set ssh keys

```sh
# on desktop: generate ansible key
ssh-keygen -t ed25519 -f ~/.ssh/ansible -C "ansible"

# temporarily allow password auth on server:
# edit /etc/ssh/sshd_config.d/20-force_publickey_auth.conf
# set: PasswordAuthentication yes / AuthenticationMethods password
# sudo systemctl restart sshd

# copy key from desktop
ssh-copy-id -i ~/.ssh/ansible.pub -p 3141 neumann@SERVERIP

# revert sshd config and restart sshd on server
```

- set syncthing shared folder

```sh
# after both machines are up and syncthing is running:
ansible-playbook playbooks/syncthing-pair.yml --ask-become-pass
```


- server post-setup (first run only)

```sh
# Jellyfin: complete wizard at http://SERVERIP:8096
# - create admin account
# - add media library pointing to /mnt/media

# Pi-hole web UI: http://SERVERIP:8080/admin
# Set password (pi-hole v6):
sudo pihole-FTL --config webserver.api.password "yourpassword"
sudo systemctl restart pihole-FTL
#
# Add blocklists via admin UI → Adlists → Add blocklist:
#   https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts  (ads, safe)
#   https://dbl.oisd.nl/                                               (ads + tracking)
#   https://raw.githubusercontent.com/hagezi/dns-blocklists/main/hosts/pro.txt  (aggressive)
# Then run: sudo pihole -g
#
# Point router DNS to SERVERIP: router admin → DHCP → DNS server
# Verify: pihole status

# Nginx (if running unexpectedly):
sudo systemctl disable --now nginx
```

- run fwup

```bash
fwupdmgr get-devices
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update
```
