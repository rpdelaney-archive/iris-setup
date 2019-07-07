# OS installation

[Installation guide](https://wiki.archlinuxorg/index.php/Installation_guide)

## Bootstrapping

Mount the btrfs filesystem on `/mnt`.

1. [ ] Pacstrap the filesystem: `pacstrap /mnt base btrfs-progs`

1. [ ] Generate the fstab: `genfstab -U /mnt >> /mnt/etc/fstab`

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

## Done :)

`reboot` and go on to post-install stuffs.
