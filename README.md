# Nitro AN515-45 on Linux — fan and RGB control

NitroSense equivalent for the **Acer Nitro 5 AN515-45** (AMD) on Linux: fan
speeds, temperatures, keyboard RGB and battery limiter from the command line.

The kernel work belongs to [Linuwu-Sense](https://github.com/0x7375646F/Linuwu-Sense).
This repo adds the three things that were missing for this specific model:

1. **A 14-line patch** adding the AN515-45 to Linuwu-Sense's DMI table. Without
   it the model matches nothing and you get no fan or RGB control at all.
   ([PR upstream](https://github.com/0x7375646F/Linuwu-Sense/pulls) — apply the
   patch here until it merges.)
2. **`nitro`** — a small wrapper so you aren't writing to 90-character sysfs
   paths by hand.
3. **[FINDINGS.md](FINDINGS.md)** — exactly which features this laptop's
   firmware supports, and which it silently ignores, all verified on hardware.

Tested on AN515-45, BIOS V1.08, Ubuntu kernel 7.0.0-34-generic.

## What works on this model

| feature | status |
|---|---|
| Fan RPM, both fans | works |
| 3 temperature sensors | works |
| Set fan speed 0-100%, per fan | works |
| Keyboard RGB effects: neon, wave, shifting, zoom | works |
| Battery charge limit (80%), USB charging while off | works |
| Keyboard static / per-zone colour | **firmware ignores it** |
| Performance / turbo profile | **firmware returns -EIO** |
| LCD override, boot animation sound | **not implemented in firmware** |

The last three are not fixable in software. The WMI calls exist but the firmware
either errors or accepts and discards them. Details and test method in
[FINDINGS.md](FINDINGS.md).

## Install

```bash
# 1. build Linuwu-Sense with the AN515-45 patch
sudo apt install -y git build-essential linux-headers-$(uname -r)
git clone https://github.com/0x7375646F/Linuwu-Sense.git
cd Linuwu-Sense
git apply /path/to/0001-add-an515-45-support.patch   # until upstream merges
sudo make install

# 2. the wrapper
sudo install -m 755 nitro /usr/local/bin/nitro
```

Then **log out and back in** — `make install` puts you in a new `linuwu_sense`
group, and group membership only applies to new sessions. Without that, every
command asks for your password.

Uninstall with `sudo make uninstall` in the Linuwu-Sense directory.

## Usage

```
nitro status                      temps, fan rpm, current modes
nitro fan auto | max | 70 [60]    fan percent (cpu [gpu]), auto = firmware curve
nitro off                         keyboard lights off
nitro neon  [speed] [bright]      effects that pick their own colours
nitro wave  [speed] [bright]
nitro shift <colour> [speed] [bright]   effects that take your colour
nitro zoom  <colour> [speed] [bright]
nitro bat on|off                  cap charging at 80%
```

Colours: `red green blue cyan pink yellow orange white`, or `"R,G,B"`.
Speed `0-9`, brightness `0-100`.

```bash
nitro fan max        # before a long compile
nitro fan auto       # hand control back to the firmware
nitro zoom red
nitro status
```

## Raw sysfs, if you'd rather skip the script

```bash
B=/sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi
echo "70,70" > $B/nitro_sense/fan_speed              # 70% both fans
echo "0,0"   > $B/nitro_sense/fan_speed              # auto
echo "3,4,100,1,0,0,0" > $B/four_zoned_kb/four_zone_mode   # wave
```

`four_zone_mode` is `mode,speed,brightness,direction,R,G,B`. Working modes on
this laptop are 2 (neon), 3 (wave), 4 (shifting) and 5 (zoom); 4 and 5 use the
colour you pass. Modes 0, 1, 6 and 7 do nothing — see FINDINGS.md.

## Known issue

The module is out-of-tree, so **a kernel update stops it loading** until you
re-run `sudo make install`. You'll notice because `nitro` reports
`linuwu_sense not loaded`. Packaging it with DKMS would fix this permanently and
is the obvious next job.

## Credit

All the hard reverse engineering is [Linuwu-Sense](https://github.com/0x7375646F/Linuwu-Sense)'s,
which was itself derived from Acer's PredatorSense. This repo is a model quirk,
a shell script, and a day of poking at firmware to find out what it really does.
