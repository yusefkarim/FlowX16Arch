# Arch Linux on Asus ROG Zephyrus G14 (G401II)
Guide to install Arch Linux with btrfs, disc encryption, auto-snapshots, no-noise fan-curves on Asus ROG Zephyrus G14. Credits to [Unim8rix](https://github.com/Unim8trix/G14Arch), this guide is a fork of their guide with some variation.

![image](https://user-images.githubusercontent.com/28199865/134336064-1a3bcf91-46d0-434e-9118-78345193c9d5.png)



## Basic Install

### Prepare and Booting ISO

Boot Arch Linux using a prepared USB stick. [Rufus](https://rufus.ie/en/) can be used on windows, [Etcher](https://www.balena.io/etcher/) can be used on Windows or Linux.
***

### Networking

For Network i use wireless, if you need wired please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration). 

Launch `iwctl` and connect to your AP `station wlan0 connect YOURSSID` 
Type `exit` to leave.

Update System clock with `timedatectl set-ntp true`

### Format Disk

* My Disk is `nvme0n1`, check with `lsblk` 
* Format Disk using `gdisk /dev/nvme0n1` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+512M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI Partition

`mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1` 


### Create encrypted filesystem 

```
cryptsetup luksFormat /dev/nvme0n1p2  
cryptsetup open /dev/nvme0n1p2 luks
```

### Create and Mount btrfs Subvolumes

`mkfs.btrfs -f -L ROOTFS /dev/mapper/luks` btrfs filesystem for root partition


Mount Partitions und create Subvol for btrfs. I dont want home, etc in my snapshots, so create subvol for them.

* `mount -t btrfs LABEL=ROOTFS /mnt` Mount root filesystem to /mnt
* `btrfs sub create /mnt/@`
* `btrfs sub create /mnt/@home`
* `btrfs sub create /mnt/@snapshots`
* `btrfs sub create /mnt/@swap`

### Create a btrfs swapfile and remount subvols

```
truncate -s 0 /mnt/@swap/swapfile
chattr +C /mnt/@swap/swapfile
btrfs property set /mnt/@swap/swapfile compression none
fallocate -l 16G /mnt/@swap/swapfile
chmod 600 /mnt/@swap/swapfile
mkswap /mnt/@swap/swapfile
mkdir /mnt/@/swap
```

Just unmount with `umount /mnt/` and remount with subvolumes

```
mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/boot
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/btrfs

mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,space_cache,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,space_cache,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,space_cache,commit=120,subvolid=5 /dev/mapper/luks /mnt/btrfs/
```


Check mountmoints with `df -Th` 

### Install the system using pacstrap

```
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs nano networkmanager amd-ucode
```

After this, generate the filesystem table using 
`genfstab -Lp /mnt >> /mnt/etc/fstab` 

Add swapfile 
`echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab `

### Chroot into the new system and change language settings
You can use a hostname of your choice, I have gone with zephyrus-g14.
```
arch-chroot /mnt
echo zephyrus-g14 > /etc/hostname
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
ln -sf /usr/share/zoneinfo/Asia/Karachi /etc/localtime
hwclock --systohc
```

Modify `nano /etc/hosts` with these entries. For static IPs, remove 127.0.1.1. Replace zephyrus-g14 with your hostname.

```
127.0.0.1		localhost
::1				localhost
127.0.1.1		zephyrus-g14.localdomain	zephyrus-g14
```

`nano /etc/locale.gen` to uncomment the following line

```
en_US.UTF-8
```
Execute `locale-gen` to create the locales now

Add a password for root using `passwd root`


### Add btrfs and encrypt to Initramfs

`nano /etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between block/filesystems

`HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck `

Also include `amdgpu` in the MODULES section

create Initramfs using `mkinitcpio -p linux`

### Install Systemd Bootloader

`bootctl --path=/boot install` installs bootloader

`nano /boot/loader/loader.conf` delete anything and add these few lines and save

```
default	arch.conf
timeout	3
editor	0
```

` nano /boot/loader/entries/arch.conf` with these lines and save. 

```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux.img
```

copy boot-options with
` echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):luks root=/dev/mapper/luks rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf` 

### Set nvidia-nouveau onto blacklist 

using `nano /etc/modprobe.d/blacklist-nvidia-nouveau.conf` with these lines

```
	blacklist nouveau
	options nouveau modeset=0
```

### Leave Chroot and Reboot

Type `exit` to exit chroot

`umount -R /mnt/` to unmount all volumes

Now its time to `reboot` into the new system!

## Finetuning after first Reboot

### Enable Networkmanager

Configure WiFi Connection.

```
systemctl enable NetworkManager
systemctl start NetworkManager
nmcli device wifi connect YOURSSID password SSIDPASSWORD
```

### Create a new user 

First create my new local user and point it to bash

```
useradd -m -g users -G wheel,power,audio -s /bin/bash MYUSERNAME
passwd MYUSERNAME
```

Edit `nano /etc/sudoers` and uncomment `%wheel ALL=(ALL) ALL`

Now `exit` and relogin with the new MYUSERNAME


Install some Deamons before we reboot

```
sudo pacman -Sy acpid dbus 
sudo systemctl enable acpid
```

## Setup Automatic Snapshots for pacman:
To setup automatic snapshots everytime system updates, follow the section from Unim8rix's [guide](https://github.com/Unim8trix/G14Arch)


## Install Desktop Environment

### Get X.Org and KDE Plasma

Install xorg and kde packages

```
pacman -S xorg 
sudo pacman -Sy xorg plasma kde-applications pulseaudio
```

SDDM Loginmanager

```
sudo pacman -S sddm
sudo systemctl enable sddm
```

Reboot and login to your new Desktop.


### Oh-My-ZSH

I like to use oh-my-zsh with Powerlevel10K theme

```
sudo pacman -Sy git git-lfs curl wget zsh zsh-completions firefox-i18n-de
chsh -s /bin/zsh # (relogin to activate)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mkdir .local/share/fonts
cd .local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -v
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`

### Setup Plymouth for nice Password Prompt during Boot

Plymouth is in [AUR](https://aur.archlinux.org) , just clone the Repo and make the Package (i create a subfolder AUR within my homefolder)

```
cd AUR
git clone https://aur.archlinux.org/plymouth-git.git
makepkg -is
```

Now modify the Hooks for the Initramfs, Plymouth must be right after "base udev". Delete encrypt hook, it will be replaced by plymouth-encrypt

```
HOOKS="base udev plymouth plymouth-encrypt autodetect modconf block btrfs filesystems keyboard fsck
```

Run `mkinitcpio -p linux` 

Add some kernel-parameters to make boot smooth. Edit `/boot/loader/entries/arch.conf` and append to options

```
...rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```


For Plymouth Theming and Options, check [Plymouth on Arch Wiki](https://wiki.archlinux.org/title/plymouth). Issue the following command to set ROG logo as the plymouth theme.
```
sudo plymouth-set-default-theme -R bgrt
```

For KDE Theming you could check this nice [Youtube Video from Linux Scoop](https://www.youtube.com/watch?v=2GYT7BK41zk)

## Useful Customizations:

### Install asusctl tool from [Luke Jones](https://asus-linux.org/)

Add his Repo to pacman.conf
```
sudo bash -c "echo -e '\r[g14]\nSigLevel = DatabaseNever Optional TrustAll\nServer = https://arch.asus-linux.org\n' >> /etc/pacman.conf"

sudo pacman -Sy asusctl
```
Optional: Copy my asusd profile to `/etc/asusd/asusd.conf` or make your own profiles and Fan-Curves

Activate DBUS Messaging for the new asus deamon to get asus notifications upon changing fan profile etc.

```
systemctl --user enable asus-notify
systemctl --user start asus-notify
```

For fine-tuning read the [Arch Linux Wiki](https://wiki.archlinux.org/title/ASUS_GA401I#ASUSCtl) or the [Repository from Luke](https://gitlab.com/asus-linux/asusctl)

### Battery limit:
Set battery charge limit to 85% in asusd.conf to prevent battery wear. Charging to 100% quickly drops battery health.

### Installing a custom kernel:
I use the linux-g14 kernel, which is available in the repo we used for asusctl tool. Install the kernel using
```
sudo pacman -S linux-g14 linux-g14-headers
```
Edit `/boot/loader/loader.conf` and replace the contents with the following. 
```
default arch-g14.conf
timeout 3
editor 0
```
Run `sudo nano /boot/loader/entries/arch-g14.conf` and add the following lines:
```
title   Arch Linux (g14)
linux   /vmlinuz-linux-g14
initrd  /amd-ucode.img
initrd  /initramfs-linux-g14.img
options cryptdevice=UUID=04ea2e96-fd5d-446c-bb6f-c83c0cc44158:luks root=/dev/mapper/luks rootflags=subvol=@ quiet splash loglevel=3 rd.systemd.show_status=auto
```
and finally, run 
```
sudo mkinitcpio -p linux-g14
```
### ROG Key Map
Go to KDE Settings->Shortcuts->Custom Shortcuts. Click Edit->New->Global Shortcut->Command/URL. Name it `NvidiaSettings`. Set trigger to `ROG Key` and set action to `nvidia-settings`  

### Change Fan Profile:
Go to KDE Settings->Shortcuts->Custom Shortcuts. Click Edit->New->Global Shortcut->Command/URL. Name it  `ChangeFanProfile`, set trigger to `fn + f5` and action to `asusctl profile -n`

### Mic Mute Key:
Run `usb-devices` and look for the device that says `Product=N-KEY Device`. Note the vendor id. For my zephyrus it is `0b05`.  Run 
```
sudo find /sys -name modalias | xargs grep -i 0b05
```

Find the line that goes like:
```
.../input/input18/modalias:input:b0003v0B05p1866e0110-e0...
```
Copy the part after `input:`, before the first `-e`. In my case, it is `b0003v0B05p1866e0110`.  Create a file named `/etc/udev/hwdb.d/90-nkey.hwdb` and add
```
/etc/udev/hwdb.d/90-nkey.hwdb
evdev:input:b0003v0B05p1866*
 KEYBOARD_KEY_ff31007c=f20 # x11 mic-mute, space in start is important in this line
```
	After that, update `hwdb`.
```
sudo systemd-hwdb update
sudo udevadm trigger
```

### Powertop
Install powertop and calibrate first by running:
```
sudo powertop -c
```
Running `powertop --autotune` can improve battery life. Add the following to /etc/rc.local
```
#!/bin/sh -e
sudo powertop --auto-tune
exit 0
```

Then add an rc-local.service to systemd using:
```
[Install]
WantedBy=multi-user.target

[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99
```

## Nvidia
Install `nvidia` package from official repos. Double check to see if linux-headers and/or linux-g14-headers (depending on the kernel) are installed to avoid blackscreen.  Do not run `nvidia-xconfig` on Arch as it results in black screen. Install the following packages:
```
sudo pacman -S nvidia-dkms nvidia-settings nvidia-prime acpi_call
```
## Optimus Manager:
Install optimus-manager and optimus-manager-qt. After rebooting, it should work fine. 

- **Type C external display:**
HDMI works out of the box, displays can be connected to type C port but require switching to dedicated graphics. Can be done either through asusctl or Optimus Manager QT.
	
- **Optimus Manager configuration**
In optimus manager qt, go to optimus tab, and select ACPI call switching method and set PCI reset to No. Enable PCI remove. Startup mode is integrated. In Nvidia, set dynamic power management to fine. Modeset, Pat and overclocking options.

- **Sleep/Shutdown issues**
System was only going to sleep once and after that got stuck on shutdown and sleep. This happened because I had set switching method to bbswitch in optimus manager. Swithced to acpi_call to fix.

## Miscellaneous
### Fetch on Terminal Start
After installing and enabling zsh and oh-my-zsh with powerlevel10k, create file `~/.zshenv` and do the following:
- Install fastfetch-git from pamac.
- Add `fastfetch --load-config paleofetch` in `~/.zshenv`
### Key delay
Reduce key input delay to 250 ms for a better keyboard experience.

## KDE Tweaks:
### Open In First Virtual Desktop:
Add Window Rule, name it `Open in first VD`. Set the following rules

- Window Class: Unimportant
- Match whole window class: No
- Window Types: Normal window (deselect all others)
- Size: Apply Initially, 1280x720
- Virtual Desktop: Apply Initially, Desktop 1.
- Ignore Requested Geometry: Force, Yes.
- Window behavior: Focus Stealing prevention Low -> None.

To make brave launch in 1280x720, do the following:
- Edit `/usr/share/applications/brave-browser.desktop`
- line 111: change `Exec=brave %U` to `Exec=brave %U --window-size="1280,720"`
- Repeat for line 111 and 223

### Touchpad Gestures:
 Use [fusuma](https://github.com/iberianpig/fusuma) to get touchpad gestures. Create `/home/slim/.local/share/scripts/fusuma.sh` and add
 ```
 #!/bin/bash
 fusuma -d #for running in daemon mode
```
Add this scrpit to autostart in KDE settings. For macOS like gestures use [this config](https://github.com/iberianpig/fusuma/wiki/KDE-to-mimic-MacOS.). 4 finger gestures are not working. My config is in the repo.

### Yet Another Magic Lamp:

A better [magic lamp](https://github.com/zzag/kwin-effects-yet-another-magic-lamp) effect. In latest plasma versions, exclude "disable unsupported effects" next to the search bar in settings for the effect to appear.

### Maximize to new desktop ([git](https://github.com/Aetf/kwin-maxmize-to-new-desktop#window-class-blacklist-in-configuration-is-blank)): 
In Kwin scripts, install "kwin-maximize-to-new-desktop" and run:
```
mkdir -p ~/.local/share/kservices5
ln -s ~/.local/share/kwin/scripts/max2NewVirtualDesktop/metadata.desktop ~/.local/share/kservices5/max2NewVirtualDesktop.desktop
```
Then install kdesignerplugin through `pacman -S kdesignerplugin`. Logout and login again after configuration changes. My config:
```
Trigger: Maximize only
Position: Next to current
```

