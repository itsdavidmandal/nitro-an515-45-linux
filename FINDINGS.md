# AN515-45 firmware findings

How this laptop was made to work on Linux, in plain language, and exactly what
its firmware does and does not support. Written 2026-10-08.

Test machine: Acer Nitro AN515-45 (AMD), BIOS V1.08, Ubuntu kernel 7.0.0-34-generic.

---

## The situation

Your laptop's fans and keyboard lights aren't controlled by Windows or Ubuntu
directly. They're controlled by a tiny chip inside the laptop. On Windows,
NitroSense was just a pretty window that sent that chip messages like "spin the
fan to 70%". The chip speaks a language called WMI, and it's always been there —
it doesn't know or care which operating system you run.

Ubuntu had no translator for it. The lights and fans weren't broken; nothing was
sending the chip any messages.

## Step 1 — checked what the laptop offers

Linux keeps hardware info in fake files you can read, like `/sys/class/hwmon/`.
Reading them showed: no fan speed anywhere, no "performance mode" setting. But
the chip was advertising two capability codes:

- `79772EC5-04B1-4BFD-843C-61E7F77B6CC9` — fans and turbo
- `7A4DDFE7-5B5D-40B4-8595-4408E0CC7F56` — 4-zone keyboard RGB

Translation: the hardware was ready and listening, Ubuntu just wasn't talking.

## Step 2 — got a translator

Someone already wrote one: **Linuwu-Sense**. People figured out what NitroSense
says to the chip by watching it, then wrote a Linux version. This is why we
didn't start a new project — rebuilding a working translator from scratch would
be weeks of work for something that already exists.

## Step 3 — the translator didn't know this exact laptop

The real problem. Acer sells dozens of Nitro models and they don't all speak
identically. The translator has a list: "AN515-55 speaks dialect A, AN515-58
speaks dialect B...". The **AN515-45 was not on that list**, so it got ignored
entirely.

## Step 4 — found the dialect by testing

Forced the newest dialect (`nitro_v4=1`). Result: fan speeds appeared (1807 rpm,
real numbers), but "performance mode" failed 10 times with error -5 (EIO).

That was precise information: this laptop speaks the *older* dialect, the same
one as the AN515-55. Its firmware genuinely has no performance-mode feature —
a hardware limit, not a bug we can fix.

## Step 5 — added the laptop to the list

14 lines of code saying "AN515-45 exists, it speaks the older dialect
(`nitro_sense = 2`), and it has the 4-zone keyboard." Rebuilt, tested:
fans went from 1785 to **4347 rpm** on command and back to automatic.

---

## Why pushing it upstream matters

Right now those 14 lines exist only on this laptop, in a folder.

**For you:** Ubuntu replaces its core system every few weeks. Each time, the
translator stops working and you'd rebuild it by hand, forever.

**For everyone else:** every other AN515-45 owner on Linux has the same dead
fans and dark keyboard, and no idea why. These 14 lines fix it for all of them.

"Pushing" means sending the lines to the people who maintain the code so they
become part of it. Then it arrives automatically for everyone, including you,
and you stop maintaining it yourself. Two places:

1. **Linuwu-Sense** (the translator project) — easy, works in days.
2. **Linux itself** — the fan-reading ability can go into Linux's own code,
   which already has a smaller built-in Acer translator (`acer-wmi`). Harder,
   slower, reviewed by strangers over email. But then it just works on every
   Linux computer forever, with no extra software, and your name is permanently
   in the Linux kernel history. A far bigger flex than a GitHub repo, because
   it's 14 real lines that fix a real thing.

---

## Confirmed status on this machine

| feature | result |
|---|---|
| fan RPM + 3 temps | works (`hwmon` named `acer`) |
| per-fan speed write | works — 1785>4347 CPU, 2000>5172 GPU |
| RGB effects: modes 2,3,4,5 | works — neon, wave, shifting, zoom |
| RGB modes 0,1,6,7 | dead — static, breath, meteor, twinkling do nothing |
| per-zone static colour | NOT reachable on this firmware (tested thoroughly) |
| battery limiter / calibration / USB charging | exposed |
| platform profile / turbo | NOT supported — firmware returns EIO |
| lcd override, boot animation sound | NOT supported by firmware |

Installed permanently on 2026-10-08 via `sudo make install`: module loads at
boot, `acer_wmi` is blacklisted, `linuwu_sense.service` restores the keyboard
effect, and the `linuwu_sense` group makes fan/RGB writes password-free.

The `nitro` helper script in this repo wraps it all (`nitro status`,
`nitro fan auto`, `nitro zoom red`, `nitro off`); it is symlinked into
`~/.local/bin` so it runs from anywhere.

If permissions come back root-only after a boot, re-apply them with:
`sudo systemd-tmpfiles --create /etc/tmpfiles.d/linuwu_sense.conf`

## The patch

`Linuwu-Sense/src/linuwu_sense.c`, 14 lines, two hunks:

- new quirk `quirk_acer_nitro_an515_45 = { .nitro_sense = 2, .four_zone_kb = 1 }`
  (added just after `quirk_acer_nitro_legacy`)
- DMI entry matching `DMI_PRODUCT_NAME` "Nitro AN515-45", pointing at it
  (added just before the AN515-55 entry)

### RGB findings for the PR (all tested on real hardware)

The keyboard IS genuine 4-zone RGB - a multi-colour wave was visually confirmed,
so `.four_zone_kb = 1` is correct.

Mode sweep 0-7 (`four_zone_mode` = mode,speed,brightness,direction,R,G,B):

| mode | name | lights? | note |
|---|---|---|---|
| 0 | static | NO | firmware silently resets `speed` to 0 |
| 1 | breath | NO | firmware silently resets `speed` to 0 |
| 2 | neon | yes | generates its own colours, RGB args zeroed |
| 3 | wave | yes | generates its own colours, RGB args zeroed |
| 4 | shifting | yes | honours the RGB argument |
| 5 | zoom | yes | honours the RGB argument |
| 6 | meteor | NO | accepts speed, no light |
| 7 | twinkling | NO | accepts speed, no light |

The `speed`-reset on exactly the two dead low modes is the firmware's tell for
"mode not implemented".

`direction` is discarded (always reads back 0) for every mode except 3 and 4.
Tested direction 0/1/2 on static - no effect.

`per_zone_mode` writes are accepted and read back correctly, but never display.
Tested further by patching `set_per_zone_color()` to skip its hardcoded
`set_kb_status(0, 0, brightness, 0,0,0,0)` call (mode 0 is dead here, so the
theory was that it switched the backlight off before colours landed) and writing
zone colours while an animated mode was already running. The animation simply
continued - the per-zone colour registers store values this firmware never
renders. So per-zone static is genuinely unavailable, not a driver bug. That
experiment was reverted; it is not part of the patch.

Practical upshot for any GUI on this model: offer modes 2-5 only, and do not
offer per-zone or static colour.

## Useful commands

```bash
# build
cd Linuwu-Sense && make

# load by hand (deps first - unloading acer_wmi takes them with it)
sudo modprobe -r acer_wmi
sudo modprobe sparse_keymap && sudo modprobe platform_profile
sudo insmod src/linuwu_sense.ko

# back to stock
sudo rmmod linuwu_sense && sudo modprobe acer_wmi

# make permanent (also creates the linuwu_sense group so sudo isn't needed after)
sudo make install

# sysfs root
B=/sys/module/linuwu_sense/drivers/platform:acer-wmi/acer-wmi
cat $B/nitro_sense/fan_speed              # "0,0" = auto
echo "70,70" | sudo tee $B/nitro_sense/fan_speed   # 70% both fans
echo "0,0"   | sudo tee $B/nitro_sense/fan_speed   # back to auto
echo "3,4,100,1,0,0,0" | sudo tee $B/four_zoned_kb/four_zone_mode  # wave
# four_zone_mode = mode,speed,brightness,direction,R,G,B
#   mode 0-7, speed 0-9, brightness 0-100, direction 1-2 (required for modes 3,4)
# per_zone_mode  = RRGGBB,RRGGBB,RRGGBB,RRGGBB,brightness
```

## Known gap

The module must be rebuilt after every kernel update. A DKMS wrapper fixes that
and is the Ubuntu packaging gap worth filling.

## Links


- Linuwu-Sense: https://github.com/0x7375646F/Linuwu-Sense
- GUI frontend: https://github.com/kumarvivek1752/Linuwu-Sense-GUI
- CLI frontend: https://github.com/Order52/linuwu-sense-cli
- older RGB-only module: https://github.com/JafarAkhondali/acer-predator-turbo-and-rgb-keyboard-linux-module
- kernel list for the upstream patch: platform-driver-x86@vger.kernel.org
