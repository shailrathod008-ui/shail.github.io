---
layout: post
title: "Ath driver Architecture and Design Documentation"
date: 2026-03-07 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "Ath driver Architecture and Design Documentation in  Depth"

---

# ATH Wireless Driver Family — High-Level Design & Architecture
## Linux Kernel 6.12 | `drivers/net/wireless/ath`

---

## Table of Contents
1. [Overview & Driver Family Map](#1-overview--driver-family-map)
2. [Layered Architecture](#2-layered-architecture)
3. [Shared Common Library (`ath/`)](#3-shared-common-library-ath)
4. [Sub-Driver Profiles](#4-sub-driver-profiles)
5. [Key Data Structures](#5-key-data-structures)
6. [Interfaces](#6-interfaces)
7. [State Machines](#7-state-machines)
8. [Control Flow](#8-control-flow)
9. [Data Flow (TX / RX)](#9-data-flow-tx--rx)
10. [DFS — Dynamic Frequency Selection Subsystem](#10-dfs--dynamic-frequency-selection-subsystem)
11. [Cryptographic Key Management](#11-cryptographic-key-management)
12. [Regulatory Domain Management](#12-regulatory-domain-management)
13. [Power Management](#13-power-management)
14. [Algorithms & Calibration](#14-algorithms--calibration)
15. [Cross-Cutting Concerns](#15-cross-cutting-concerns)

---

## 1. Overview & Driver Family Map

The `ath` directory is the **Atheros/Qualcomm Atheros wireless driver family** inside the Linux kernel. It implements IEEE 802.11 (Wi-Fi) support for a broad range of Atheros chipsets spanning nearly two decades of hardware generations.

```
drivers/net/wireless/ath/
│
├── ath.h               ← Shared common types (ath_common, ath_ops, ath_ani …)
├── main.c              ← Shared utility functions (rxbuf_alloc, printk …)
├── key.c               ← Shared hardware key-cache management
├── regd.c / regd.h     ← Shared regulatory domain logic
├── hw.c                ← Shared HW utilities (cycle counters, listen time)
├── debug.c             ← Shared debug helpers
├── trace.c / trace.h   ← Shared ftrace tracepoints
├── dfs_pattern_detector.c/.h ← Shared DFS radar pattern detection
├── dfs_pri_detector.c/.h     ← Shared DFS PRI (Pulse Repetition Interval) detector
│
├── ath5k/     ← AR5xxx series (legacy PCI/CardBus, 802.11a/b/g)
├── ath9k/     ← AR9xxx series (PCIe/AHB/USB, 802.11n with HT)
├── ath6kl/    ← AR6xxx USB/SDIO thin-client (offload firmware model)
├── ath10k/    ← QCA9880/QCA988x/QCA99xx (PCIe/USB/SDIO/SNOC, 802.11ac VHT)
├── ath11k/    ← IPQ807x/QCN9074 (AHB/PCIe, 802.11ax HE, Wi-Fi 6)
├── ath12k/    ← QCN9274/WCN7850 (PCIe/AHB, 802.11be EHT, Wi-Fi 7)
├── ar5523/    ← USB dongle driver for AR5523/AR5524
├── carl9170/  ← USB 802.11n driver for AR9170/AR9271
├── wcn36xx/   ← Qualcomm WCN3600/3620/3660 SDIO (integrated Snapdragon)
└── wil6210/   ← 802.11ad (60 GHz WiGig) — Wilocity/Qualcomm
```

### Hardware Generation Summary

| Driver  | Chipsets       | Standards       | Bus Interface   | Architecture Model |
|---------|----------------|-----------------|-----------------|-------------------|
| ath5k   | AR5001–AR5416  | 802.11a/b/g     | PCI/CardBus     | Full-MAC on host  |
| ath9k   | AR9160–AR9580  | 802.11n (HT40)  | PCIe/AHB/USB    | Full-MAC on host  |
| ath6kl  | AR6003/6004    | 802.11n         | USB/SDIO        | Offload to FW     |
| ath10k  | QCA988x–QCA99x | 802.11ac (VHT)  | PCIe/USB/SNOC   | Offload to FW     |
| ath11k  | IPQ807x/QCN9074| 802.11ax (HE)   | AHB/PCIe        | Offload to FW     |
| ath12k  | QCN9274/WCN7850| 802.11be (EHT)  | PCIe/AHB        | Offload to FW     |

---

## 2. Layered Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                USER SPACE  (iw, wpa_supplicant, hostapd …)           │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ nl80211 / cfg80211
┌────────────────────────────────▼─────────────────────────────────────┐
│                        mac80211 (net/mac80211/)                        │
│   Virtual interface management, MLME state, rate control, aggregation │
└────────┬───────────────────────────────────────────┬─────────────────┘
         │ ieee80211_ops callbacks                    │ ieee80211_hw
         │                                            │ ieee80211_vif
         │                                            │ ieee80211_sta
┌────────▼────────────────────────────────────────────▼───────────────┐
│                   ATH Driver Layer (per sub-driver)                   │
│                                                                       │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐  ┌─────────┐ │
│  │  ath9k  │  │  ath10k  │  │  ath11k │  │  ath12k  │  │  ath5k  │ │
│  │ (mac.c) │  │ (mac.c)  │  │ (mac.c) │  │  (mac.c) │  │ (main.c)│ │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └────┬─────┘  └────┬────┘ │
│       │            │             │             │              │       │
│  ┌────▼────────────▼─────────────▼─────────────▼──────────────▼───┐  │
│  │              Shared Common Library (ath/)                        │  │
│  │  key.c  regd.c  dfs_*.c  hw.c  debug.c  trace.c  main.c        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │   Transport / HIF Layer (Hardware Interface)                     │  │
│  │   PCI / AHB / USB / SDIO / SNOC                                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────▼──────────────────┐
              │        Hardware / Firmware         │
              │  RF PHY · Baseband · MAC Engine   │
              └────────────────────────────────────┘
```

### Architecture Models

**Host-MAC (ath5k, ath9k)**
- All MAC operations (MLME, QoS, BA, key management) run in the Linux kernel.
- Driver programs the hardware directly via MMIO register access (`ath_ops`).
- Tight coupling between driver and hardware register view.

**Firmware-Offload (ath6kl, ath10k, ath11k, ath12k)**
- A firmware image runs on the embedded ARM/RISC-V core inside the chip.
- Host driver communicates via a structured control protocol (WMI) over a transport (HTC/HIF/CE).
- Data plane uses descriptor-based DMA rings (HTT / DP).
- Driver is thinner; much logic lives in firmware.

---

## 3. Shared Common Library (`ath/`)

### 3.1 `struct ath_common` — The Universal Root Object

Defined in `ath.h`. Every sub-driver embeds or points to this structure as its common state.

```c
struct ath_common {
    void *ah;                          // opaque HW abstraction handle
    void *priv;                        // driver-private pointer
    struct ieee80211_hw *hw;           // mac80211 HW handle
    int debug_mask;                    // active debug categories
    enum ath_device_state state;       // ATH_HW_UNAVAILABLE | ATH_HW_INITIALIZED
    unsigned long op_flags;            // ATH_OP_* bitmask flags

    struct ath_ani ani;                // Adaptive Noise Immunity timers

    u16 cachelsz;                      // CPU cache-line size (for RX buf alignment)
    u16 curaid;                        // current AID (Station mode)
    u8  macaddr[ETH_ALEN];
    u8  curbssid[ETH_ALEN];
    u8  bssidmask[ETH_ALEN];

    u32 rx_bufsize;
    u32 keymax;                        // key cache size
    DECLARE_BITMAP(keymap, 128);       // allocated key indices
    DECLARE_BITMAP(tkip_keymap, 128);
    DECLARE_BITMAP(ccmp_keymap, 128);
    enum ath_crypt_caps crypt_caps;

    spinlock_t cc_lock;
    struct ath_cycle_counters cc_ani;
    struct ath_cycle_counters cc_survey;

    struct ath_regulatory regulatory;
    struct ath_regulatory reg_world_copy;
    const struct ath_ops    *ops;       // register I/O ops (read/write/rmw)
    const struct ath_bus_ops *bus_ops;  // bus-specific ops
    const struct ath_ps_ops  *ps_ops;   // power-save ops (wakeup/restore)

    bool btcoex_enabled;
    bool disable_ani;
    bool bt_ant_diversity;
    int  last_rssi;

    struct ieee80211_supported_band sbands[NUM_NL80211_BANDS];
};
```

### 3.2 `struct ath_ops` — Register I/O Abstraction

```c
struct ath_ops {
    unsigned int (*read)(void *, u32 reg_offset);
    void (*multi_read)(void *, u32 *addr, u32 *val, u16 count);
    void (*write)(void *, u32 val, u32 reg_offset);
    void (*enable_write_buffer)(void *);
    void (*write_flush)(void *);
    u32  (*rmw)(void *, u32 reg_offset, u32 set, u32 clr);
    void (*enable_rmw_buffer)(void *);
    void (*rmw_flush)(void *);
};
```

This abstraction allows write-coalescing (buffered register writes) and atomic read-modify-write — critical for correctness over PCIe.

### 3.3 `struct ath_ani` — Adaptive Noise Immunity

```c
struct ath_ani {
    bool caldone;
    unsigned int longcal_timer;     // period for long calibration
    unsigned int shortcal_timer;    // period for short calibration
    unsigned int resetcal_timer;    // period for calibration reset
    unsigned int checkani_timer;    // period for ANI check
    struct timer_list timer;        // periodic timer
};
```

ANI is a HW-level algorithm that dynamically tunes noise immunity thresholds to balance sensitivity vs. false-positive protection in different RF environments.

### 3.4 `struct ath_regulatory` — Regulatory Context

```c
struct ath_regulatory {
    char alpha2[2];                         // country code (ISO 3166)
    enum nl80211_dfs_regions region;         // DFS domain (FCC/ETSI/MKK)
    u16 country_code;
    u16 max_power_level;
    u16 current_rd;                          // current regulatory domain
    int16_t power_limit;                     // tx power cap (dBm)
    struct reg_dmn_pair_mapping *regpair;    // domain pair (2G/5G CTLs)
};
```

---

## 4. Sub-Driver Profiles

### 4.1 `ath9k` — AR9xxx (802.11n)

**Source tree:** `ath9k/`

**Key files:**

| File | Purpose |
|------|---------|
| `init.c` | PCI/AHB device probe, `ieee80211_alloc_hw`, IRQ setup |
| `main.c` | mac80211 ops (start/stop/tx/rx/config/…), ISR, tasklet |
| `hw.c` / `hw.h` | `struct ath_hw` — master hardware object, reset, init |
| `mac.c` / `mac.h` | TX/RX descriptor management, queue management |
| `xmit.c` | TX MPDU aggregation (A-MPDU), rate control |
| `recv.c` | RX ring management, frame decapsulation |
| `beacon.c` | Beacon construction and scheduling |
| `calib.c` | NF, IQ, ADC-gain calibration |
| `ani.c` | ANI algorithm implementation |
| `btcoex.c` | Bluetooth coexistence (2-wire/3-wire/MCI) |
| `dfs.c` | Per-chip DFS pulse detection feeding `dfs_pattern_detector` |
| `eeprom*.c` | EEPROM read and calibration data parsing |
| `pci.c` / `ahb.c` | Bus-specific probe/remove |
| `hif_usb.c` | USB transport for AR9271 |
| `htc_*.c` | HTC protocol for USB variant |
| `mci.c` | Message Coexistence Interface (AR9462+) |
| `wow.c` | Wake-on-Wireless |
| `dynack.c` | Dynamic ACK timeout estimation |

**Central structure: `struct ath_softc`** (`ath9k.h`)
- Contains `struct ath_hw *sc_ah` (hardware abstraction)
- TX queues: array of `struct ath_txq`
- RX: `struct ath_rx` with HP and LP queues (EDMA)
- Channel context list for multi-channel operation
- Tasklets: `intr_tq` (main interrupt), `bcon_tasklet` (beacon)
- Power-save spinlock, PS flags, sleep timer
- Work queues: hw_reset_work, paprd_work, hw_check_work

**Central structure: `struct ath_hw`** (`ath9k/hw.h`)
- Embeds `struct ath_common`
- HAL-style callbacks: `ath_hw_private_ops` (PHY ops), `ath_hw_ops` (calibration, spectral scan)
- `eeprom` union: supports 4 EEPROM formats (def, 4k, 9287, AR9300)
- `caldata`: per-channel calibration state
- `caps`: hardware capability bitmask
- `ath9k_channel curchan`: current channel

### 4.2 `ath10k` — QCA98xx/QCA99xx (802.11ac)

**Source tree:** `ath10k/`

**Key files:**

| File | Purpose |
|------|---------|
| `core.c/h` | `struct ath10k` master, device lifecycle, thread model |
| `mac.c/h` | mac80211 ops, vdev/peer management |
| `wmi.c/h` / `wmi-ops.h` | WMI control protocol (host→FW commands, FW→host events) |
| `wmi-tlv.c/h` | TLV-encoded WMI for newer firmware variants |
| `htt.c/h` / `htt_tx.c` / `htt_rx.c` | Host-Target Transport (data path) |
| `htc.c/h` | HTC — transport layer over Copy Engine |
| `hif.h` | HIF vtable — bus-agnostic transport interface |
| `pci.c/h` | PCIe HIF implementation |
| `usb.c/h` | USB HIF |
| `sdio.c/h` | SDIO HIF |
| `snoc.c/h` | SNOC (Snapdragon NoC) HIF for IPQ/WCN |
| `bmi.c/h` | Boot Management Interface — FW download |
| `swap.c/h` | Code swap (firmware patching) |
| `qmi.c/h` | QMI-based FW load (SNOC variant) |
| `txrx.c/h` | TX completion and RX frame delivery |
| `spectral.c/h` | FFT spectral scan |
| `thermal.c/h` | Thermal management |
| `p2p.c/h` | P2P NoA scheduling |

**Central structure: `struct ath10k`** (`ath10k/core.h`)
```
struct ath10k {
    struct ath_common ath_common; // embedded shared context
    struct ieee80211_hw *hw;
    struct device *dev;

    // Hardware info
    enum ath10k_hw_rev hw_rev;
    u32 chip_id, target_version;
    struct ath10k_hw_params hw_params;

    // Protocol stacks
    struct ath10k_bmi bmi;    // Boot Management Interface
    struct ath10k_wmi wmi;    // WMI state (completions, ops)
    struct ath10k_htc htc;    // HTC transport state
    struct ath10k_htt htt;    // HTT data path state

    // HIF (bus abstraction)
    struct { enum ath10k_bus bus; const struct ath10k_hif_ops *ops; } hif;

    // State
    enum ath10k_state state;  // OFF/ON/RESTARTING/RESTARTED/WEDGED
    unsigned long dev_flags;  // ATH10K_FLAG_* bitmask
    enum ath10k_scan_state scan.state;

    // VIF/STA/Peer management
    struct list_head arvifs;  // list of ath10k_vif
    struct list_head peers;   // list of ath10k_peer
    spinlock_t data_lock;
    struct mutex conf_mutex;

    // DFS
    struct dfs_pattern_detector *dfs_detector;
    ...
};
```

### 4.3 `ath11k` — IPQ807x/QCN9074 (802.11ax / Wi-Fi 6)

**Source tree:** `ath11k/`

**Key additions over ath10k:**
- HAL (Hardware Abstraction Layer): `hal.c/h`, `hal_rx.c`, `hal_tx.c`, `hal_desc.h`
- Data-plane (DP): `dp.c/h`, `dp_rx.c`, `dp_tx.c` — descriptor ring management
- Direct ring DMA with hardware parser (not firmware queues)
- Copy Engine: `ce.c/h` — same CE concept as ath10k but newer rings
- DB Ring (Direct Buffer Ring): `dbring.c/h` — for monitor/spectral

Architecture: **HAL-DP split**
- HAL owns ring descriptors and HW-specific register access
- DP owns buffer lifecycle, refill, completion
- mac80211 → DP → HAL → hardware

### 4.4 `ath12k` — QCN9274/WCN7850 (802.11be / Wi-Fi 7)

**Source tree:** `ath12k/`

Structurally similar to ath11k with these additions:
- Multi-link operation (MLO) support
- `dp_mon.c/h`: Monitor ring dedicated to RX statistics and sniffing
- ACPI integration via `acpi.c/h` for board data
- Enhanced CE (Copy Engine) descriptor formats

---

## 5. Key Data Structures

### 5.1 Shared Structures (Common Library)

| Structure | File | Purpose |
|-----------|------|---------|
| `struct ath_common` | `ath.h` | Root common context |
| `struct ath_ops` | `ath.h` | Register I/O vtable |
| `struct ath_ps_ops` | `ath.h` | Power-save vtable |
| `struct ath_bus_ops` | `ath.h` | Bus-type vtable |
| `struct ath_ani` | `ath.h` | ANI calibration timer state |
| `struct ath_cycle_counters` | `ath.h` | PHY channel busy counters |
| `struct ath_regulatory` | `ath.h` | Regulatory domain state |
| `struct ath_keyval` | `ath.h` | Key value (TK + MIC keys) |
| `struct reg_dmn_pair_mapping` | `ath.h` | Domain → CTL mapping |
| `struct dfs_pattern_detector` | `dfs_pattern_detector.h` | DFS pattern engine |
| `struct pulse_event` | `dfs_pattern_detector.h` | PHY radar pulse descriptor |
| `struct radar_detector_specs` | `dfs_pattern_detector.h` | Pattern type specification |
| `struct pri_detector` | `dfs_pri_detector.h` | Per-PRI sequence matcher |
| `struct pri_sequence` | `dfs_pri_detector.h` | Candidate radar pulse sequence |

### 5.2 ath9k Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `struct ath_hw` | `ath9k/hw.h` | Master HW object (embeds ath_common) |
| `struct ath9k_channel` | `ath9k/hw.h` | Channel + flags + noise floor |
| `struct ath9k_hw_capabilities` | `ath9k/hw.h` | Capability bitmask |
| `struct ath_hw_private_ops` | `ath9k/hw.h` | PHY/calibration internals vtable |
| `struct ath_hw_ops` | `ath9k/hw.h` | Calibration + spectral vtable |
| `struct ath9k_hw_cal_data` | `ath9k/hw.h` | Per-channel calibration data |
| `struct ath9k_beacon_state` | `ath9k/hw.h` | Beacon timing scheduling |
| `struct ath_gen_timer` | `ath9k/hw.h` | Generic TSF-based timers |
| `struct ath_hw_radar_conf` | `ath9k/hw.h` | Radar detection HW thresholds |
| `struct ath_softc` | `ath9k/ath9k.h` | Driver software context |
| `struct ath_txq` | `ath9k/ath9k.h` | TX hardware queue |
| `struct ath_atx_tid` | `ath9k/ath9k.h` | Per-TID aggregation state |
| `struct ath_node` | `ath9k/ath9k.h` | Per-station state |
| `struct ath_chanctx` | `ath9k/ath9k.h` | Per-channel context (multi-channel) |

### 5.3 ath10k Structures

| Structure | File | Purpose |
|-----------|------|---------|
| `struct ath10k` | `ath10k/core.h` | Master device context |
| `struct ath10k_vif` | `ath10k/core.h` | Virtual interface (per-VIF state) |
| `struct ath10k_sta` | `ath10k/core.h` | Per-station driver state |
| `struct ath10k_peer` | `ath10k/core.h` | firmware peer entry |
| `struct ath10k_wmi` | `ath10k/core.h` | WMI control protocol state |
| `struct ath10k_htc` | `ath10k/htc.h` | HTC transport state |
| `struct ath10k_htt` | `ath10k/htt.h` | HTT data path state |
| `struct ath10k_hif_ops` | `ath10k/hif.h` | Bus-agnostic HIF vtable |
| `struct ath10k_hif_sg_item` | `ath10k/hif.h` | Scatter-gather DMA item |
| `struct ath10k_fw_file` | `ath10k/core.h` | Firmware binary components |
| `struct ath10k_fw_stats_pdev`| `ath10k/core.h` | PDEV-level statistics |
| `struct ath10k_fw_stats_vdev`| `ath10k/core.h` | VDEV-level statistics |
| `struct ath10k_fw_crash_data`| `ath10k/core.h` | Crash-dump snapshot |
| `struct ath10k_skb_cb` | `ath10k/core.h` | SKB TX control block |
| `struct ath10k_txq` | `ath10k/core.h` | Per-TID software TX queue |
| `struct ath10k_debug` | `ath10k/core.h` | debugfs state |

---

## 6. Interfaces

### 6.1 Upward: mac80211 (`ieee80211_ops`)

All ath drivers register a `struct ieee80211_ops` with mac80211. Key callbacks:

| Callback | Trigger | Driver Action |
|----------|---------|---------------|
| `start()` | Interface up | HW reset, interrupt setup, RX enable |
| `stop()` | Interface down | Drain queues, disable HW, free IRQ |
| `tx()` | Frame to transmit | Queue to HW TX, power-save wake |
| `set_key()` | Key install/remove | Program key cache or WMI key cmd |
| `config()` | Channel/BW change | HW reset to new channel |
| `bss_info_changed()` | BSS state change | BSSID, beacon interval, PS mode |
| `sta_add/remove()` | Station join/leave | Alloc/free peer state |
| `conf_tx()` | WMM AC parameters | Program EDCA registers |
| `hw_scan()` | Scan request | Channel hopping, probe req TX |
| `sw_scan_start/complete()` | Scan lifecycle | Pause beacons, update filters |
| `get_survey()` | Channel survey | Return channel-busy counters |
| `set_rts_threshold()` | RTS policy | Update HW register |
| `ampdu_action()` | A-MPDU start/stop | BA session management |
| `set_antenna()` | Antenna control | Update chainmask registers |
| `remain_on_channel()` | Off-channel | Off-channel mgmt (P2P, DFS CAC) |

### 6.2 Downward: HIF — Hardware Interface (`ath10k/hif.h`)

Bus-agnostic vtable for ath10k and newer:

```c
struct ath10k_hif_ops {
    int  (*tx_sg)(…);              // Scatter-gather DMA TX
    int  (*diag_read)(…);          // Diagnostic memory read
    int  (*diag_write)(…);         // Diagnostic memory write
    int  (*exchange_bmi_msg)(…);   // BMI sync message
    int  (*start)(…);              // Post-BMI operational start
    void (*stop)(…);               // Operational stop
    int  (*map_service_to_pipe)(…);// Service → CE pipe mapping
    void (*get_default_pipe)(…);   // Default UL/DL pipes
    void (*send_complete_check)(…);// Poll completion (USB/SDIO)
    u32  (*read32)(…);             // MMIO read
    void (*write32)(…);            // MMIO write
    int  (*power_up)(…);           // Power on + BMI mode
    void (*power_down)(…);         // Power off
    int  (*suspend)(…);
    int  (*resume)(…);
    int  (*fetch_cal_eeprom)(…);   // EEPROM calibration read
};
```

### 6.3 ath9k Register I/O (`ath_ops`)

Direct MMIO via the `ath_ops` vtable stored in `ath_common.ops`. Macros in `ath9k/hw.h`:

```c
REG_READ(ah, reg)         → (ah)->reg_ops.read(ah, reg)
REG_WRITE(ah, reg, val)   → (ah)->reg_ops.write(ah, val, reg)
REG_RMW(ah, reg, set, clr)→ (ah)->reg_ops.rmw(ah, reg, set, clr)
```

Buffered write mode (`ENABLE_REGWRITE_BUFFER` / `REGWRITE_BUFFER_FLUSH`) coalesces multiple register writes into a single PCIe transaction burst for performance.

### 6.4 WMI Control Channel (ath10k/ath11k/ath12k)

```
Host Driver                          Firmware
    │           WMI Command               │
    │─────────────────────────────────────▶│
    │    (HTT/HTC transport, CE pipe #1)   │
    │                                      │
    │           WMI Event                  │
    │◀─────────────────────────────────────│
    │    (HTC transport, CE pipe #2)       │
```

WMI uses a **completion-based** synchronous request/response model for configuration and an **event-driven** asynchronous model for notifications (scan done, RX stats, radar detect, …).

---

## 7. State Machines

### 7.1 `ath_device_state` — Device Availability (Shared)

```
   ┌──────────────────┐      probe_success()     ┌──────────────────┐
   │ ATH_HW_UNAVAILABLE│ ──────────────────────▶  │ ATH_HW_INITIALIZED│
   └──────────────────┘                           └──────────────────┘
           ▲                                               │
           │          device_removed / fatal_error         │
           └───────────────────────────────────────────────┘
```

### 7.2 `ath_op_flags` — Operational Flags (Shared bitmask)

Bits cleared/set atomically via `set_bit/clear_bit` in `ath_common.op_flags`:

| Flag | Meaning |
|------|---------|
| `ATH_OP_INVALID` | HW not yet initialized or being reset |
| `ATH_OP_BEACONS` | Beaconing is active |
| `ATH_OP_ANI_RUN` | ANI algorithm is running |
| `ATH_OP_PRIM_STA_VIF` | Primary STA VIF is associated |
| `ATH_OP_HW_RESET` | HW reset in progress (blocks ISR) |
| `ATH_OP_SCANNING` | Active scan in progress |
| `ATH_OP_MULTI_CHANNEL` | Multi-channel context active |
| `ATH_OP_WOW_ENABLED` | Wake-on-Wireless enabled |

### 7.3 ath9k Power State Machine

```
                 ath9k_ps_wakeup()
   ┌────────────────────────────────────────┐
   │                                        │
   ▼                                        │
ATH9K_PM_AWAKE ──────────────────────────▶ ATH9K_PM_NETWORK_SLEEP
   │  (active, interrupts on)                   (PS mode, HW dozing)
   │                                        ▲
   │  ath9k_ps_restore()                   │
   │  (when ps_usecount → 0                │
   │   and no pending flags)               │
   │                                        │
   ▼                                        │
ATH9K_PM_FULL_SLEEP ─────────────────────▶─┘
   (sleeping timer expired, cpu deep sleep)  wakeup on frame

ATH9K_PM_UNDEFINED ← initial / error state
```

#### Power Save Flags (`sc->ps_flags`)

| Flag | Meaning |
|------|---------|
| `PS_WAIT_FOR_BEACON` | Staying awake to receive beacon |
| `PS_WAIT_FOR_CAB` | Waiting for buffered CAB frames |
| `PS_WAIT_FOR_PSPOLL_DATA` | Waiting for PS-Poll response |
| `PS_WAIT_FOR_TX_ACK` | Waiting for TX ACK |
| `PS_WAIT_FOR_ANI` | ANI requires radio active |
| `PS_BEACON_SYNC` | Re-syncing TSF with AP |

### 7.4 ath10k Device State Machine

```
     alloc_hw()
         │
         ▼
   ATH10K_STATE_OFF
         │
         │ core_start() / start()
         ▼
   ATH10K_STATE_ON ──── firmware_crash ────▶ ATH10K_STATE_RESTARTING
         │                                            │
         │                                    mac80211 calls start()
         │                                            │
         │                                            ▼
         │                                   ATH10K_STATE_RESTARTED
         │                                            │
         │                              reconfig_complete() succeeds
         │                                            │
         │◀───────────────────────────────────────────┘        
         │
         │ core_stop()
         ▼
   ATH10K_STATE_OFF

   ATH10K_STATE_WEDGED ← crash during restart (blocks further recovery)
   ATH10K_STATE_UTF     ← factory test mode
```

### 7.5 ath10k Scan State Machine

```
ATH10K_SCAN_IDLE
    │  hw_scan() from mac80211
    ▼
ATH10K_SCAN_STARTING  ── WMI scan start cmd ──▶ (FW ack)
    │
    ▼
ATH10K_SCAN_RUNNING
    │  \───── abort triggered ──▶ ATH10K_SCAN_ABORTING ──▶ ATH10K_SCAN_IDLE
    │
    │  WMI scan complete event
    ▼
ATH10K_SCAN_IDLE
```

### 7.6 ath10k Beacon State (Per VIF)

```
ATH10K_BEACON_SCHEDULED → ATH10K_BEACON_SENDING → ATH10K_BEACON_SENT
        ▲                                                   │
        └───────────────────────────────────────────────────┘
                      (on next DTIM period)
```

---

## 8. Control Flow

### 8.1 Device Initialization (ath9k)

```
pci_driver.probe()
    │
    ├── ieee80211_alloc_hw(sizeof(ath_softc))
    ├── ath9k_init_device()
    │       ├── ath9k_hw_init(ah)            ← detect chip, read EEPROM
    │       ├── ath_init_channels()           ← build channel list from EEPROM
    │       ├── ath9k_init_queues()           ← alloc 10 HW TX queues
    │       ├── ath_init_softc()             ← alloc RX ring, init locks
    │       └── ath_regd_init()              ← register regulatory notifier
    │
    └── ieee80211_register_hw(hw)            ← visible to cfg80211/nl80211
```

### 8.2 Device Initialization (ath10k)

```
pci_driver.probe()
    │
    ├── ath10k_core_create()                 ← alloc struct ath10k
    ├── ath10k_hif_power_up()               ← power on, enter BMI mode
    ├── ath10k_bmi_get_target_info()        ← read chip ID
    ├── ath10k_core_fetch_firmware_files()  ← load FW from filesystem
    ├── ath10k_bmi_load_firmware()          ← DMA FW to chip RAM
    ├── ath10k_hif_start()                  ← exit BMI, start HTC
    ├── ath10k_wmi_wait_for_service_ready() ← wait for FW WMI ready
    ├── ath10k_mac_register()               ← alloc ieee80211_hw, register
    └── ath10k_core_start_recovery_timer()  ← watchdog
```

### 8.3 Channel Change Flow (ath9k)

```
mac80211 → ath9k_config() / chanctx_change()
    │
    ├── ath9k_ps_wakeup()
    ├── ath_prepare_reset()
    │       ├── ieee80211_stop_queues()
    │       ├── ath_stop_ani()
    │       ├── ath9k_hw_disable_interrupts()
    │       ├── ath_stoprecv()          ← drain RX
    │       └── ath_drain_all_txq()    ← drain TX
    │
    ├── ath9k_hw_reset(ah, new_chan, caldata, fastcc)
    │       ├── rf_set_freq()           ← tune RF synthesizer
    │       ├── process_ini()           ← write register arrays
    │       ├── init_cal()              ← run initial calibration
    │       └── restore_chainmask()
    │
    └── ath_complete_reset()
            ├── ath_startrecv()         ← re-enable RX
            ├── ath9k_hw_set_interrupts()
            ├── ath9k_hw_enable_interrupts()
            └── ieee80211_wake_queues()
```

### 8.4 Interrupt / Tasklet Flow (ath9k)

```
Hardware IRQ fires
    │
    ▼
ath_isr()  [interrupt context]
    ├── ath9k_hw_getisr()        ← read & clear ISR
    ├── mask status with imask
    ├── sc->intrstatus |= status
    │
    ├── [SWBA] tasklet_schedule(bcon_tasklet)
    ├── [FATAL/BB_WATCHDOG] → ath9k_queue_reset()
    └── [SCHED_INTR set] → ath9k_hw_kill_interrupts() + tasklet_schedule(intr_tq)

ath9k_tasklet()  [softirq context]
    ├── [INT_FATAL]      → queue_reset(RESET_TYPE_FATAL_INT)
    ├── [INT_BB_WATCHDOG]→ queue_reset(RESET_TYPE_BB_WATCHDOG)
    ├── [INT_GTT]        → check alive, maybe reset(RESET_TYPE_TX_GTT)
    ├── [INT_RX/RXHP]    → ath_rx_tasklet()
    ├── [INT_TX]         → ath_tx_tasklet() / ath_tx_edma_tasklet()
    └── ath9k_hw_resume_interrupts()
```

---

## 9. Data Flow (TX / RX)

### 9.1 TX Data Path (ath9k)

```
Application write() → socket → net_device → mac80211 scheduler
        │
        ▼ ieee80211_ops.tx()
ath9k_tx()
    ├── Form ath_tx_control (txctl)
    ├── ath_tx_start(sc, skb, txctl)
    │       ├── ath_tx_classify()      ← assign AC / TID
    │       ├── A-MPDU check:
    │       │     ├── ath_tx_send_ampdu()  ← aggregate into A-MPDU
    │       │     └── ath_tx_send_normal() ← single MPDU
    │       └── ath_tx_txqaddbuf()
    │               ├── Build TX descriptor (ath9k_hw_set_txdesc)
    │               └── REG_WRITE(ah, AR_QTXDP(q), bf->bf_daddr) ← tell HW
    │
    HW fetches descriptor via DMA
    Transmit over air
    HW generates TX completion interrupt (INT_TX)
    │
    ▼ ath_tx_tasklet() / ath_tx_edma_tasklet()
    ath_tx_process_buffers()
        ├── ath9k_hw_gettxbuf() or read EDMA TX status
        ├── ath_tx_complete_buf()    ← update stats
        └── ieee80211_tx_status()   ← report to mac80211
```

### 9.2 RX Data Path (ath9k)

```
Frame arrives over air
HW DMA into pre-posted RX buffer
HW generates INT_RX / INT_RXHP interrupt
    │
    ▼ ath_rx_tasklet()
    ath_rx_edma_buf_link()   ← or legacy path
    │
    ▼ ath_rx_fn() per descriptor
    ├── ath9k_hw_proc_rxdesc_edma()  ← parse RX status
    ├── ath_rx_ps()                   ← handle power save state
    ├── ath_check_rxbuf_len()
    ├── ath_rx_radiotap()             ← if monitor mode
    │
    ├── ath_rx_filter()               ← drop unwanted frames
    ├── ath9k_cmn_rx_skb_postprocess()← strip HW padding
    │
    └── ieee80211_rx()               ← deliver to mac80211
```

### 9.3 TX Data Path (ath10k — firmware offload)

```
mac80211 → ath10k_tx()
    ├── ath10k_tx_h_*() processing hooks (PS, CRYPTO, HT, channel ctx)
    └── ath10k_htt_tx()
            ├── Build HTT_MSG_TYPE_TX_FRM descriptor
            ├── DMA map skb
            └── ath10k_hif_tx_sg() → Copy Engine push

Firmware:
    ← copies frame from CE → TX ring → air

Completion: CE interrupt → ath10k_htc_rx_completion_handler()
    → HTT TX completion msg → ath10k_txrx_tx_unref()
    → ieee80211_tx_status()
```

### 9.4 RX Data Path (ath10k — firmware offload)

```
Air → firmware RX ring → CE delivery
    │
    ▼ CE RX interrupt → ath10k_htc_rx_completion_handler()
    → HTT_T2H_MSG_TYPE_RX_IND / RX_FRAG_IND
    → ath10k_htt_rx_handler()
        ├── ath10k_htt_rx_h_csum_offload()
        ├── ath10k_htt_rx_h_ampdu()
        ├── ath10k_htt_rx_h_decrypt()    ← or HW crypto done by FW
        └── ieee80211_rx() to mac80211
```

---

## 10. DFS — Dynamic Frequency Selection Subsystem

DFS enables the use of 5 GHz radar-restricted channels by detecting radar pulses and vacating the channel. The ATH common library provides a full software DFS pattern detector.

### 10.1 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      mac80211 / cfg80211                             │
│   ieee80211_radar_detected() ←───────────────────────────────────── │
└──────────────────────────────────────────────────────────────────────┘
                                    ▲
                          radar detected!
                                    │
┌───────────────────────────────────┴─────────────────────────────────┐
│              dfs_pattern_detector  (dfs_pattern_detector.c)          │
│                                                                       │
│   dpd->add_pulse(dpd, &pulse_evt)                                    │
│      │                                                               │
│      ▼                                                               │
│   foreach channel_detector (per active channel):                     │
│      foreach radar_type (FCC/ETSI/MKK patterns):                    │
│         pri_detector.add_pulse() ──▶ match PRI sequence?             │
│                                          │ YES                       │
│                                          └──▶ return match (radar!)  │
└───────────────────────────────────────────────────────────────────────┘
                          ▲
              pulse_event (ts, freq, width, rssi, chirp)
                          │
┌─────────────────┴─────────────────────────────────────────────────┐
│           Per-chip DFS pulse detection (e.g. ath9k/dfs.c)          │
│                                                                     │
│   ath9k_rx_indication() detects PHY_ERROR_RADAR                    │
│   → ath9k_dfs_process_phyerr()                                     │
│           ├── parse HW radar event registers                        │
│           ├── fill struct pulse_event                               │
│           └── dpd->add_pulse()                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.2 Key Structures for DFS

```c
// PHY pulse from hardware
struct pulse_event {
    u64  ts;      // timestamp in µs
    u16  freq;    // channel freq MHz
    u8   width;   // pulse duration µs
    u8   rssi;    // radar RSSI
    bool chirp;   // frequency chirp detected
};

// Pattern detector (singleton per driver instance)
struct dfs_pattern_detector {
    bool (*set_dfs_domain)(dpd, region);  // change regulatory domain
    bool (*add_pulse)(dpd, pe, rs);       // add pulse, returns TRUE on detection
    void (*exit)(dpd);                    // destructor

    enum nl80211_dfs_regions region;
    u8 num_radar_types;
    u64 last_pulse_ts;
    const struct radar_detector_specs *radar_spec;  // array per DFS domain
    struct list_head channel_detectors;              // per-channel state
};

// Per-radar-type pulse sequence being tracked
struct pri_sequence {
    u32 pri;             // Pulse Repetition Interval µs
    u32 dur;             // duration
    u32 count;           // matching pulses
    u32 count_falses;    // non-matching pulses
    u64 first_ts, last_ts, deadline_ts;
};
```

### 10.3 Radar Pattern Specifications

Each DFS domain (FCC/ETSI/MKK) defines radar types with:
- `width_min/max` — pulse duration bounds (µs)
- `pri_min/pri_max` — PRI bounds (µs)
- `ppb` — pulses per burst
- `ppb_thresh` — detection threshold
- `max_pri_tolerance` — timing jitter tolerance
- `chirp` — whether chirp is required

---

## 11. Cryptographic Key Management

### 11.1 Key Cache

The ATH hardware has a **128-entry hardware key cache** (`ATH_KEYMAX = 128`). Each entry stores a cipher key, type, and MAC address for hardware-assisted decryption/encryption.

### 11.2 Cipher Support

| `ath_cipher` enum | Value | IEEE Suite |
|--------------------|-------|-----------|
| `ATH_CIPHER_WEP` | 0 | WEP-40/104/128 |
| `ATH_CIPHER_AES_OCB` | 1 | AES-OCB |
| `ATH_CIPHER_AES_CCM` | 2 | WPA2-CCMP |
| `ATH_CIPHER_CKIP` | 3 | CKIP |
| `ATH_CIPHER_TKIP` | 4 | WPA-TKIP |
| `ATH_CIPHER_CLR` | 5 | No encryption |
| `ATH_CIPHER_MIC` | 127 | TKIP Michael MIC |

### 11.3 TKIP Key Cache Layout

TKIP requires MIC keys in addition to the cipher key:

**Combined MIC mode** (`ATH_CRYPT_CAP_MIC_COMBINED` set):
- Entry at index `N`: TKIP cipher key
- Entry at index `N+64`: TX MIC + RX MIC combined

**Split MIC mode** (older hardware):
- Entry at `N`: TKIP key (TX)
- Entry at `N+32`: MIC RX
- Entry at `N+64`: MIC TX secondary
- Entry at `N+64+32`: MIC additional

### 11.4 Key Allocation Algorithm (`ath_reserve_key_cache_slot`)

```
ath_key_config()
    │
    ├── TKIP cipher → ath_reserve_key_cache_slot_tkip()
    │       Finds a slot where neither N nor N+64 are used
    │       Avoids collision with WEP4 reserved slots [0..3]
    │
    └── Other ciphers → ath_reserve_key_cache_slot()
            Priority: recycle partially-allocated slots first
            Fallback: scan from slot 4 upward
            Exclude reserved TKIP companion slot regions
```

### 11.5 TKIP Key Write Sequence

The hardware requires a **specific write ordering** to prevent MIC errors on in-flight frames:
1. Write **inverted** key[47:0] first (disables encryption briefly)
2. Write rest of key (key[95:48] and key type)
3. Write MIC keys
4. Write MAC address
5. Write **correct** key[47:0] last (re-enables)

---

## 12. Regulatory Domain Management

### 12.1 Structure

The `regd.c` / `regd.h` files implement Atheros proprietary regulatory domain mapping on top of cfg80211/nl80211 regulatory infrastructure.

```
cfg80211 regulatory_request
         │
         ▼
ath_reg_notifier_apply()
    ├── Map country_code → ath_regulatory
    ├── ath_regd_find_country_by_name(alpha2)
    └── Apply power limits and CTL (Channel Tx Level) rules
```

### 12.2 Domain Hierarchy

```
WorldWide Domain (0x00)
    │
    ├── CountryCode → RegDomainEnum
    │       (country_code_to_enum_rd table)
    │
    └── RegDomainEnum → reg_dmn_pair_mapping
                ├── reg_5ghz_ctl (5G transmit power class)
                └── reg_2ghz_ctl (2G transmit power class)

CTL Groups:
    CTL_FCC  = 0x10   (North America)
    CTL_ETSI = 0x30   (Europe)
    CTL_MKK  = 0x40   (Japan)
```

### 12.3 Channel Restrictions

Each allowed channel has an associated **CTL value**:
- `CTL_11A` (5 GHz OFDM)
- `CTL_11B` (2.4 GHz DSSS)
- `CTL_11G` (2.4 GHz OFDM)
- `CTL_2GHT20/40`, `CTL_5GHT20/40` (HT rates)

These map to transmit power entries in the EEPROM power calibration tables.

---

## 13. Power Management

### 13.1 ath9k Power Save Reference Count

```
ath9k_ps_wakeup(sc)   → increments sc->ps_usecount
                         if count was 0: wake HW (PM_AWAKE)

ath9k_ps_restore(sc)  → decrements sc->ps_usecount
                         if count → 0 and ps_enabled:
                           set PM_NETWORK_SLEEP
                         if sc->ps_idle:
                           arm sleep_timer (100ms)

ath_ps_full_sleep()   → timer callback:
                         PM_FULL_SLEEP (deep sleep)
```

### 13.2 Conditions Blocking Sleep

The driver stays awake (`ps_flags` set) for:
- `PS_WAIT_FOR_BEACON`: TSF hasn't synced
- `PS_WAIT_FOR_CAB`: AP has buffered multicast for us
- `PS_WAIT_FOR_TX_ACK`: a TX frame is in flight
- `PS_WAIT_FOR_PSPOLL_DATA`: waiting for PS-Poll response
- Active ANI calibration

### 13.3 Wake-on-Wireless (WoW)

Supported in ath9k for:
- Magic packet pattern
- User-defined patterns  
- Link status change
- Beacon miss

HW enters deep sleep; pattern matching engine keeps running.
WoW trigger restores full operation.

### 13.4 ath10k Power Management

ath10k uses **WMI power save commands** sent to firmware:
- `WMI_STA_PS_PARAM_UAPSD`: UAPSD configuration
- `WMI_STA_PS_PARAM_TX_WAKE_THRESHOLD`: wake on TX threshold
- `ath10k_wmi_pdev_suspend/resume()`: suspend entire platform

---

## 14. Algorithms & Calibration

### 14.1 Adaptive Noise Immunity (ANI)

ANI dynamically tunes 5 PHY parameters to reduce false-positive frame reception in high-noise environments:

| Parameter | Direction | Effect |
|-----------|-----------|--------|
| Spur Immunity | ↑ | Ignore narrow-band interference |
| OFDM Weak Signal | disable | Only detect strong signals |
| CCK Weak Signal | disable | Only detect strong CCK |
| First Step | ↑ | Require stronger correlation burst |
| OFDM/CCK Error Thresholds | adjust | Trigger on error counts |

**ANI loop (ath9k):**
1. `checkani_timer` fires (100ms interval)
2. Read PHY error counters (OFDM errors, CCK errors, beacon RSSI)
3. If errors > upper threshold → tighten sensitivity
4. If errors < lower threshold → relax sensitivity
5. Write new ANI settings to HW registers

### 14.2 Noise Floor Calibration

- NF = Noise Floor value (dBm) measured when the channel is idle
- Sampled periodically and stored in a history buffer (`nfCalHist`)
- Median of recent values used to avoid transient spikes
- Applied to the PHY's AGC reference level

### 14.3 IQ Calibration (ath9k)

- Corrects I/Q imbalance caused by analog imperfections
- Measures correlation coefficients (`iCoff`, `qCoff`)
- Applied via internal correction registers

### 14.4 PAPRD — Pre-Distortion (ath9k AR9300+)

Power Amplifier Pre-distortion linearizes the PA:
1. Transmit a known test signal (`paprd_work`)
2. Measure the loopback response
3. Compute a distortion table (`pa_table[])`)
4. Load the table into HW for real-time compensation

### 14.5 Dynamic ACK Timeout (DynACK — ath9k)

Dynamically adjusts the ACK wait timeout based on measured round-trip times. This maximizes throughput in long-range outdoor links by avoiding premature timeouts without wasting slots on short-range links.

### 14.6 DFS PRI Detection Algorithm

```
New pulse arrives with timestamp ts
    │
    ├── For each known pri_sequence:
    │     if |ts - sequence.last_ts - sequence.pri| ≤ tolerance:
    │         → extend sequence (count++)
    │         if count ≥ ppb_thresh: RADAR DETECTED!
    │
    └── Create new candidate pri_sequence for all possible PRIs
        that could explain the timestamp relative to all prior pulses.
        (O(n_pulses × n_pri_candidates) per pulse event)

Sequences older than window_size are expired.
Pool allocator manages pulse_elem and pri_sequence objects.
```

---

## 15. Cross-Cutting Concerns

### 15.1 Locking Model (ath9k)

| Lock | Scope | Protects |
|------|-------|---------|
| `sc->mutex` | Process context | All config operations |
| `sc->sc_pcu_lock` (spin) | IRQ + task context | PCU register access |
| `sc->sc_pm_lock` (spin, irqsave) | ISR + process | Power save state, ps_usecount |
| `sc->intr_lock` (spin) | ISR | intrstatus snapshot |
| `txq->axq_lock` (spin) | Task context | Per-queue TX descriptor lists |
| `common->cc_lock` (spin) | IRQ | Cycle counter snapshot |
| `sc->chan_lock` (spin) | Mixed | cur_chandef |

### 15.2 Locking Model (ath10k)

| Lock | Scope | Protects |
|------|-------|---------|
| `ar->conf_mutex` | Process (mutex) | All configuration |
| `ar->data_lock` | IRQ (spin, irqsave) | Peer list, RX buffers, TX queues |
| `ar->ce_lock` | IRQ (spin) | Copy Engine TX/RX rings |
| `ar->htt.tx_lock` | Mixed | HTT TX index bitmap |

### 15.3 Debug Infrastructure

Unified debug system via `ath_dbg()` macro:

```c
ath_dbg(common, RESET, "Resetting hw...\n");
// expands to: if (common->debug_mask & ATH_DBG_RESET) ath_printk(…)
```

Categories (`enum ATH_DEBUG`):
`RESET, QUEUE, EEPROM, CALIBRATE, INTERRUPT, REGULATORY, ANI, XMIT, BEACON, CONFIG, FATAL, PS, BTCOEX, WMI, BSTUCK, MCI, DFS, WOW, CHAN_CTX, DYNACK, SPECTRAL_SCAN`

debugfs entries expose: register dumps, TX/RX statistics, calibration data, ANI levels, DFS pool stats, TPC tables, firmware crash dumps.

### 15.4 Tracing

`trace.c` / `trace.h` define `ftrace` tracepoints:
- `ath_log`: wraps `ath_printk` output into trace ring buffer
- Used for post-mortem analysis without live logging overhead

### 15.5 Spectral Scan

The ATH hardware can perform **FFT spectral scans** — capturing raw power spectra across the channel without disrupting normal operation:
- `ath_spec_scan` structure configures: enabled, count, period, fft_period
- `spectral_scan_config() / trigger() / wait()` HW ops
- Output delivered via a dedicated debugfs relay file
- Used for interference detection and spectrum management tools

### 15.6 Bluetooth Coexistence (btcoex)

ath9k supports three BT coexistence mechanisms:
- **2-wire**: Simple WLAN/BT priority arbitration (GPIO)
- **3-wire**: Grant arbitration (common for Atheros combo chips)
- **MCI** (Message Coexistence Interface): Full protocol between WLAN MAC and BT MAC for AR9462+ combo chips. Uses dedicated MMIO messaging registers, supports A2DP/SCO/BLE traffic classification.

---

## Summary

The ATH driver family demonstrates a clean **layered abstraction** with a shared common library covering regulatory, key management, DFS, ANI, and debug infrastructure. Sub-drivers plug into this common framework while differing in their transport model:

- **ath5k/ath9k**: Direct register programming, host-driven MAC, tight latency coupling
- **ath10k/ath11k/ath12k**: Firmware offload model with WMI control and bus-abstracted HIF transport, enabling more sophisticated firmware capabilities (beamforming, MU-MIMO, MLO) while decoupling the host driver from rapid hardware evolution.

The `dfs_pattern_detector` / `dfs_pri_detector` tandem implements a domain-agnostic radar detection engine shared across all modern sub-drivers. The key cache management (`key.c`) provides a single correct implementation of the subtle TKIP MIC splitting/combining logic.
