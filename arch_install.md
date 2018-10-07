### Step 1: Disk preparation

Set keyboard layout:

```
loadkeys i386/qwerty/sv-latin1.map.gz
```

Check internet status:

ping google.com

If the ping is successful, you may proceed. If you need a wireless connection, use networkmanager for the time being

Check drives:

```
lsblk
```

Partition drive(s):

```
cfdisk /dev/sdx
```

Where x is replaced by the device to partition.

Choose gpt for hdd > 2TB or if using UEFI, else use DOS
(Source: https://unix.stackexchange.com/questions/235467/arch-linux-cfdisk-asking-for-disk-label-type)

Select free space, create partitions as needed.
(I don't do swapfiles.)
You need at least one primary partition with the bootable flag and type set to Linux.
When satisfied, press write, type yes and then exit `cfdisk`.
Repeat this process for all drives you wish to partition.

Check drives again, sdx1 should now appear:

```
lsblk
```

Create file systems:

```
mkfs.ext4 /dev/sdxy
```

Where x is replaced by the device, and y is replaced by the partition number.

Create and enable swap (if appropriate):

```
mkswap /dev/sdxy
swapon /dev/sdxy
```

Mount main drive:
```
mount /dev/sdxy /mnt
```

Run installation script:
```
pacstrap -i /mnt base base-devel
```

Select `systemd-resolv` if you get the option to do so.

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
pacman -S neovim bash-completition git
```

Enable your locale(s) of choice:
```
nvim /etc/locale.gen
```
Uncomment the locale(s) you want to enable.
My locales include:
```
sv_SE.UTF8 UTF-8
en_GB.UTF8 UTF-8
```
Then generate them:
```
locale-gen
```
Set your primary language:
```
echo LANG=en_GB.UTF8 > /etc/locale.conf
export LANG=en_GB.UTF8
```

Find your language code:
```
find /usr/share/kbd/keymaps/ type f
```
Alternatively, if you know what to look for:
```
find /usr/share/kbd/keymaps/ type f | grep "sv"
```
Make a mental note of it, and open vconsole.conf:
```
nvim /etc/vconsole.conf
```
Add your keymap:
```
KEYMAP=sv-latin1
```

Remove localtime:
```
rm /etc/localtime
```
Symlink your desired timezone instead:
```
ln -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
```
Set hardware clock to system time:
```
hwclock --systohc
```
Set hostname:
```
echo Hostname > etc/hostname
```
Edit hosts:
```
nvim /etc/hosts
```
When done, it should look something like this:
```
#	<ip-address>	<hostname.domain.org>	<hostname>
	127.0.0.1		localhost.localdomain	localhost	Hostname
	::1				localhost.localdomain	localhost	Hostname
```

This is where you install your network manager of choice, for me, that is `connman`.
Miss this step and you'll have to `chroot` in again unless you got a wired connection.
```
pacman -S wpa-supplicant
```
I need to install this beforehand as a wifi dependency for connman
```
pacman -S connman
```
If you are met with a invalid or corrupted package, execute:
```sh
pacman-key --list-sigs Lastname # pipe this through grep once you know their name
```
Find the appropriate user's key and delete it:
```
pacman-key --delete XXXXXXXXXXXXXXXX
```
Then rebuild the keyring:
```
pacman-key --populate archlinux
```
And then, try installing the network manager once more.

Once the manager is installed, you need to enable it with systemctl. First find the name of the service with:
```
systemctl list-unit-files
```
And then enable it if it wasn't already:
```
systemctl enable connman.service
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

Install grub and os prober:
```
pacman -S grub os-prober
```
Run grub-install on your boot drive:
```
grub-install --target=i386-pc --recheck /dev/sdx
```
If you recieve 'Installation finished. No error reported.' you're golden.

Create grub config:
```
grub-mkconfig -o /boot/grub/grub.conf
```
Here you'll get a warning `'Failed to connect to lvmetad'`.
This is because you ran grub from `chroot`.
Once again, no need to worry.

Now shutdown the pc:
```
umount /dev/sdxy
exit
shutdown -P now
```

Before turning it on, make sure you've changed the boot media to the drive you marked as bootable.
Remove all bootable discs, usbs, etc.
If everything works as it's supposed to, you'll now be able to log in as the user root and the password you provided.

If it doesn't, mount the installation media again and jump back into the main partition:
```
mount /dev/sdxy /mnt
arch-chroot /mnt /bin/bash
```
Rerun the last two `grub` steps. It should work now.

### Step 3: Customize the system

Create your user account and set password:
```sh
useradd -m -g users -G wheel,storage,power -s /bin/bash temmel
passwd temmel
```

Set up user rights:
```
visudo
```
Uncomment the line that looks like:
```
# %wheel ALL=(ALL) ALL
```
This allows all members of the wheel group to use all commands.

Now logout from the root account:
```
exit
```
Login with your new user, if you can run the following line everything works as intended:
```
sudo pacman -Syyu
```

From here, you'll want to import your dots:
```sh
git clone https://github.com/atemmel/dotfiles.git

# Copy the contents:
cp -R dotfiles/. .
# Clean up:
rm -rf .git dotfiles
```

Install basic software:
```
sudo pacman -S i3-gaps rofi xorg xorg-xinit rxvt-unicode compton dunst mpd ncmpcpp ranger neofetch youtube-dl
```

Download and install `aurman` so we can install more software:
```sh
git clone https://aur.archlinux.org/aurman.git
cd aurman
gpg --recv-keys 465022E743D71E39
makepkg -si
cd ../;rm -rf aurman
```

Install graphics drivers: (vesa should be installed by default)
```
sudo pacman -S xf86-video-vesa
```
Also install `xf86-video-intel` and `xf86-video-nouveau` if applicable.

Download and install fonts:
```
# Enable bitmap fonts:
sudo rm /etc/fonts/conf.d/10*

# Download the main font:
git clone https://github.com/NerdyPepper/scientifica.git

cp scientifica/regular/scientifica-11.bdf  .local/share/fonts/
cp scientifica/bold/scientificaBold-11.bdf .local/share/fonts/

# Rebuild font cache:
fc-cache
# Check that it is installed with:
fc-list | grep scientifica

# Now for the icon font:
aurman -S nerd-fonts-complete
# This is a 2 GB install, might take some time.
```

Install more software:
```
aurman -S python-pywal flameshot polybar pyenv
```

Enter X:
```
startx
```

Set keyboard layout:
```
# Try out a layout:
setxkbmap -model pc104 -layout se
# Set consistent layout:
localectl --no-convert set-x11-keymap se pc104
```


Install `discord`:
```
aurman -S discord
```
`libc++abi` may or may not install at first attempt, the key is sometimes hard to get verified. (No idea why this is.)
Stay persistent, and try again later, this is why I install `discord` in a separate command.
Furthermore, `libc++abi` runs almost 6K tests on installation, meaning it can take a while to install. Do this last.

If it absolutely refuses to work, clone it and build it without tests:
```sh
git clone https://aur.archlinux.org/libc++.git
cd libc++
makepkg -si --nocheck
cd ../
rm -rf libc++

# Now try installing it again:
aurman -S discord
```

### Todo: urxvt resize font