# Arch Setup

> [!WARNING]
> This is an advanced guide using some core system alternatives. Only follow if you know what you're doing!!!

This guide is for UEFI systems.

## Change keyboard layout

To list available options:
```
localectl list-keymaps
```

For example, to change the keyboard layout to Italian, you can do:

```
loadkeys it
```

## Connect to WiFi

Heres some basic commands to connect to WiFi. Replace `name` with your device name -Typically `wlan0`- and `SSID` with the name of the network you want to connect. If the network name has spaces in it, you'll have to wrap it inside of quotation marks (E.g. `WiFi`, "My Network"`).

```
iwctl
```
```
device list
device name set-property Powered on
station name scan
station name get-networks
station name connect SSID
```

To test if you're connected to the Internet:

```
ping archlinux.org
```

## Partitioning

```
cfdisk
```

If you see a screen to select patition type. Select the `gpt` option.

Next, you'll see a screen with partitions on it and, if you have multiple partitions, you can delete them if you have backed up the data on them.

Then, go to the `New` tab and type in how much you need for a boot partition. In this case, it will be **512M - 2G** or greater depending on how many snapshots/kernels you want to save.

After that, create a partition that's **4G**. That one will be used as a swap partition. You can make this bigger if you would like.

Finally, create a partition that will use the rest of the space you have and hit `Write` and type `yes` when you're finished with everything.

## Format Partitions

To list the partions, you have to type:

```
lsblk
```

Format the root partition (the biggest partition) to `btrfs`:

```
mkfs.btrfs /dev/<drive and partition>
```

Next, format the boot partition (usally the smallest partition):

```
mkfs.fat -F 32 /dev/<drive and partition>
```

## Mount partitions

First, mount the root partition:

```
mount /dev/<drive and partition> /mnt
```

Now, we need to mount the boot partition but the `/boot/efi` is not a directory by default so we will create that first

```
mkdir -p /mnt/boot/efi
mount /dev/<drive and partition> /mnt/boot/efi
```

To check if you got everything right, run and verify:

```
lsblk
```

## Create btrfs sub volumes

Now, to create subpartitions run the following:

```
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var_log
umount /mnt
```

Next, we need to mount the root directory:

```
mount -o noatime,compress=zstd,subvol=@ /dev/<drive and partition> /mnt
```

Next, we need to create the mounting points:

```
mkdir -p /mnt/{boot,home,var/log}
```

Next, we can mount the sub volumes:

```
mount -o noatime,compress=zstd,subvol=@home /dev/<drive and partition> /mnt/home
mount -o noatime,compress=zstd,subvol=@var_log /dev/<drive and partition> /mnt/var/log
```

Finally, lets mount the boot partition:

```
mount /dev/<drive and partition> /mnt/boot
```

## Edit pacman.conf and update mirrorlist

First, we need a text editor so we will use `helix`.

```
pacman -S helix
```

Now to edit `pacman.conf`

```
helix /etc/pacman.conf
```

Uncomment the line with `Color` and `ParallelDownloads = 5` and save.

Now, to update the mirrorlist, we need to get a package called `reflector` to generate a fresh mirror list and to generate the mirrorlist.

```
pacman -Sy reflector
reflector --latest 10 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Finally, we need to copy the configuration to the installed system and we can do that by:

```
cp /etc/pacman.conf /mnt/etc/pacman.conf
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
```

## Install base system

```
pacstrap /mnt base booster linux linux-lts linux-firmware base-devel btrfs-progs limine helix networkmanager
```

## Fstab

```
genfstab /mnt > /mnt/etc/fstab
```

## Chroot

```
arch-chroot /mnt
```

## Booster

```
# example
compress: zstd -9 -T0
modules: btrfs
strip: true
```

If you're using the btrfs filesystem, you need the `btrfs` module
If you have an Intel GPU, you dont need a GPU module.
If you have an AMD GPU, you need to have the `amdgpu` module.
If you have an Nvidia GPU, you need these modules `nvidia,nvidia_modeset,nvidia_uvm,nvidia_drm`

To run booster run:

```
sudo /usr/lib/booster/regenerate_images
```

## Time zone

```
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

To verify it's the correct time, run:

```
date
```

Now, we need to sync the the system clock:

```
hwclock --systohc
```

If you're dualbooting with Windows and don't want to use `RealTimeIsUniversal` registry hack, you can replace `--systohc` with `--hctosys` instead, and Arch will follow and set BIOS clock instead, just like Windows.

## Localization

```
helix /etc/locale.gen
```

Now you will edit a file that contains all the available locales on the system.

In most cases, if you are in the US, you will need to uncomment the line that contains `en_US.UTF-8`.

Now, to finish, run:

```
locale-gen
```

Also, you will need to add `LANG=en_US.UTF-8` to `/etc/locale.conf` if you want to use US locale.

If you want to change the keyboard layout (default is US), edit `/etc/vconsole.conf` with `KEYMAP=us`.

## Network hostname

Now, edit `/etc/hostname` with whatever hostname you would like.

## Create user

Now, create a username you would like to use.

```
useradd -mG wheel -s /bin/bash yourusername
```

Now, create a password that you would like to use:

```
passwd yourusername
```

To be able to use sudo, we need to do the following:

```
EDITOR=hx visudo
```

Uncomment the following line:

```
%wheel ALL=(ALL:ALL) ALL
```

Finally, to test, switch to your user using `su username` and then try to run a `sudo` command such as updating the system.

Some apps may need you to be apart of the `video` and `input` group so just runn the following command:

```
sudo usermod -aG input,video $USER
```

Optionally, if you want to, you can now disable the root user by running this command:

```
passwd -l root
```

## Enable core services

```
systemctl enable networkmanager
```

## Microcode

Install your microcode for whatever CPU brand you're using:

`intel-ucode` or `amd-ucode`

## Boot loader

UEFI systems:

```
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTIA32.EFI /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT
```

BIOS systems:

```
mkdir -p /boot/limine
cp /usr/share/limine/limine-bios.sys /boot/limine/
```

Getting `PARTUUID` and File Type and add it to `/boot/limine.conf`:

```
lsblk -o PARTUUID -nr /dev/<disk and pertition> | sudo tee /boot/limine.conf > /dev/null
```

Please add this to the `limine.conf` file and, if its not there, create it and move the `PARTUUID` to the required spot:

```
TIMEOUT: 5
default_entry: 2
# If you want a background add this

wallpaper: boot():/wallpaper.png # wallpaper.png needs to be in /boot

/+Arch Linux

//Arch Linux (linux) - Booster
  protocol: linux
  kernel_path: boot():/vmlinuz-linux
  kernel_cmdline: root=PARTUUID=<PARTUUID> rootflags=subvol=@ rw rootfstype=<filetype> quiet
  module_path: boot():/<brand>-ucode.img
  module_path: boot():/booster-linux.img

```

If you have zswap enabled, add this to the kernel_cmdline:

```
zswap.enabled=0
```

## Reboot

Now, you can reboot, but before we do, we need to unmount the drive and we can do that by:

```
umount -a
```

To reboot, just run the `reboot` command.

## Connect to the internet

Use command `nmtui` to connect to the internet if you're using WiFi.

## AUR wrapper

To install an AUR wrapper, we first need to install `git`, and then we can install the aur wrapper of our choosing. In our case, we will use `yay`.

```
sudo pacman -Sy git
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
```

## Helix

Now, it would be nice to install a package to be able to invoke `helix` each time you want to run the text edit, but instead you can now run `hx`.

```
yay -S helix helixbinhx
```

To view the opitional dependencies for helix syntax highlighting for x launguages run `hx --health` to view what packages you need for x launguage

## Add audio services

Some core PipeWire packages are:

```
yay -S pipewire pipewire-jack pipewire-audio pipewire-pulse pipewire-alsa wireplumber
```

To have Bluetooth support, you will need:

```
yay -S bluez bluez-utils
```

To enable/start audio and Bluetooth support, run the following:

```
sudo systemctl enable bluetooth.service --now
systemctl --user enable pipewire.service wireplumber.service --now
```

## Limine shapshots setup

https://gitlab.com/Zesko/limine-snapper-sync

Install these packages:

```
yay -S limine-entry-tool limine-snapper-sync snap-pac
```

Check if an ESP path is detected by running: `bootctl --print-esp-path`:
If detected, no further configuration for `ESP_PATH` is needed.
If not detected, create and edit `/etc/default/limine` and set `ESP_PATH=` to the correct ESP path.
Then run `limine-update` to confirm it worked.

To initialize snapper for limine run:

```
sudo limine-snapper-sync
```

To enable automatic syncing:

```
systemctl enable --now limine-snapper-sync.service
sudo systemctl enable --now snapper-cleanup.timer
```

Further reading: https://gitlab.com/Zesko/limine-snapper-sync#tool-configuration

## Install and configure Zram

Install zram:

```
sudo pacman -S zram-generator
```

Create file `/etc/systemd/zram-generator.conf` with contents:

```
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
```

Now reboot to see the effects!

to confirm that they have applied you can run `swapon` or `zramctl`

-----

Congrats! You have set up your base system, and can now install your preferred Window Manager or Desktop Manager of your choice!
