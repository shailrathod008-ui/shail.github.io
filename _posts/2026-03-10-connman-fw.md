---
layout: post
title: "connman to fw"
date: 2026-03-07 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "connman to fw"

---
---

## Complete Qualcomm-Specific Call Chain
```
ConnMan service_connect()
    │
    └─▶ __connman_network_connect()
            │
            └─▶ wifi_connect()  [plugins/wifi.c]
                    │
                    └─▶ g_supplicant_interface_connect()  [gsupplicant]
                            │  D-Bus: fi.w1.wpa_supplicant1.Interface.SelectNetwork
                            ▼
                        wpa_supplicant_associate()
                            │
                            └─▶ wpa_driver_nl80211_connect()
                                    │  Netlink: NL80211_CMD_CONNECT
                                    ▼
                                nl80211_connect()  [net/wireless/nl80211.c]
                                    │
                                    └─▶ cfg80211_connect()  [net/wireless/sme.c]
                                            │
                                            └─▶ rdev_connect()
                                                    │
                                    ┌───────────────┼────────────────────┐
                                    ▼               ▼                    ▼
                              ath10k_connect   ath11k_connect    wlan_hdd_cfg80211_connect
                              [mac.c]          [mac.c]           [qcacld/hdd/cfg80211.c]
                                    │               │                    │
                                    ▼               ▼                    ▼
                             ath10k_wmi_cmd   ath11k_wmi_cmd      wmi_unified_cmd_send
                                    │               │                    │
                                    ▼               ▼                    ▼
                              ath10k_htc_send  ath11k_htc_send     HTT/HTC layer
                                    │               │                    │
                                    ▼               ▼                    ▼
                              Copy Engine TX   Copy Engine TX      Copy Engine TX
                                    │               │                    │
                                    └───────────────┴────────────────────┘
                                                    │
                                              PCIe / SNOC / USB
                                                    │
                                                    ▼
                                        ┌─────────────────────┐
                                        │   QCA Chip Firmware  │
                                        │  (WLAN FW / LMAC)   │
                                        │                      │
                                        │  802.11 Auth Frame   │
                                        │  802.11 Assoc Frame  │
                                        │  EAPOL 4-way H/S     │
                                        │  → Connected!        │
                                        └─────────────────────┘
                                                    │
                                         WMI_CONNECT_EVENTID
                                                    │
                                                    ▼
                                         cfg80211_connect_result()
                                                    │
                                         nl80211 event → wpa_supplicant
                                                    │
                                         D-Bus event → ConnMan
                                                    │
                                         Service state = ONLINE ✓
```

---

## Key Qualcomm-Specific Structures

| Component | Key Structure | File |
|---|---|---|
| ath10k device | `struct ath10k` | `ath10k/core.h` |
| ath10k vif | `struct ath10k_vif` | `ath10k/core.h` |
| ath11k device | `struct ath11k` | `ath11k/core.h` |
| WMI handle | `struct wmi_unified` | `wmi_unified_priv.h` |
| Copy Engine | `struct ath10k_ce_pipe` | `ath10k/ce.h` |
| HTC endpoint | `struct ath10k_htc_ep` | `ath10k/htc.h` |
| qcacld HDD | `struct hdd_adapter` | `qcacld/hdd/inc/wlan_hdd_main.h` |
| VDEV start | `struct wmi_vdev_start_request_arg` | `wmi.h` |

---

## Firmware Communication Summary
```
Host Driver          QCA Firmware (DSP/ARM inside chip)
    │                          │
    │── WMI CMD_CONNECT ──────▶│  Process connect
    │                          │  Run 802.11 state machine
    │                          │  TX Auth / Assoc frames over RF
    │                          │
    │◀── WMI EVENT_CONNECT ────│  Report result
    │                          │
    │── WMI SET_KEY ──────────▶│  Install PTK/GTK
    │                          │  Enable data path encryption
    │                          │
  Data path                  Data path
  (HTT protocol)           (LMAC/UMAC)

```
