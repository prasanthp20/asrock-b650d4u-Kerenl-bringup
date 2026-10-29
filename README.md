# ASRock Rack B650D4U (ASPEED AST2600) — Bare-Metal Silicon Bring-Up & Upstream Enablement

A lightweight, bare-metal platform validation lab and upstream Linux kernel Device Tree enablement project for the **ASRock Rack B650D4U** server management processor (ASPEED AST2600 Dual Cortex-A7).

---

## 1. What This Project Is

This repository contains the Device Tree sources, minimal static `/init` test harnesses, boot scripts, and kernel configurations used to achieve:
1. **Sub-second DDR RAM booting** of a pure upstream Linux kernel (6.18+) via U-Boot TFTP without distribution overhead (no Yocto, systemd, or OpenBMC abstractions).
2. **Direct register-level hardware validation** (MMIO, GIC interrupts, hardware watchdog timers, I2C/SMBus, and eSPI/LPC host telemetry).
3. **Upstream Linux Kernel Enablement** submitted directly to the Linux Kernel Mailing List (LKML) and maintainers.

---

## 2. Why Pure Kernel Bring-Up?

In semiconductor and platform firmware engineering, relying on full user-space distributions (like OpenBMC or Yocto) during early validation introduces high compile times (1–3 hours) and obscuring user-space layers (D-Bus, systemd units, daemons).

By utilizing a pure kernel paired with an in-memory static ARM32 `/init` test harness:
* **Sub-Second Boot Time:** Drops into userspace PID 1 in **~1.01 seconds** directly on physical silicon.
* **Deterministic Fault Isolation:** Eliminates software race conditions; any bus hang or memory abort corresponds directly to an MMIO register or Device Tree misconfiguration.
* **Rapid Turnaround:** Build iteration loop is under **20 seconds** (`make zImage` -> TFTP load -> execution).
* **Direct Silicon Telemetry:** Services physical SoC timers (`/dev/watchdog`), probes hardware FIFOs, and manages DMA cache boundaries directly.

---

## 3. Platform Architecture

* **Host Processor:** AMD AM5 Socket (Ryzen 7000 / EPYC 4004)
* **Baseboard Management Controller (BMC):** ASPEED AST2600 (Dual ARM Cortex-A7 @ 1.2 GHz, ARMv7-A)
* **Active Kernel:** Linux 6.18+ (Upstream `aspeed_g5_defconfig` + AST2600 G6 extensions)
* **Console Interface:** UART5 @ `0x1e784000` (`ttyS0`, 115200 8n1)
* **Ethernet MAC/PHY:** `ftgmac100` (`0x1e660000`) paired with Realtek RTL8211F Gigabit RGMII PHY
* **Storage:** 64MB SPI NOR Flash (`w25q512jvq`)

---

## 4. Hardware Bring-Up & Mainline Status

| Subsystem / Controller | Physical MMIO Base | Hardware Verification | Upstream Mainline Status |
| :--- | :--- | :--- | :--- |
| **UART5 Console** | `0x1e784000` | Verified (Earlycon + 8250 driver) | Standard upstream |
| **GIC Interrupt Controller** | `0x40461000` | Dual-core SMP IRQ dispatching ok | Standard upstream |
| **Hardware WDT** | `0x1e785000` | Managed & serviced via PID 1 | Subsystem verified |
| **Ethernet (eth0)** | `0x1e660000` | RTL8211F link up (1000 Mbps Full) | Included in DTS |
| **I2C Bus 7 (EEPROM)** | `0x1e78a400` | `24c128` detected & readable @ `0x57` | Included in DTS |
| **eSPI / LPC Port 0x80** | `0x1e789000` | Host POST code snooping harness | In Progress |
| **Host AM5 Power GPIOs** | `0x1e780000` | Signal line pinmux mapping | In Progress |

---

## 5. Boot Flow

```text
[ Power-On / 5VSB ]
       │
       ▼
[ AST2600 U-Boot SPL ]  --> Initializes DRAM (496 MiB usable)
       │
       ▼
[ U-Boot Prompt ]       --> Loads zImage & DTB into DDR RAM via TFTP
       │
       ▼
[ Linux 6.18 Kernel ]   --> Maps MMIO, probes RTL8211F, unpacks RAM-based CPIO rootfs
       │
       ▼
[ Static PID 1 (/init) ] --> Mounts devtmpfs, services watchdog, runs hardware tests
