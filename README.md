![Arch Linux](https://archlinux.org/static/logos/archlinux-logo-dark-90dpi.ebdee92a15b3.png)
# Arch Linux Installation Guide  
Updated for [Arch 2024.12.01](https://archlinux.org/releng/releases/2024.12.01/)  

## Introduction
The goal of this Arch Linux installation guide is to provide an easier to interpret, while still comprehensive how-to for installing Arch Linux on x86_64 architecture devices. This guide assumes you are technically inclined and have a basic understanding of Linux. This installation guide was made with intentions of sticking to systemd (systemd-boot, systemd-networkd). Due to the vast amount of options/preferences for networking tools & boot loaders, instructions for NetworkManager & GRUB were included to their respective sections.  

This guide is a mix of knowledge and information as listed in the [Appendix A - Resources](#appendix-a---resources) section.  At the end of the day, this is how I do a install of ARCH, your mileage may vary.

**What won't be covered is some basics, like how to get the image and write it to USB, or PXE boot, or however you want to get the target system booted to the Arch install ISO.**  Look in the [Appendix B - Downloads](#appendix-b---downloads)

## Verify Boot Mode
Verify the boot mode and UEFI bitness (if booted in UEFI mode):
```shell
ls /sys/firmware/efi/efivars
```

If the command returns the directory without error, then the system is booted in [uefi](https://wiki.archlinux.org/title/UEFI). if the directory doesn't exist the system may be booted in [bios](https://wiki.archlinux.org/title/Arch_boot_process#Under_BIOS) or [csm](https://en.wikipedia.org/wiki/Compatibility_Support_Module) mode

To see/verify the bitness of UEFI that you are booted into:  
```shell
cat /sys/firmware/efi/fw_platform_size
```

## Initial Network Setup
Check if the [network interface](https://wiki.archlinux.org/title/Network_interface#Network_interfaces) is enabled with [iplink](https://man.archlinux.org/man/ip-link.8)
```shell
ip link
```

If disabled, check the device driver -- see [ethernet#device-driver](https://wiki.archlinux.org/title/Network_configuration/Ethernet#Device_driver) or [wireless#device-driver](https://wiki.archlinux.org/title/Network_configuration/Wireless#Device_driver)
### _Ethernet_
Plug in an ethernet cable, it's that simple

### _WiFi_
Make sure the card isn't blocked with [rfkill](https://wiki.archlinux.org/title/Network_configuration/Wireless#Rfkill_caveat)
```shell
rfkill list
```

If the card is soft-blocked by the kernel, use this command:
```shell
rfkill unblock wifi
```

The card could be hard-blocked by a hardware button or switch, e.g. a laptop. Make sure this is enabled.  
Authenticate to a wireless network in an [iwd](https://wiki.archlinux.org/title/Iwctl) interactive prompt  
```bash
iwctl
```

List all wireless devices
```shell
[iwd] device list
```

Scan for networks
```shell
[iwd] station <device> scan
```

List scanned networks
```shell
[iwd] station <device> get-networks
```

Finally, connect to specified network
```shell
[iwd] station <device> connect <SSID>
```

Verify connection
```shell
[iwd] station <device> show
```

Exit prompt (ctrl+c)
```shell
exit
```

_*for wwan (mobile broadband) -- see [nmcli](https://wiki.archlinux.org/title/Mobile_broadband_modem#ModemManager)_

## SSH Install (Optional)
At this point, you can [Install Arch Linux via SSH](https://wiki.archlinux.org/title/Install_Arch_Linux_via_SSH) if you so choose


## System Clock
Set system clock [timedatectl](https://man.archlinux.org/man/timedatectl.1)
```shell
timedatectl set-ntp true
```

Check status
```shell
timedatectl status
```

## Disk Partitioning
List disk and block devices
```shell
lsblk
```

Using the most desirable [partitioning](https://wiki.archlinux.org/title/Partition) tool([gdisk](https://wiki.archlinux.org/title/Gdisk), [fdisk](https://wiki.archlinux.org/title/Fdisk), [parted](https://wiki.archlinux.org/title/parted), etc) for your system, create a new gpt or mbr partition table, if one does not exist. A gpt table is required in uefi mode; an mbr table is required if the system is booted in legacy bios mode

If booted in uefi, create a [efi](https://wiki.archlinux.org/title/EFI_system_partition) [system partition](https://wiki.archlinux.org/title/EFI_system_partition), otherwise known as _esp_ of atleast 512MB in size.  
Create an optional swap partition (half or equal to RAM) and the required [root](https://wiki.archlinux.org/title/Root_directory#/)[partition](https://wiki.archlinux.org/title/Root_directory) of no less than 10GB

If the system has an existing efi partition, don't create a new one. Do not format it or all data on the partition will be lost, rendering other operating systems potentially un-bootable. Skip the 'mkfs.vfat' command in the [Format partitions](#format-partitions) section below and mount the already existing efi partition if available

To create any stacked block devices for [lvm](https://wiki.archlinux.org/title/LVM), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID), do it now

<details markdown='1'><summary>partitioning with gdisk</summary>

<h3 style="color:aquamarine;">gdisk /dev/sdX</h3>

**new gpt partition table (erases any existing partitions)**
<p style="color:aquamarine;">o</p>

**efi system partition**
<p style="color:aquamarine;">n</p>
<p style="color:aquamarine;">+512M</p>
<p style="color:aquamarine;">ef00</p>


**(swap partition)**
<p style="color:aquamarine;">n</p>
<p style="color:aquamarine;">+4G</p>
<p style="color:aquamarine;">8200</p>

**root partition**
<p style="color:aquamarine;">n</p>
<p style="color:aquamarine;">+10G</p>
<p style="color:aquamarine;">8300</p>

**write changes**
<p style="color:aquamarine;">w</p>
</details>


## _Swap Space (Optional)_
In order to create [swap](https://wiki.archlinux.org/title/Partitioning_tools#Swap) space consider creating either a [swap partition](https://wiki.archlinux.org/title/Swap#Swap_partition) or a [swapfile](https://wiki.archlinux.org/title/Swap#Swap_file) now. To share the [swap space](https://wiki.archlinux.org/title/Swap_file#Swap_space) system-wide with other operating systems or enable hibernation; create a linux swap partition. in comparison, a swapfile can change size on-the-fly and is more easily removed, which may be more desirable for a modestly-sized ssd but you cannot hybernate the system with a swap file.

### _Swap Partition_
If a swap partition was made, format it by replacing 'swap_partition' with it's  assigned block device path, e.g. sda2
```shell
mkswap /dev/<swap_partition>
```

Then activate it
```shell
swapon /dev/<swap_partition>
```

----

### _Swapfile_
To create a swapfile instead, use dd. the following command will create a 4gb swapfile
```shell
dd if=/dev/zero of=/swapfile bs=1M count=<4096> status=progress
```

Format it
```shell
mkswap /swapfile
```

Then activate it
```shell
swapon /swapfile
```

A swapfile without the correct permissions is a big security risk. set the file permissions to 600 with [chmod](https://wiki.archlinux.org/title/File_permissions_and_attributes#Changing_permissions)
```shell
chmod 600 /swapfile
```

## Format Partitions
Format the root partition just created with preferred [filesystem](https://wiki.archlinux.org/title/File_systems) and replace 'root_partition' with it's assigned [block device](https://en.m.wikipedia.org/wiki/Device_file#Block_devices) path, e.g. sda3
```shell
mkfs.<ext4> /dev/<root_partition>
```

For uefi systems, format the efi partition by replacing 'efi_partition' with it's assigned block device path, e.g. sda1

_*if an efi partition already exist, skip formatting_
```shell
mkfs.vfat -F32 /dev/<efi_partition>
```

## Mount Partitions
[mount](https://wiki.archlinux.org/title/Mount) root partition to /mnt
```shell
mount /dev/<root_partition> /mnt
```

Mount the efi system [partition](https://wiki.archlinux.org/title/Partitioning#Example_layouts) to [/boot](https://wiki.archlinux.org/title/Partitioning#/boot)
```shell
mkdir /mnt/boot
mount /dev/<efi_partition> /mnt/boot
```

## Install Essential Packages
Use _pacstrap_ to bootstrap essential packages into your now mounted root partition, choosing the [kernel](https://wiki.archlinux.org/title/Kernel) and if the system has an amd or intel cpu, install the coinciding [microcode](https://wiki.archlinux.org/title/Microcode) updates.  See [Appendix A - Resources](#appendix-a---resources) for more options of kernels and setups that are not covered in the official wiki. 
```shell
pacstrap /mnt base linux linux-firmware nano sudo <cpu_manufacturer>-ucode vim
```

## Fstab
Generate an [fstab](https://wiki.archlinux.org/title/Fstab) file from the currently mounted block devices, defined by uuid's
```shell
genfstab -U /mnt > /mnt/etc/fstab
```

Check generated fstab
```shell
cat /mnt/etc/fstab
```

## Change Root
To continue, [chroot](https://wiki.archlinux.org/title/Change_root) into freshly installed system
```shell
arch-chroot /mnt
```

## Time Zone
Set [time zone](https://wiki.archlinux.org/title/Time_zone)
```shell
ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime
```

generate /etc/adjtime with [hwclock](https://man.archlinux.org/man/hwclock.8)
```shell
hwclock --systohc
```

_*this assumes the hardware clock is set to [utc](https://en.m.wikipedia.org/wiki/UTC). for more, see [system time#time standard](https://wiki.archlinux.org/title/System_time#Time_standard)_

## Localization
Using your favorite editor, [edit](https://wiki.archlinux.org/title/Textedit) /etc/locale.gen and un-[comment](https://linuxhint.com/bash_comments/) 'en_US.UTF-8 UTF-8' or any other necessary [locales](https://wiki.archlinux.org/title/Locale) by removing the '#'
```shell
vim /etc/locale.gen
```

_* for a different editor -- see [documents#editors](https://wiki.archlinux.org/title/List_of_applications/Documents#Text_editors)_

Generate the locales
```shell
locale-gen
```

[create](https://wiki.archlinux.org/title/Textedit) the [locale.conf](https://man.archlinux.org/man/locale.conf.5) File and set the [system locale](https://wiki.archlinux.org/title/Locale#Setting_the_system_locale)
```shell
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

## Network Configuration
Create [hostname](https://wiki.archlinux.org/title/Hostname) file
```shell
echo <hostname> > /etc/hostname
```

Add matching entries to [hosts](https://man.archlinux.org/man/hosts.5)
```shell
vim /etc/hosts
```
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    <hostname>.localdomain <hostname>
```

_*if the system has a permanently assigned ip address, use it instead of '127.0.1.1'_

Install any desired [network managment](https://wiki.archlinux.org/title/Network_configuration) software. for this guide [systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd) is used. if you prefer a gui and configureless setup -- check out networkmanager below

### _Systemd-networkd_
Install wpa_supplicant
```shell
pacman -S wpa_supplicant
```

Connect to the network with [wpa_passphrase](https://wiki.archlinux.org/title/Wpa_supplicant#Connecting_with_wpa_passphrase)
_*Note, the interface name may differ when in the chroot from the booted kernel_
```shell
wpa_passphrase <ssid> <password> > /etc/wpa_supplicant/wpa_supplicant-<interface>.conf
```

Instead of enabling _wpa_supplicant_ in systemd, we will use dhcpcd to start wpa_supplicant and get the DHCP address
Add the following line to the beginning of the /etc/wpa_supplicant-<interface>.conf file
```shell
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=wheel
```

Now, install dhcpcd and enable the service for the wireless adapter
```shell
pacman -S dhcpcd
```

Configure dhcpcd to start wpa_supplicant using [dhcpcd hook](https://wiki.archlinux.org/title/Dhcpcd#10-wpa_supplicant)
```shell
ln -s /usr/share/dhcpcd/hooks/10-wpa_supplicant /usr/lib/dhcpcd/dhcpcd-hooks/
```

Enable the dhcpcd service
```shell
systemctl enable --now dhcpcd@<interface>.service
```

Additionally, review the [Speed up DHCP by disabling ARP probing](https://wiki.archlinux.org/title/Dhcpcd#Speed_up_DHCP_by_disabling_ARP_probing) page for a way to speed up boot times.

Setup systemd-networkd for wireless networking
```shell
vim /etc/systemd/network/25-wireless.network
```

Add this to the contents of the file  
```
[Match]
Name=<interface>

[Network]
DHCP=yes
```
Enable and start systemd-network service daemon
```shell
systemctl enable systemd-networkd
```

_* for a static connection -- see [#static](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_a_static_IP); for ethernet -- see [#wired adapter](https://wiki.archlinux.org/title/Systemd-networkd#Wired_adapter_using_DHCP)_

----

### _NetworkManager_
Install [networkmanager](https://wiki.archlinux.org/title/NetworkManager)
```shell
pacman -S networkmanager
```

Enable and start networkmanager service daemon
```shell
systemctl enable NetworkManager
```

## _Initramfs_
_*creating an initramfs image isn't necessary since [mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) was ran when pacstrap installed the kernel unless you installed a custom kernel outside of pacman_

For [lvm](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), [system encryption](https://wiki.archlinux.org/title/Dm-crypt) or [raid](https://wiki.archlinux.org/title/RAID#Configure_mkinitcpio) modify [mkinitcpio.conf](https://man.archlinux.org/man/mkinitcpio.conf.5) then recreate the initramfs image with:
```shell
mkinitcpio -P
```

## Users and Passwords
Create a new user
```shell
useradd -m <username>
```

Add created user to the wheel group
```shell
usermod -aG wheel <username>
```

Uncomment '%wheel', which one depends if you want _sudo_ with or without a password.  I suggest with for added security.
```shell
EDITOR=vim visudo
```

Set created user [password](https://wiki.archlinux.org/title/Password)
```shell
passwd <username>
```

Set [root user](https://wiki.archlinux.org/title/Root_user) password
```shell
passwd
```

To disable login for [superuser/root](https://en.m.wikipedia.org/wiki/Root_user), use the locking password entry for root user. This will give the system increased security and a user can still be elevated within the wheel group to superuser priveleges with [sudo](https://wiki.archlinux.org/title/Sudo) and [su](https://wiki.archlinux.org/title/Su) commands but prevent login as root similar to Ubuntu style distros  
```shell
passwd -l root
```

## Boot Loader
Install a linux-capable [boot loader](https://wiki.archlinux.org/title/Boot_loader). for simplicity and ease-of-use I recommend _systemd-boot_, not _grub_ for uefi. _systemd-boot_ will boot any configured EFI image including windows operating systems. _systemd-boot_ is not compatible with systems booted in legacy bios mode, for that you will have to use _grub_.

### _Systemd-boot_
Install [systemd-boot](https://wiki.archlinux.org/title/Systemd-boot) in the _esp_ system partition
```shell
bootctl --path=/boot install
```

Add a [boot entry](https://wiki.archlinux.org/title/Systemd-boot#Adding_loaders) and load installed microcode updates
```shell
vim /boot/loader/entries/<entry>.conf
```
Add the following entry.  Repeat this, giving each conf file a different name for each kernel you want to boot.
```
title <Arch Linux>
linux /vmlinuz-linux
initrd /<cpu_manufacturer>-ucode.img
initrd /initramfs-linux.img
options root=/dev/<root_partition> rw quiet log-level=0
```

_*if a different kernel was installed such as linux-zen, you would add '-zen' above to 'vmlinuz-linux' and 'initframs-linux' lines. this will boot the system using the selected kernel. for more -- see [kernel paramters](https://wiki.archlinux.org/title/Kernel_parameters#systemd-boot)_

Edit [loader config](https://man.archlinux.org/man/loader.conf.5)
```shell
vim /boot/loader/loader.conf
```
Add the following lines
```
default <entry>.conf
timeout <3>
console-mode <max>
editor <no>
```

verify entry is bootable
```shell
bootctl list
```

----

### _GRUB_
install [grub](https://wiki.archlinux.org/title/GRUB). also, install  [os-prober](https://archlinux.org/packages/?name=os-prober) to automatically add boot entries for other operating systems
```shell
pacman -S grub os-prober
```
#### _UEFI_
for systems booted in uefi mode,
install [efibootmgr](https://wiki.archlinux.org/title/EFISTUB#efibootmgr)
```shell
pacman -S efibootmgr
```

and install grub to the efi partition
```shell
grub-install --target=x86_64-efi --efi-directory=/boot/grub --bootloader-id=GRUB
```

----

#### _BIOS_
othwerwise, if booted in bios mode; where path is the entire disk, not just a partition or path to a directory
install grub to the disk
```shell
grub-install --target=i386-pc /dev/sdx
```

----

generate the [grub config](https://wiki.archlinux.org/title/GRUB#Configuration) file. this will automatically detect the arch linux installation
```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

_* if a warning that os-prober will not be executed appears then, un-comment 'GRUB_DISABLE_OS_PROBER=false'_
```shell
nano /etc/default/grub
```
## Customization
This is a good time to do any other customizations such as installing other software or performing any configuration prior to the initial boot.

## Arch Linux Installation Complete :partying_face:
exit (ctrl+d) chroot
```shell
exit
```

unmount all partitions; check if busy
```shell
umount -R /mnt
```

reboot the system
```shell
reboot
```

## Appendix A - Resources  
[ArchWiki](https://wiki.archlinux.org/title/Installation_guide)  
[Surface Setup](https://github.com/linux-surface/linux-surface/wiki/Installation-and-Setup)

## Appendix B - Downloads  
[Arch Releases](https://archlinux.org/releng/releases/)