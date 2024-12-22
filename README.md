# Install Artix Linux to a machine with UEFI

1. Check if your machine uses UEFI:

```
ls /sys/firmware/efi/efivars
```

2. Connect to the Internet using wpa_supplicant

```
// Check for available network interfaces (search for wlan0 if using wifi)
ip link

// Create a config file for wpa_supplicant.conf specifying
nano /etc/wpa_supplicant/wpa_supplicant.conf

// Add the following to the wpa_supplicant.conf

network = {
  ssid="your_SSID"
  psk="your_pw"
}

// Unblock all wireless devices on you pc
rfkill unblock all

// Activate connection using wpa_supplicant
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf

dhcpcd

// verify the connection by pinging to a website
ping -c 4 www.google.com
```

3. '''lsblk''' to check the disk for installing the OS

4. Format the disk

```
cfdisk /dev/nvme0n1
```

- Create three partitions:

Boot partition (/dev/nvme0n1p1): 1GB / Type - EFI system 
Root partition (/dev/nvme0n1p2): 30GB / Type - Linux filesystem
Home partition (/dev/nvme0n1p3): Remaining space / Type - Linux filesystem

Additionally may consider creating a swap partition

5. Put filesystems on the created partitions

```
// Format the root partition to fat for UEFI systems
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

6. Mount the partitions

```
// Mount root  partition to /mnt
mount /dev/nvme0n1p2 /mnt
// Make directories to mount home and boot partitions
mkdir /mnt/home
mkdir /mnt/boot
mount /dev/nvme0n1p3 /mnt/home
mount /dev/nvme0n1p1 /mnt/boot
```

7. Install the OS

```
// Install efibootmgr only for UEFI machines
basestrap -i /mnt base base-devel runit elogind-runit linux linux-firmware grub networkmanager vim neovim networkmanager-runit cryptsetup lvm2 lvm2-runit efibootmgr
```

8. Now the OS is installed but we can't boot into it. So, we have to install bootloader. When our system reboots we have to tell it how to remount the partitions. 

- The command for this is ```fstabgen```
- If you run it and give it a location it generates a file which tells you how to mount everything when you start. 

```
fstabgen -U /mnt >> /mnt/etc/fstab
```

- The machine reads this file on start

9. Change root directory to the newly installed system

```
artix-chroot /mnt bash
```

10. Set the time zone

```
// Check available locations
ln -s /usr/share/zoneinfo
// Set the timezone
ln -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime
// Check if the time zone was set correctly
ls -l /etc/localtime
```

11. Sync clock: ```hwclock --systohc```

12. Generate locales

- Check your locale in locale.gen file and uncomment it

```
vim /etc/locale.gen
```

- Open the locale.conf file

```
vim /etc/locale.conf
```

- Add the following to the locale file

```
export LANG="en_US.UTF-8"
export LC_COLLATE="C"
```

```
locale-gen
```

13. Set hostname

```
echo "lada" > /etc/hostname
vim /etc/hosts
```

- Add the following to /etc/hosts file

```
127.0.0.1  localhost
::1        localhost
127.0.1.1  lada.localdomain lada
```

14. Enable networkmanager to automatically start on boot up

```
ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/current
```

15. Create a password: ```passwd```

16. Add a user

```
useradd -G wheel -m lada
passwd lada
```

17. Additionally install dhcpcd and wpa_supplicant to manage internet connection

```
pacman -S dhcpcd wpa_supplicant
```

18. Configure grub

```
// Below is the command only for UEFI systems
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub

grub-mkconfig -o /boot/grub/grub.cfg
```

19. Exit and reboot

```
exit (ctrl+D)
reboot
```

20. After reboot install LARBS

```
su
curl -LO larbs.xyz/larbs.sh
sh larbs.sh
```

- ctrl+D again and login as the new user generated during LARBS install

```
// To start X11 session
startx
```

****

## TODO

* Customize the installation of LARBS by adding more programs I need for basic install

