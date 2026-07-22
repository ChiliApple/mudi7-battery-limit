# mudi7-battery-limit

Charge limit for the **GL.iNet Mudi 7 (GL-E5800)**. Stops charging at a
configurable state of charge while the router keeps running from USB-C power,
so you do not have to pull the battery out for stationary 24/7 operation.

Tested on firmware **4.8.5**. No kernel module, no patched firmware — a single
POSIX shell script writing to sysfs attributes the stock driver already exposes.

```
# glbattlimit on 80
Charge limit active: 80 %   (currently 78 %)
Released automatically when the charger is unplugged.
```

---

## Important: the percentage is NOT the number on the screen

All values used and shown by this script come from the **CW2217 fuel gauge**
(`/sys/class/power_supply/cw221X-bat/capacity`). The GL.iNet GUI and the
touchscreen show a **different, consistently higher** number — observed offsets
ranged from about 5 to 11 points.

```
GUI 80 %  ->  gauge 71 %
GUI 89 %  ->  gauge 78 %
GUI 67 %  ->  gauge 62 %
```

The offset is not constant, so there is no simple conversion. The likely reason:
`quec_battery` contains a `v42_cap` symbol, suggesting a second capacity value
referenced to a 4.2 V full charge, while the gauge reports against the cell's
actual 4.4 V full charge.

**Consequence:** `glbattlimit on 80` limits to 80 % *gauge*, which the GUI will
display as roughly 85–90 %. Always compare against `glbattlimit status`, never
against the screen. If you want the screen to read 80 %, set the limit lower —
around 70–75 — and verify with `status`.

---

## Why

The Mudi 7 has an integrated battery, and in stationary use it sits at 100 %
around the clock. The only workaround so far was to physically remove the
battery, which also removes the UPS behaviour during a power blip.

A previously published workaround caps `constant_charge_voltage` on the
SGM41542S buck charger via a hotplug script. That works for QC / fixed-PDO /
BC1.2 charging, but **not for USB-PD PPS** — which is what the stock Mudi 7
charger negotiates. This script covers the PPS case.

## How it works

The Mudi 7 uses a two-stage charging architecture:

| Chip | Role | sysfs |
|---|---|---|
| **SGM41600** | switched-capacitor charger, 2:1 divider, fed directly from the PPS source | `/sys/class/power_supply/sgm41600-standalone/device/sgm41600/` |
| **SGM41542S** | buck charger with NVDC power path, supplies the system, charges on non-PPS sources | `/sys/class/power_supply/charger/device/sgm41542s/` |
| **CW2217** | fuel gauge | `/sys/class/power_supply/cw221X-bat/` |
| **AW35615** | USB-C / PD3.0 controller (2 ports) | `/sys/bus/i2c/drivers/aw35615/*/AW35615-*/` |

Under PPS the switched-cap charger does the work, so the buck's CV setting is
bypassed entirely. Measured: `VBUS 8792 mV / VBAT 4288 mV = 2.05`, and
`ibus 1536 mA` vs `ibat 2586 mA` — the classic 2:1 signature.

The script therefore does two things when the limit is reached:

1. `charge_en = 0` on the **SGM41600** — turns the charge pump off.
   `quec_battery` (the vendor daemon) then re-enables the buck, as required
   by the SGM41600 datasheet: *"the host must disable the main charger before
   enabling the SGM41600"*.
2. `vreg` on the **SGM41542S** set below the current cell voltage — so the
   buck does not charge either.

Result: **0 mA into the cell**, router still running from the adapter.

```
18:56:26 vnow=4008000 inow=0 cap=68
18:56:31 vnow=4008000 inow=0 cap=68
18:56:36 vnow=4008000 inow=0 cap=68
...held for 60 s, no drift
```

Both writes are restrictive only — `vreg` stays far below the cell's
`bat_ovp_uv` of 4.4 V. Overcharging is not possible; the worst failure mode is
that the device charges normally.

## Install

On the router (SSH):

```sh
wget -O /usr/bin/glbattlimit https://raw.githubusercontent.com/ChiliApple/mudi7-battery-limit/main/glbattlimit && chmod +x /usr/bin/glbattlimit && glbattlimit status
```

A firmware update overwrites `/usr/bin`, so re-run the line after flashing.

## Uninstall

```sh
glbattlimit off; rm -f /usr/bin/glbattlimit /tmp/glbattlimit.pid /tmp/glbattlimit.limit; echo "glbattlimit removed"
```

`off` stops the watcher and restores the factory CV value before the file is
deleted. Nothing else is left behind — the script never writes to flash, never
creates a UCI section and never installs an init script. If the file is deleted
while a limit is still active, unplug and replug the charger once to restore the
factory state.

## Usage

| Command | Effect |
|---|---|
| `glbattlimit on [PERCENT]` | enable limit, default 80, valid range 50–100 |
| `glbattlimit off` | release limit, normal charging |
| `glbattlimit status` | current state, capacity, voltage, current, chip state |

The watcher started by `on` **exits by itself when you unplug the charger** and
restores the factory state. Plugging in again charges normally until you run
`on` once more. Nothing runs in the background when the limit is not active.

## Notes and caveats

- Percentages are fuel gauge values, not GUI values — see the section at the top.
- The cell is a **4.4 V high-voltage cell** (`vreg` factory default 4400000,
  `bat_ovp_uv` 4400000). Voltage/SoC rules of thumb for 4.2 V cells do not
  apply.
- `quec_battery` rewrites the charger registers on every plug **uevent**, not
  periodically. The watcher re-applies the gate every 15 s, which covers it.
- Nothing is written to flash. State lives in `/tmp` and disappears on reboot.
- `set_cap` / `gl_otg.typec1.threshold` is **not** a charge limit — that is the
  GUI setting for the minimum SoC at which USB power output is cut. Setting it
  does not affect charging.
- `pd_full_mv` in `/etc/config/qlbattery` has no effect under PPS (tested:
  set to 4000, cell still charged to 4.29 V at full current).

## Disclaimer

Unofficial. Not affiliated with GL.iNet. Use at your own risk — running
third-party scripts may affect warranty support. The script only ever lowers
charge voltage and disables charging; it never raises any limit.

## License

MIT
