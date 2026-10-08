# ThinkPad E16 Gen 1 — Reliable ACPI S3 Suspend/Wake on CachyOS

> **Status:** Working configuration validated on 8 October 2026  
> **Hardware:** Lenovo ThinkPad E16 Gen 1  
> **OS:** CachyOS  
> **CPU:** AMD Ryzen 7 7730U  
> **GPU:** AMD Barcelo  
> **Wi-Fi:** Realtek RTL8852BE  
> **Ethernet:** Realtek RTL8111/8168/8211/8411

## Overview

This document describes how I restored and stabilized **ACPI S3 / Suspend-to-RAM** on a ThinkPad E16 Gen 1 running CachyOS.

The original firmware exposed only:

```text
/sys/power/mem_sleep
[s2idle]
```

so Linux could only use **Suspend-to-Idle (S2Idle)** for `mem` suspend.

S2Idle was unreliable on this system. After some lid-close suspend attempts, the machine could fail to wake and require a reboot.

The solution was:

1. Identify that the firmware DSDT contained existing S3-related AML but did not expose `_S3`.
2. Create a minimal DSDT override that exposes the existing S3 package.
3. Increase the DSDT OEM revision so Linux accepts the initramfs override.
4. Configure `deep` as the selected suspend mode.
5. Identify a problematic ACPI wake source associated with **GPP4**.
6. Disable GPP4's ACPI wake source.
7. Make both settings persistent with systemd services.

The resulting system uses actual **ACPI S3 / Suspend-to-RAM** and has successfully resumed from lid-close suspend repeatedly.

---

## 1. Original problem

Before the fix:

```bash
cat /sys/power/mem_sleep
```

returned:

```text
[s2idle]
```

There was no `deep` option.

The system could therefore enter S2Idle, but lid-close suspend was not reliable. Some suspend attempts resulted in:

- blank/non-responsive display,
- keyboard no longer responding,
- Caps Lock not responding,
- inability to wake normally,
- requiring a reboot.

The desired behavior was **Suspend-to-RAM**, not hibernation or hybrid sleep.

---

## 2. Inspecting the firmware DSDT

The system DSDT was extracted and decompiled using ACPICA tools.

Original DSDT:

```text
/tmp/dsdt.dat
```

Size:

```text
74847 bytes
```

Original OEM revision:

```text
0x00001310
```

The original DSDT contained:

```asl
Name (_S0, Package (0x04)
{
    0x00,
    0x00,
    0x00,
    0x00
})

Name (_S4, Package (0x04)
{
    0x04,
    0x04,
    0x00,
    0x00
})

Name (_S5, Package (0x04)
{
    0x05,
    0x05,
    0x00,
    0x00
})
```

There was no `_S3` object.

However, the DSDT contained substantial S3-specific AML. Among other things, `_PTS`, `_WAK`, `PPTS`, and `PWAK` contained logic checking for sleep state `3`.

The DSDT also contained S3-related objects/functions including:

- `S3ST`
- `S3RM`
- `SLTP()`
- `SLPS`
- S3-specific EC handling
- SMI calls associated with sleep transitions

Most importantly, there was an existing object:

```asl
Name (NOS3, Package (0x04)
{
    0x03,
    0x03,
    0x00,
    0x00
})
```

This was the basis for the minimal modification.

---

## 3. Minimal DSDT modification

The modification was intentionally kept minimal.

Original:

```asl
Name (NOS3, Package (0x04)
{
    0x03,
    0x03,
    0x00,
    0x00
})
```

Changed to:

```asl
Name (_S3, Package (0x04)
{
    0x03,
    0x03,
    0x00,
    0x00
})
```

No new S3 implementation was created.

The goal was to expose the existing firmware S3 definition to Linux.

---

## 4. Increasing the DSDT OEM revision

The first initramfs override was ignored because its OEM revision was identical to the firmware table:

```text
0x00001310
```

The DSDT definition was therefore changed to:

```asl
DefinitionBlock ("", "DSDT", 1, "LENOVO", "TP-R2C  ", 0x00001311)
```

The only relevant change here was:

```text
0x00001310 → 0x00001311
```

The patched DSDT compiled successfully:

```text
Compilation successful.
0 Errors
23 Warnings
157 Remarks
69 Optimizations
```

---

## 5. Installing the ACPI override

The compiled AML was placed at:

```text
/etc/initcpio/acpi_override/dsdt.aml
```

The `acpi_override` initramfs hook was enabled in `/etc/mkinitcpio.conf`:

```text
HOOKS=(base systemd acpi_override autodetect microcode kms modconf block keyboard sd-vconsole plymouth filesystems fsck)
```

The initramfs was rebuilt:

```bash
sudo mkinitcpio -P
```

The generated initramfs was verified to contain:

```text
kernel/firmware/acpi/dsdt.aml
```

After reboot, the kernel reported:

```text
ACPI: DSDT ... (v01 LENOVO TP-R2C   00001311 INTL 20251212)
```

The important point is that the active OEM revision became:

```text
00001311
```

which confirmed that the patched DSDT was actually loaded.

---

## 6. S3 became available

After the DSDT override:

```bash
cat /sys/power/mem_sleep
```

returned:

```text
[s2idle] deep
```

`deep` is the Linux suspend-to-RAM path corresponding to ACPI S3 on this system.

It was selected with:

```bash
echo deep | sudo tee /sys/power/mem_sleep
```

Result:

```text
s2idle [deep]
```

---

## 7. Making `deep` persistent

A systemd service was created:

```text
/etc/systemd/system/set-suspend-mode.service
```

Contents:

```ini
[Unit]
Description=Set Linux suspend mode to S3/deep
After=systemd-modules-load.service
Before=suspend.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo deep > /sys/power/mem_sleep'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now set-suspend-mode.service
```

Verify:

```bash
cat /sys/power/mem_sleep
```

Expected:

```text
s2idle [deep]
```

---

## 8. Confirming that `deep` really enters S3

Manual suspend was tested with:

```bash
echo mem | sudo tee /sys/power/state
```

The kernel reported:

```text
PM: suspend entry (deep)
ACPI: PM: Preparing to enter system sleep state S3
```

On resume:

```text
ACPI: PM: Low-level resume complete
ACPI: PM: Waking up from system sleep state S3
PM: suspend exit
```

This confirmed that the machine was entering **actual ACPI S3**, rather than S2Idle.

The AMD GPU also resumed successfully during these tests.

---

## 9. Investigating unreliable lid wake

The ACPI wake table included:

```text
Device  S-state  Status
GPP3    S3       *enabled
GPP4    S4       *enabled
GPP5    S3       *disabled
GP17    S3       *enabled
XHC0    S3       *enabled
XHC1    S3       *enabled
GP19    S3       *disabled
LID     S4       *enabled
SLPB    S3       *enabled
```

The Linux lid device was:

```text
PNP0C0D:00
```

and:

```bash
cat /sys/devices/platform/PNP0C0D:00/power/wakeup
```

returned:

```text
enabled
```

Therefore the lid itself remained enabled as a wake source.

---

## 10. Identifying GPP4

PCIe topology showed:

```text
00:02.2-[02]----00.0
```

`00:02.2` was:

```text
AMD Renoir/Cezanne PCIe GPP Bridge
```

and its downstream device was:

```text
02:00.0 Realtek RTL8111/8168/8211/8411 Ethernet
```

So:

```text
GPP4 (00:02.2)
    |
    +-- 02:00.0 Realtek Ethernet
```

The Ethernet device had:

```text
/sys/bus/pci/devices/0000:02:00.0/power/wakeup
disabled
```

The GPP4 bridge itself had:

```text
/sys/bus/pci/devices/0000:00:02.2/power/wakeup
disabled
```

Therefore this workaround does not disable Ethernet operation or Linux PCI device wake through those sysfs controls.

The relevant change is specifically to the **ACPI wake entry exposed through `/proc/acpi/wakeup`**.

---

## 11. GPP4 workaround

GPP4 was disabled using:

```bash
echo GPP4 | sudo tee /proc/acpi/wakeup
```

The state became:

```text
GPP4    S4    *disabled    pci:0000:00:02.2
```

The lid remained enabled.

No other wake source was changed.

---

## 12. Validation

With GPP4 disabled, the following test was performed three consecutive times:

```bash
echo mem | sudo tee /sys/power/state
```

Then:

1. Close the lid.
2. Wait approximately 30–40 seconds.
3. Open the lid.

All three tests resumed normally.

A longer test was then performed.

Suspend began at approximately:

```text
12:55:44 PM
```

The lid was opened at approximately:

```text
1:23 PM
```

The system therefore remained suspended for approximately **27 minutes**.

It resumed normally.

This provided a longer-duration validation of the working configuration.

---

## 13. Making GPP4 disabling persistent

Because `/proc/acpi/wakeup` state is not inherently persistent across boots, a systemd service was created:

```text
/etc/systemd/system/disable-gpp4-wakeup.service
```

Contents:

```ini
[Unit]
Description=Disable GPP4 ACPI wake source
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'if grep -q "^GPP4.*enabled" /proc/acpi/wakeup; then echo GPP4 > /proc/acpi/wakeup; fi'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable disable-gpp4-wakeup.service
```

Apply it immediately without rebooting:

```bash
sudo systemctl start disable-gpp4-wakeup.service
```

Verify:

```bash
grep GPP4 /proc/acpi/wakeup
```

Expected:

```text
GPP4    S4    *disabled    pci:0000:00:02.2
```

Verify the service:

```bash
systemctl status disable-gpp4-wakeup.service
```

Expected:

```text
Active: active (exited)
```

The service was tested and returned:

```text
status=0/SUCCESS
```

The conditional check is intentional: it only toggles GPP4 when it is currently enabled, preventing the service from accidentally re-enabling it.

---

## 14. Final working configuration

The important final state is:

```bash
cat /sys/power/mem_sleep
```

```text
s2idle [deep]
```

and:

```bash
grep GPP4 /proc/acpi/wakeup
```

```text
GPP4    S4    *disabled    pci:0000:00:02.2
```

The lid remains enabled as a wake source.

### Final suspend path

```text
Lid close
    ↓
systemd suspend
    ↓
Linux "deep"
    ↓
ACPI S3
    ↓
Suspend-to-RAM
    ↓
Lid open
    ↓
ACPI wake
    ↓
Normal resume
```

---

## 15. What this fix does

### DSDT override

Exposes the firmware's existing S3 definition:

```text
NOS3 → _S3
```

and makes the override newer:

```text
OEM revision 0x1310 → 0x1311
```

### Suspend selection

Selects:

```text
deep
```

instead of:

```text
s2idle
```

### GPP4 workaround

Disables:

```text
GPP4 S4 ACPI wake
```

while leaving the actual Ethernet controller operational.

### Persistence

Two systemd services maintain the configuration:

```text
set-suspend-mode.service
disable-gpp4-wakeup.service
```

---

## 16. What was NOT changed

This solution does not require:

- hibernation
- hybrid sleep
- disabling the Ethernet controller
- disabling the Ethernet driver
- disabling the lid wake source
- disabling all PCIe wake sources
- disabling AMDGPU
- changing CPU C-states
- replacing the kernel
- increasing swap
- using a reboot as the wake mechanism

The fix is targeted at the ACPI/firmware suspend configuration.

---

## 17. Limitations

This is a **hardware/firmware-specific workaround**.

The exact internal reason why GPP4 interfered with reliable S3 lid resume was not proven. The practical finding was more important for this case:

- S3 was unavailable before the DSDT modification.
- The firmware DSDT already contained S3-related AML.
- Exposing `_S3` made `deep` available.
- Actual S3 suspend/resume worked.
- Disabling GPP4's ACPI wake source produced repeatable reliable lid wake.
- The configuration was validated with three consecutive short tests and one approximately 27-minute suspend.

Therefore this should not be treated as a universal ThinkPad or AMD fix.

---

## 18. Recovery

If the DSDT override needs to be removed, remove the override AML and rebuild the initramfs:

```bash
sudo rm /etc/initcpio/acpi_override/dsdt.aml
sudo mkinitcpio -P
```

Disable the custom services if necessary:

```bash
sudo systemctl disable set-suspend-mode.service
sudo systemctl disable disable-gpp4-wakeup.service
```

If GPP4 needs to be re-enabled temporarily:

```bash
echo GPP4 | sudo tee /proc/acpi/wakeup
```

---

## 19. Useful references

- [ArchWiki — DSDT](https://wiki.archlinux.org/title/DSDT)
- [ArchWiki — Power management / Suspend and hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate)
- Linux kernel documentation: ACPI table override through initrd

---

## Conclusion

The ThinkPad E16 Gen 1 originally exposed only S2Idle even though its DSDT contained existing S3-related firmware logic.

A minimal DSDT override exposing `_S3`, combined with selecting `deep`, restored access to actual ACPI S3 / Suspend-to-RAM.

Reliable lid wake was then achieved by disabling the GPP4 ACPI wake source while leaving the lid wake source enabled.

The resulting configuration was validated with repeated short suspend/resume cycles and an approximately 27-minute S3 suspend, all of which resumed normally.

**Known-good configuration:**

```text
ACPI DSDT:
    NOS3 → _S3
    OEM revision: 0x1310 → 0x1311

Linux:
    /sys/power/mem_sleep = s2idle [deep]

ACPI wake:
    GPP4 = disabled
    LID  = enabled

Persistent services:
    set-suspend-mode.service
    disable-gpp4-wakeup.service
```
