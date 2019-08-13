# Pre-OS prep

Review the [hardware](https://github.com/rpdelaney/iris-setup/blob/master/HARDWARE.md).

We're putting the O/S on the NVME disk, `/dev/nvme0n1`.

```
+---------------------------+-----------------------------------------------+
| Boot loader               | Logical volume #2     | Logical volume #2     |
| mountpoint: /boot         | mountpoint: swap      | mountpoint: /         |
|                           | /dev/volgroup0/swap   | /dev/volgroup0/root   |
|                           |_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _|
|                           |                                               |
| LUKS2 encrypted partition |              LUKS2 encrypted partition        |
| /dev/nvme0n1p1            |                /dev/nvme0n1p2                 |
+---------------------------+-----------------------------------------------+
```

## Installation media

1. [ ] Prepare the installation media: keyboard layout, system clock, and optimize the mirrorlist.

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
gdisk /dev/nvme0n1
```

Device         | Type Hex Code | Role      | First sector | Last sector
---------------|---------------| ----------|--------------|------------
/dev/nvme0n1p1 | EF02          | BIOS boot | 2048         | +1M
/dev/nvme0n1p2 | 8300          | Linux FS  | default      | default

## Create LUKS container

1. [ ] Format the LUKS container:

```
cryptsetup luksFormat --type luks1 --use-random --hash whirlpool --iter-time 5000 /dev/nvme0n1p2
```

1. [ ] [Backup LUKS headers](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Backup_and_restore)

## O/S Filesystem

Decrypt and mount the encrypted root partition.

```
cryptsetup open --type luks1 /dev/nvme0n1p2 cryptlvm
```

### Create and mount LVM volumes

_Main article: [LVM](https://wiki.archlinux.org/index.php/LVM#Create_file_systems_and_mount_logical_volumes)_

1. [ ] Create LVM group and volumes for root and swap.

```
vgcreate volgroup0 /dev/mapper/cryptlvm
lvcreate -L 32G -n lvswap
lvcreate -L 100%FREE -n lvroot
```

* To display created volume groups, `vgdisplay`
* To display created logical volumes, `lvdisplay`

### Create btrfs filesystem

1. [ ] Format the root LVM volume and create the swap:

```
mkfs.btrfs -L root /dev/volgroup0/lvroot
mkswap -L swap /dev/volgroup0/lvswap
swapon -L swap
```

Now we will create the following subvolumes:

```
subvolid=5 (/dev/volgroup0/lvroot)
   └──| @ (mounted as /)
      ├── /home
      ├── /var/abs
      ├── /var/tmp
      └── /var/cache/pacman/pkg
```

We will create a subvolume for snapshots later, when setting up snapper.

Mount the newly created filesystem with zstd compression.

```
# mount -o compress=zstd /dev/volgroup0/lvroot /mnt
```

1. [ ] Now, create the top-level subvolume:

```
btrfs subvolume create /mnt/@
umount /mnt
```

Next mount the top-level subvolume:

```
mount -o compress=zstd,subvol=@ /dev/volgroup0/lvroot /mnt
```

1. [ ] Create nested subvolumes that we do **not** want to have snapshots of when taking snapshots of `/`.

```
btrfs subvolume create /mnt/home
mkdir -p /mnt/var
btrfs subvolume create /mnt/var/abs
btrfs subvolume create /mnt/var/tmp
mkdir -p /mnt/var/cache/pacman
btrfs subvolume create /mnt/var/cache/pacman/pkg
```

Mount the nested subvolumes:

```
mount -o compress=zstd,subvol=@/var/abs /dev/volgroup0/lvroot /mnt/var/abs
mount -o compress=zstd,subvol=@/var/tmp /dev/volgroup0/lvroot /mnt/var/tmp
mount -o compress=zstd,subvol=@/var/cache/pacman/pkg /dev/volgroup0/lvroot /mnt/var/cache/pacman/pkg
```

* To see a list of current subvolumes: `btrfs subvolume list -a /mnt`
* To delete a subvolume: `btrfs subvolume delete /path/to/subvolume`

## Update the fstab

Now that we can get the UUID for the root and swap LVM volumes, we are ready to get fstab into its final form.  Note that although this is an SSD, we don't set 'discard' on these disks since that enforces continuous TRIM, but we are going to set up periodic TRIM with `fstrim`.

1. [ ] Define the fstab:

```
genfstab -U /mnt >> /mnt/etc/fstab
```

Check out the file it created. We want it to be like this:

```
/mnt/etc/fstab
---
# <file system>                             <dir>       <type>      <options>                                                                   <dump> <pass>
# /dev/volgroup0/lvroot LABEL=root
UUID=<LVROOT UUID>                          /           btrfs       rw,relatime,compress=zstd:3,ssd,space_cache,subvolid=256,subvol=/@,subvol=@ 0       0

# /dev/volgroup0/swap LABEL=swap
LABEL=SWAP                                  none        swap        defaults                                                                    0       0
```

## O/S

_Continue to [OS Installation Plan](https://github.com/rpdelaney/iris-setup/blob/master/BOOTSTRAP.md)_

<!--- vim: set nospell: -->
