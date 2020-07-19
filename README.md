# Arch installation with btrfs filesystem 

Boot the live environment with USB  

You will be logged in on the first virtual console as the root user, and presented with a Zsh shell prompt.

# Verify the boot mode

If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:

`ls /sys/firmware/efi/efivars`

If the directory does not exist, the system may be booted in BIOS or CSM mode. Refer to your motherboard's manual for details.

# Connect to the internet

The connection may be verified with ping:

`ping -c 3 www.google.co.in`

If no internet connection, ensure your network interface is listed and enabled

`ip link`

If enp4s0 (or whatever your network interface name is) is down, type: 

`ip link set up enp4s0`


# Update the system clock 

Use timedatectl to ensure the system clock is accurate: 

`timedatectl set-ntp true`

To check the service status, use:

`timedatectl status`

  
# Partition the disks 

When recognized by the live system, disks are assigned to a block device such as /dev/sda or /dev/nvme0n1. To identify these devices, use `lsblk` or `fdisk`. 

`fdisk -l`

Use fdisk, gdisk, parted, cfdisk or cgdisk to modify partition tables, for example 
`cgdisk /dev/sdb`

## UEFI with GPT  


### Mount point

| Mount point | Partition | Partition type | Suggested size |
|---|---|---|---|
| /mnt/boot | /dev/sda2 | EFI system partition | 1GB (if dual booting) |
| /mnt | /dev/sda4 | / |  |
| /mnt/home | /dev/sdb4 | /home |  |
| /mnt/var | /dev/sdb5 | var |  |
| /tmp | /dev/sdb6 | tmp |  |


# Format the partitions 

Once the partitions have been created, each must be formatted with an appropriate file system. 

`mkfs.btrfs -L “Arch” /dev/sda5`

  
# Create Btrfs subvolumes: 

After formatting the disk with btrfs, we can now proceed to create subvolumes as follows: 

`mount /dev/sda5 /mnt`

`btrfs su cr /mnt/@`

`btrfs su cr /mnt/@home`

`btrfs su cr /mnt/@var`

`btrfs su cr /mnt/@srv`

`btrfs su cr /mnt/@opt`

`btrfs su cr /mnt/@tmp`

`btrfs su cr /mnt/@swap`

`btrfs subvolume create /mnt/@log`

`btrfs subvolume create /mnt/@pkg`

`btrfs su cr /mnt/@snapshots`

  
# Mount subvolumes: 

`umount /mnt` 

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,ssd,subvol=@ /dev/sda5 /mnt`

`mkdir -p /mnt/{boot,home,var,srv,opt,tmp,.snapshots,btrfs}`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@home /dev/sdb4 /mnt/home`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@srv /dev/sdb5 /mnt/srv`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@opt /dev/sdb5 /mnt/opt`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvolid=5 /dev/sda5 /mnt/btrfs`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@tmp /dev/sda5 /mnt/tmp`

`mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@snapshots /dev/sda5 /mnt/.snapshots`

`mount -o nodatacow,subvol=@swap /dev/sda5 /mnt/swap`

`mount -o nodatacow,subvol=@var /dev/sda5 /mnt/var`

`mount /dev/sda2 /mnt/boot`

  
# Check if partition mounted correctly: 

`df –Th`

`free –h`

  
# Install the base packages: 

## Use the pacstrap script to install the base package group: 

`pacstrap /mnt base base-devel linux linux-headers linux-firmware bash-completion btrfs-progs dosfstools grub grub-btrfs efibootmgr sysfsutils usbutils e2fsprogs mtools inetutils dhcpcd nano less man-db man-pages texinfo vim git networkmanager network-manager-applet reflector bluex bluez-utils xdg-utils xdg-user-dirs snapper`

 
# Fstab 

`genfstab -U -p /mnt >> /mnt/etc/fstab`

## Check the resulting file in /mnt/etc/fstab afterwards, and edit it in case of errors: 

`cat /mnt/etc/fstab`

## change options to only have the following: 

rw,relatime,compress=lzo,space_cache=v2,subvol=@log

remove / mount point 


# Chroot 

Change root into the new system:  

`arch-chroot /mnt`

# Swap file

`truncate -s 0 /swap/swapfile`

`chattr +C /swap/swapfile`

`btrfs property set /swap/swapfile compression none`

`dd if=/dev/zero of=/swap/swapfile bs=4G count=2 statys=progress`

`chmod 600 /swap/swapfile`

`mkswap /swap/swapfile`

`swapon /swap/swapfile`

`nano /etc/fstab`

Enter swap file mount point in fstab: 

`/swap/swapfile none swap defaults 0 0`

# Time zone 

## Set the time zone:  

`ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime`

## Run hwclock to generate /etc/adjtime:  

`hwclock –systohc --utc`

This command assumes the hardware clock is set to UTC.


*** To dual boot with Windows it is recommended to configure Windows to use UTC, rather than Linux to use localtime. (Windows by default uses localtime 

You can do this from an Administrator Command Prompt in Windows 10 running:  

reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_QDWORD /f 

  
# Localization 

`nano /etc/locale.gen`

## Uncomment `en_US.UTF-8` and other needed locales (en_IN.UTF-8) in `/etc/locale.gen`, and generate them with: 

`locale-gen`

`echo LANG=en_US.UTF-8 >> /etc/locale.conf`

Check with `locale` if correct locales set.

## Verify the locale.conf file, and set the LANG variable accordingly:  

`nano /etc/locale.conf`

LANG=en_US.UTF-8
LC_MESSAGES=en_IN.UTF-8
LC_CTYPE=en_IN.UTF-8
LC_NUMERIC=en_IN.UTF-8
LC_TIME=en_IN.UTF-8
LC_COLLATE=en_IN.UTF-8
LC_MONETARY=en_IN.UTF-8
LC_PAPER=en_IN.UTF-8
LC_NAME=en_IN.UTF-8
LC_ADDRESS=en_IN.UTF-8
LC_TELEPHONE=en_IN.UTF-8
LC_MEASUREMENT=en_IN.UTF-8
LC_IDENTIFICATION=en_IN.UTF-8
  

# Network configuration 

Create the hostname file:  

`nano /etc/hostname`

archpc


# Alternatively, using hostnamectl:  

`hostnamectl set-hostname archpc`

 
# Add matching entries to hosts: 

`nano /etc/hosts`
  

|127.0.0.1| localhost | 
|---|---|---|
|::1| localhost | archpc |
|127.0.1.1 | myhostname.localdomain | archpc |

  
Add arhcpc next to localhost on line# 2 & 3

If the system has a permanent IP address, it should be used instead of 127.0.1.1


# Root password 

Set the root password:  

`passwd`

`nano /etc/mkinitcpio.conf`

Add in : MODULES=(btrfs) 

`mkinitcpio -p linux`
  
# Boot loader 

Choose and install a Linux-capable boot loader. If you have an Intel or AMD CPU, enable microcode updates in addition.  

`grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`

`grub-mkconfig –o /boot/grub/grub.cfg`

Reboot and check if booting properly. 

 
If you see an error message about var log while shutting down: 

`nano /etc/systemd/journald.conf`

Add under: 

[Journal] 

Storage=volatile
 
# Enable services

`systemctl enable NetworkManager`

# Add User

`useradd -mG wheel khaleel`

`passwd khaleel`

`EDITIOR=nano visudo`

uncomment line - wheel ALL=(ALL) ALL

`pacman -S openssh`

`systemctl enable sshd`

`sudo btrfs subvol list -p /`

`sudo pacman -S nvidia nvidia-utils nvidia-settings xorg-server-devel plasma-meta sddm dolphin kate`

`exit`

`umount -a`

`reboot`

# Install yay

`sudo pacman -S git`

`git clone https://aur.archlinux.org/yay.git`

`cd yay`

`makepkg -sirc`


 
# Configure Snapper

# Check if btrfs partition properly mounted:

`sudo btrfs sub list /`

`df -Th`

`sudo umount /.snapshots/`

`sudo rm -rf /.snapshots/`


# Install snapper (if not already installed while installing Arch):

`sudo pacman -S snapper`

# Create config

 `sudo snapper -c root create-config /`

`kate /etc/snapper/configs/root`

`sudo chmod a+rx /.snapshots/`

`yay -S snap-pac`

`kate /etc/default/grub` 

Add the following under `GRUB_CMDLINE_LINUX=""`
 
`GRUB_BTRFS_CREATE_ONLY_HARMONIZED_ENTRIES="true"`

`GRUB_BTRFS_LIMIT="10"`

Then save and close this file.

`kate /etc/snapper/configs/root`

`NUMBER_CLEANUP="yes"`

`NUMBER_MIN_AGE="0"`

`NUMBER_LIMIT="12"`

`NUMBER_LIMIT_IMPORTANT="3"`

`TIMELINE_CREATE="no"`

Then save and close this file.

`sudo systemctl start snapper-timeline.timer`

`sudo systemctl enable snapper-timeline.timer`

`sudo systemctl start snapper-cleanup.timer`

`sudo systemctl enable snapper-cleanup.timer`
 

# Check snapshots created
 
`sudo snapper list`

Install any app

Reboot & check if grub has snapshot entry for pre & post installations.
