# Pre-OS prep

We're putting the O/S on the NVME disk, `/dev/nvme0n1`.

## Installation media

Prepare the installation media: keyboard layout, system clock, and optimize the mirrorlist.

```
# loadkeys dvorak
# timedatectl set-ntp true
# pacman -Syu
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist."$(date +%Y%m%d-%H%m%S)".bak
# reflector --verbose --connection-timeout 3 --age --protocol https --protocol rsync --sort rate --sort score --save /etc/pacman.d/mirrorlist
```

_(No dm-crypt [drive preparation](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation) since the disk is blank and factory-new.)_

## Create partitions

[ ] Create MBR partition tables: [GPT](https://wiki.archlinux.org/index.php/GPT)
  - `/dev/nvme0n1p0` A swap partition of 32G
  - `/dev/nvme0n1p1` A `/` partition for the rest, since we don't need EFI and we will be using btrfs subvolumes under LUKS

[ ] Activate swap

```
mkswap /dev/nvme0n1p0
swapon /dev/nvme0n1p0
```

## Create LUKS container

[ ] Format the LUKS container:

```
# cryptsetup luksFormat /dev/nvme0n1p1
```

[ ] [Backup LUKS headers](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Backup_and_restore)

## O/S Filesystem

- [btrfs](https://wiki.archlinux.org/index.php/Btrfs) with [full disk encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap) on both drives, including encrypted bootloader

Unlock the LUKS container and format it.

```
# cryptsetup open --type luks1 /dev/nvme0n1p1 cryptroot
# mkfs.btrfs -L root /dev/mapper/cryptroot
```

### Create btrfs subvolumes

Mount the newly created filesystem with zstd compression.

```
# mount -o compress=zstd /dev/mapper/cryptroot /mnt
```

Now we will create the following subvolumes:

```
subvolid=5 (/dev/nvme0n1p1)
   ├── @ (mounted as /)
   |       ├── /home (mounted @home subvolume)
   |       ├── /.snapshots (mounted @snapshots subvolume)
   |       ├── /var/cache/pacman/pkg (nested subvolume)
   |       ├── ... (other directories and nested subvolumes)
   |       └──
   ├── @snapshots (mounted as /.snapshots)
   ├── @home (mounted as /home)
   └── @... (additional subvolumes you wish to use as mount points)
```

[ ] First, create the top-level subvolumes:

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

[ ] Finally, create nested subvolumes that we do **not** want to have snapshots of when taking snapshots of `/`.

```
# btrfs subvolume create /mnt/var/cache/pacman/pkg
# btrfs subvolume create /mnt/var/abs
# btrfs subvolume create /mnt/var/tmp
```

To see a list of current subvolumes: `btrfs subvolume list -p /`
To delete a subvolume: `btrfs subvolume delete /path/to/subvolume`

### LUKS keyfile

[ ] Now we generate a LUKS keyfile to embed in GRUB.

```
# dd bs=1024 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
# chmod 000 /crypto_keyfile.bin
# chmod 600 /boot/initramfs-linux*
# cryptsetup luksAddKey /dev/nvme0n1p2 /crypto_keyfile.bin
```

### Configure mkinitcpio.conf

[ ] Include the keyfile in [mkinitcpio's FILES array](https://wiki.archlinux.org/index.php/Mkinitcpio#BINARIES_and_FILES):

```
/etc/mkinitcpio.conf
---
FILES=(/crypto_keyfile.bin)
```

[ ] Enable btrfs-check to run on the unmounted filesystem:

```
/etc/mkinitcpio.conf
---
BINARIES=("/usr/bin/btrfs")
```

[ ] Add the systemd hooks:

```
/etc/mkinitcpio.conf
---
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck)
```

[ ] Finally, regenerate the initramfs:

`# mkinitcpio -p linux`

We should only have to enter the container decryption passphrase once.

## Bootloader

[GRUB2](https://wiki.archlinux.org/index.php/GRUB) in BIOS mode (no UEFI)

<!--- vim: nospell -->
