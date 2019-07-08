# OS installation

[Installation guide](https://wiki.archlinuxorg/index.php/Installation_guide)

## Bootstrapping

Mount the btrfs filesystem on `/mnt`.

1. [ ] Pacstrap the filesystem: `pacstrap /mnt base btrfs-progs grub vim`

1. [ ] Generate the fstab: `genfstab -U /mnt >> /mnt/etc/fstab`.

Note that it will create entries for all the subvolumes. Since btrfs subvolumes are recursively mounted automatically, we only need fstab entries for `/` and swap. Delete the others.

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

1. [ ] Since we are using LVM, recreate the initramfs image: `mkinitcpio -p linux`
1. [ ] Set the root password: `passwd`

## Bootloader

[GRUB2](https://wiki.archlinux.org/index.php/GRUB) in BIOS mode (no UEFI)

1. [ ] Configure /etc/default/grub using additional arguments for encrypted boot as described [here](https://wiki.archlinux.org/index.php/GRUB#Additional_arguments) and [here](https://wiki.archlinux.org/index.php/GRUB#Encrypted_/boot)
   - Uncomment `GRUB_ENABLE_CRYPTODISK=y`
   - Set kernel parameters allowing the kernel module to decrypt the root partition:

```
/etc/default/grub
---
GRUB_CMDLINE_LINUX="rd.luks.name=_device-UUID_=cryptlvm"
```

1. [ ] Install GRUB `grub-install --target=i386-pc --recheck /dev/nvme0n1` <-- # failed here with "GPT partition label contains no BIOS boot partition"
1. [ ] Finally, generate the GRUB configuration file

## Encrypt the swap

1. [ ] Configure encrypted swap as described [here](https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption)

## Encrypt the boot partition

1. [ ] Encrypt the boot partition as described [here](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))

## Done :)

`reboot` and go on to post-install stuffs.
