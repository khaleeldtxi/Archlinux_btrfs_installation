# Arch installation with btrfs filesystem 

Boot the live environment with USB  

You will be logged in on the first virtual console as the root user, and presented with a Zsh shell prompt.

# Verify the boot mode

If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:

`ls /sys/firmware/efi/efivars`

If the directory does not exist, the system may be booted in BIOS or CSM mode. Refer to your motherboard's manual for details.

# Connect to the internet

The connection may be verified with ping:

$ ping -c 3 www.google.co.in

Ensure your network interface is listed and enabled

$ ip link

If enp4s0 is down, type: 

$ ip link set up enp4s0


# Update the system clock 

Use timedatectl to ensure the system clock is accurate: 

$ timedatectl set-ntp true

To check the service status, use: timedatectl status 

  
# Partition the disks 

When recognized by the live system, disks are assigned to a block device such as /dev/sda or /dev/nvme0n1. To identify these devices, use lsblk or fdisk. 

$ fdisk -l 

Use fdisk, gdisk, parted, cfdisk or cgdisk to modify partition tables, for example cgdisk /dev/sdb. 

UEFI with GPT  


# Mount point  

![Screenshot_20191124_145857](https://user-images.githubusercontent.com/54496531/69492750-3b589600-0ecc-11ea-953b-035bc4c11412.png)


# Format the partitions 

Once the partitions have been created, each must be formatted with an appropriate file system. 

$ mkfs.btrfs -L “Arch” /dev/sda5

$ mkswap –L swap /dev/sda6

$ swapon /dev/sda6

  
# Create Btrfs subvolumes: 

After formatting the disk with btrfs, we can now proceed to create subvolumes as follows: 

$ mount /dev/sda5 /mnt

$ btrfs subvolume create /mnt/@

$ btrfs subvolume create /mnt/@home

$ btrfs subvolume create /mnt/@log

$ btrfs subvolume create /mnt/@pkg

$ btrfs subvolume create /mnt/@snapshots

  
# Mount subvolumes: 

$ umount /mnt

$ mount -o relatime,ssd,compress=zstd,space_cache=v2,commit=120,subvol=@ /dev/sda5 /mnt

$ mkdir -p /mnt/{boot/efi,home,var/log,var/cache/pacman/pkg,btrfs}

$ mount -o relatime,compress=zstd,space_cache=v2,commit=120,subvol=@home /dev/sda5 /mnt/home

$ mount -o relatime,compress=zstd,space_cache=v2,commit=120,subvol=@log /dev/sda5 /mnt/var/log

$ mount -o relatime,compress=zstd,space_cache=v2,commit=120,subvol=@pkg /dev/sda5 /mnt/var/cache/pacman/pkg/

$ mount -o relatime,compress=zstd,space_cache=v2,commit=120,subvolid=5 /dev/sda5 /mnt/btrfs

$ mount /dev/sda2 /mnt/boot/efi

  

# Check if partition mounted correctly: 

$ df –Th

$ free –h

  
# Install the base packages: 

# Use the pacstrap script to install the base package group: 

$ pacstrap /mnt base base-devel linux linux-firmware bash-completion btrfs-progs dosfstools grub efibootmgr sysfsutils usbutils e2fsprogs mtools inetutils dhcpcd nano less man-db man-pages texinfo

 
# Fstab 

$ genfstab -U -p /mnt >> /mnt/etc/fstab

# Check the resulting file in /mnt/etc/fstab afterwards, and edit it in case of errors: 

$ cat /mnt/etc/fstab

# change options to only have the following: 

rw,relatime,compress=lzo,space_cache=v2,subvol=@log

remove / mount point 

  

# Chroot 

# Change root into the new system:  

$ arch-chroot /mnt

  

# Time zone 

# Set the time zone:  

$ ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime 

# Run hwclock to generate /etc/adjtime:  

$ hwclock –systohc --utc

This command assumes the hardware clock is set to UTC.  

*** To dual boot with Windows it is recommended to configure Windows to use UTC, rather than Linux to use localtime. (Windows by default uses localtime 

You can do this from an Administrator Command Prompt in Windows 10 running:  

reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_QDWORD /f 

  
# Localization 

$ nano /etc/locale.gen

 
# Uncomment en_US.UTF-8 and other needed locales (en_IN.UTF-8) in /etc/locale.gen, and generate them with: 

$ locale-gen

  
# Create the locale.conf file, and set the LANG variable accordingly:  

$ nano /etc/locale.conf

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


# Check with #locale if correct locales set:

$ locale
  

# Network configuration 

Create the hostname file:  

$ nano /etc/hostname

archpc


# Alternatively, using hostnamectl:  

$ hostnamectl set-hostname archpc

 
# Add matching entries to hosts: 

$ nano /etc/hosts
  

127.0.0.1    localhost

::1        localhost archpc

127.0.1.1    myhostname.localdomain    archpc

  
*** Add arhcpc next to localhost on line# 2 & 3 *** 

If the system has a permanent IP address, it should be used instead of 127.0.1.1
  

$ systemctl enable dhcpcd@enp4s0.service


# Root password 

Set the root password:  

# passwd

  
# Boot loader 

Choose and install a Linux-capable boot loader. If you have an Intel or AMD CPU, enable microcode updates in addition.  

$ grub-mkconfig –o /boot/grub/grub.cfg

$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub

  

Or follow the steps mentioned in https://wiki.archlinux.org/index.php/REFInd 

$ pacman-S refind-efi

$ refind-install

  

Reboot and check if booting properly. 

 
If you see an error message about var log while shutting down: 

 

$ nano /etc/systemd/journald.conf

 
Add under: 

[Journal] 

Storage=volatile 

 

BTRFS: open_ctree failed 

As of November 2014 there seems to be a bug in systemd or mkinitcpio causing the following error on systems with multi-device Btrfs filesystem using the btrfs hook in mkinitcpio.conf:  

 

BTRFS: open_ctree failed 
mount: wrong fs type, bad option, bad superblock on /dev/sdb2, missing codepage or helper program, or other error 

In some cases useful info is found in syslog - try dmesg|tail or so. 

You are now being dropped into an emergency shell. 

 

A workaround is to remove btrfs from the HOOKS array in /etc/mkinitcpio.conf and instead add btrfs to the MODULES array. Then regenerate the initramfs with mkinitcpio -p linux (adjust the preset if needed) and reboot.  

 
# Configure Snapper

# Check if btrfs partition properly mounted: 
 

$ sudo btrfs sub list /

$ df -Th


# Install snapper:

$ sudo pacman -S snapper

# Create config 

$ sudo snapper -c root create-config /
 

We'll get another subvoulme named .snapshots, check it by running: 
 

$ sudo btrfs sub list /

This is a problem, as we don't need this created subvolme as it's below / directory 

So we'll delete this .snapshots subvolume, and manually create a new directory snapshots 


$ sudo btrfs sub del /.snapshots/

$ sudo mkdir /.snapshots

Create snapshot mount point n fstab:

$ sudo leafpad /etc/fstab
 

Copy first line (/home) then edit it to look like this: 

/dev/sdb4	

/.snapshots 
		

btrfs 
	

Rw,relatime,compress=zstd,space_cache=v2,subvol=@snapshots
	
0	

0 

 

$ mount /.snapshots/


Check if newly created snapshots directory is mounted as subvolume:

$ df –Th


$ sudo snapper list

(no snapshots)
 

# Download these packages: 

$ yay -S grub-btrfs

$ yay -S snap-pac
 

$ less /etc/grub.d/41_snaphots-btrfs
 

$ sudo leafpad /etc/default/grub
 

Add the following under GRUB_CMDLINE_LINUX=""
 

GRUB_BTRFS_CREATE_ONLY_HARMONIZED_ENTRIES="true"

GRUB_BTRFS_LIMIT="10"
 

Then save and close this file.
 

$ sudo leafpad /etc/snapper/configs/root 

NUMBER_CLEANUP="yes" 

NUMBER_MIN_AGE="0" 

NUMBER_LIMIT="12" 

NUMBER_LIMIT_IMPORTANT="3" 

TIMELINE_CREATE="no" 
 

Then save and close this file.


$ sudo systemctl start snapper-timeline.timer

$ sudo systemctl enable snapper-timeline.timer

$ sudo systemctl start snapper-cleanup.timer

$ sudo systemctl enable snapper-cleanup.timer

 

$ systemctl status cronie.service

 

$ yay -S snap-pac-grub

 

# Check snapshots created 
 

$ sudo snapper list



 
