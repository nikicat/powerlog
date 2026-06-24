# powerlog

A single-file Bash tool that shows **CPU power / thermal / charging / clock / load**
state on Linux laptops — as a one-shot report or a live one-line-per-second table.

It pulls everything from `sysfs`, `turbostat`, `rdmsr` and the battery gauge, and
focuses on telling you *why* the CPU is doing what it's doing (which power/thermal
limit is active, what the real effective clock is, where the watts are going).

> A small personal tool that grew out of chasing down why a Dell XPS 13 kept
> throttling. See [Notes](#notes--caveats).

## Example

```
$ sudo powerlog -w 1
TIME         MHz busy   pkgW   psys   sysW  restW  pkgT   fan   bl  chgW  SOURCE bat% mode   LIMITER  thr+
22:31:10    1547  12%   4.39    5.1    n/a    n/a   51°C  6096  55%    45  AC     100% bal    none        0
22:31:12    2790  31%   9.88    7.4    n/a    n/a   71°C  6105  55%    45  AC     100% perf   PWR         0
22:31:14    1518  10%   3.92    3.0    6.8    2.9   50°C  5476  55%     -  BATT    98% bal    none        0
```

```
$ sudo powerlog            # one-shot detailed report
===== POWER / CHARGING =====
  source        : AC
  battery       : 100%  flow +0.0W  (+charge / -discharge)
  ucsi-source-psy-USBC000:001  online=1 45W
  platform(psys): 6.1 W
  RAPL limits   : PL1=64W  PL2=64W
===== CLOCK =====
  power mode    : balanced  (choices: low-power balanced performance)
  driver/gov    : intel_pstate / powersave
  ...
```

## Usage

```
powerlog              one-shot detailed report (sections, self-labeled)
powerlog -w [SEC]     periodic: one summary line every SEC seconds (default 1)
powerlog -h           help, including a description of every column
```

## Columns (periodic mode)

| col | meaning |
|-----|---------|
| `MHz` | effective core frequency, busy-averaged (turbostat `Bzy_MHz`) — the clock the cores *actually* ran at, not the requested one |
| `busy` | mean CPU utilization across all cores (%) |
| `pkgW` | CPU package power, RAPL (W) |
| `psys` | raw RAPL `psys` domain (W) — see caveat below; shown for completeness, not used in `restW` |
| `sysW` | true whole-system power from **battery discharge** (W); `n/a` on AC |
| `restW` | `sysW − pkgW` = non-CPU system power (display + SSD + board + …); battery only |
| `pkgT` | package temperature (°C) |
| `fan` | fastest fan speed (RPM) |
| `bl` | backlight brightness (% of `actual_brightness`) |
| `chgW` | charger PD-contract watts offered by the online USB-C/Mains adapter; `-` on battery |
| `SOURCE` | `AC` or `BATT` |
| `bat%` | battery charge level |
| `mode` | OS power profile (`/sys/firmware/acpi/platform_profile`): `perf`, `bal`, `lowpwr`, … |
| `LIMITER` | active perf cap from `IA32_PACKAGE_THERM_STATUS` (MSR `0x1b1`): `PWR` (power/RAPL), `THERM` (Tjmax), `PROCHOT` (external), `none` |
| `thr+` | package thermal-throttle events since the previous line (delta) |

The one-shot report additionally shows RAPL PL1/PL2 limits, all thermal zones,
cumulative throttle counts, governor/EPP/base-max clocks, and the top CPU consumers.

## Requirements

- Linux with an Intel CPU (uses `intel_pstate`, `intel-rapl`, `IA32_PACKAGE_THERM_STATUS`).
- `turbostat` (from `linux-tools` / `kernel-tools`) — for effective MHz + package/psys watts.
- `rdmsr` (from `msr-tools`) — for the `LIMITER` column.
- Everything degrades gracefully: without these (or without privilege) the affected
  columns show `n/a` / `?` and `MHz` falls back to the *requested* frequency.

## Privilege

Effective MHz, package watts and the limiter reason read MSRs/RAPL, which need root:

```bash
sudo powerlog -w
```

To run **without** sudo, grant the two binaries the capability and relax MSR access:

```bash
sudo setcap cap_sys_rawio+ep "$(command -v turbostat)"
sudo setcap cap_sys_rawio+ep "$(command -v rdmsr)"
echo 'KERNEL=="msr[0-9]*", GROUP="msr", MODE="0640"' | sudo tee /etc/udev/rules.d/55-msr.rules
sudo groupadd -f msr && sudo usermod -aG msr "$USER" && echo msr | sudo tee /etc/modules-load.d/msr.conf
```

## Install

```bash
install -Dm755 powerlog ~/.local/bin/powerlog
```

## Notes & caveats

A few things this tool learned the hard way on a **Dell XPS 13 (9310, Tiger Lake)**;
they're documented here because they're easy to misread:

- **`psys` is not whole-system power on every machine.** The RAPL `psys` domain is
  only meaningful if the OEM wired the platform power sense. On the XPS 9310 it reads
  *at or below* package power (impossible for a true platform sensor), so it's shown
  raw, for completeness, and **never** used to derive `restW`. The honest whole-system
  number is `sysW`, taken from battery discharge — which is why `sysW`/`restW` are
  `n/a` on AC (no trustworthy whole-system sensor while plugged in).

- **`bl` (backlight) depends on HDR mode.** With HDR off, the panel is driven by the
  sysfs LED backend and `actual_brightness` is the live value. With HDR on, the
  compositor (Mutter) dims via the GPU color pipeline ("ref-white"), pins the sysfs
  LED, and `bl` no longer reflects reality (reads ~100%). On such panels HDR brightness
  isn't exposed in any sysfs/DRM/D-Bus register at all.

- **`LIMITER` is the most useful column for "why is it slow?":** `PWR` means a power
  limit (RAPL PL1/PL2, or a charger/EC budget) is holding the clock down — often the
  real culprit on thin laptops, not temperature. `THERM` means it actually hit Tjmax.

Portable to most Intel laptops; the machine-specific notes above only affect the
interpretation of `psys` and `bl`.
