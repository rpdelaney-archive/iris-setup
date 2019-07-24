# Laptop setup plan

I'm calling it 'Iris' (after the ancient Greek goddess of the rainbow and messenger of the gods) because it has one of those stupid rainbow backlit keyboards.

## Hardware

Hardware specs [here](./HARDWARE.md).

## MVP Goals

### Pre-OS

[Execution plan](https://gist.github.com/rpdelaney/f4b789d7b9c6b0fba24c524f5829cc04#file-mvp-pre_os-md)

1. Partition the disks
   1. A `swap` partition of 32G
   1. A `/` partition for the rest, since we don't need EFI and we will be using btrfs subvolumes under LUKS
1. Create LUKS container
1. Format O/S filesystem: [btrfs](https://wiki.archlinux.org/index.php/Btrfs) with [full disk encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap) on both drives, including an encrypted bootloader

### OS

[Execution plan](https://gist.github.com/rpdelaney/f4b789d7b9c6b0fba24c524f5829cc04#file-mvp-os-md)

1. [ ] [Archlinux](https://wiki.archlinux.org/index.php/Installation_guide)
1. [ ] Bootloader: [GRUB2](https://wiki.archlinux.org/index.php/GRUB) in BIOS mode (no UEFI)[[1]](http://techrights.org/wiki/index.php/UEFI)[[2]](http://bytesmedia.co.uk/2012/07/17/richard-stallman-uefi/)[[3]](https://yarchive.net/comp/linux/efi.html)

### Post-OS

Execution plan (TBD)

1. [ ] Keyboard layout in the [console](https://wiki.archlinux.org/index.php/Linux_console/Keyboard_configuration)
1. [ ] Time synchronization: [Chrony](https://wiki.archlinux.org/index.php/Chrony)

## Post-MVP Goals

Take a btrfs snapshot here.

### Pre-WM

1. [ ] Separate `ryan` user role in sudoers group
1. [ ] Create XDG [default directories](https://wiki.archlinux.org/index.php/XDG_user_directories#Creating_default_directories) with `pacman -S xdg-user-dirs && xdg-user-dirs-update`
   * [ ] Set the `XDG_*_DIR` env vars to stop creating directory names I hate (like "Downloads")
1. [ ] AUR helper `yay`
1. [ ] [Power Management](https://wiki.archlinux.org/index.php/Power_management)
   * [ ] Install `acpid`
   * [ ] Set systemd [ACPI Events](https://wiki.archlinux.org/index.php/Power_management#ACPI_events)
   * [ ] Disable Wake-on-LAN
   * [ ] [Hybrid sleep](https://wiki.archlinux.org/index.php/Power_management#Hybrid-sleep_on_suspend_or_hibernation_request) on suspend
1. [ ] [Security](https://wiki.archlinux.org/index.php/Security) recommendations, especially:
   * [ ] No remote access as root
   * [ ] Public key authentication only
   * [ ] Passworded BIOS
   * Consider disabling the front-facing camera by [blacklisting](https://wiki.archlinux.org/index.php/Kernel_module#Blacklisting) the `uvcvideo` module
1. [ ] Domain name resolution: [dnsmasq](https://wiki.archlinux.org/index.php/Dnsmasq)
1. [ ] Network manager: [Wicd](https://wiki.archlinux.org/index.php/Wicd)
1. [ ] NVidia [graphics drivers](https://wiki.archlinux.org/index.php/NVIDIA)
   * [ ] Deal with [Optimus](https://wiki.archlinux.org/index.php/NVIDIA_Optimus)
1. [ ] Touchpad support with [libinput](https://wiki.archlinux.org/index.php/Libinput)

### Post-WM

Take a btrfs snapshot here.

1. [ ] WM: [i3-gaps](https://wiki.archlinux.org/index.php/I3)
1. [ ] Terminal emulator: [kitty](https://sw.kovidgoyal.net/kitty/)
1. [ ] Keyboard layout in [Xorg](https://wiki.archlinux.org/index.php/Xorg/Keyboard_configuration) using `xorg-setxkbmap`
1. [ ] Mouse sensitivity, keyboard repeat rate etc using `xorg-xset`
1. [ ] Multi-head display configuration using [arandr](https://wiki.archlinux.org/index.php/Multihead#Configuration_using_arandr)
1. [ ] Secure DNS resolution:
   * [ ] [DNSCrypt](https://wiki.archlinux.org/index.php/Dnscrypt-proxy)
   * [ ] [DNSSEC](https://wiki.archlinux.org/index.php/DNSSEC)
   * [ ] Swiss Privacy Foundation [DNS Servers](https://web.archive.org/web/20140209065424/http://anonymous-proxy-servers.net/wiki/index.php/Censorship-free_DNS_servers)
1. [ ] Mirrorlist automation with [Reflector](https://wiki.archlinux.org/index.php/Reflector#Automation)
1. [ ] CPU [microcode](https://wiki.archlinux.org/index.php/Microcode)

## Decisions

1. [ ] BIOS: coreboot or libreboot? (Is this hardware even supported by either?)
1. [ ] [Kernel patchset](https://wiki.archlinux.org/index.php/Kernel#Patches_and_patchsets)?
1. [ ] Filesystem: autodefrag or no autodefrag?
1. [ ] Shell: bash or zsh?
1. [ ] Windowing system: X11 (i3-gaps) or Wayland (sway)?
1. [ ] cli file manager: ranger or vifm?
1. [ ] Screenshots: scrot or flameshot?
1. [ ] Launcher: dmenu or rofi?
1. [ ] Backlight control: xbacklight or light?
1. [ ] Status bar: i3blocks or polybar?

## "Nice to have"s

This stuff is out of scope for this execution plan.

See also: [privacytools.io](https://www.privacytools.io/)

1. [ ] [Snapper](https://wiki.archlinux.org/index.php/Snapper)
1. [ ] [ILoveCandy](https://www.reddit.com/r/archlinux/comments/6r8lk0/i_love_candydo_you/)
1. [ ] [Activate](https://wiki.archlinux.org/index.php/Activating_Numlock_on_Bootup) NumLock on bootup
1. [ ] Local NordVPN connection with non-VPN traffic dropped by `iptables`
1. [ ] [zswap](https://wiki.archlinux.org/index.php/Zswap)
1. [ ] [USBGuard](https://usbguard.github.io/)
1. [ ] [Backlight](https://wiki.archlinux.org/index.php/Backlight) control using `light`
1. [ ] [AppArmor](https://wiki.archlinux.org/index.php/AppArmor) for closed-source commercial stuff that I (a) don't completely trust and yet (b) don't want to run in a sandbox/VM
1. [ ] [CPU frequency scaling](https://wiki.archlinux.org/index.php/CPU_frequency_scaling)
1. [ ] Autostart management with [dex](https://www.archlinux.org/packages/community/any/dex/)
1. [ ] [coreboot](https://coreboot.org/) or [libreboot](https://libreboot.org/)
1. [ ] Bootloader partition / decryption headers on [separate device](https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Encrypted_system_using_a_detached_LUKS_header)
1. [ ] [Hard disk shock protection](https://wiki.archlinux.org/index.php/Laptop#Hard_disk_shock_protection)
1. [ ] Maybe [archstrike](https://archstrike.org/)
1. [ ] Local mail delivery using [Postfix](https://wiki.archlinux.org/index.php/Postfix)
1. [ ] Use [systemd timers](https://wiki.archlinux.org/index.php/Systemd/Timers#As_a_cron_replacement) over cron
1. [ ] Pimp out GRUB with [fancy colors](https://www.gnu.org/software/grub/manual/grub/html_node/Theme-file-format.html) and [terminal candy](https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#Visual_configuration)

### Maintenance

[System maintenance](https://wiki.archlinux.org/index.php/System_maintenance)

1. [ ] Periodically [clear pacman cache](https://wiki.archlinux.org/index.php/Pacman#Cleaning_the_package_cache)
1. [ ] Periodically [remove orphaned packages](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Removing_unused_packages_(orphans))
1. [ ] `pacdiff` or [something like it](https://wiki.archlinux.org/index.php/Pacman/Pacnew_and_Pacsave#pacdiff)

### Optimizations

[Improving performance](https://wiki.archlinux.org/index.php/Improving_performance)

1. [ ] systemd boot time (`systemd-analyze`)
   * [ ] [Systemd optimizations](https://freedesktop.org/wiki/Software/systemd/Optimizations/)
1. [ ] Consider a different [Kernel patchset](https://wiki.archlinux.org/index.php/Kernel#Patches_and_patchsets)
   * [ ] Research and consider MuQSS and BFQ
   * [ ] [Scheduling policies](https://ck.fandom.com/wiki/SchedulingPolicies)
1. [ ] [zram](https://wiki.archlinux.org/index.php/Improving_performance#Zram_or_zswap) ?

<!--- vim: set nospell: -->
