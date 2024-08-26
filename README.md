# arch_x13s
How to install Arch on a Lenovo x13s


1. Boot from USB stick
2. Change keyboard layout: `loadkeys de-latin1`
3. Connect to WiF
   1. iwctl
   2. station wlan0 connect <your-wifi-ssid>
   3. exit
4. Set the time: `timedatectl set-ntp true`
5. Setup pacman
   1. `nano /etc/pacman.conf` uncomment line: #ParallelDownloads = 5 -> ParallelDownloads = 5
   2. `nano /etc/pacman.d/mirrorlist` - uncomment a mirror, the automatic mirror selection sucks sometimen :)
6. setup the repo: `ironrobin-setup`
7. setup the partitions:
   1. Create a partition layout in Windows - shrink your windows partition, leaves an empty one
   2. `fdisk -l` - choose an empty one
   3. or: `fdisk` - `n` - `enter`- `enter `- `w`
8. `mkfs.ext4 /dev/nvme0n1p7` (change for your empty partition!)
9. mount the filesystems:
   1.  `mount /dev/nvme0n1p7 /mnt`
   2.  `mount --mkdir /dev/nvme0n1p1/ /mnt/boot` 
       1.  this mounts the EFI Partition to the new filesystem 
       2.  this is where GRUB boots the linux kernel and the initram! 
       3.  make sure, it has enough space
10. Bootstrap your mounted system 
    1.  `pacstrap -K /mnt base linux-x13s alsa-ucm-conf grub git vim arch-install-scripts efibootmgr networkmanager network-manager-applet dialog os-prober mtools dosfstools base-devel archlinuxarm-keyring`
    2.  `cp /etc/pacman.conf /mnt/etc/pacman.conf`
    3.  `genfstab -U /mnt >> /mnt/etc/fstab`
    4.  ``
    5.  ``
    6.  ``
    7.  
11. Hop into the chroot, trust my keys again, and install a bootloader
    1. `arch-chroot /mnt`
    2. Repeat steps in step 8 to trust my gpg key
    3.  Give root user a password using passwd command. I am choosing root so the credentials are root/root
    4.   Install grub: `grub-install --target=arm64-efi --efi-directory=/boot --bootloader-id=GRUB`
    5.   `nano /etc/default/grub` Here we use vim to modify the grub default config:
         1.  Edit `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 efi=noruntime clk_ignore_unused pd_ignore_unused arm64.nopauth`
         2.  Add `GRUB_EARLY_INITRD_LINUX_CUSTOM=initramfs-linux-x13s.img` so the config generator can find the initrd image
         3.  For dual boot, uncomment `GRUB_DISABLE_OS_PROBER=false`
     6.  Generate grub config file `grub-mkconfig -o /boot/grub/grub.cfg` 
     7.  Output should look like:
```bash
        Generating grub configuration file ...
        Found linux image: /boot/vmlinuz-linux
        Found initrd image: /boot/initramfs-linux-x13s.img
```
    8. small changes on the system
       1. `locale-gen` 
       2. Create the locale.conf(5) file, and set the LANG variable accordingly:
   ```bash
   /etc/locale.gen
de_DE.UTF-8 UTF-8
```
```bash
/etc/vconsole.conf
KEYMAP=de-latin1
```
```bash
/etc/hostname
yourhostname
```
13.  Now take out the USB stick and reboot into your root partition. login is user: root password: root
    1.  `cd /opt`
    2.  Connect to internet `sudo systemctl enable --now NetworkManager` and `nmtui` to choose your Wi-Fi connection
    3.  Sync datetime `timedatectl set-ntp true`

14. install gnome & gdm
    1.  `pacman -S gnome gdm`
        1.  I had to "CTRL-C" on the appstream cache step
    2.  `systemctl enable gdm` `sudo systemctl start gdm`
15. reboot!





## Enable Bluetoott

### Bluetooth
working as of [kernel 6.2.0-rc7](https://github.com/ironrobin/x13s-alarm/tree/trunk/linux-x13s), install [firmware](https://github.com/ironrobin/x13s-alarm/tree/trunk/x13s-firmware) and set an address with btmgmt. e.g., `sudo btmgmt public-addr F4:A8:0D:30:A3:47`

To make it just work, copy this file to `/etc/systemd/system/bluetooth.service.d/override.conf`
(It is courteous to use your own public-addr, put a random string of numbers to make it unique.)

```
[Service]
ExecStartPre=/bin/bash -c 'sleep 5 && yes | btmgmt public-addr 00:24:81:17:62:36'
# Blank ExecStart line to clear the service file, overrides are additive.
ExecStart=
ExecStart=/usr/lib/bluetooth/bluetoothd
```
