# OS installation

_Continued from [Pre-OS installation plan](https://github.com/rpdelaney/iris-setup/blob/master/PRE_OS.md)_

[Archlinux Installation guide](https://wiki.archlinuxorg/index.php/Installation_guide)

## Bootstrapping

Mount the btrfs filesystem on `/mnt`.

1. [ ] Pacstrap the filesystem:

```
pacstrap /mnt $(curl -Ss http://ix.io/1Rtk)
```

1. [ ] Generate the fstab:

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Now that we can get the UUID for the root and swap LVM volumes, we are ready to get fstab into its final form.  Note that although this is an SSD, we don't set 'discard' on these disks since that enforces continuous TRIM, but we are going to set up periodic TRIM with `fstrim`.

Check out the file it created. `genfstab` will attempt to add records for all the subvolumes as well -- but they will be mounted automatically, so they should be removed.  We want it to be like this:

```
/mnt/etc/fstab
---
# <file system>                             <dir>       <type>      <options>                                                                   <dump> <pass>
# /dev/mapper/volgroup0-lvroot LABEL=root
UUID=<LVROOT UUID>                          /           btrfs       rw,relatime,compress=zstd:3,ssd,space_cache,subvolid=256,subvol=/@,subvol=@ 0       0

# /dev/mapper/volgroup0-lvswap LABEL=swap
LABEL=SWAP                                  none        swap        defaults                                                                    0       0
```

Change root into the new system:

```
# arch-chroot /mnt
```

1. [ ] Set the timezone:

```
# ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```

1. [ ] Generate `/etc/adjtime`:

```
# hwclock --systohc
```

1. [ ] Set up `/etc/locale.gen`:

```
# echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen
```

1. [ ] Generate the locales with `locale-gen`
1. [ ] Set up locale.conf:

```
# echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

1. [ ] Set keyboard layout:

```
echo 'KEYMAP=dvorak' > /etc/vconsole.conf
```

1. [ ] Set the hostname:

```
echo 'iris' > /etc/hostname
```

1. [ ] Add matching entries to hosts:

```
/etc/hosts
---
127.0.0.1   localhost
::1         localhost
127.0.1.1   iris.localdomain    iris
```

1. [ ] Set the root password: `passwd`

## fstrim

util-linux provides `fstrim.service` and `fstrim.timer` systemd units. Enabling the timer activates the periodic fstrim service:

```
systemctl enable fstrim.timer
```

_See also:_ `man fstrim`

Q: How does it know not to trim magnetic disks?
A: It trims everything in fstab. Since the magnetic disk is going in /etc/crypttab later, that is okay for us.

Q: How do I tune the period of trimming?
A: `/usr/lib/systemd/system/fstrim.service` defines periodicity but the default of 1 week is probably fine

## Bootloader

[GRUB2](https://wiki.archlinux.org/index.php/GRUB) in BIOS mode (no UEFI)

### Configure mkinitcpio.conf

We need to configure and recreate the initramfs.

#### LUKS keyfile

Decrypting the bootloader will require entering a passphrase. Since the root partition is also encrypted, and we don't want to enter that long passphrase twice, we decrypt the root volume with a keyfile.

1. [ ] Create a LUKS keyfile to embed in the kernel when we generate the initramfs later:

```
# dd bs=1024 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock
# chmod 000 /crypto_keyfile.bin
# chmod 600 /boot/initramfs-linux*
# cryptsetup luksAddKey /dev/nvme0n1p2 /crypto_keyfile.bin
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

### Pacman Hook

Since we have an Nvidia graphics card (arg) we need to add a pacman hook to regenerate the initramfs every time the nvidia graphics driver is updated (or removed).

**WARNING** If we use a different kernel, such as (for example) linux-ck, both `Target=` lines will need to be updated accordingly.

```
/etc/pacman.d/hooks/nvidia.hook
---
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux
# Change the linux part above and in the Exec line if a different kernel is used

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

#### Generate initramfs

1. [ ] Finally, regenerate the initramfs:

```
mkinitcpio -p linux
```

We should only have to enter the container decryption passphrase once.

### GRUB

1. [ ] Configure /etc/default/grub using additional arguments for encrypted boot as described [here](https://wiki.archlinux.org/index.php/GRUB#Additional_arguments) and [here](https://wiki.archlinux.org/index.php/GRUB#Encrypted_/boot)
   - [ ] Uncomment `GRUB_ENABLE_CRYPTODISK=y`
   - [ ] Set kernel parameters allowing the kernel module to decrypt the root partition:

```
/etc/default/grub
---
UUID_ROOT=""    # UUID of encrypted disk partition (/dev/nvme0n1p2)
UUID_LVM2=""    # UUID of the root LVM volume (/dev/volgroup0/lvroot)
UUID_SWAP=""    # UUID of swap volume (/dev/volgroup0/lvswap)

GRUB_CMDLINE_LINUX="rd.luks.name=$UUID_ROOT=cryptlvm root=UUID=$UUID_LVM2 resume=UUID=$UUID_SWAP"
```

1. [ ] Install GRUB: `grub-install --target=i386-pc --recheck /dev/nvme0n1`
1. [ ] Finally, generate the GRUB configuration file: `grub-mkconfig -o /boot/grub/grub.cfg`

## Pacman boot hook

1. [ ] Add [pacman hooks](http://archive.is/jRuC3) to automount the boot partition when upgrades need to access related files

## Create our user role

1. [ ] Create the user role.

```
# useradd --create-home --shell /bin/bash ryan
# passwd ryan
# groupadd sudoers
# usermod -a -G sudoers ryan
```

1. [ ] Add the sudoers group in `visudo`.

## Snapper

Once we have created our user role, create a group for managing snapper snapshots and add ourselves to it:

```
# groupadd snapper
# usermod -a -G snapper ryan
```

Install snapper and snap-pac:

```
# pacman -S snapper snap-pac
```

snap-pac creates a pacman hook that automatically creates pre/post snapshots if the timer is enabled.

```
# systemctl enable snapper-timeline.timer
```

Create a snapper config for a `.snapshots` subvolume:

```
# snapper create-config --template default /
```

Edit the configurations to set up scheduled snapshots:

```
/etc/snapper/configs/root
---
# create hourly snapshots
TIMELINE_CREATE="yes"
# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="3"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="1"
TIMELINE_LIMIT_MONTHLY="12"
TIMELINE_LIMIT_YEARLY="2"
```

1. [ ] Also do this for the homedir, which I guess is `/etc/snapper/configs/home`

## Done :)

Exit the chroot and `reboot` and go on to post-install stuffs.

<!--- vim: set nospell: -->
