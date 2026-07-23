# mudi7-battery-limit

Charge limit for the **GL.iNet Mudi 7 (GL-E5800)**. Stops charging at a
configurable state of charge while the router keeps running from USB-C power —
so you no longer have to pull the battery out for stationary 24/7 operation,
and the UPS behaviour during a power blip stays intact.

Tested on firmware **4.8.5**. No kernel module, no patched firmware, nothing
written to flash: a single POSIX shell script writing to sysfs attributes that
the stock drivers already expose.

```
# glbattlimit on 80 gui
GUI 80 % -> gauge 71 % (estimate, see README)
Charge limit active: 71 % gauge / ~80 % GUI
Currently: 65 % gauge / ~71 % GUI
Released automatically when the charger is unplugged.
```

> **The router displays a different percentage than the fuel gauge reports.**
> The script understands both scales. See
> [The two percentage scales](#the-two-percentage-scales) before picking a number.

---

## Install

### Option A — package, no SSH needed

Download
[`glbattlimit_1.0.0-1_all.ipk`](https://github.com/ChiliApple/mudi7-battery-limit/releases/latest)
and install it from the router's web interface:

**LuCI** (Admin Panel → System → Advanced Settings) → **System** → **Software** →
**Upload Package…**

LuCI will warn that the source is untrusted and show the checksums — that is
normal for any manually uploaded package, since opkg only verifies signatures on
feed indexes, not on individual files. Compare the MD5 against the release page
and continue.

Verified working on firmware 4.8.5: the package then appears in the GL.iNet
Admin Panel under **APPLICATIONS → Plug-ins** with an uninstall button.
Uninstalling releases the limit first, so the charger returns to its factory
state even when removed from the GUI. The size column may read *unknown* there —
that value comes from a feed index, which a manually installed package has none
of.

Or install over SSH:

```sh
cd /tmp && wget -O gbl.ipk https://github.com/ChiliApple/mudi7-battery-limit/releases/download/v1.0.0/glbattlimit_1.0.0-1_all.ipk && opkg install gbl.ipk
```

### Option B — script, via SSH

```sh
wget -O /usr/bin/glbattlimit https://raw.githubusercontent.com/ChiliApple/mudi7-battery-limit/main/glbattlimit && chmod +x /usr/bin/glbattlimit && glbattlimit status
```

`status` must print a complete block ending with the `Buck vreg` line. If it
does not, the download was incomplete — do not use the file.

`wget` on OpenWrt may report a **smaller** transfer size than the file actually
has, because GitHub serves the raw file gzip-compressed. Check the file, not
the transfer counter:

```sh
wc -c /usr/bin/glbattlimit; sh -n /usr/bin/glbattlimit && echo "SYNTAX OK"
```

A firmware update overwrites `/usr/bin`, so re-run the install line after
flashing. Nothing else needs to be restored.

## Usage

| Command | Effect |
|---|---|
| `glbattlimit on [PERCENT]` | enable limit on the **gauge** scale (default 80, range 50–100) |
| `glbattlimit on [PERCENT] gui` | enable limit on the **GUI** scale, converted automatically |
| `glbattlimit off` | release the limit, resume normal charging |
| `glbattlimit status` | current state in both scales, plus voltage, current and charger state |

```
# glbattlimit status
Limit     : active (71 % gauge / ~80 % GUI, PID 26302)
Capacity  : 65 % gauge / ~71 % GUI (estimated)
Voltage   : 4145 mV
Current   : 0 mA  (+charging -discharging 0=blocked)
Charger   : online=1
Pump      : charge_en=0  (0=off 1=bypass 2=2:1)
Buck vreg : 3900000 uV  (factory 4400000)
```

`Current : 0 mA` together with `online=1` is the working state: the charger is
connected, the router runs from it, and nothing flows into the cell.

The watcher started by `on` **terminates itself when the charger is unplugged**
and restores the factory state on the way out. Plugging in again charges
normally until you run `on` once more. When no limit is active, nothing runs in
the background.

## The two percentage scales

Every value this script reads and prints comes from the **CW2217 fuel gauge**
(`/sys/class/power_supply/cw221X-bat/capacity`). The GL.iNet web GUI and the
touchscreen show a **different and consistently higher** number.

The relationship is not a fixed offset — it is linear with a slope of about
1.39, which is why the apparent gap grows with the state of charge. Four
measured pairs:

| gauge | GUI measured | linear fit | deviation |
|---|---|---|---|
| 62 | 67 | 67.0 | 0.0 |
| 65 | 71 | 71.2 | +0.2 |
| 71 | 80 | 79.5 | −0.5 |
| 78 | 89 | 89.2 | +0.2 |

```
GUI = 1.3867 * gauge - 18.93
```

Extrapolated, GUI 100 % corresponds to gauge ~86 and GUI 0 % to gauge ~14 — the
GUI stretches the usable window onto a 0–100 range. That is consistent with the
`v42_cap` symbol inside `quec_battery`: a capacity referenced to a 4.2 V full
charge on a cell that actually charges to 4.4 V, with the lower end near
`min_shutdown_mv` (3400 mV).

The script implements this conversion, so you can work in either scale:

```sh
glbattlimit on 71          # 71 % gauge  (~80 % GUI)
glbattlimit on 80 gui      # 80 % GUI    -> converted to 71 % gauge
```

| GUI target | `glbattlimit on` (gauge) |
|---|---|
| 70 % | 64 |
| 75 % | 68 |
| **80 %** | **71** |
| 85 % | 75 |
| 90 % | 79 |

One gauge point is worth about 1.4 GUI points, so not every GUI value can be
hit exactly. Round trips are stable up to roughly 85 %; above that they can
land one point high (GUI 90 → gauge 79 → GUI 91). `status` always prints both
numbers, so the value that actually applies is never in doubt.

### Wanted: data points near the edges

The formula is validated only between gauge 62 and 78. Outside that band it is
an extrapolation, and fuel gauges are typically non-linear near the ends — the
line may bend below ~40 % and above ~85 %. It may also differ between units and
with cell age.

If you use this script, please help: read the number shown by the GUI or
touchscreen, run `glbattlimit status` at the same moment, and report both.
Pairs around **40–50 %** and **82–95 %** are the most useful.

Open an [issue](https://github.com/ChiliApple/mudi7-battery-limit/issues) with:

```
GUI: 80    gauge: 71    charger plugged: yes    firmware: 4.8.5
```

## Update

**Disable the limit first.** A running watcher is an executing shell script,
and `ash` reads scripts incrementally rather than loading them into memory —
replacing the file underneath a running instance leads to undefined behaviour.

```sh
glbattlimit off
```

Fetch and check the new version:

```sh
wget -O /usr/bin/glbattlimit https://raw.githubusercontent.com/ChiliApple/mudi7-battery-limit/main/glbattlimit && chmod +x /usr/bin/glbattlimit && sh -n /usr/bin/glbattlimit && glbattlimit status
```

Re-enable with your limit:

```sh
glbattlimit on 80 gui
```

`off` restores the factory CV value before the file is replaced, so an
interrupted update cannot leave charging blocked. Should that ever happen
anyway, unplugging and replugging the charger once restores the factory state
unconditionally.

> GitHub's raw CDN caches for a few minutes. Right after a release,
> `wget -qO- <raw-url> | wc -c` may still return the previous size.

## Uninstall

```sh
glbattlimit off; rm -f /usr/bin/glbattlimit /tmp/glbattlimit.pid /tmp/glbattlimit.limit; echo "glbattlimit removed"
```

`off` stops the watcher and restores the factory CV value before the file is
removed. Nothing else is left behind: the script never writes to flash, never
creates a UCI section and never installs an init script. If the file is deleted
while a limit is still active, unplug and replug the charger once.

## Why

The Mudi 7 has an integrated battery, and in stationary use it sits at 100 %
around the clock. Until now the only mitigation was to physically remove the
battery — which also removes the UPS behaviour, and means opening the cover
repeatedly for anyone who alternates between stationary and travel use.

A previously published workaround caps `constant_charge_voltage` on the
SGM41542S buck charger from a hotplug script. That works for QC, fixed-PDO and
BC1.2 charging, but **not for USB-PD PPS** — which is what the stock Mudi 7
charger negotiates. This script covers the PPS case.

## How it works

The Mudi 7 charges through a two-stage architecture:

| Chip | Role | sysfs |
|---|---|---|
| **SGM41600** | switched-capacitor charger, 2:1 divider, fed directly from the PPS source | `/sys/class/power_supply/sgm41600-standalone/device/sgm41600/` |
| **SGM41542S** | buck charger with NVDC power path; supplies the system, charges on non-PPS sources | `/sys/class/power_supply/charger/device/sgm41542s/` |
| **CW2217** | fuel gauge | `/sys/class/power_supply/cw221X-bat/` |
| **AW35615** | USB-C / PD 3.0 controller (two ports) | `/sys/bus/i2c/drivers/aw35615/*/AW35615-*/` |

Under PPS the switched-capacitor charger does the work, which bypasses the
buck's CV setting entirely. Measured: `VBUS 8792 mV / VBAT 4288 mV = 2.05`, and
`ibus 1536 mA` against `ibat 2586 mA` — the classic 2:1 signature.

When the limit is reached the script therefore does two things:

1. `charge_en = 0` on the **SGM41600** turns the charge pump off. The vendor
   daemon `quec_battery` then re-enables the buck, exactly as the SGM41600
   datasheet requires: *"the host must disable the main charger before enabling
   the SGM41600"*.
2. `vreg` on the **SGM41542S** is set below the current cell voltage, so the
   buck does not charge either.

The result is **0 mA into the cell** while the router keeps running from the
adapter:

```
18:56:26 vnow=4008000 inow=0 cap=68
18:56:31 vnow=4008000 inow=0 cap=68
18:56:36 vnow=4008000 inow=0 cap=68
...held for 60 s, no drift
```

Both writes are restrictive only — `vreg` stays far below the cell's
`bat_ovp_uv` of 4.4 V. Overcharging is not possible; the worst failure mode is
that the device charges normally.

## Notes and caveats

- The cell is a **4.4 V high-voltage cell** (`vreg` factory default 4400000,
  `bat_ovp_uv` 4400000). Voltage-to-SoC rules of thumb for 4.2 V cells do not
  apply to it.
- `quec_battery` rewrites the charger registers on every plug **uevent**, not
  periodically. The watcher re-applies the gate every 15 s, which covers that.
- State lives in `/tmp` and is gone after a reboot. Re-run `on` if you want the
  limit back.
- `set_cap` / `gl_otg.typec1.threshold` is **not** a charge limit. It is the GUI
  setting for the minimum SoC at which USB power output is cut, and changing it
  has no effect on charging.
- `pd_full_mv` in `/etc/config/qlbattery` has no effect under PPS. Tested: set
  to 4000, the cell still charged to 4.29 V at full current.

## Disclaimer

Unofficial, not affiliated with GL.iNet.

The script only ever lowers the charge voltage and disables charging — it never
raises any limit, and `vreg` stays far below the cell's own over-voltage
threshold. Even so: it writes to charger registers that the vendor daemon also
manages, and GL.iNet's warranty terms exclude devices where *"the software has
been manipulated"*. Their warranty covers hardware rather than software
elements, and third-party plug-ins are explicitly outside their support scope.
Use at your own risk.

## License

MIT
