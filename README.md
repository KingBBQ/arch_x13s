# arch_x13s - How to install Arch on a Lenovo x13s. 
Using Arch has several advantages, such as better support for hardware (display drivers, etc.) and the ability to install newer kernels.

It took me quite some time to get this up and running, find my collected brain dump here!

# Step by step instructions

1. Boot from USB stick
   1. Get your ISO and instructions here: https://github.com/ironrobin/archiso-x13s
2. Change keyboard layout: `loadkeys de-latin1`
3. Connect to WiF
   1. `iwctl`
   2. `station wlan0 connect <your-wifi-ssid>`
   3. `exit`
4. Set the time: `timedatectl set-ntp true` - this is needed, so pacman can verify the keys...
5. Setup pacman
   1. `nano /etc/pacman.conf` uncomment line: 
      1. `#ParallelDownloads = 5 -> ParallelDownloads = 5` - much faster downloads 
   2. `nano /etc/pacman.d/mirrorlist` - uncomment a mirror, the automatic mirror selection sucks sometimes
6. setup the repo: `ironrobin-setup`
7. setup the partitions:
   1. Create a partition layout in Windows - shrink your windows partition, leaves an empty one
   2. `fdisk -l` - choose an empty one
   3. or: `fdisk` - `n` - `enter`- `enter `- `w`
8. `mkfs.ext4 /dev/nvme0n1p7` (**change for your empty partition!**)
9.  mount the filesystems:
   1.  `mount /dev/nvme0n1p7 /mnt`
   2.  `mount --mkdir /dev/nvme0n1p1/ /mnt/boot` 
       1.  this mounts the EFI Partition to the new filesystem 
       2.  this is where GRUB boots the linux kernel and the initram! 
       3.  make sure, it has enough space - the partition is quite small by default and can only hold one kernel image.
10. Bootstrap your mounted system 
    1.  `pacstrap -K /mnt base linux-x13s alsa-ucm-conf grub git vim arch-install-scripts efibootmgr networkmanager network-manager-applet dialog os-prober mtools dosfstools base-devel archlinuxarm-keyring`
    2.  `cp /etc/pacman.conf /mnt/etc/pacman.conf`
    3.  `genfstab -U /mnt >> /mnt/etc/fstab` - generate the mapping for the filesystem, the -U flag uses UUIDs
11. Hop into the chroot, trust my keys again, and install a bootloader
    1. `arch-chroot /mnt` 
    2. Repeat these steps to trust the ironrobin pgp keys: 
       1. `sudo pacman-key --recv-keys 6ED02751500A833A`
       2. `sudo pacman-key --lsign-key 6ED02751500A833A`
    3.  Give root user a password using passwd command. I am choosing root so the credentials are root/root
    4.   Install grub: `grub-install --target=arm64-efi --efi-directory=/boot --bootloader-id=GRUB`
    5.   `nano /etc/default/grub` Here we use vim to modify the grub default config:
         1.  Edit `GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 efi=noruntime clk_ignore_unused pd_ignore_unused arm64.nopauth`
         2.  Add `GRUB_EARLY_INITRD_LINUX_CUSTOM=initramfs-linux-x13s.img` so the config generator can find the initrd image
         3.  For dual boot, uncomment `GRUB_DISABLE_OS_PROBER=false`
     6.  Generate grub config file `grub-mkconfig -o /boot/grub/grub.cfg` 
     7.  Output should look like:
        `Generating grub configuration file ...`
        `Found linux image: /boot/vmlinuz-linux`
        `Found initrd image: /boot/initramfs-linux-x13s.img`
  
12.  Now take out the USB stick and reboot into your root partition. login is user: root password: root
    1.  `cd /opt`
    2.  Connect to internet `sudo systemctl enable --now NetworkManager` and `nmtui` to choose your Wi-Fi connection
    3.  Sync datetime `timedatectl set-ntp true`

13. install gnome & gdm
    1.  `pacman -S gnome gdm`
        1.  I had to "CTRL-C" on the appstream cache step
    2.  `systemctl enable gdm` `sudo systemctl start gdm`
14. reboot!

# Additional Hardware & drivers

## Install proper video drivers

This makes use of a better video driver - speeds up the video output (Minecraft > 60 fps) and adds support for 60hz 4k display on external monitors. 

for example: https://gitlab.com/TheOneWithTheBraid/sc8280xp-alarm/

```bash
git clone https://gitlab.com/TheOneWithTheBraid/sc8280xp-alarm/
cd sc8280xp-alarm/sc8280xp-firmware/
makepkg -si # build & install
cd linux-x13s/
makepkg -si # build & install
```


## Fingerprint reader

The driver works by default, all you need is the support in arch:

https://wiki.archlinux.org/title/Fprint

```bash
sudo pacman -S fprintd imagemagick
```

reboot and add your finger print on the users section in settings. 

## Enable Bluetooth

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
