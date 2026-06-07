# .tmux.conf
My tmux setup.

```
ln -s (pwd)/.tmux.conf ~/.tmux.conf
```

## Backlight off (`tela-off`)

`bin/tela-off` turns off the screen backlight until any key is pressed
(chord `Ctrl+G`). It's hardware-level — it writes to the backlight `brightness`
sysfs node, so it does not touch the framebuffer, WiFi, or running processes;
background work keeps going. The bind opens it in a throwaway window so its
`dd` read has its own stdin; the window closes when the script exits.

It needs the `brightness` node to be group-writable. A udev rule grants that
to the `video` group; since the node already exists at boot (an `add` rule
won't fire retroactively), also fix it in place once:

```sh
echo 'SUBSYSTEM=="backlight", ACTION=="add", KERNEL=="intel_backlight", MODE="0664", GROUP="video"' \
  | sudo tee /etc/udev/rules.d/90-backlight.rules
sudo udevadm control --reload-rules
sudo udevadm trigger -s backlight

sudo chgrp video /sys/class/backlight/intel_backlight/brightness
sudo chmod g+w   /sys/class/backlight/intel_backlight/brightness

sudo usermod -aG video "$USER"   # if not already in the video group
```

`~/permitir-backlight.sh` automates exactly this. The group change only
applies to new sessions.