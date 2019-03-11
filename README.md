# Arch

## Prerequisitses:
- An USB drive with the Arch.iso.
- A computer booted into the USB drive.

## 1 - Keymaps:
Use `loadkeys` to set a keymap:
```
loadkeys no-latin1
```
The keymaps are stored under `/usr/share/kbd/keymaps/` if you want to check out your options.

## 2 - Partitioning:
Use `fdisk` to get an overview of availiable disks and their partitions:
```
fdisk -l
```
The following instructions will use `/dev/sda`, which already has three partitions:
```
/dev/sda1
/dev/sda2
/dev/sda3
```
We want to do a clean install, i.e. delete the partitions and create new ones.
### 2.1 - Deleting Partitions:
Use `cgdisk` to view current partitions on a volume:
```
cgdisk /dev/sda
```
Select everything that is not "free space" and select "delete". Finish with selecting "write" and then "quit".
### 2.2 - Create new partitions:
Use `parted` to set up the new partition table and partitions:
```
parted /dev/sda
mklabel gpt # This deletes all data on the disk!
mkpart ESP fat32 1049kB 538MB
set 1 boot on
mkpart primary ext4 538MB 112GB
mkpart primary linux_swap 112GB 100%
```
Type `print` and you should see something like this:
```
Model: ATA APPLE SSD SD0128 (scsi)
Disk /dev/sda: 121GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File System     Name    Flags
1       1049kB  538MB   537MB   fat32                   boot, esp
2       538MB   112GB   111GB   ext4
3       112GB   121GB   9331MB  linux-swap(v1)
```
Leave `parted` by typing `quit`.
### 2.3 Create file systems on the partitions:
```
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
```
### 2.4 Mount the partitions:
Mount the primary partition to `/mnt`, which is the Arch installer's default target location:
```
mount /dev/sda2 /mnt
```
Next, create a directory for the boot partition under `/mnt`:
```
mkdir -p /mnt/boot
```
The `-p` will create the `/mnt` directory if it doesn't already exist. Finally mount the boot partition to `/mnt/boot`:
```
mount /dev/sda1 /mnt/boot
```
### 2.5 - Installing Arch onto a partition:
Installing the base and base-devel packages of Arch onto `/mnt` is very simple. Note that this requires an network connection.
```
pacstrap /mnt base base-devel
```
This should take a while, depending on the network bandwidth.
### 2.5  - Optimize the partitions for SSDs:
Unix systems uses a file system table aka. an fstab file to determine how partitions should be initialized and generally integrated into the larger file system [[wikipedia]](https://www.wikiwand.com/en/Fstab) and in this case we want to optimize the partitions for SSD.

To generate the fstab file do:
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```
and edit it:
```
nano /mnt/etc/fstab
```
There is some variation in recommended fstab configurations. See the [Arch wiki on solid state drives](https://wiki.archlinux.org/index.php/Solid_state_drive#Partition_alignment) for a full description. 

Meanwhile, use the recommendations from [this guide](https://medium.com/@philpl/arch-linux-running-on-my-macbook-2ea525ebefe3). A different configuration for dual booting OSX and Arch can be found in [this guide](https://github.com/pandeiro/arch-on-air).

The fstab file should look something like this:
```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/sda2
UUID=0da9886f-450e-465b-8cc0-eee32d12cf03       /       ext4        rw,relatime,data=ordered,discard    0   2

# /dev/sda1
UUID=1E41-DFD2      /boot       vfat        rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro       0   1

# /dev/sda3
UUID=555a927a-6be7-4a8e-8aeb-a738aae1735b       none        swap        defaults        0   0
```
## 3 - Configuring the base system:
The following section covers basic configuration, including user + password, language and keymap.

Start by setting the new `/mnt` directory as the root directory:
```
arch-chroot /mnt
```
### 3.1 - Set language aka. locale:
Open the locale.gen file:
```
nano /etc/locale.gen
```
Uncomment the language you want the OS to run in, i.e. for english:
```
#en_SG ISO-8859-1
en_US.UTF-8 UTF-8
#en_US ISO-8859-1
```
Generate the new locale:
```
locale-gen
```
Apply the language for the current session:
```
export LANG=en_US.UTF-8
```
To make the language load on boot do:
```
echo LANG=en_US.UTF-8 > /etc/locale.conf
```
Remember that the first thing you did was setting the keymap. We want that to happen on boot so do:
```
nano /etc/vconsole.conf
```
And add:
```
KEYMAP=no-latin1
```
### 3.2 - Set the time-zone and enable clock:
You can look for availiable time-zones under `/usr/share/zoneinfo/`. In this case we are using Oslo, so do:
```
ln -s /usr/share/zoneinfo/Europe/Oslo /etc/localtime
```
If you get the "File exists" warning do:
```
rm -f /etc/localtime
```
and apply the time-zone again.

Finally, enable the hardware clock:
```
hwclock --systohc --utc
```
### 3.3 - Apple kernel modules:
For fan speed, temperature sensors etc we need a couple of hardware specific modules. For a MacBook, we need coretemp and applesmc.

Open the modules file:
```
nano /etc/modules
```
And add the two lines:
```
coretemp
applesmc
```
### 3.4 Naming the system:
Set the host-name by doing:
```
echo ninja_host > /etc/hostname
```
Open up the hosts file:
```
nano /etc/hosts
```
And either append your hostname to the existing lines or add the lines to make it look like:
```
127.0.0.1 localhost.localdomain localhost ninja_host
::1 localhost.localdomain localhost ninja_host
```
### 3.5 - Maintain the network connection:
Currently, we are running from an ethernet cable using the dhcpcd package with is part of the Archs base package group.

To maintain our connection, we want this to run at boot until we have a better network manager set-up.

Download dhcpcd and enable it on boot:
```
pacman -S dhcpcd
systemctl enable dhcpcd
```
### 3.6  - Set the admin password:
Set the passowrd by using the `passwd` utility:
```
passwd
```
## 4 - Install a bootloader:
There are many bootloaders, and this step will probably feature some alternatives in the future.

Remember that the boot partition was set up as a FAT-32 partition, so we need dosfstools to install a bootloader on it:
```
pacman -S dosfstools
```
Then install systemd-boot onto the boot partition:
```
bootctl --path=/boot install
```
Next, create a boot entry for Arch:
```
nano /boot/loader/entries/arch.conf
```
Again, the boot entry is configured according to [this guide](https://medium.com/@philpl/arch-linux-running-on-my-macbook-2ea525ebefe3), which promises some SSD optimization.

Add the folowing to arch.conf:
```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/sda2 rw elevator=deadline quiet splash resume=/dev/sda3 nmi_watchdog=0
```
Finally, set the config as the default boot configuration:
```
echo "default arch" > /boot/loader/loader.conf
```
That was it for base configuration. Reboot into Arch bo doing:
```
exit
reboot
```










