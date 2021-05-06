# Arch Linux installation on Dell XPS 13 (9360) with UEFI and disk encryption
This document details all the steps necessary to get Arch Linux up and running on a brand new Dell XPS 13 (9360). The specs for this machine are as follows:
- Intel i5-8250U (Kaby Lake)
- 8GB 1866 MHz DDR3 RAM
- 256 GB NMVe SSD
- Killer 1535 network adapter
- 13.3 inch FHD monitor

The model in question is a European one with a UK keyboard, so please adjust various commands (`locale-gen`, `loadkeys`, ...) to your personal requirements.

The general choices made for this install are as follows:
- The "LVM on LUKS" encryption style will be used.
    - This means there will be a single encrypted partition with multiple logical volumes on top, representing `/`, `/home` and `swap`, as well as a separate, unencrypted partition for `/boot`.
- rEFInd will be used as a boot manager.
- The boot process will use the `systemd` boot style. This will affect how `mkinitcpio.conf` is configured.
- X11 display stack.
- PulseAudio audio stack.

## Resources
- https://wiki.archlinux.org/index.php/Installation_guide
- https://wiki.archlinux.org/index.php/Dell_XPS_13_(9360)
- https://github.com/kelp/arch-matebook-x-pro/blob/master/README.md
- https://gitlab.com/cryptsetup/cryptsetup/wikis/FrequentlyAskedQuestions

## Preliminary steps
### Update the BIOS
First things first. Boot into Windows, download the latest BIOS from Dell's website, run the installer. **Do not trash your Windows install before doing this!!**

### Important BIOS settings
**NOTE:** Do not do this before booting into Windows, as the pre-installed version that comes with the computer won't boot.

Hold down F2 at boot to enter the BIOS. From there:
- `Secure Boot` > `Secure Boot Enable: Disabled`
- `System Configuration` > `SATA Operation: AHCI`

### Create bootable media
Download an [Arch ISO](https://www.archlinux.org/download) and find a suitable USB stick. These commands can be run on Mac OS in order to burn the ISO.
```
$ diskutil unmountDisk /dev/disk2
$ sudo dd if=/path/to/iso of=/dev/sdX bs=1m status=progress
```

## Create a base installation
### Boot into removable media
Hold down F12 at boot to enter the boot menu. Choose `EFI USB Device`.

### Make TTY font readable and set keyboard layout
Substitute an appropriate value from `/usr/share/kbd/keymaps/i386`.
```
# setfont Lat2-Terminus16
# loadkeys uk
```

### Get online and update the system clock
We'll install the network manager later, for now we can use the live USB's built-in utility to connect to a wireless network.
```
# wifi-menu
```

Quickly check that your connection works.
```
# ping -c 3 www.archlinux.org
```

Tell the system clock to get its time from the web.
```
# timedatectl set-ntp true
```

### Wipe the disk
We're going to be using the [LVM on LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) encryption style, which means we only need to create a boot partition, and a separate partition that will hold all of our LVM volumes.

First, we prep the drive as described [here](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_wipe_on_an_empty_disk_or_partition), by creating a temporary encrypted container and writing random numbers to it. This makes it so that unused space on the drive can't easily be told apart from the encrypted data.

Create the container and write random numbers to it:
```
# cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped
```
Go make a cup of coffee. When it's done, you can use `lsblk` here to check that it was created correctly.

Write zeroes all over the container:
```
# dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress
```

Finally, close the container:
```
# cryptsetup close to_be_wiped
```

### Partition the disk
Now, create a 512 MB EFI boot partition, and make the rest a Linux Filesystem for use with LVM later.

```
# gdisk /dev/nvme0n1
```

```
GPT fdisk (gdisk) version 1.0.1

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1):
First sector (34-500118192, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-500118192, default = 500118192) or {+-}size{KMGTP}: +512M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): EF00
Changed type of partition to 'EFI System'

Command (? for help): n
Partition number (2-128, default 2):
First sector (34-500118192, default = 1050624) or {+-}size{KMGTP}:
Last sector (1050624-500118192, default = 500118192) or {+-}size{KMGTP}:
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8e00
Changed type of partition to 'Linux LVM'

Command (? for help): p
Disk /dev/sda: 500118192 sectors, 238.5 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 5680395C-3F93-4B81-944F-A3E68AAABD2E
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 500118192
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI System
   2         1050624       500118192   238.5 GiB   8e00  Linux LVM

Command (? for help): w
```

There should now be two partitions on the disk:
```
nvme0n1
├─nvme0n1p1   # EFI System (boot partition)
├─nvme0n1p2   # Linux LVM (LVM partition)
```

### Set up disk encryption
Here we create a LUKS encryption container on our "Linux" partition (`nvme0n1p2`) and use LVM to create root and swap volumes on top, as well as any other ones we may want. I personally like to keep `/home` on a separate volume.

First, create a LUKS container:
```
# cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
```

Open the container and name it (`cryptlvm` in this example):
```
# cryptsetup open /dev/nvme0n1p2 cryptlvm
```

Create a physical volume on top of the opened container:
```
# pvcreate /dev/mapper/cryptlvm
```

Create an LVM volume group (I name it `archvg`) and add the physical volume to it:
```
# vgcreate archvg /dev/mapper/cryptlvm
```

Next, create your standard Linux partitions. Make sure SWAP is as big as your RAM so you can hibernate safely. Note that the `-L` flag specifies a fixed size, whereas `-l` takes a percent.
```
# lvcreate -L 8G archvg -n swap
# lvcreate -L 32G archvg -n root
# lvcreate -l 100%FREE archvg -n home
```

Format the partitions accordingly:
```
# mkfs.ext4 /dev/archvg/root -L root
# mkfs.ext4 /dev/archvg/home -L home
# mkfs.fat -F32 /dev/nvme0n1p1
# mkswap /dev/archvg/swap
```

Mount the filesystems and activate the swap partition:
```
# mkdir -p /mnt/{home,boot}
# mount /dev/archvg/root /mnt
# mount /dev/archvg/home /mnt/home
# mount /dev/nvme0n1p1 /mnt/boot
# swapon /dev/archvg/swap
```

Partitioning is now finished. We can move on to installing stuff.

### Select mirrors
Picking mirrors closer to you will greatly speed up the installation of packages. To make things easier, install the `pacman-contrib` package:
```
# pacman -S pacman-contrib
```

This provides the `/usr/bin/rankmirrors` script, which does as it says.

Make a backup of your mirror list:
```
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Then, use `rankmirrors` to populate a new list:
```
# rankmirrors -n 20 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

### Install base system
```
# pacstrap /mnt base base-devel
```

### Generate fstab
The `-p` flag skips pseudo-filesystems.
```
# genfstab -pU /mnt >> /mnt/etc/fstab
```

```
# cat /etc/fstab

# /dev/archvg/root
UUID=3fe9c0a6-d2a1-4523-b235-877b0456ba1d       /               ext4            rw,relatime 0 1

# /dev/nvme0n1p1
UUID=70C7-C498          /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro   0 2

# /dev/archvg/home
UUID=8ba422b5-0fa0-4a73-8ecf-6800a3b629f9       /home           ext4            rw,relatime 0 2

# /dev/archvg/swap
UUID=756a6a08-29eb-448d-974e-e20c3a8f0689       none            swap            defaults,pri=-2     0 0
```

### chroot into new system
```
# arch-chroot /mnt
```

You are now inside your fresh new install. Just a couple of steps before we can remove the USB stick and boot from the SSD.

### System time, locale and hostname
Set timezone and hardware clock:
```
# ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
# hwclock --systohc --utc
```

Uncomment `en_GB.UTF-8 UTF-8` in `/etc/locale.gen` and generate:
```
# locale-gen
# echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

Set console keyboard layout and font permanently (note `>>` to append).
```
# echo CHARMAP=UTF-8 > /etc/vconsole.conf
# echo KEYMAP=uk >> /etc/vconsole.conf
# echo FONT=ter-v18n >> /etc/vconsole.conf
```

Set hostname:
```
# echo $HOSTNAME > /etc/hostname
```

Add matching entries to `/etc/hosts`:
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	$hostname.localdomain	$HOSTNAME
```

### Install boot manager
**NOTE: Do not fuck this up. This and the next step are the most important part of the installation procedure. If you get it wrong, your system won't boot.**

First, install Intel Microcode Updates, which will create an `initrd` image that we'll add to the boot manager config:
```
# pacman -S intel-ucode
```

Now install the actual manager:
```
# pacman -S refind-efi parted sbsigntools imagemagick
# refind-install
```

Get the UUIDs of the volumes we made earlier, to use in rEFInd's config:
```
# lsblk -f

or

# blkid /dev/nvme0n1p1
# blkid /dev/nvme0n1p2
```

Paste the following into `/boot/EFI/refind/refind.conf`, substituting the two UUIDs where noted:
```
use_graphics_for linux
...
showtools shell, memtest, gdisk, mok_tool, about, shutdown, reboot, firmware, fwupdate
...
scanfor manual
...
menuentry "Arch Linux" {
    icon     /EFI/refind/icons/os_arch.png
    volume   <boot-partition-UUID>
    loader   /vmlinuz-linux
    initrd   /initramfs-linux.img
    options  "rd.luks.name=<luks-partition-UUID>=cryptlvm root=/dev/archvg/root resume=/dev/archvg/swap initrd=/intel-ucode.img mem_sleep_default=deep rw quiet splash"
    submenuentry "Boot using fallback initramfs" {
        initrd /initramfs-linux-fallback.img
    }
    submenuentry "Boot to terminal" {
        add_options "systemd.unit=multi-user.target"
    }
}
```

Several things are happening here:
- A number of utilities (`showtools`) are being made available in the boot manager menu.
    - These need to be installed separately. See documentation [here](https://wiki.archlinux.org/index.php/REFInd#Tools).
- The boot manager is being told where the boot partition is (`volume`).
- The LUKS disk `nvme0n1p2` is being unlocked and made available at `/dev/mapper/cryptlvm`.
- The correct LVM partition is being mounted to `/`.
- The default suspend state is being set to `deep`, so that battery is preserved.
- The system is being told to read from the swap partition when resuming from suspend.

### Create new initial  ramdisk environment
In order to provide encryption and LVM support, the appropriate packages need to be installed inside the `arch-chroot` environment. See the following official docs:
- https://wiki.archlinux.org/index.php/Mkinitcpio#Common_hooks
- https://wiki.archlinux.org/index.php/LVM#Configure_mkinitcpio
- https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration

Just in case:
```
# pacman -S cryptsetup lvm2
```

Now, edit `/etc/mkinitcpio.conf` and add `dm-crypt` and LVM support.
**NOTE:** The order of the hooks matters!
```
HOOKS=(base systemd autodetect sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck)
```

Because this machine uses an NVMe SSD, you also need to load the `nvme` module at boot time in order for it to boot properly. Same goes for `i915` (Intel graphics driver) and `ext4` (filesystem support). Further down:

```
MODULES=(nvme i915 ext4)
```

Also, create a keyfile and add it as a LUKS key. This is to avoid having to enter your encryption password twice -- once for rEFInd, once for initramfs.
```
# mkdir /root/secrets && chmod 700 /root/secrets
# head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
# cryptsetup -v luksAddKey -i 1 /dev/nvme0n1p2 /root/secrets/crypto_keyfile.bin
```

Then, add the file to `mkinitcpio.conf`:
```
FILES=(/root/secrets/crypto_keyfile.bin)
```

Finally, recreate the `initramfs` image:
```
# mkinitcpio -p linux
```

### Final steps
Set root password:
```
# passwd
```

Exit the `chroot` environment and reboot:
```
# exit
# reboot
```

## Post-install configuration
From this point onwards, you already have a fully functioning Arch install, and everything else is just gravy. The rest of this document is very opinionated, so feel free to follow any part of it at your discretion.

### Create an admin user
We'll need to install a couple of useful packages before we create an admin user:
```
# pacman -S sudo zsh
```

Create the user, adding them to appropriate groups, and using `zsh` as default:
```
# useradd -m -G wheel,users,audio,network,power,input,storage -s /usr/bin/zsh $username
```
Give them a password:
```
# passwd $username
```
Now edit the `sudoers` file to grant privileges to `wheel` group users. This includes things like not needing to input your password in order to shut down the machine, or update packages.
```
# visudo
```
Find and uncomment the following line:
```
%wheel  ALL=(ALL)   ALL
```
as well as any lines containing the `NOPASSWD` option - read the comments for explanations. You might want to learn how to quit `vim` before doing this \^_\^.

### Install core tools
Very quick installation of a couple of tools that will make the next few steps more efficient:
```
# pacman -S neovim git openssh yay
```

Now we can access the AUR through `yay`, as well as edit config files with `nvim`. Even though we'll be pulling our dotfiles later, it may be worth copying your `init.vim` over now, as you'll be spending some time staring at it.

We're also installing a few niceties for the terminal:
```
# pacman -S tree tmux jq htop cowsay sl
```

### Install power management stack
We'll be using `tlp` for power management, and `powertop` as an inspection tool.
```
# pacman -S tlp tlp-rdw powertop
# systemctl enable tlp.service
# systemctl enable tlp-sleep.service
# systemctl mask systemd-rfkill.service
# systemctl mask systemd-rfkill.socket
# systemctl enable NetworkManager-dispatcher.service
```

While we're at it, create a `udev` rule to hibernate the system on low battery (as per [official docs](https://wiki.archlinux.org/index.php/Laptop#Hibernate_on_low_battery_level)). Edit or create `/etc/udev/rules.d/99-lowbat.rules`:
```
# Suspend the system when battery level drops to 4% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-4]", RUN+="/usr/bin/systemctl hibernate"
```

### Install basic networking
I don't use `nm-applet` as I run a custom `rofi` script instead, but you may want to install that here:
```
# pacman -S networkmanager
# systemctl enable NetworkManager.service --now
# nmcli device wifi connect <network> password <pwd>
```

Now that we can automatically connect to networks, we'll get the laptop to automatically update the system clock every time the network manager connects to a network.
Create `/etc/NetworkManager/dispatcher.d/09-timezone`:
```
#!/bin/sh
case "$2" in
    up)
        timedatectl set-timezone "$(curl --fail https://ipapi.co/timezone)"
    ;;
esac
```

### Install audio stack
This is a bog-standard audio stack for the average user. If you're planning on doing any kind of music production or instrument recording, you'll want to replace PulseAudio with JACK.

```
# pacman -S alsa-lib alsa-plugins alsa-utils pulseaudio pulseaudio-alsa
```
PulseAudio actually starts by itself if it's present, so reboot here if you want functioning audio during the next steps.

While we're at it, might as well install our music-playing tools:
```
# pacman -S mpd ncmpcpp
```

### Install a basic graphical environment
First off, the X11 server:
```
# pacman -S xorg-server xorg-font-util xorg-fonts-75dpi xorg-fonts-100dpi
# pacman -S xorg-mkfontdir xorg-mkfontscale xorg-xdpyinfo xorg-xrandr xorg-xset2
# pacman -S xdotool xev
```

Next, the desktop environment. Or rather, the window manager. And a few basic utilities:
```
# pacman -S i3-gaps xss-lock rofi feh picom dunst conky
# pacman -S kitty redshift ranger vivaldi xscreensaver
# yay -S polybar xsecurelock
```

And finally, the display manager and greeter:
```
# pacman -S lightdm lightdm-slick-greeter
# systemctl enable lightdm --now
```
Some fonts to give it that A E S T H E T I C look:
```
pacman -S nerd-fonts-complete terminus-font adobe-base-14-fonts noto-fonts-emoji
pacman -S adobe-source-code-pro-fonts adobe-source-sans-pro-fonts adobe-source-serif-pro-fonts
```

Don't forget to rebuild the font cache after installation:
```
# fc-cache -vf
```

Before we actually log into the graphical environment, there's another couple of steps we need to take.

### (Optional) Clone dotfiles
Now is a good time to pull any dotfiles you have, so that the graphical environment looks the way you're used to. You'll need to be logged in as your regular user for this.

Make sure that your remote has the intended destination directory in its `.gitignore` to avoid recursion:
```
$ echo ".dotfiles/" >> .gitignore
```
Then, clone your dotfiles into a bare repo:
```
$ git clone --bare <repo-url> ~/.dotfiles
```
Temporarily set the `dotfiles` command alias in the current shell:
```
$ alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
```
Now, checkout the actual repo into your home directory:
```
$ dotfiles checkout
```
If the above command throws an error, it's because files already exist with the same name as the ones you're trying to pull. Delete them, or back them up if you're really paranoid, then checkout again.

Finally, tell your dotfiles repo to not show untracked files:
```
$ dotfiles config --local status.showUntrackedFiles no
```

One more step before rebooting into X.

### Install and configure touchpad driver
We're installing `libinput`, because `synaptics` is old and decrepit:
```
# pacman -S xf86-input-libinput libinput-gestures
```

Paste into `/etc/X11/xorg.conf.d/40-libinput.conf`:
```
Section "InputClass"
        Identifier "libinput touchpad catchall"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"
        Option "ClickMethod" "clickfinger"
        Option "Tapping" "True"
        Option "TappingButtonMap" "lrm"
        Option "TappingDrag" "True"
        Option "TappingDragLock" "True"
        Option "DisableWhileTyping" "True"
        Option "MiddleEmulation" "True"
        Option "ScrollMethod" "twofinger"
EndSection
```
These options give you:
- Two-finger right click
- Three-finger middle click
- Two-finger scroll
- Tap-to-click
- Drag lock

Later, paste into user's `$HOME/.config/libinput-gestures.conf`:
```
# Back and Forward
gesture swipe left  3   xdotool key alt+Left
gesture swipe right 3   xdotool key alt+Right

# Home and End
gesture swipe up    3   xdotool key Home
gesture swipe down  3   xdotool key End

# Pinch to zoom
gesture pinch in    2   xdotool key ctrl+minus
gesture pinch out   2   xdotool key ctrl+plus
```
This  last step may be redundant if you've got `libinput-gestures.conf` in your dotfiles -- see section on dotfiles.

### Add some spice to the graphical environment
After rebooting into i3, we can now install a few bells and whistles to the graphical environment. First off, graphical theme control:
```
$ sudo pacman -S lxappearance-gtk3 kvantum-qt5 qt5ct oh-my-zsh
```
Next, a GUI file manager and extensions:
```
$ sudo pacman -S thunar thunar-archive-plugin thunar-media-tags-plugin thunar-volman tumbler
```
Utilities:
```
$ sudo pacman -S ncdu fzf ripgrep
$ sudo pacman -S gsimplecal simplenote-electron-bin mutt calcurse w3m scrot
$ yay -S tlpui
```
Multimedia:
```
$ sudo pacman -S mpv vlc okular shotwell rhythmbox
```
Removable media:
```
$ sudo pacman -S gvfs gvfs-mtp simple-mtpfs udisks2
```
Code/text editors:
```
$ sudo pacman -S code pycharm-community-edition gedit
```

TODO:
- bleachbit

### Pimp out your boot manager
### Install GTK/Qt themes
- Windows 10 icons: https://github.com/B00merang-Artwork/
- Breeze theme (KDE default)
- Tela icons
- Orchis GTK theme


- video drivers xf86-video-intel

### Install Conda

Suppress Conda modifying PS1, let oh-my-zsh decide what the prompt looks like:
`conda config --set changeps1 False`