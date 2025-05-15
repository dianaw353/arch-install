# Arch Setup

> [!WARNING]
> This is a advance guide usning some core system alternitives. Only follow if you know what your doing!!!

This guide is for UEFI systems

## Change keyboard layout

To list available options(): 
`localectl list-keymaps`

For exsample you chane launguage to Italian you can do:
`loadkeys it`

## Connect to wifi

Heres some basic commands to connect to wifi:

`iwctl`
`device list`
`device name set-property Powered on`
`station name scan`
`station name get-networks`
`station name connect SSID`

To test if your connected to wifi:

`ping archlinux.org`

## Partition System

`cfdisk`

If you see a screen to select patition type select the gpt option
Next, You'll see a screen with partions on it and if you have multiple partitions you can delete them if you have backed up the data on them
Then go to the new tab and enter in how much you need for a boot partition and in this case it will be **200M**
After create a partition thats **4G** and that will be used for a swap partition (you can make this bigger if you would like)
Finally, create a partition that will use the rest of the space you have and hit write and type `yes` when your finish with everything

## Format Partions

To list the partions you have made type:

`lsblk`

With the root partitons format the partition to btrfs (biggest partition):

`mkfs.btrfs /dev/<drive and partition>` 

Next format the boot partition (usally smallest partition):

`mkfs.fat -F 32 /dev/<drive and partition>`

## Mount partitions

First mount root partition:

`mount /dev/<drive and partition> /mnt`

Now we need to mount the boot partition but the `/boot/EFI` is not a directory by default so we will create that first

`mkdir -p /mnt/boot/EFI`
`mount /dev/<drive and partition> /mnt/boot/EFI`

To check if you got everything right run and verify:

`lsblk`

## Create btrfs sub partitions

Comming soon!

## Install base system

`pacstrap /mnt base booster linux linux-lts linux-firmware base-devel btrfs-progs limine helix networkmanager`

## Fstab

`genfstab /mnt > /mnt/etc/fstab`

## Chroot

`arch-chroot /mnt`

## Booster

```
# example
compress: zstd -9 -T0
modules: btrfs
strip: true
```
If your using the btrfs filesystem you need the `btrfs` module
If you have a Intel gpu you dont need a gpu module.
If you have a Amd gpu you need to have the `amdgpu` module.
If you have a Nvidia you need these modules `nvidia,nvidia_modeset,nvidia_uvm,nvidia_drm`

To run booster run:

`sudo /usr/lib/booster/regenerate_images`

## Time zone

`ln -sf /usr/share/zoneinfo/Region/City /etc/localtime`

To verify its the correct time run:

`date`

Now we need to sync the the system clock:

`hwclock --systohc`

## Localization

`helix /etc/locale.gen`

Now you will edit a file that contains all the available locals on the system
In most case if you are in america you will need to uncommet the line that contains `en_US.UTF-8 UTF-8`

Now to finish run:

`locale-gen`

Also you will need to add this `LANG=en_US.UTF-8` to this file `/etc/locale.conf` if you want to use US local

If you want to change keyboard layout (default is US) edit this file `/etc/vconsole.conf` with `KEYMAP=us`

## Network hostname

Now edit `/etc/hostname` with whatever hostname you would like

## Create user

Now create a username you would like to use
`useradd -m -G wheel -s /bin/bash yourusername`

Now create a password that you would like to use

`passwd yourusername`

To be able to use sudo we need to do the following:

`EDITOR=helix visudo`

Uncomment the following line:
`%wheel ALL=(ALL:ALL) ALL`

Finally to test seitch to your user using `su username` and then try to run a sudo command such as updating the system

Optional, if you want to you can now disable the root user by running this command

`passwd -l root`

## Enable core services

`systemctl enable networkmanager`

## Boot loader

`cd /boot/EFI/BOOT`
`cp BOOTIA32.EFI /boot/EFI/BOOT`
`cp BOOTX64.EFI /boot/EFI/BOOT`

How to get PARTUUID and File Type:

`lsblk -o NAME,MOUNTPOINT,FSTYPE,UUID,PARTUUID /dev/<disk>`

Please add this to the limine.conf file and if its not there create it:
```
TIMEOUT=5

/Arch Linux (linux) - Booster
    protocol: linux
    kernel_path: boot():/vmlinuz-linux
    kernel_cmdline: root=PARTUUID=<PARTUUID> rootflags=subvol=@ rw rootfstype=<filetype> quiet
    module_path: boot():/booster-linux.img

/Arch Linux (linux-lts) - Booster
    protocol: linux
    kernel_path: boot():/vmlinuz-linux-lts
    kernel_cmdline: root=PARTUUID=<PARTUUID> rootflags=subvol=@ rw rootfstype=<filetype> quiet
    module_path: boot():/booster-linux-lts.img

```
If you have zram enabled add this to the kernel_cmdline:

`zswap.enabled=0`

## Microcode

Install your microcode for whatever cpu brand your using

`intel-ucode` or `amd-ucode`

## Reboot

Now you can reboot but before we do we need to unmount the drive and we can do that by:

`umount -a`

To reboot just run the `reboot command`

## Connect to the internet

Use command `nmtui` to connect to the internet if your using wifi

## AUR wrapper

To install a AUR wrapper we first need to install `git` and then we can install the aur wrapper of our choosing but in our case we will use `yay`

`sudo pacman -S git`
`git clone https://aur.archlinux.org/yay-bin.git`
`cd yay-bin`
`makepkg -si`

## Helix

Now would be nice to install a package to now have to type `helix` each time you want to run the text edit but instead you can now run `hx`.

`yay -S helix helixbinhx`

And if you want systex highlighting for more programming launguages you can install all or some of these packages

Optional Dependencies:
bash-language-server clang dart elvish gopls haskell-language-server julia lua-language-server marksman python-lsp-server r racket rust-analyzer taplo texlab typescript-language-server typst-lsp vue-language-server vscode-css-languageserver vscode-html-languageserver vscode-json-languageserver yaml-language-server zls

ansible-language-server: for Ansible language support
bash-language-server: for Bash language support
clang: for C/C++ language support
dart: for Dart language support
elvish: for elvish language support
gopls: for Go language support
haskell-language-server: for Haskell language support
julia: for Julia language support
lua-language-server: for Lua language support
marksman: for Marksman language support
python-lsp-server: for Python language support
r: for R and rmarkdown language support
racket: for racket language support
rust-analyzer: for Rust language support
taplo: for TOML language support
texlab: for LaTeX language support
typescript-language-server: for jsx, tsx, typescript language support
typst-lsp: for Typst language support
vue-language-server: for Vue language support
vscode-css-languageserver: for CSS and SCSS support
vscode-html-languageserver: for HTML language support
vscode-json-languageserver: for JSON language support
yaml-language-server: for YAML language support
zls: for Zig language support

## Add audio services

Some core pipewire packages are:

`yay -S pipewire pipewire-jack pipewire-audio pipewire-pulse pipewire-alsa wireplumber`

To have bluetooth support you will need:

`yay -S bluez bluez-utils`

To enable/start audio and bluetooth support run the following:

```
sudo systemctl enable bluetooth.service
sudo systemctl start bluetooth.service
systemctl --user enable pipewire.service
systemctl --user enable pipewire-pulse.service
systemctl --user enable wireplumber.service
systemctl --user start pipewire.service
systemctl --user start pipewire-pulse.service
systemctl --user start wireplumber.service

```

## Set up snapshots

Comming soon!

## Limine shapshots setup

Comming soon!

## Install and configure Zram

Comming soon!

Concrats you have set up your base system and can now install your prefered Window Manager or Desktop Manager of your choice!
