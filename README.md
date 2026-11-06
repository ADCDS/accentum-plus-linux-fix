# ACCENTUM Plus Linux fix

Stops Sennheiser ACCENTUM Plus headphones from pausing and stuttering your media on Linux when they're connected over USB-C.

This is an unofficial fix, not affiliated with Sennheiser or Sonova.

## Symptoms

- Media pauses, resumes or stutters by itself while the headphones are wired.
- On KDE Plasma, opening the volume popup in the taskbar sets it off. The stuttering starts about 5 seconds after you close the popup.
- The sound gets worse while the popup is open, or while any app is using the headphones' mic.

It looks like a Bluetooth problem, but it isn't. Disconnecting, blocking or forgetting the headphones in your Bluetooth settings won't fix it. In testing, the bug came back with the headphones forgotten and no Bluetooth link at all.

## Check that you have this bug

Confirm the USB ID matches:

```sh
lsusb -d 3542:1002
```

Start your media, then run this to watch media commands:

```sh
dbus-monitor --session "type='method_call',interface='org.mpris.MediaPlayer2.Player'"
```

Open and close the volume popup, then wait about 10 seconds. With this bug, you'll see a fast stream of `member=Pause` and `member=Play` lines. Press Ctrl+C to stop watching.

## Fix

```sh
git clone https://github.com/ADCDS/accentum-plus-linux-fix
cd accentum-plus-linux-fix
sudo install -m 644 70-accentum-plus-latched-media-keys.hwdb /etc/udev/hwdb.d/
sudo systemd-hwdb update
sudo udevadm control --reload
sudo udevadm trigger --subsystem-match=input --action=change
```

To confirm the rule is applied, run this and check that it prints two `KEYBOARD_KEY` lines:

```sh
EVENT=$(grep -A4 'ACCENTUM Plus Consumer Control' /proc/bus/input/devices | grep -oE 'event[0-9]+')
udevadm info "/dev/input/$EVENT" | grep KEYBOARD_KEY
```

The rule only matches this USB ID, and it only ignores the Play (`0xB0`) and Pause (`0xB1`) signals. The regular Play/Pause (`0xCD`), next and previous signals stay mapped, and the touch controls kept working in testing.

## What causes it

1. Something opens the headphones' mic. Plasma's volume popup does this to draw its level meters.
2. The headphones send a Pause signal over USB (HID report `01 20 00`).
3. When the mic closes, they send a Play signal (`01 10 00`) with no key release. PipeWire keeps the mic open for about 5 seconds after the last app stops using it, which is why this comes late.
4. Linux reports that as a Play key being held down.
5. KDE auto-repeats the held key as its play/pause shortcut, about 600 ms after the press and then every 40 ms. Your player gets told to pause and play 25 times a second.

In testing, that added up to 973 commands in 39 seconds, until something cancelled the repeat, most likely another key press. These tests all had audio playing when the mic closed.

The headphones' USB descriptor marks these two signals as latched states rather than momentary buttons, so this looks deliberate: pause the music during a call and resume it afterwards.

## Optional: stop the quality drop

The rule above doesn't change the sound getting worse while the mic is in use. That sounds like a built-in call mode, and it looks like the PC can't turn it off. If you don't use this mic, you can remove it so nothing can open it:

```sh
CARD=$(pactl list short cards | awk '/ACCENTUM/{print $2}')
pactl set-card-profile "$CARD" output:analog-stereo
```

To turn the mic back on:

```sh
CARD=$(pactl list short cards | awk '/ACCENTUM/{print $2}')
pactl set-card-profile "$CARD" output:analog-stereo+input:mono-fallback
```

If you turn it off, make sure you have another mic connected. With no other mic available, voice apps may seem fine but send silence.

## Undo

```sh
sudo rm /etc/udev/hwdb.d/70-accentum-plus-latched-media-keys.hwdb
sudo systemd-hwdb update
```

Then unplug and replug the headphones. The old key mapping stays until they reconnect.

## Tested on

So far this has only been tested on one setup:

- ACCENTUM Plus, USB ID `3542:1002`, device version 29.11
- Debian 13, kernel 6.12, systemd 257
- KDE Plasma 6.3.6 on Wayland, PipeWire 1.4.2, WirePlumber 0.5.8
- Brave as the media player

Other desktops and other Sennheiser models haven't been tested. The call mode explanation is based on how it sounds, not on a measurement.

If you try this on a different setup, please open an issue and say whether it worked.

## License

[MIT](LICENSE)
