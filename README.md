# mudi7-battery-limit

Charge limit for the **GL.iNet Mudi 7 (GL-E5800)**.

Stops charging at a configurable state of charge while the router keeps running
on USB-C power. The battery stays in the device, so it still works as a UPS
during a power blip — no need to pull the cell out for stationary 24/7 use.

Verified on firmware **4.8.5**.

---

## Why this is needed

The only workaround discussed so far is physically removing the battery, which
also removes the UPS behaviour and means opening the battery cover over and over.
There is currently no GUI setting and no UCI option for a charge limit.

The earlier community workaround — writing `constant_charge_voltage` on the buck
charger — does **not** work when the power supply negotiates **PD PPS**, because
in that mode the buck is not the active charge path at all.

## How it works

The Mudi 7 has a two-stage charging architecture:

| Chip | Role |
|---|---|
| **SGM41600** | switched-capacitor charger, 2:1 divider, fed directly from the PPS source. This is the fast-charge path. |
| **SGM41542S** | buck charger with NVDC power path. Supplies the system, charges when the pump is not running. |
| **CW2217** | fuel gauge |
| **AW35615** | USB-C / PD3.0 controller (two ports) |

`quec_battery` is the userspace daemon that arbitrates between them. Per the
SGM41600 datasheet, the host must disable the main charger before enabling the
slave — which is why the buck reports `charge_en = 0` while the pump is running.

The limit therefore needs **both** levers:

1. `charge_en = 0` on the SGM41600 — turns the pump off.
   `quec_battery` responds by re-enabling the SGM41542S buck.
2. `vreg` on the SGM41542S set below the current cell voltage — so the buck
   does not charge either.

Result: charge current exactly **0 mA**, system still powered from the adapter,
battery neither charged nor discharged.

Both writes are restrictive only. `vreg` is always far below `bat_ovp_uv`
(4,400,000 µV), so overcharging is impossible. If the script dies, the device
simply charges normally again.

### Measured evidence

Charging via the pump, 8.8 V PPS input:

```
charge_en=2  ibat_adc=2586 mA  ibus_adc=1536 mA  vbus_adc=8792 mV  vbat_adc=4288 mV
```

8792 / 4288 = 2.05 — the 2:1 divider. Input current is roughly half the battery
current, exactly as the switched-capacitor topology specifies.

With both levers applied:

```
18:56:26 vnow=4008000 inow=0 cap=68
18:56:31 vnow=4008000 inow=0 cap=68
...
18:58:33 vnow=4009000 inow=0 cap=68
```

Flat voltage, zero current, stable capacity, router still online.

### Notes on the cell

This is a **4.4 V** cell, not a 4.2 V one — both `vreg` (4,400,000 µV) and
`bat_ovp_uv` (4,400,000 µV) confirm it. Percentages derived from 4.2 V cell
curves will be wrong. Measured internal resistance is roughly **52 mΩ**, so at
2.5 A charge current the terminal voltage sits ~130 mV above the open-circuit
voltage.

The GUI percentage and the fuel gauge percentage differ by a roughly constant
offset. The script always uses the gauge
(`/sys/class/power_supply/cw221X-bat/capacity`).

---

## Install

```sh
wget -O /usr/bin/glbattlimit https://raw.githubusercontent.com/ChiliApple/mudi7-battery-limit/main/glbattlimit
chmod +x /usr/bin/glbattlimit
glbattlimit status
```

`/usr/bin` is not preserved across firmware updates — reinstall after a flash.

## Usage

```sh
glbattlimit on          # limit at 80 %
glbattlimit on 75       # limit at 75 %
glbattlimit off         # back to normal charging
glbattlimit status      # show state
```

`on` starts a small background watcher. It exits by itself when the charger is
unplugged and restores the factory state, so the next plug-in charges normally
unless you enable the limit again. Nothing runs when the limit is off.

Log output goes to syslog:

```sh
logread | grep glbattlimit
```

---

## Caveats

- Not an official GL.iNet feature. Firmware updates may change the sysfs layout;
  the script checks for the required paths and refuses to run if they are missing.
- Tested only on GL-E5800 firmware 4.8.5.
- The watcher polls every 15 s, so charging can continue for up to 15 s past the
  threshold after a plug event.
- `glbattlimit off` restores `vreg` and charging resumes via the buck immediately.
  The faster pump path returns on the next unplug/plug cycle.

## What GL.iNet would need to do

Nothing in the hardware or the drivers is missing. The levers are already there
and already writable. A supported implementation would be a GUI setting backed by
a UCI option, applied by `quec_battery` itself — which already reads
`/etc/config/qlbattery` for `max_current_ma`, `min_shutdown_mv`, `max_pd_vbus_mv`
and `pd_full_mv`. A fifth option would be enough.

## Licence

MIT
