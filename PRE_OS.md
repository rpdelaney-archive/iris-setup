# Pre-OS prep

Review the [hardware](https://github.com/rpdelaney/iris-setup/blob/master/HARDWARE.md).

We're putting the O/S on the NVME disk, `/dev/nvme0n1`.

```
+---------------------------+-----------------------------------------------+
| Boot loader               | Logical volume #2     | Logical volume #2     |
| mountpoint: /boot         | mountpoint: swap      | mountpoint: /         |
|                           | /dev/MyVolGroup/swap  | /dev/MyVolGroup/root  |
|                           |_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _|
|                           |                                               |
| LUKS2 encrypted partition |              LUKS2 encrypted partition        |
| /dev/nvme0n1p1            |                /dev/nvme0n1p2                 |
+---------------------------+-----------------------------------------------+
```

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

### Create btrfs subvolumes

Now we will create the following subvolumes:

```
subvolid=5 (/dev/mapper/cryptlvm)
   └──| @ (mounted as /)
      ├── /home
      ├── /var/abs
      ├── /var/tmp
      └── /var/cache/pacman/pkg
```

We will create a subvolume for snapshots later, when setting up snapper.

Mount the newly created filesystem with zstd compression.

```
# mount -o compress=zstd /dev/mapper/cryptlvm /mnt
```

1. [ ] Now, create the top-level subvolume:

```
btrfs subvolume create /mnt/@
umount /mnt
```

Next mount the top-level subvolume:

```
mount -o compress=zstd,subvol=@ /dev/mapper/cryptlvm /mnt
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

1. Mount the nested subvolumes

```
mount -o compress=zstd,subvol=@/var/abs /dev/mapper/cryptlvm /mnt/var/abs
mount -o compress=zstd,subvol=@/var/tmp /dev/mapper/cryptlvm /mnt/var/tmp
mount -o compress=zstd,subvol=@/var/cache/pacman/pkg /dev/mapper/cryptlvm /mnt/var/cache/pacman/pkg
```

- To see a list of current subvolumes: `btrfs subvolume list -a /mnt`
- To delete a subvolume: `btrfs subvolume delete /path/to/subvolume`

## Update the fstab

Note that although this is an SSD, we don't set 'discard' on these disks since that enforces continuous TRIM, but we are going to set up periodic TRIM with `fstrim`.

```
# <file system>                             <dir>       <type>      <options>                                                                   <dump> <pass>
# /dev/mapper/cryptlvm LABEL=root
UUID=<CRYPTLVM UUID>                        /           btrfs       rw,relatime,compress=zstd:3,ssd,space_cache,subvolid=256,subvol=/@,subvol=@ 0       0

# /dev/mapper/swap LABEL=swap
UUID=<SWAP UUID>                            none        swap        defaults                                                                    0       0
```

## Create LVM volumes

Unlock the encrypted partition and create lvm volumes for root and swap.
`https://wiki.archlinux.org/index.php/LVM#Create_file_systems_and_mount_logical_volumes`

## O/S

_Continue to [OS Installation Plan](https://github.com/rpdelaney/iris-setup/blob/master/BOOTSTRAP.md)_

<!--- vim: set nospell: -->
