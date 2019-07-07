# Pre-OS prep

We're putting the O/S on the NVME disk, `/dev/nvme0n1`.

## Installation media

Prepare the installation media: keyboard layout, system clock, and optimize the mirrorlist.

```
# loadkeys dvorak
# timedatectl set-ntp true
# pacman -Sy reflector
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
# reflector --verbose --age 12 --sort rate --sort score --save /etc/pacman.d/mirrorlist
```

_(No dm-crypt [drive preparation](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation) since the disk is blank and factory-new.)_

## Create partitions

1. [ ] Create MBR partition tables: [GPT](https://wiki.archlinux.org/index.php/GPT)

```
# gdisk /dev/nvem0n1
```

  - `/dev/nvme0n1p1` A swap partition (type hex code 8200) with first sector at 2048 and last sector at +32G
  - `/dev/nvme0n1p2` A `/` partition (type hex code 8300) with first sector default and last sector default for the rest, since we don't need EFI and we will be using btrfs subvolumes under LUKS

1. [ ] Activate swap

```
mkswap /dev/nvme0n1p1
swapon /dev/nvme0n1p1
```

## Create LUKS container

1. [ ] Format the LUKS container:

```
# cryptsetup luksFormat /dev/nvme0n1p2
```

1. [ ] [Backup LUKS headers](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Backup_and_restore)

## O/S Filesystem

- [btrfs](https://wiki.archlinux.org/index.php/Btrfs) with [full disk encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap) on both drives, including encrypted bootloader

Unlock the LUKS container and format it.

```
# cryptsetup open --type luks1 /dev/nvme0n1p2 cryptroot
# mkfs.btrfs -L root /dev/mapper/cryptroot
```

### Create btrfs subvolumes

Mount the newly created filesystem with zstd compression.

```
# mount -o compress=zstd /dev/mapper/cryptroot /mnt
```

Now we will create the following subvolumes:

```
subvolid=5 (/dev/nvme0n1p2)
   ├── @ (mounted as /)
   |       ├── /.snapshots (mounted @snapshots subvolume)
   |       ├── /home (mounted @home subvolume)
   |       └── /var/cache/pacman/pkg (nested subvolume)
   ├── @snapshots (mounted as /.snapshots)
   └── @home (mounted as /home)
```

1. [ ] First, create the top-level subvolumes:

```
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@snapshots
# btrfs subvolume create /mnt/@home
```

Now mount the top-level subvolumes:

```
# umount /mnt
# mount -o compress=zstd,subvol=@ /dev/mapper/cryptroot /mnt
# mount -o compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
# mount -o compress=zstd,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
```

1. [ ] Create nested subvolumes that we do **not** want to have snapshots of when taking snapshots of `/`.

```
# btrfs subvolume create /mnt/@var-abs
# btrfs subvolume create /mnt/@var-tmp
# btrfs subvolume create /mnt/@var-cache-pacman
```

1. [ ] Mount the nested subvolumes

```
# mount -o compress=zstd,subvol=@var-abs /dev/mapper/cryptroot /mnt/var/abs
# mount -o compress=zstd,subvol=@var-tmp /dev/mapper/cryptroot /mnt/var/tmp
# mount -o compress=zstd,subvol=@var-cache-pacman /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg
```

To see a list of current subvolumes: `btrfs subvolume list -p /`
To delete a subvolume: `btrfs subvolume delete /path/to/subvolume`

1. [ ] Add all the subvolumes to the fstab

We set `fs_passno 0` because btrfs filesystems do not need to be fsck'ed

```
/etc/fstab
---
# <file system>     <dir>           <type>      <options>                       <dump>      <pass>

# swap
/dev/nvme0n1p1      none            swap        defaults                        0           0

# root
/dev/nvme0n1p2      /               btrfs       discard,compress=zstd,ssd       0           0
```

### LUKS keyfile

1. [ ] Now we generate a LUKS keyfile to embed in GRUB later.

```
# dd bs=1024 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
# chmod 000 /crypto_keyfile.bin
# chmod 600 /boot/initramfs-linux*
# cryptsetup luksAddKey /dev/nvme0n1p2 /crypto_keyfile.bin
```

### Configure mkinitcpio.conf

1. [ ] Include the keyfile in [mkinitcpio's FILES array](https://wiki.archlinux.org/index.php/Mkinitcpio#BINARIES_and_FILES):

```
/etc/mkinitcpio.conf
---
FILES=(/crypto_keyfile.bin)
```

1. [ ] Enable btrfs-check to run on the unmounted filesystem:

```
/etc/mkinitcpio.conf
---
BINARIES=("/usr/bin/btrfs")
```

1. [ ] Add the systemd hooks:

```
/etc/mkinitcpio.conf
---
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck)
```

1. [ ] Finally, regenerate the initramfs:

`# mkinitcpio -p linux`

We should only have to enter the container decryption passphrase once.

<!--- vim: nospell -->
