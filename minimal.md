# Arch Install Setup (minimal)

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

```fule
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

## Network hostname

Now, edit `/etc/hostname` with whatever hostname you would like.

## Create user

Now, create a username you would like to use.

```
useradd -mG wheel,input,video -s /bin/bash yourusername
```

Now, create a password that you would like to use:

```
passwd yourusername
```

To be able to use sudo, we need to do the following:

```
EDITOR=helix visudo
```

Uncomment the following line:

```
%wheel ALL=(ALL:ALL) ALL
```

Finally, to test, switch to your user using `su username` and then try to run a `sudo` command such as updating the system.


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

Or you can do it automatically by running this but make sure `DISK=/dev/sdXn` is the correct drive and partion to boot off of
```
DISK=/dev/sdXn && PARTUUID=$(lsblk -o PARTUUID -nr "$DISK") && BRAND=$(lscpu | grep -qi amd && echo amd || echo intel) && sed "s|<PARTUUID>|$PARTUUID|g; s|<brand>|$BRAND|g" limine.conf | sudo tee /boot/limine.conf > /dev/null
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

## Reboot

Now, you can reboot, but before we do, we need to unmount the drive and we can do that by:

```
umount -a
```

To reboot, just run the `reboot` command.

## Connect to the internet

Use command `nmtui` to connect to the internet if you're using WiFi.

-----

Congrats! You have set up your base system, and can now install your preferred Window Manager or Desktop Manager of your choice!
