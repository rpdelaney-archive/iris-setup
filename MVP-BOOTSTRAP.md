# OS installation

_Continued from [Pre-OS installation plan](https://github.com/rpdelaney/iris-setup/blob/master/MVP-PRE_OS.md)_

[Archlinux Installation guide](https://wiki.archlinuxorg/index.php/Installation_guide)

## Bootstrapping

Mount the btrfs filesystem on `/mnt`.

1. [ ] Pacstrap the filesystem: `pacstrap /mnt base btrfs-progs grub vim`

1. [ ] Generate the fstab: `genfstab -U /mnt >> /mnt/etc/fstab`.

Change root into the new system: `arch-chroot /mnt`

1. [ ] Set the timezone: `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime`
1. [ ] Generate `/etc/adjtime`: `hwclock --systohc`
1. [ ] Set up `/etc/locale.gen`:

```
/etc/locale.gen
---
en_US.UTF-8 UTF-8
```

1. [ ] Generate the locales with `locale-gen`
1. [ ] Set up locale.conf: `echo 'LANG=en_US.UTF-8' > /etc/locale.conf`
1. [ ] Set keyboard layout: `echo 'KEYMAP=dvorak' > /etc/vconsole.conf`
1. [ ] Set the hostname: `echo 'iris' > /etc/hostname`
1. [ ] Add matching entries to hosts:

```
/etc/hosts
---
127.0.0.1   localhost
::1         localhost
127.0.1.1   iris.localdomain    iris
```

1. [ ] Set the root password: `passwd`

## Bootloader

[GRUB2](https://wiki.archlinux.org/index.php/GRUB) in BIOS mode (no UEFI)

### Configure mkinitcpio.conf

We need to configure and recreate the initramfs.

#### LUKS keyfile

1. [ ] Now we generate a LUKS keyfile to embed in GRUB

```
# dd bs=1024 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
# chmod 000 /crypto_keyfile.bin
# chmod 600 /boot/initramfs-linux*
# cryptsetup luksAddKey /dev/nvme0n1p3 /crypto_keyfile.bin
```

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

#### System hooks

1. [ ] Add the systemd hooks:

```
/etc/mkinitcpio.conf
---
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf resume block sd-encrypt sd-lvm2 filesystems fsck)
```

#### Generate initramfs

1. [ ] Finally, regenerate the initramfs: `# mkinitcpio -p linux`

We should only have to enter the container decryption passphrase once.

### GRUB

1. [ ] Configure /etc/default/grub using additional arguments for encrypted boot as described [here](https://wiki.archlinux.org/index.php/GRUB#Additional_arguments) and [here](https://wiki.archlinux.org/index.php/GRUB#Encrypted_/boot)
   - Uncomment `GRUB_ENABLE_CRYPTODISK=y`
   - Set kernel parameters allowing the kernel module to decrypt the root partition:

```
/etc/default/grub
---
UUID_NVME=""    # UUID of encrypted disk partition (/dev/nvme0n1p3)
UUID_ROOT=""    # UUID of decrypted & mounted volume (/dev/mapper/cryptroot)
UUID_SWAP=""    # UUID of swap partition (/dev/nvme0n1p1)

GRUB_CMDLINE_LINUX="rd.luks.name=$UUID_NVME=cryptlvm root=UUID=$UUID_ROOT resume=UUID=$UUID_SWAP"
```

1. [ ] Install GRUB: `grub-install --target=i386-pc --recheck /dev/nvme0n1`
1. [ ] Finally, generate the GRUB configuration file: `grub-mkconfig -o /boot/grub/grub.cfg`

## Encrypt the swap

1. [ ] Configure encrypted swap as described [here](https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption)
   - _I think we did this already?_

## Encrypt the boot partition

1. [ ] Encrypt the boot partition as described [here](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))  
   - _I think we did this already?_
1. [ ] Add [pacman hooks](http://archive.is/jRuC3) to automount the boot partition when upgrades need to access related files

## Done :)

`reboot` and go on to post-install stuffs.

<!--- vim: set nospell: -->
