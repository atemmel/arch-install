### Step 1: Disk preparation

Set keyboard layout:

```
loadkeys i386/qwerty/sv-latin1.map.gz
```

Check internet status:

```
ping google.com
```

If the ping is successful, you may proceed. If you need a wireless connection, use `iwctl` for the time being

```
iwctl
device list
station *device* scan
station *device* get-networks
station *device* connect *ssid*
```

Check drives:

```
lsblk
```

Partition drive(s):

```
cfdisk /dev/sdx
```

Where x is replaced by the device to partition.

Choose gpt, always ~~for hdd > 2TB or if using UEFI, else use DOS~~
(Source: https://unix.stackexchange.com/questions/235467/arch-linux-cfdisk-asking-for-disk-label-type)

Select free space, create partitions as needed.
(I don't do swapfiles.)
You need at least one primary partition intended for day-to-day-usage and one marked as boot (~512MB, set type to EFI).
When satisfied, press write, type yes and then exit `cfdisk`.
Repeat this process for all drives you wish to partition.

Check drives again, sdx1 and sdx2 should now appear:

```
lsblk
```

Create file systems:

```sh
# main partition
mkfs.ext4 /dev/sdxy

# boot partition
mkfs.fat -F32 /dev/sdxy
```

Where x is replaced by the device, and y is replaced by the partition number.

Mount drives:
```
# main partition
mount /dev/sdxy /mnt
# boot partition
mkdir /mnt/boot
mount /dev/sdxy
```


Run installation script:
```sh
pacstrap -i /mnt base base-devel linux linux-firmware
```

Go and make some chocolate, this might take a while.

Generate file system table:
```
genfstab -U /mnt >> /mnt/etc/fstab
```

### Step 2: The bare necessities

Change root into the new linux installation:
```
arch-chroot /mnt /bin/bash
```
Your prompt should now look a bit different.

Install vim with added bloat, completion for bash, and finally, git:
```
pacman -S nano bash-completion git iwd
```

Enable your locale(s) of choice:
```
nvim /etc/locale.gen
```
Uncomment the locale(s) you want to enable.
My locales include:
```
sv_SE.UTF-8 UTF-8
en_GB.UTF-8 UTF-8
```
Then generate them:
```
locale-gen
```
Set your primary language:
```sh
echo LANG=en_GB.UTF-8 > /etc/locale.conf
export LANG=en_GB.UTF-8
```

Find your language code:
```
find /usr/share/kbd/keymaps/
```
Alternatively, if you know what to look for:
```
find /usr/share/kbd/keymaps/ | grep "sv"
```
Make a mental note of it, and open vconsole.conf:
```
nano /etc/vconsole.conf
```
Add your keymap:
```
KEYMAP=sv-latin1
```

`localtime` is no longer created by the system

Symlink your desired timezone instead:
```
ln -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
```
Set hardware clock to system time:
```
hwclock --systohc
```
Set hostname:
```sh
# replace "Hostname" with what you wish to name your computer
echo Hostname > etc/hostname
```

Edit hosts:
```
nano /etc/hosts
```
When done, it should look something like this:
```
#	<ip-address>	<hostname.domain.org>	<hostname>
	127.0.0.1		localhost.localdomain	localhost	Hostname
	::1				localhost.localdomain	localhost	Hostname
```

Create ramdisk environment:
```
mkinitcpio -p linux
```
You should get two warnings about missing drivers.
Basically nobody owns that kind of hardware, so no need to worry about it.

Set root password:
```
passwd
```

Setup systemctl-boot
```
bootctl install
```

Create new boot entry (this file must not contain tabs, use spaces instead)
```
nano /boot/loader/entries/arch.conf
```
```
title 	Arch Linux
linux	/vmlinuz-linux
initrd	/initramfs-linux.img
options	root=/dev/sdxy rw
```

Edit bootloader to include default entry
```
nano /boot/loader/loader.conf
```
```
default arch-*
```

Now shutdown the pc:
```sh
exit
umount /dev/sdxy
umount /dev/sdxy
shutdown -P now
```

Before turning it on, make sure you've changed the boot media to the drive you marked as bootable.
Remove all bootable discs, usbs, etc.
If everything works as it's supposed to, you'll now be able to log in as the user root and the password you provided.

If it doesn't, mount the installation media again and jump back into the main partition:
```
mount /dev/sdxy /mnt
mount /dev/sdxy /mnt/boot
arch-chroot /mnt /bin/bash
```

### Step 3: Customize the system

(You may need to revisit iwctl again)

Create your user account and set password:
```sh
useradd -m -g users -G wheel,storage,power -s /bin/bash temmel
passwd temmel
```

Set up user rights:
```
EDITOR=nano visudo
```
Uncomment the line that looks like:
```
# %wheel ALL=(ALL) ALL
```
This allows all members of the wheel group to use all commands.

Now logout from the root account:
```sh
exit
```
Login with your new user, if you can run the following line everything works as intended:
```
sudo pacman -Syyu
```

From here, you'll want to import your dots:
```sh
git clone https://github.com/atemmel/dotfiles.git
git submodule update --init --recursive

# Copy the contents:
cp -r dotfiles/. .
# Clean up:
rm -rf .git dotfiles
```

Install basic software:
```
sudo pacman -S i3-gaps rofi xorg xorg-xinit kitty dunst mpd ncmpcpp neofetch feh pulseaudio pavucontrol zip unzip unrar firefox zathura
```

Download and install `yay` so we can install more software:
```sh
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ../;rm -rf aurman
```

Install graphics drivers: (vesa should be installed by default)
```
sudo pacman -S xf86-video-vesa

# If applicable
sudo pacman -S xf86-video-intel
sudo pacman -S nvidia
sudo nvidia-xconfig
```

Enter X:
```
startx
```

Set keyboard layout:
```sh
# Try out a layout:
setxkbmap -model pc104 -layout se
# Set consistent layout:
localectl --no-convert set-x11-keymap se pc104
```

### Optional: Startx on startup
Open `/etc/profile`
then append the following to the end of the file:

```sh
# Autostart systemd default session on tty1
if [[ "$(tty)" == '/dev/tty1' ]] ; then
	exec startx
fi
```
