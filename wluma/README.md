# Wluma

Wluma is a tool for Wayland compositors to automatically adjust screen brightness based on the screen contents and amount of ambient light around you.

```
yay -S wluma
```

Copy this file to the following location:

```
mkdir ~/.config/wluma
cp config.toml ~/.config/wluma
```

If needed, edit the backlight name and path in `~/.config/wluma/config.toml`

On hyprland add this line to ~/.config/hypr/hyprland.conf:

```
exec-once = wluma
```

Add this to the following file `/etc/udev/rules.d/90-wluma-backlight.rules`:

```
ACTION=="add", SUBSYSTEM=="backlight", RUN+="/bin/chgrp video /sys/class/backlight/%k/brightness"
ACTION=="add", SUBSYSTEM=="backlight", RUN+="/bin/chmod g+w /sys/class/backlight/%k/brightness"

```

See if your user is in `video` group:

```
groups
```

If not, add yourself:

```
sudo usermod -aG video $USER
```
