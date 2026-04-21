## Solution shape

### 1 Chosen approach: Two-phase hybrid

**Phase 1 — microSD recovery (remote-safe, deploy first):**
Provision a dedicated microSD with Pi OS Lite + Tailscale + recovery-only SSH key. Physically insert on next on-site visit. Phase 1 completion means any sda failure is recoverable remotely.

**Phase 2 — initramfs recovery on sda3 (RAM-only, identical kernel to main):**
Boot into phase-1 microSD to safely shrink `sda2` offline and create `sda3` (~500 MB vfat). Build a custom Debian initramfs from main OS, install to `sda3`, wire up `autoboot.txt` and watchdog-partition failover. Phase 2 completion makes the fast-path (frequent) recovery a RAM-resident env with zero version skew to main.

The microSD is **retained after phase 2** as an independent-failure-domain fallback for "sda is physically unreachable".

### 2 Alternatives considered and rejected

| Option | Why rejected |
|---|---|
| A: Same-SSD carve with full Debian on sda4 | Loses RAM-only advantage; 5 GB footprint vs 500 MB; no independent-failure-domain fallback; same shrink cost as F |
| B: microSD only | No RAM-only fast-path; every failover incurs microSD boot time; single-media recovery |
| C: `boot_ramdisk=1` with piCore `tryboot.img` | piCore lacks prebuilt ZFS; building OpenZFS against piCore kernel is fragile (kernel-version lock-in, no DKMS pattern, userspace/module version mismatch risk); different kernel from main = two maintenance surfaces |
| D: B → A hybrid | Supplanted by F (E replaces A's ext4-root with a superior initramfs) |
| E: Pure initramfs, no microSD | No disaster fallback for physical sda failure |

### 3 Final on-disk layout

```
sda (232.9 GB SATA SSD via USB3, MBR, unchanged scheme)
├── sda1   512 MB    vfat    main /boot/firmware            [existing]
│                            adds: autoboot.txt
├── sda2   ~231.4 GB ext4    main /                         [existing, shrunk ~1 GB in phase 2]
└── sda3   ~500 MB   vfat    recovery /boot/firmware-ram    [NEW, phase 2]
                             contents:
                               config.txt (recovery-specific)
                               cmdline.txt (recovery)
                               kernel_2712.img (copy from sda1)
                               bcm2712-rpi-5-b.dtb (+ overlays)
                               initramfs_recovery (~300 MB)
                               initramfs_recovery.sha256
                               initramfs_recovery.meta

microSD (phase 1, retained indefinitely)
├── mmcblk0p1  512 MB  vfat  /boot/firmware  (Pi OS Lite standard)
└── mmcblk0p2  ~7 GB   ext4  /               (Pi OS Lite + Tailscale + recovery key)
                             hostname: pi5-recovery-sd
```

MBR primaries used: 3 of 4. Fourth slot reserved.

### 4 EEPROM configuration changes

Applied via `sudo -E rpi-eeprom-config --edit`:

```
[all]
# --- existing settings preserved ---
BOOT_UART=1
BOOT_ORDER=0xf146                    # unchanged: NVMe→USB-MSD→SD→RESTART
NET_INSTALL_AT_POWER_ON=0
USB_MSD_DISCOVER_TIMEOUT=20000
USB_MSD_LUN_TIMEOUT=6000
USB_MSD_STARTUP_DELAY=5000

# --- NEW for recovery plan ---
BOOT_WATCHDOG_TIMEOUT=120            # 2 min bootloader WDT; if kernel not started, reset
BOOT_WATCHDOG_PARTITION=3            # on WDT reset, boot sda3 (initramfs recovery)
PARTITION_WALK=1                     # default-on since 2025-08-13, assert anyway
```

`BOOT_ORDER` is deliberately unchanged — SD (`0x1`) already trails USB-MSD (`0x4`), so a fully-dead sda falls through to microSD naturally.

### 5 config.txt changes (main, in `sda1/config.txt`)

Appended under the existing `[all]` section:

```ini
# --- NEW for recovery plan ---
kernel_watchdog_timeout=180          # hands over from BOOT_WATCHDOG at kernel start;
                                     # sets watchdog.open_timeout=180 on kernel cmdline
kernel_watchdog_partition=3          # runtime wedge → reboot into sda3 recovery
```

**Interaction with existing systemd watchdog:** `kernel_watchdog_timeout` arms `/dev/watchdog0` early (before systemd). When systemd starts it opens the device and takes over petting; the `open_timeout` simply closes the pre-systemd gap. No conflict. OMV's `RuntimeWatchdogSec=14s` continues to govern runtime behaviour unchanged.

### 6 New file: `sda1/autoboot.txt`

```ini
[all]
tryboot_a_b=1
boot_partition=1

[tryboot]
boot_partition=3
```

### 7 Trigger chain (resulting boot paths)

| # | Trigger | Boot mode | Partition resolution | Outcome |
|---|---------|-----------|----------------------|---------|
| 1 | Power-on / normal `reboot` | USB-MSD | `autoboot.txt [all]` → sda1 | Main: kernel + sda2 root |
| 2 | `sudo reboot '0 tryboot'` | USB-MSD | `autoboot.txt [tryboot]` → sda3 | Recovery-RAM: kernel + initramfs |
| 3 | `BOOT_WATCHDOG_TIMEOUT` fires (kernel never starts ≤120 s) | USB-MSD | `BOOT_WATCHDOG_PARTITION` → sda3 | Recovery-RAM (same as path 2) |
| 4 | `kernel_watchdog_timeout` fires (wedged after kernel start, ≤180 s) | USB-MSD | `kernel_watchdog_partition` → sda3 | Recovery-RAM (same as path 2) |
| 5 | sda unreachable (SSD dead / USB adapter failed) | USB-MSD exhausts retries → SD (`0x1`) | microSD partitions | Recovery-SD: Pi OS Lite |

---

