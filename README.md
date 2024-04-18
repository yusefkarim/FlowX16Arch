## Table of contents

- [Arch Linux on Asus ROG Flow X16 (GV601VU)](#arch-linux-on-asus-rog-flow-x16-gv601vu)
	- [Basic Install](#basic-install)
		- [Prepare and Booting ISO](#prepare-and-booting-iso)
		- [Networking](#networking)
		- [Format Disk](#format-disk)
		- [Create encrypted filesystem](#create-encrypted-filesystem)
		- [Create and Mount btrfs Subvolumes](#create-and-mount-btrfs-subvolumes)
		- [Create a btrfs swapfile and remount subvolumes](#create-a-btrfs-swapfile-and-remount-subvolumes)
		- [Install the system using pacstrap](#install-the-system-using-pacstrap)
		- [Chroot into the new system and change system and language settings](#chroot-into-the-new-system-and-change-system-and-language-settings)
		- [Add btrfs and encrypt to initramfs](#add-btrfs-and-encrypt-to-initramfs)
		- [Install Systemd bootloader](#install-systemd-bootloader)
		- [Blacklist Nouveau](#blacklist-nouveau)
		- [Leave chroot and reboot](#leave-chroot-and-reboot)
	- [Fine-tuning](#fine-tuning)
		- [Setup networking](#setup-networking)
		- [Connect to WiFi](#connect-to-wifi)
		- [Install packages](#install-packages)
		- [Install an AUR helper (paru)](#install-an-aur-helper-paru)
		- [Create a new user](#create-a-new-user)
		- [Enable the SSD trim timer service](#enable-the-ssd-trim-timer-service)
	- [Flow X16 specific customizations](#flow-x16-specific-customizations)
		- [Intel](#intel)
		- [NVIDIA](#nvidia)
		- [ASUS tools](#asus-tools)
	- [Install audio and graphical cruft](#install-audio-and-graphical-cruft)
		- [Audio](#audio)
			- [Pipewire](#pipewire)
			- [Bluetooth](#bluetooth)
		- [Wayland + Sway](#wayland--sway)
		- [Additional graphical/desktop applications](#additional-graphicaldesktop-applications)
	- [Setup automatic snapshots for pacman](#setup-automatic-snapshots-for-pacman)

# Arch Linux on Asus ROG Flow X16 (GV601VU) 
Guide to install Arch Linux with btrfs, disc encryption, auto-snapshots, no-noise fan-curves on Asus ROG Flow X16. Credits to [k-amin07](https://github.com/k-amin07/G14Arch), this guide is a fork of their guide.

## Basic Install

### Prepare and Booting ISO

Boot Arch Linux using a prepared USB stick.
[Rufus](https://rufus.ie/en/) can be used on windows.
[Etcher](https://www.balena.io/etcher/) can be used on Windows or Linux.

You can also use `dd`:

```sh
dd bs=4M if=archlinux-x86_64.iso of=/dev/sdX status=progress oflag=sync
```

### Networking

To connect to a wireless network launch `iwctl` and connect to your AP like this:
* `station wlan0 scan`
* `station wlan0 get-networks`
* `station wlan0 connect YOURSSID` 

Type `exit` to leave.

If using a wired connection please check the [Arch WiKi](https://wiki.archlinux.org/index.php/Network_configuration). 

Once connected, update system clock with `timedatectl set-ntp true`

### Format Disk

* Check for the name of the main disk with `lsblk`, we'll assume it's `nvme0n1` going forward.
* Format the disk using `gdisk /dev/nvme0n1` with this simple layout:

	* `o` for new partition table
	* `n,1,<ENTER>,+1024M,ef00` for EFI Boot
	* `n,2,<ENTER>,<ENTER>,8300` for the linux partition
	* `w` to save layout


Format the EFI partition:

```sh
mkfs.vfat -F 32 -n EFI /dev/nvme0n1p1
```

### Create encrypted filesystem 

```sh
cryptsetup luksFormat /dev/nvme0n1p2  
cryptsetup open /dev/nvme0n1p2 luks
```

### Create and Mount btrfs Subvolumes

Create btrfs filesystem for root partition

```sh
mkfs.btrfs -f -L ROOTFS /dev/mapper/luks
```


Mount Partitions and create subvolumes for btrfs.
A separate subvolume is created for the home directory so that system snapshots
and home directory snapshots are independent of each other.

```sh
mount -t btrfs LABEL=ROOTFS /mnt
btrfs sub create /mnt/@
btrfs sub create /mnt/@home
btrfs sub create /mnt/@snapshots
btrfs sub create /mnt/@swap
```

### Create a btrfs swapfile and remount subvolumes

```sh
btrfs filesystem mkswapfile --size ${SWAP_SIZE} /mnt/@swap/swapfile
```

Replace `${SWAP_SIZE}` with the amount of swap space you want. Typically you should have the same amount of swap as RAM. So if you have 16GB of ram, you should have 16GB of swap space. Note that the size in GB is denoted with a G as a suffix and **NOT** GB
([Source](https://btrfs.readthedocs.io/en/latest/Swapfile.html)).

After that, just unmount with `umount /mnt/` and remount with subvolumes

```
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/{boot,home,.snapshots,swap,btrfs}

mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@home /dev/mapper/luks /mnt/home/
mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots/
mount -o noatime,space_cache=v2,commit=120,subvol=@swap /dev/mapper/luks /mnt/swap/

mount /dev/nvme0n1p1 /mnt/boot/

mount -o noatime,compress=zstd,space_cache=v2,commit=120,subvolid=5 /dev/mapper/luks /mnt/btrfs/
```

Check mountpoints with `df -Th` and enable swap file

```sh
swapon /mnt/swap/swapfile
```

### Install the system using pacstrap

```sh
pacstrap /mnt base base-devel linux linux-firmware iwd btrfs-progs man-db neovim curl ldns python intel-ucode
```

After this, generate the filesystem table using 

```sh
genfstab -Lp /mnt >> /mnt/etc/fstab
```

### Chroot into the new system and change system and language settings

```sh
arch-chroot /mnt
echo laptop01 > /etc/hostname
echo LANG=en_US.UTF-8 > /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
ln -sf /usr/share/zoneinfo/America/Toronto /etc/localtime
hwclock --systohc
```

Modify `/etc/hosts`:

```sh
127.0.0.1		localhost
::1				localhost
127.0.1.1		zephyrus-g14.localdomain	zephyrus-g14
```

Edit `/etc/locale.gen` and uncomment the following line

```sh
en_US.UTF-8 UTF-8
```

Execute `locale-gen` to create the locales.

Add a password for root using `passwd root`

### Add btrfs and encrypt to initramfs

Edit `/etc/mkinitcpio.conf` and add `encrypt btrfs` to hooks between `block filesystems`.

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt btrfs filesystems fsck)
```

Run `mkinitcpio -P` to generate a new image.

### Install Systemd bootloader

Run `bootctl --path=/boot install` to install the bootloader.

Edit the bootloader config `/boot/loader/loader.conf`, replacing the existing text with the following lines.

```
default	arch.conf
timeout	3
editor	0
```

Then, replace the contents of `/boot/loader/entries/arch.conf` with the following

```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
```

Finally, copy boot-options with
```
echo "options	cryptdevice=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2):luks root=/dev/mapper/luks rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf
```

### Blacklist Nouveau

Edit `/etc/modprobe.d/blacklist-nvidia-nouveau.conf` and add the following:

```
blacklist nouveau
options nouveau modeset=0
```

### Leave chroot and reboot

Type `exit` to exit chroot and unmount all the volumes using
```
umount -R /mnt/
```
and reboot the system.

## Fine-tuning

After a successful reboot above, we will perform some necessary initial setup.

### Setup networking

Enable essential systemd units

```sh
systemctl enable --now iwd systemd-networkd systemd-resolved systemd-timesyncd
```

Make sure the systemd stub `resolv.conf` file is used:

```sh
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Create the following systemd-networkd files:

`/etc/systemd/network/20-ethernet.network`
```sh
[Match]
Name=en*
Name=eth*

[Network]
DHCP=yes
IPv6PrivacyExtensions=yes

# systemd-networkd does not set per-interface-type default route metrics
# Explicitly set route metric, so that Ethernet is preferred over Wi-Fi.
[DHCPv4]
RouteMetric=100

[IPv6AcceptRA]
RouteMetric=100
```

`/etc/systemd/network/20-wlan.network`
```sh
[Match]
Name=wl*

[Network]
DHCP=yes
IPv6PrivacyExtensions=yes

# systemd-networkd does not set per-interface-type default route metrics
# Explicitly set route metric, so that Ethernet is preferred over Wi-Fi.
[DHCPv4]
RouteMetric=600

[IPv6AcceptRA]
RouteMetric=600
```

Restart networking with `systemctl restart systemd-networkd`.

### Connect to WiFi

```sh
iwctl station wlan0 connect <SSID>
```

### Install packages

Ensure that the system is up to date:
```
pacman -Syu
```

```sh
pacman -S acpid dbus sudo docker openssh picocom rsync pass python-pip rustup git \
	ttf-dejavu ttf-hack ttf-font-awesome ttf-nerd-fonts-symbols noto-fonts-cjk \
    zip unzip ripgrep wireguard-tools bash-completion
```

Enable and start `acpid`:

```sh
systemctl enable --now acpid
```

### Install an AUR helper ([paru](https://github.com/Morganamilo/paru))

Install paru:

```sh
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
cd .. && rm -rf paru
```

### Create a new user 

Create a new local user with the relevant groups:

```sh
useradd -m -G wheel,docker,power,audio -s /usr/bin/bash USERNAME
passwd USERNAME
```

Enable root access for the user by running the command below and uncommenting the line `%wheel ALL=(ALL) ALL`:

```sh
EDITOR=nvim visudo
```

### Enable the SSD trim timer service

Per the Arch Wiki [[ref](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)]:

> The util-linux package provides fstrim.service and fstrim.timer systemd unit files.
> Enabling the timer will activate the service weekly.
> The service executes fstrim(8) on all mounted filesystems on devices that support the discard operation. 

```sh
systemctl enable --now fstrim.timer
```

## Flow X16 specific customizations

### Intel

Install the `mesa` driver (likely already installed as a dependency of other packages):

```sh
pacman -S mesa
```

Enable [GuC/HuC firmware loading](https://wiki.archlinux.org/title/intel_graphics#Enable_GuC_/_HuC_firmware_loading), create the file `/etc/modprobe.d/i915.conf` with contents:

```sh
/etc/modprobe.d/i915.conf
```

### NVIDIA
Install the following packages

```sh
sudo pacman -S nvidia-dkms acpi_call linux-headers
```

Add NVIDIA-specific kernel options to systemd-boot:

`/boot/loader/entries/arch.conf`
```sh
...
options ... nvidia_drm.modeset=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1
```

Add NVIDIA-specific kernel modules to the `MODULES` section of `mkinitcpio.conf`:

`/etc/mkinitcpio.conf`
```sh
...
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
...
```

Generate a new `initrd`:

```sh
mkinitcpio -P
```

Create a new file modprobe file to enable DRM (Direct Rendering Manager):

`/etc/modprobe.d/nvidia.conf`
```sh
options nvidia-drm modeset=1
```

Enable the suspend/hibernate/resume NVIDIA services:

```sh
systemctl enable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
```

Reboot the system and confirm nothing explodes.

### ASUS tools

You can install these tools by following official asus-linux [Arch Setup Guide](https://asus-linux.org/guides/arch-guide/)
or just use an AUR helper and get them from the AUR repos:

```sh
paru -S asusctl
```

You can also install `rog-control-center` and `supergfxctl`. I opted not to.

Enable services (`supergfxd` only if you installed `supergfxctl`):
```sh
systemctl enable --now power-profiles-daemon supergfxd
```

Run the following commands to set charge limit and enable Quiet, Performance and Balanced Profiles:
```sh
asusctl -c 85 		# Sets charge limit to 85% if you do not want this, do not execute this line
asusctl fan-curve -m Quiet -f cpu -e true
asusctl fan-curve -m Quiet -f gpu -e true 
asusctl fan-curve -m Performance -f cpu -e true
asusctl fan-curve -m Performance -f gpu -e true
asusctl fan-curve -m Balanced -f cpu -e true
asusctl fan-curve -m Balanced -f gpu -e true
```

Reboot the system and confirm nothing explodes.

## Install audio and graphical cruft

### Audio

#### Pipewire
Install Pipewire and [Easy Effects](https://github.com/wwmm/easyeffects) (GTK-based audio effects/manager application):

```sh
sudo pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack gst-plugin-pipewire wireplumber easyeffects
```

Enable the Pipewire services:
```sh
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```

#### Bluetooth

Install bluetooth related packages:

```sh
pacman -S bluez bluez-utils
systemctl enable --now bluetooth
```

### Wayland + Sway

```sh
pacman -S sway swaybg swaylock swayidle waybar wl-clipboard grim slurp gammastep bemenu-wayland brightnessctl
```

Add your user to the `seat` group:

```sh
usermod -aG seat cuyler
```

Enable the `seatd` service:

```sh
systemctl enable --now seatd
```

If you want screen rotation handling, you can install iio-sensormanually compile and install [iio-sway](https://github.com/okeri/iio-sway/tree/master):

```sh
pacman -S iio-sensor-proxy meson
git clone https://github.com/okeri/iio-sway.git
cd iio-sway
meson setup build
cd build
meson install
# Remove meson once done (optional)
sudo pacman -Rns meson
```

You can then add, `exec iio-sway` to the top of your sway config.

### Additional graphical/desktop applications

```sh
pacman -S firefox alacritty imv mpv
```

## Setup automatic snapshots for pacman

At this point, we have installed everything we need. Reboot the system once to make sure everything works fine and set up BTRFS snapshots to ensure we always have a restore point in case something breaks in the future. To do so, first create a snapshot manually as follows
```sh
sudo -i
btrfs sub snap / /.snapshots/STABLE
cp /boot/vmlinuz-linux /boot/vmlinuz-linux-stable
cp /boot/intel-ucode.img /boot/intel-ucode-stable.img
cp /boot/initramfs-linux.img /boot/initramfs-linux-stable.img
cp /boot/loader/entries/arch.conf /boot/loader/entries/arch-stable.conf
```
Edit `/boot/loader/entries/arch-stable.conf` to boot from `STABLE` snapshot
```sh
title Arch Linux Stable
linux /vmlinuz-linux-stable
initrd /intel-ucode-stable.img
initrd /initramfs-linux-stable.img
options cryptdevice=UUID=62b5e6e0-6376-46d8-9faf-fe391a58c6b1:luks root=/dev/mapper/luks rootflags=subvol=@snapshots/STABLE quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```
Now edit `/.snapshots/STABLE/etc/fstab` to change the root of the STABLE snapshot.
```sh
 ... LABEL=ROOTFS / btrfs rw,noatime,.....subvol=@snapshots/STABLE ...
```
Reboot the system, in the boot menu, select `Arch Linux Stable` to see if it boots correctly. If it does, boot back into `Arch Linux`. 

Copy the script from repo to `/usr/bin/autosnap` and make it executable with `chmod +x /usr/bin/autosnap`. Then copy the pacman hook script from the repo to `/etc/pacman.d/hooks/00-autosnap.hook`. 

Now every time pacman installs or upgrades something, the oldest snapshot would be removed a new one will be created. Let's test everything one more time to ensure nothing breaks. To do so, install any package from pacman, e.g.
```sh
sudo pacman -S neofetch
```
The output should look something like this
```sh
...
:: Running pre-transaction hooks...
(1/1) Creating btrfs snapshot
Delete subvolume (no-commit): '/.snapshots/STABLE'
Create a snapshot of '/' in '/.snapshots/STABLE'
...
```

Note that in pre-transaction hooks, it deletes the STABLE snapshot, takes the snapshot of the current system in `/.snapshots/STABLE` before proceeding to install the package. Boot back into the stable snapshot and run `neofetch` in the terminal. It should say `command not found`. Now boot back into the normal system and try running `neofetch` again, it would work without issues.

The script maintains five recent snapshots, allowing you to boot back into an older one if something breaks.
