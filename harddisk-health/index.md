# Running a SMART Long Test on a USB HDD — Troubleshooting Documentation

**Date:** 2026-04-07
**OS:** Ubuntu 24 (kernel 6.8.0-106-generic)
**Drive:** TOSHIBA MQ01UBB200, 2TB, 5400rpm, USB 3.0 external
**Drive stats at start:** 6309 power-on hours, SMART overall health: PASSED

---

## Background

The goal was to run a SMART extended self-test (`smartctl -t long`) on a USB-attached 2TB Toshiba
external HDD to verify drive health. What seemed like a simple task turned into a multi-attempt
troubleshooting session due to USB power management interfering with the test repeatedly.

---

## Step 1 — Read SMART data

```bash
sudo smartctl -a /dev/sda
```

**Sample output (abridged):**
```
SMART overall-health self-assessment test result: PASSED

ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  1 Raw_Read_Error_Rate     0x000b   100   100   050    Pre-fail  Always       -       0
  5 Reallocated_Sector_Ct   0x0033   100   100   050    Pre-fail  Always       -       0
  9 Power_On_Hours          0x0032   085   085   000    Old_age   Always       -       6309
197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0030   100   100   000    Old_age   Offline      -       0

Self-test execution status:      ( 249)	Self-test routine in progress...
					90% of test remaining.
```

**What to look for:**
- `Reallocated_Sector_Ct`, `Current_Pending_Sector`, `Offline_Uncorrectable` should all be 0 — anything
  above 0 is a warning sign.
- `Power_On_Hours` (ID 9) tells total lifetime hours. 6309h ≈ ~263 days of continuous use.
- The `Self-test execution status` value of 249 here was confusing — it says "in progress" but the test
  was already aborted from a previous session. This is a firmware quirk (see Step 2).

**Monitoring progress live (run as root):**
```bash
sudo su
watch -n 1 'smartctl -a /dev/sda | grep -E "test remaining|Self-test execution"'
```

This refreshes every 1 second so you can watch the percentage tick down without running the command
manually each time. Exit with Ctrl+C.

---

## Step 2 — First attempt: start the long test

```bash
sudo smartctl -t long /dev/sda
```

**Failed — output:**
```
Can't start self-test without aborting current test (90% remaining),
add '-t force' option to override, or run 'smartctl -X' to abort test.
```

**Why:** A previous extended test had been aborted (likely from an earlier session where the drive was
unplugged). The drive's firmware still reported status byte 249 ("in progress") even though the test
log showed it as aborted. This is an inconsistent firmware state — the drive thinks a test is still
running.

**Check the test log:**
```bash
sudo smartctl -l selftest /dev/sda
```

```
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Aborted by host               90%      6303         -
```

**What the columns mean:**
- `Status` — result of the test. "Aborted by host" = interrupted by software/disconnect. "Interrupted
  (host reset)" = USB connection reset at hardware level.
- `Remaining` — how much of the test was left when it stopped. 90% remaining = only 10% completed.
- `LifeTime(hours)` — the drive's total power-on hours at the time the test ran. Used to correlate
  which test happened when.

---

## Step 3 — Abort the stuck test, then restart

Tried aborting three times in a row (the test kept re-registering as aborted but the firmware state
persisted):

```bash
sudo smartctl -X /dev/sda
```

```
Sending command: "Abort SMART off-line mode self-test routine".
Self-testing aborted!
```

After three abort attempts, the log showed two new entries:

```
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Aborted by host               90%      6309         -
# 2  Extended offline    Aborted by host               90%      6303         -
```

Finally, starting the test succeeded:

```bash
sudo smartctl -t long /dev/sda
```

```
Drive command "Execute SMART Extended self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 496 minutes for test to complete.
Test will complete after Tue Apr  7 11:54:31 2026 WIB
Use smartctl -X to abort test.
```

**Note:** Right after starting, the test log does not immediately show a new entry. A new log entry
only appears after the test completes or is aborted — not when it starts. So seeing only old entries
right after `smartctl -t long` is normal.

---

## Step 4 — Test aborted again (USB autosuspend)

Checked status a few minutes later and the drive had gone quiet:

```bash
sudo smartctl -a /dev/sda | grep -E "test remaining|Self-test execution"
```

```
Self-test execution status:      (  25)	The self-test routine was aborted by the host
```

The test log now showed a new failure with a different status:

```
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Interrupted (host reset)      90%      6310         -
# 2  Extended offline    Aborted by host               90%      6309         -
# 3  Extended offline    Aborted by host               90%      6309         -
# 4  Extended offline    Aborted by host               90%      6309         -
# 5  Extended offline    Aborted by host               90%      6303         -
```

**"Interrupted (host reset)"** is more serious than "Aborted by host" — it means the USB connection
itself reset at the hardware level, not just a software abort. This points to USB power management
cutting power to the drive mid-test.

---

## Step 5 — Diagnose with dmesg

```bash
sudo dmesg | grep -i "reset\|disconnect\|error" | tail -30
```

**Key output:**
```
[  219.503506] usb 2-1: USB disconnect, device number 3
[  836.477513] usb 2-1: new SuperSpeed USB device number 4 using xhci_hcd
[  836.490275] usb 2-1: New USB device found, idVendor=0480, idProduct=a200, bcdDevice= 0.00
[  836.490290] usb 2-1: Product: External USB 3.0
[  836.490293] usb 2-1: Manufacturer: TOSHIBA
[22391.734233] usb 2-1: reset SuperSpeed USB device number 4 using xhci_hcd
[25453.948046] usb 2-1: reset SuperSpeed USB device number 4 using xhci_hcd
```

**What this tells us:**
- The Toshiba drive is on USB path `2-1` (USB 3.0 SuperSpeed port).
- There were two USB resets at ~22391s and ~25453s — these are what killed the test.
- There was also a separate Seagate FreeAgent GoFlex drive disconnecting and reconnecting several
  times earlier — unrelated to the Toshiba, but confirms USB instability on this machine.

---

## Step 6 — Fix 1: prevent system sleep

The initial attempt was to prevent the OS from suspending:

```bash
systemd-inhibit --what=sleep --who="smartctl" --why="HDD self-test running" sleep infinity
```

**How it works:** `systemd-inhibit` holds a lock that blocks the specified system action (`sleep` =
suspend/hibernate). The `sleep infinity` at the end is a dummy process — `systemd-inhibit` holds the
lock only as long as the wrapped command is running. Press Ctrl+C to release.

**Why this alone wasn't enough:** The drive was still going quiet even while the system was awake.
The culprit was USB autosuspend — Linux's aggressive USB power management that suspends USB devices
independently of system sleep.

---

## Step 7 — Fix 2: check and disable HDD APM (spindown)

```bash
# Check current APM level
sudo hdparm -B /dev/sda
```

```
/dev/sda:
 APM_level	= 128
```

```bash
# Disable spindown
sudo hdparm -B 255 /dev/sda
```

```
/dev/sda:
 setting Advanced Power Management level to disabled
 APM_level	= off
```

**APM levels explained:**
- 1–127: allow spindown (lower = more aggressive power saving)
- 128–254: no spindown, but still some power management
- 255: disable APM entirely

**Verdict:** APM at 128 actually doesn't allow spindown, so this wasn't the root cause. But disabling
it anyway removes one variable.

**Undo:**
```bash
sudo hdparm -B 128 /dev/sda
```

> Temporary — resets on reboot.

---

## Step 8 — Fix 3: disable USB autosuspend (all devices)

```bash
# Check current autosuspend delay for all USB devices
cat /sys/bus/usb/devices/*/power/autosuspend_delay_ms
```

```
2000
2000
2000
2000
2000
0
0
```

Default is 2000ms (2 seconds) for most devices. A value of `-1` disables autosuspend.

```bash
# Disable for all USB devices
echo -1 | sudo tee /sys/bus/usb/devices/*/power/autosuspend_delay_ms
```

**Undo:**
```bash
echo 2000 | sudo tee /sys/bus/usb/devices/*/power/autosuspend_delay_ms
```

> Temporary — resets on reboot.

---

## Step 9 — Fix 4: disable USB power control for the specific device (root cause fix)

The dmesg output showed the Toshiba drive at USB path `2-1`. The `power/control` file for a USB
device controls whether Linux applies automatic power management to it.

**Find the device path by idVendor:**
```bash
grep -l "0480" /sys/bus/usb/devices/*/idVendor 2>/dev/null
```

```
/sys/bus/usb/devices/2-1/idVendor
```

Confirms the Toshiba is at `2-1`.

```bash
# Disable autosuspend for this specific device
echo on | sudo tee /sys/bus/usb/devices/2-1/power/control
```

The difference between this and Step 8:
- Step 8 sets `autosuspend_delay_ms = -1` — tells the kernel "don't autosuspend", but `power/control`
  may still be set to `auto`, meaning the kernel can still decide to suspend the device.
- Step 9 sets `power/control = on` — tells the kernel "keep this device permanently on, no automatic
  power management at all." This is the more definitive fix.

**Undo:**
```bash
echo auto | sudo tee /sys/bus/usb/devices/2-1/power/control
```

> Temporary — resets on reboot.

---

## Step 10 — Final attempt (all fixes applied)

With all three fixes in place:
1. `hdparm -B 255` — APM disabled
2. `echo -1` to `autosuspend_delay_ms` — USB autosuspend delay disabled
3. `echo on` to `power/control` for `2-1` — USB power management disabled for the Toshiba

```bash
sudo smartctl -t long /dev/sda
```

Then monitor with:
```bash
sudo su
watch -n 1 'smartctl -a /dev/sda | grep -E "test remaining|Self-test execution"'
```

Expected output while running:
```
Self-test execution status:      ( 249)	Self-test routine in progress...
					60% of test remaining.
```

Expected output on completion:
```
Self-test execution status:      (   0)	The previous self-test routine completed without error
```

Expected log on completion:
```
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed without error       00%      6310         -
```

---

## Quick reference — all commands

```bash
# --- SMART test ---
sudo smartctl -t long /dev/sda           # start long test
sudo smartctl -t long -t force /dev/sda  # force start (if firmware thinks test is running)
sudo smartctl -X /dev/sda               # abort test
sudo smartctl -l selftest /dev/sda      # view test log
sudo smartctl -a /dev/sda | grep -E "test remaining|Self-test execution"  # check progress

# Monitor live (run as root)
sudo su
watch -n 1 'smartctl -a /dev/sda | grep -E "test remaining|Self-test execution"'

# --- Prevent system sleep ---
systemd-inhibit --what=sleep --who="smartctl" --why="HDD self-test running" sleep infinity
# Ctrl+C to release

# --- APM / spindown ---
sudo hdparm -B /dev/sda          # check
sudo hdparm -B 255 /dev/sda      # disable (run during test)
sudo hdparm -B 128 /dev/sda      # undo (restore default)

# --- USB autosuspend (all devices) ---
cat /sys/bus/usb/devices/*/power/autosuspend_delay_ms         # check
echo -1   | sudo tee /sys/bus/usb/devices/*/power/autosuspend_delay_ms  # disable
echo 2000 | sudo tee /sys/bus/usb/devices/*/power/autosuspend_delay_ms  # undo

# --- USB power control (specific device) ---
grep -l "0480" /sys/bus/usb/devices/*/idVendor 2>/dev/null   # find device path by idVendor
echo on   | sudo tee /sys/bus/usb/devices/2-1/power/control  # disable autosuspend (run during test)
echo auto | sudo tee /sys/bus/usb/devices/2-1/power/control  # undo

# --- Debug USB resets ---
sudo dmesg | grep -i "reset\|disconnect\|error" | tail -30
```

> All changes to `/sys/bus/usb/` and `hdparm` are **temporary** and reset automatically on reboot.

---

## Lessons learned

- A SMART test runs entirely on the drive's firmware — no Linux process to keep alive. But the drive
  must stay powered and connected throughout.
- "Aborted by host" and "Interrupted (host reset)" are different: the former is a software abort
  (e.g. `smartctl -X`, or OS shutdown), the latter is a USB-level reset which is harder to prevent
  and points to power management or cable/hub issues.
- USB autosuspend is aggressive on Linux. Even with `autosuspend_delay_ms = -1`, setting
  `power/control = on` is the more reliable way to keep a USB device fully awake.
- `dmesg` is the fastest way to confirm whether USB resets are happening and which device path is
  affected.
- The `LifeTime` column in `smartctl -l selftest` is power-on hours, useful for correlating which
  test run corresponds to which attempt.
