---
layout: post
title: "BlueZ Architecture and Design Documentation"
date: 2026-01-30 9:00:00 +0530
categories: [connman]
tags: [connman,wlan]
description: "BlueZ Architecture and Design Documentation in  Depth"

---

# BlueZ Architecture and Design Documentation

**Version:** 1.0  
**Date:** January 2026  
**Purpose:** Comprehensive architectural documentation of the BlueZ Bluetooth protocol stack for Linux

---

## Table of Contents

1. [Overview & High-Level Architecture](#1-overview--high-level-architecture)
2. [Core Components](#2-core-components)
3. [State Machines](#4-state-machines)
4. [Control and Data Flows](#5-control-and-data-flows)
5. [Data Structures](#6-data-structures)
6. [Algorithms and Design Decisions](#7-algorithms-and-design-decisions)
7. [Architecture Diagrams](#8-architecture-diagrams)

---

## 1. Overview & High-Level Architecture

### 1.1 Purpose and Scope

BlueZ is the official Linux Bluetooth protocol stack, providing comprehensive support for:
- **Bluetooth Classic (BR/EDR)**: Traditional Bluetooth for audio, file transfer, and serial communication
- **Bluetooth Low Energy (LE)**: Power-efficient Bluetooth for IoT, wearables, and sensors
- **Bluetooth Mesh**: Multi-hop networking for smart home and industrial applications

### 1.2 System Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Applications                              │
│  (bluetoothctl, audio players, file managers, custom apps)  │
└─────────────────────────────────────────────────────────────┘
                            ↕ D-Bus
┌─────────────────────────────────────────────────────────────┐
│                BlueZ User-Space Daemon                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  bluetoothd  │  │  bluetooth-  │  │    obexd     │      │
│  │   (main)     │  │    meshd     │  │  (OBEX)      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↕ HCI/MGMT
┌─────────────────────────────────────────────────────────────┐
│                  Linux Kernel                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Bluetooth Subsystem (net/bluetooth/)                │   │
│  │  - HCI Core  - L2CAP  - RFCOMM  - BNEP  - HIDP      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↕ USB/UART/SDIO
┌─────────────────────────────────────────────────────────────┐
│              Bluetooth Hardware Controller                   │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 Component Interaction Overview

The BlueZ architecture follows a layered design:

1. **Hardware Layer**: Bluetooth controllers (USB, UART, SDIO)
2. **Kernel Layer**: HCI protocol, L2CAP, RFCOMM, and other core protocols
3. **User-Space Daemon**: `bluetoothd` - main daemon managing adapters, devices, and profiles
4. **D-Bus IPC**: Inter-process communication for applications
5. **Application Layer**: User applications and system services

**Key Communication Paths:**
- **MGMT Interface**: Kernel ↔ bluetoothd for adapter management
- **HCI Socket**: Kernel ↔ bluetoothd for device communication
- **D-Bus**: bluetoothd ↔ Applications for high-level operations

---

## 2. Core Components

### 2.1 Main Daemon (`bluetoothd`)

**File**: `src/main.c`

The `bluetoothd` daemon is the central component of BlueZ, responsible for:
- Adapter enumeration and management
- Device discovery and connection management
- Profile registration and service discovery
- D-Bus service provisioning
- Plugin loading and management

**Initialization Sequence:**
1. Parse configuration file (`/etc/bluetooth/main.conf`)
2. Initialize D-Bus connection
3. Setup MGMT interface to kernel
4. Load plugins
5. Initialize adapter subsystem
6. Register D-Bus services
7. Enter GMainLoop event loop

**Event Loop Architecture:**
- Uses **GLib's GMainLoop** for event-driven operation
- File descriptor monitoring via `g_io_add_watch()`
- Timer callbacks via `g_timeout_add()`
- Idle callbacks via `g_idle_add()`
- D-Bus message dispatch integrated into main loop

**Configuration Management:**
The daemon supports extensive configuration through `main.conf`:
- General settings (Name, Class, DiscoverableTimeout, PairableTimeout)
- BR/EDR parameters (PageScanInterval, InquiryScanInterval, etc.)
- LE parameters (ScanInterval, ConnectionInterval, etc.)
- Policy settings (AutoEnable, ReconnectAttempts, etc.)
- GATT settings (Cache, KeySize, ExchangeMTU)
- Profile-specific settings (AVDTP, AVRCP, CSIS, etc.)

### 2.2 Adapter Management

**Files**: `src/adapter.h`, `src/adapter.c`

The adapter subsystem manages Bluetooth controllers (HCI devices).

**Key Responsibilities:**
- Adapter power management
- Discovery (inquiry and LE scanning)
- Pairable/Discoverable modes
- Device tracking and management
- Service registration (SDP)
- GATT database management
- Advertisement management

**Adapter Structure** (`struct btd_adapter`):
```c
struct btd_adapter {
    uint16_t dev_id;              // HCI device index
    struct mgmt *mgmt;            // Management interface
    bdaddr_t bdaddr;              // Bluetooth address
    uint32_t current_settings;    // Current adapter settings
    uint32_t power_state;         // Power state (off/on/transitioning)
    
    GSList *devices;              // List of discovered/paired devices
    GSList *connections;          // Currently connected devices
    GSList *discovery_list;       // Active discovery sessions
    
    struct btd_gatt_database *database;  // GATT server database
    struct btd_adv_manager *adv_manager; // LE advertising manager
    struct btd_adv_monitor_manager *adv_monitor_manager;
    
    bool discovering;             // Discovery in progress
    uint8_t discovery_type;       // BREDR/LE/DUAL
    // ... (many more fields)
};
```

**Adapter Power States:**
```c
enum {
    ADAPTER_POWER_STATE_OFF,           // Powered off
    ADAPTER_POWER_STATE_ON,            // Powered on
    ADAPTER_POWER_STATE_ON_DISABLING,  // Transitioning to off
    ADAPTER_POWER_STATE_OFF_ENABLING,  // Transitioning to on
    ADAPTER_POWER_STATE_OFF_BLOCKED,   // Blocked by rfkill
};
```

**Discovery Management:**
- Supports multiple concurrent discovery sessions from different clients
- Merges discovery filters from all active sessions
- Handles both BR/EDR inquiry and LE scanning
- RSSI filtering and pathloss calculation
- Duplicate device detection

### 2.3 Device Management

**Files**: `src/device.h`, `src/device.c`

The device subsystem represents remote Bluetooth devices.

**Key Responsibilities:**
- Device lifecycle management (creation, pairing, removal)
- Connection state tracking (per-bearer: BR/EDR and LE)
- Service discovery (SDP for BR/EDR, GATT for LE)
- Pairing and bonding
- Persistent storage of device information
- Auto-connection management

**Device Structure** (`struct btd_device`):
```c
struct btd_device {
    bdaddr_t bdaddr;              // Device Bluetooth address
    uint8_t bdaddr_type;          // Address type (public/random)
    char *path;                   // D-Bus object path
    
    struct btd_bearer *bredr;     // BR/EDR bearer
    struct btd_bearer *le;        // LE bearer
    
    struct bearer_state bredr_state;  // BR/EDR connection state
    struct bearer_state le_state;     // LE connection state
    
    GSList *services;             // List of btd_service
    GSList *uuids;                // Discovered service UUIDs
    
    struct gatt_db *db;           // GATT database cache
    struct bt_gatt_client *client; // GATT client instance
    struct bt_gatt_server *server; // GATT server instance
    
    bool trusted;                 // User trust setting
    bool blocked;                 // Blocked device
    bool temporary;               // Temporary (not stored)
    bool connectable;             // Device is connectable
    
    // Pairing/bonding
    struct bonding_req *bonding;
    struct authentication_req *authr;
    
    // Storage
    uint8_t prefer_bearer;        // Preferred connection bearer
    // ... (many more fields)
};
```

**Bearer State** (per BR/EDR and LE):
```c
struct bearer_state {
    bool paired;                  // Pairing completed
    bool bonded;                  // Bonding info stored
    bool connected;               // Currently connected
    bool svc_resolved;            // Services discovered
    bool initiator;               // We initiated connection
    bool connectable;             // Device is connectable
    time_t last_seen;             // Last advertisement/inquiry
    time_t last_used;             // Last connection time
};
```

**Device Storage:**
Devices are persisted in `/var/lib/bluetooth/<adapter_addr>/<device_addr>/`:
- `info`: General device information (name, class, UUIDs, trusted, etc.)
- `attributes`: GATT attribute cache
- `cache`: Temporary cached data (name resolution failures)

### 2.4 GATT Layer

**Files**: `src/gatt-database.h`, `src/gatt-database.c`, `src/gatt-client.h`, `src/gatt-client.c`

The GATT layer provides Generic Attribute Profile support for Bluetooth LE.

**GATT Database** (`struct btd_gatt_database`):
- Manages the local GATT server database
- Handles service registration from profiles
- Processes GATT read/write requests
- Manages characteristic notifications/indications
- Implements GATT caching (Service Changed characteristic)

**GATT Client** (`struct bt_gatt_client`):
- Performs service discovery on remote devices
- Handles characteristic read/write operations
- Manages notifications/indications subscriptions
- Implements ATT protocol with automatic retries
- MTU negotiation

**Shared GATT Implementation** (`src/shared/gatt-*`):
- `gatt-db`: Generic GATT database structure
- `gatt-client`: GATT client implementation
- `gatt-server`: GATT server implementation
- `gatt-helpers`: Utility functions for GATT operations
- `att`: ATT protocol implementation

### 2.5 Profile Implementations

**File**: `src/profile.h`

Profiles define how BlueZ interacts with specific Bluetooth services.

**Profile Structure** (`struct btd_profile`):
```c
struct btd_profile {
    const char *name;             // Profile name
    int priority;                 // Connection priority
    int bearer;                   // BREDR/LE/ANY
    
    const char *local_uuid;       // Local service UUID
    const char *remote_uuid;      // Remote service UUID
    
    bool auto_connect;            // Auto-connect on discovery
    bool external;                // Exposed via GATT API
    bool experimental;            // Experimental profile
    
    // Callbacks
    int (*device_probe)(struct btd_service *service);
    void (*device_remove)(struct btd_service *service);
    int (*connect)(struct btd_service *service);
    int (*disconnect)(struct btd_service *service);
    int (*adapter_probe)(struct btd_profile *p, struct btd_adapter *adapter);
    void (*adapter_remove)(struct btd_profile *p, struct btd_adapter *adapter);
};
```

**Profile Categories:**

1. **Audio Profiles** (`profiles/audio/`):
   - A2DP (Advanced Audio Distribution Profile)
   - AVRCP (Audio/Video Remote Control Profile)
   - HFP (Hands-Free Profile)
   - HSP (Headset Profile)
   - BAP (Basic Audio Profile - LE Audio)
   - MCP (Media Control Profile)
   - VCP (Volume Control Profile)

2. **Network Profiles** (`profiles/network/`):
   - PANU (Personal Area Networking User)
   - NAP (Network Access Point)
   - GN (Group Network)

3. **Input Profiles** (`profiles/input/`):
   - HID (Human Interface Device)
   - HoG (HID over GATT)

4. **Other Profiles**:
   - Battery Service (`profiles/battery/`)
   - Device Information Service (`profiles/deviceinfo/`)
   - MIDI (`profiles/midi/`)
   - Ranging (`profiles/ranging/`)

**Profile Registration:**
Profiles register with BlueZ using `btd_profile_register()`. When a device is discovered with matching UUIDs, the profile's `device_probe()` is called to create a service instance.

### 2.6 Supporting Components

**SDP Server and Client:**
- **SDP Server** (`src/sdpd-*.c`): Manages Service Discovery Protocol records for BR/EDR
- **SDP Client** (`src/sdp-client.c`): Performs service discovery on remote BR/EDR devices

**Agent Management** (`src/agent.h`, `src/agent.c`):
- Handles user interaction for pairing (PIN, passkey, confirmation)
- Supports multiple agent capabilities (DisplayOnly, KeyboardOnly, NoInputNoOutput, etc.)
- Agent priority system for multiple registered agents

**Storage Subsystem** (`src/storage.h`, `src/storage.c`):
- Persistent storage of adapter and device information
- Uses GKeyFile format for configuration files
- Storage location: `/var/lib/bluetooth/`

**Advertisement Management** (`src/advertising.h`, `src/advertising.c`):
- LE Advertisement registration and management
- Multiple concurrent advertisements
- Advertisement data and scan response configuration

**Advertisement Monitor** (`src/adv_monitor.h`, `src/adv_monitor.c`):
- Monitors LE advertisements matching specific patterns
- RSSI-based filtering
- Integration with kernel advertisement monitoring

**Battery Provider** (`src/battery.h`, `src/battery.c`):
- Battery service provider for external battery information
- Exposes battery level via D-Bus

---

## 3. D-Bus Interface Architecture

### 3.1 D-Bus Service Structure

**Service Name**: `org.bluez`  
**Bus Type**: System Bus

BlueZ exposes its functionality through D-Bus, allowing applications to:
- Manage adapters (power, discovery, pairing)
- Connect to devices
- Access GATT services
- Register agents for user interaction
- Register profiles for custom services

### 3.2 Object Path Hierarchy

```
/org/bluez
├── /org/bluez/hci0                          (Adapter)
│   ├── /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX  (Device)
│   │   ├── /org/bluez/hci0/dev_.../serviceXXXX  (GATT Service)
│   │   │   ├── /org/bluez/hci0/dev/.../charYYYY  (GATT Characteristic)
│   │   │   │   └── /org/bluez/hci0/dev/.../descZZZZ  (GATT Descriptor)
│   │   └── /org/bluez/hci0/dev_.../playerX      (Media Player)
│   └── /org/bluez/hci0/adv_monitorX         (Advertisement Monitor)
└── /org/bluez/hci1                          (Another Adapter)
```

### 3.3 Key D-Bus Interfaces

#### 3.3.1 org.bluez.Adapter1

**Purpose**: Represents a Bluetooth adapter (controller)

**Methods:**
- `StartDiscovery()`: Begin device discovery
- `StopDiscovery()`: Stop device discovery
- `RemoveDevice(object device)`: Remove a device
- `SetDiscoveryFilter(dict filter)`: Set discovery filters
- `GetDiscoveryFilters()`: Get supported filters
- `ConnectDevice(dict properties)`: Direct connection without discovery

**Properties:**
- `Address` (string, readonly): Adapter Bluetooth address
- `Name` (string, readonly): System name
- `Alias` (string, readwrite): User-friendly name
- `Class` (uint32, readonly): Class of device
- `Powered` (boolean, readwrite): Power state
- `Discoverable` (boolean, readwrite): Discoverable mode
- `Pairable` (boolean, readwrite): Pairable mode
- `PairableTimeout` (uint32, readwrite): Pairable timeout
- `DiscoverableTimeout` (uint32, readwrite): Discoverable timeout
- `Discovering` (boolean, readonly): Discovery active
- `UUIDs` (array{string}, readonly): Supported services
- `Modalias` (string, readonly, optional): Device ID

#### 3.3.2 org.bluez.Device1

**Purpose**: Represents a remote Bluetooth device

**Methods:**
- `Connect()`: Connect all auto-connectable profiles
- `Disconnect()`: Disconnect device
- `ConnectProfile(string uuid)`: Connect specific profile
- `DisconnectProfile(string uuid)`: Disconnect specific profile
- `Pair()`: Initiate pairing
- `CancelPairing()`: Cancel ongoing pairing

**Properties:**
- `Address` (string, readonly): Device Bluetooth address
- `Name` (string, readonly, optional): Device name
- `Alias` (string, readwrite): User-friendly name
- `Class` (uint32, readonly, optional): Class of device
- `Appearance` (uint16, readonly, optional): GAP appearance
- `UUIDs` (array{string}, readonly, optional): Service UUIDs
- `Paired` (boolean, readonly): Pairing completed
- `Bonded` (boolean, readonly): Bonding info stored
- `Connected` (boolean, readonly): Connection state
- `Trusted` (boolean, readwrite): Trust setting
- `Blocked` (boolean, readwrite): Block setting
- `RSSI` (int16, readonly, optional): Signal strength
- `ServicesResolved` (boolean, readonly): Services discovered

**Signals:**
- `Disconnected(string reason, string message)`: Device disconnected

#### 3.3.3 org.bluez.GattManager1

**Purpose**: Manages GATT service registration

**Methods:**
- `RegisterApplication(object application, dict options)`: Register GATT application
- `UnregisterApplication(object application)`: Unregister GATT application

#### 3.3.4 org.bluez.GattService1

**Purpose**: Represents a GATT service

**Properties:**
- `UUID` (string, readonly): Service UUID
- `Primary` (boolean, readonly): Primary service flag
- `Includes` (array{object}, readonly, optional): Included services

#### 3.3.5 org.bluez.GattCharacteristic1

**Purpose**: Represents a GATT characteristic

**Methods:**
- `ReadValue(dict options)`: Read characteristic value
- `WriteValue(array{byte} value, dict options)`: Write characteristic value
- `StartNotify()`: Enable notifications
- `StopNotify()`: Disable notifications

**Properties:**
- `UUID` (string, readonly): Characteristic UUID
- `Service` (object, readonly): Parent service
- `Value` (array{byte}, readonly, optional): Current value
- `Flags` (array{string}, readonly): Properties (read, write, notify, etc.)
- `Notifying` (boolean, readonly, optional): Notification state

#### 3.3.6 org.bluez.LEAdvertisingManager1

**Purpose**: Manages LE advertisements

**Methods:**
- `RegisterAdvertisement(object advertisement, dict options)`: Register advertisement
- `UnregisterAdvertisement(object advertisement)`: Unregister advertisement

**Properties:**
- `ActiveInstances` (byte, readonly): Number of active advertisements
- `SupportedInstances` (byte, readonly): Maximum supported advertisements

#### 3.3.7 org.bluez.Media1

**Purpose**: Media endpoint management

**Methods:**
- `RegisterEndpoint(object endpoint, dict properties)`: Register media endpoint
- `UnregisterEndpoint(object endpoint)`: Unregister media endpoint
- `RegisterPlayer(object player, dict properties)`: Register media player
- `UnregisterPlayer(object player)`: Unregister media player

#### 3.3.8 org.bluez.AgentManager1

**Purpose**: Manages pairing agents

**Methods:**
- `RegisterAgent(object agent, string capability)`: Register pairing agent
- `UnregisterAgent(object agent)`: Unregister agent
- `RequestDefaultAgent(object agent)`: Set default agent

**Agent Capabilities:**
- `DisplayOnly`: Can only display
- `DisplayYesNo`: Display + yes/no input
- `KeyboardOnly`: Keyboard input only
- `NoInputNoOutput`: No I/O
- `KeyboardDisplay`: Full I/O

### 3.4 Signal and Property Change Notifications

BlueZ uses D-Bus signals for asynchronous notifications:
- `PropertiesChanged`: Property value changes
- `InterfacesAdded`: New objects created
- `InterfacesRemoved`: Objects removed

Applications use `org.freedesktop.DBus.Properties.PropertiesChanged` to monitor state changes.

---

## 4. State Machines

### 4.1 Adapter Power State Machine

**States:**
```
OFF ←→ OFF_ENABLING ←→ ON ←→ ON_DISABLING ←→ OFF
 ↓
OFF_BLOCKED (rfkill)
```

**State Descriptions:**
- **OFF**: Adapter is powered off
- **OFF_ENABLING**: Transitioning from off to on
- **ON**: Adapter is powered on and operational
- **ON_DISABLING**: Transitioning from on to off
- **OFF_BLOCKED**: Adapter is blocked by rfkill

**Transitions:**
- User sets `Powered` property → triggers state transition
- Kernel sends `new_settings` event → confirms state change
- rfkill event → forces to OFF_BLOCKED

**Implementation**: `src/adapter.c` - `adapter_set_power_state()`

### 4.2 Device Connection State Machine

**States (per bearer):**
```
DISCONNECTED ←→ CONNECTING ←→ CONNECTED ←→ DISCONNECTING ←→ DISCONNECTED
```

**State Tracking:**
Each device maintains separate connection states for BR/EDR and LE bearers in `struct bearer_state`.

**Transitions:**
- `Connect()` method → DISCONNECTED → CONNECTING
- Connection complete event → CONNECTING → CONNECTED
- `Disconnect()` method → CONNECTED → DISCONNECTING
- Disconnection complete → DISCONNECTING → DISCONNECTED
- Connection failure → CONNECTING → DISCONNECTED

**Auto-Connect:**
Devices with `auto_connect` flag automatically transition to CONNECTING when discovered.

### 4.3 Pairing/Bonding State Machine

**Pairing Process:**
```
IDLE → PAIRING_REQUESTED → AUTHENTICATING → PAIRED → BONDING → BONDED
                                ↓
                            PAIRING_FAILED
```

**States:**
1. **IDLE**: No pairing in progress
2. **PAIRING_REQUESTED**: User initiated pairing
3. **AUTHENTICATING**: Exchanging authentication data
   - PIN Code entry
   - Passkey entry/display
   - Just Works confirmation
   - OOB data exchange
4. **PAIRED**: Pairing successful, link encrypted
5. **BONDING**: Storing bonding information
6. **BONDED**: Bonding information persisted
7. **PAIRING_FAILED**: Pairing failed (timeout, rejection, etc.)

**Authentication Methods:**
- **Legacy Pairing** (pre-2.1): PIN code
- **Secure Simple Pairing** (2.1+):
  - Numeric Comparison
  - Just Works
  - Passkey Entry
  - Out of Band (OOB)
- **LE Pairing**:
  - Just Works
  - Passkey Entry
  - Numeric Comparison
  - OOB

**Implementation**: `src/device.c` - bonding_req structure and related functions

### 4.4 Service Discovery State Machine

**BR/EDR (SDP) Discovery:**
```
IDLE → SDP_SEARCH → SDP_COMPLETE → SERVICES_RESOLVED
         ↓
    SDP_FAILED
```

**LE (GATT) Discovery:**
```
IDLE → GATT_DISCOVERY → GATT_COMPLETE → SERVICES_RESOLVED
          ↓
    GATT_FAILED
```

**Service Resolution:**
1. Connection established
2. Initiate service discovery (SDP or GATT)
3. Parse discovered services
4. Match services to profiles
5. Probe matching profiles
6. Set `ServicesResolved` property
7. Auto-connect enabled profiles

**Caching:**
- **GATT**: Services cached in `gatt_db`, invalidated by Service Changed indication
- **SDP**: Records cached in device storage

---

## 5. Control and Data Flows

### 5.1 Initialization Flow

**Daemon Startup Sequence:**

1. **Parse Command Line Arguments**
   - Debug options
   - Configuration file path
   - Plugin directory

2. **Load Configuration** (`load_config()`)
   - Parse `/etc/bluetooth/main.conf`
   - Set global options in `btd_opts`

3. **Initialize D-Bus** (`dbus_conn_init()`)
   - Connect to system bus
   - Request `org.bluez` service name

4. **Initialize MGMT Interface** (`mgmt_new_default()`)
   - Open `/dev/bluetooth/mgmt` socket
   - Register for management events

5. **Initialize Subsystems**
   - `plugin_init()`: Load plugins
   - `adapter_init()`: Initialize adapter subsystem
   - `device_init()`: Initialize device subsystem
   - `profile_init()`: Initialize profile subsystem
   - `btd_gatt_database_init()`: Initialize GATT database

6. **Register D-Bus Interfaces**
   - Register object manager
   - Register adapter interfaces
   - Register agent manager

7. **Enumerate Adapters** (`mgmt_send(MGMT_OP_READ_INDEX_LIST)`)
   - Query kernel for available adapters
   - Create `btd_adapter` for each

8. **Enter Main Loop** (`g_main_loop_run()`)
   - Process events indefinitely

### 5.2 Device Discovery Flow

**Discovery Initiation:**

1. **Application calls `StartDiscovery()`** via D-Bus
2. **BlueZ creates discovery session**
   - Allocate `discovery_client` structure
   - Add to adapter's `discovery_list`
3. **Merge discovery filters** from all active sessions
4. **Send discovery command to kernel**
   - BR/EDR: `MGMT_OP_START_DISCOVERY`
   - LE: `MGMT_OP_START_SERVICE_DISCOVERY` with filters
5. **Kernel starts scanning/inquiry**
6. **Set `Discovering` property to true**

**Device Found:**

1. **Kernel sends `MGMT_EV_DEVICE_FOUND` event**
2. **BlueZ processes event** (`btd_adapter_device_found()`)
3. **Check if device already exists**
   - If yes: Update RSSI, manufacturer data
   - If no: Create new `btd_device`
4. **Apply discovery filters**
   - UUID matching
   - RSSI threshold
   - Pathloss calculation
5. **Emit `InterfacesAdded` signal** for new device
6. **Emit `PropertiesChanged` for updated device**
7. **Trigger auto-connect** if configured

**Discovery Stop:**

1. **Application calls `StopDiscovery()`**
2. **Remove discovery session** from list
3. **If no more sessions**, send `MGMT_OP_STOP_DISCOVERY` to kernel
4. **Set `Discovering` property to false**

### 5.3 Connection Establishment Flow

**Outgoing Connection (User-Initiated):**

1. **Application calls `Connect()` on Device**
2. **BlueZ determines bearer to use**
   - Check `PreferredBearer` setting
   - Check bonding status per bearer
   - Check last seen/used timestamps
3. **Initiate connection**
   - BR/EDR: `MGMT_OP_CONNECT`
   - LE: `MGMT_OP_ADD_DEVICE` + kernel auto-connect
4. **Wait for connection event**
5. **On `MGMT_EV_DEVICE_CONNECTED`**:
   - Update bearer state to connected
   - Set `Connected` property
   - Emit `PropertiesChanged` signal
6. **Initiate service discovery**
   - BR/EDR: SDP search
   - LE: GATT service discovery
7. **Probe matching profiles**
8. **Connect auto-connect profiles**
9. **Reply to D-Bus method call**

**Incoming Connection:**

1. **Kernel sends `MGMT_EV_DEVICE_CONNECTED`**
2. **Find or create `btd_device`**
3. **Update connection state**
4. **Emit `PropertiesChanged` signal**
5. **Initiate service discovery**
6. **Probe and connect profiles**

### 5.4 Pairing Flow

**Pairing Initiation:**

1. **Application calls `Pair()` on Device**
2. **BlueZ creates bonding request** (`bonding_req`)
3. **Initiate pairing**
   - BR/EDR: `MGMT_OP_PAIR_DEVICE`
   - LE: `MGMT_OP_PAIR_DEVICE` with LE address type
4. **Kernel initiates pairing procedure**

**Authentication:**

1. **Kernel requests authentication** (`MGMT_EV_USER_CONFIRM_REQUEST`, etc.)
2. **BlueZ determines agent to use**
   - Application-specific agent
   - Default agent
3. **Call agent method** based on request type:
   - `RequestPinCode()`: PIN entry
   - `RequestPasskey()`: Passkey entry
   - `DisplayPasskey()`: Display passkey
   - `RequestConfirmation()`: Numeric comparison
4. **User provides input**
5. **BlueZ sends response to kernel**
   - `MGMT_OP_USER_CONFIRM_REPLY`
   - `MGMT_OP_USER_PASSKEY_REPLY`
   - `MGMT_OP_PIN_CODE_REPLY`

**Pairing Complete:**

1. **Kernel sends `MGMT_EV_NEW_LINK_KEY` or `MGMT_EV_NEW_LONG_TERM_KEY`**
2. **BlueZ stores bonding information**
   - Link keys in `/var/lib/bluetooth/<adapter>/<device>/info`
3. **Set `Paired` and `Bonded` properties**
4. **Emit `PropertiesChanged` signal**
5. **Continue with service discovery**
6. **Reply to `Pair()` method call**

### 5.5 Data Transfer Paths

**GATT Read/Write:**

1. **Application calls `ReadValue()` or `WriteValue()`**
2. **BlueZ forwards to `bt_gatt_client`**
3. **GATT client sends ATT request** over L2CAP
4. **Kernel forwards to controller**
5. **Response received**
6. **BlueZ returns value to application**

**GATT Notifications:**

1. **Application calls `StartNotify()`**
2. **BlueZ registers notification handler**
3. **Send ATT Write Request** to enable notifications (CCCD)
4. **On notification received**:
   - Update cached value
   - Emit `PropertiesChanged` signal

**RFCOMM Data:**

1. **Profile opens RFCOMM socket** (`bt_io_connect()`)
2. **Data written to socket**
3. **Kernel RFCOMM layer** handles segmentation
4. **L2CAP transmission**
5. **Data received** via socket read

**A2DP Audio Streaming:**

1. **Audio profile negotiates codec**
2. **Opens L2CAP media channel**
3. **Audio data encoded** (SBC, AAC, etc.)
4. **Transmitted over L2CAP**
5. **Kernel handles flow control**

### 5.6 Event Loop Architecture

**GMainLoop Integration:**

BlueZ uses GLib's main loop for event-driven operation:

```c
// Main loop setup
GMainLoop *main_loop = g_main_loop_new(NULL, FALSE);

// File descriptor monitoring
GIOChannel *io = g_io_channel_unix_new(fd);
g_io_add_watch(io, G_IO_IN, io_callback, user_data);

// Timers
g_timeout_add_seconds(timeout, timer_callback, user_data);

// Idle callbacks
g_idle_add(idle_callback, user_data);

// D-Bus integration
dbus_connection_setup_with_g_main(connection, NULL);

// Run loop
g_main_loop_run(main_loop);
```

**Event Sources:**
- **MGMT socket**: Adapter management events
- **HCI sockets**: Device communication
- **D-Bus**: Application requests
- **Timers**: Discovery timeouts, temporary device cleanup
- **Signals**: SIGTERM, SIGINT for graceful shutdown

---

## 6. Data Structures

### 6.1 Core Structures

**`struct btd_adapter`** (src/adapter.c):
- Represents a Bluetooth controller
- Tracks power state, settings, devices
- Manages discovery sessions
- Owns GATT database and advertisement manager

**`struct btd_device`** (src/device.c):
- Represents a remote Bluetooth device
- Maintains per-bearer connection state
- Stores service UUIDs and GATT database
- Handles pairing and bonding

**`struct btd_service`** (src/service.c):
- Represents a service instance on a device
- Links device to profile
- Tracks connection state

**`struct btd_profile`** (src/profile.h):
- Defines a Bluetooth profile
- Provides callbacks for device probe/remove
- Specifies UUIDs and bearer type

### 6.2 GATT Structures

**`struct gatt_db`** (src/shared/gatt-db.h):
- Generic GATT database structure
- Tree of services, characteristics, descriptors
- Supports attribute caching

**`struct bt_gatt_client`** (src/shared/gatt-client.h):
- GATT client implementation
- Handles service discovery
- Manages read/write operations
- Notification/indication handling

**`struct bt_gatt_server`** (src/shared/gatt-server.h):
- GATT server implementation
- Processes incoming ATT requests
- Manages authorization and security

**`struct bt_att`** (src/shared/att.h):
- ATT protocol layer
- Request/response matching
- MTU negotiation
- Error handling

### 6.3 Connection Structures

**`struct bearer_state`** (src/device.c):
```c
struct bearer_state {
    bool paired;
    bool bonded;
    bool connected;
    bool svc_resolved;
    bool initiator;
    bool connectable;
    time_t last_seen;
    time_t last_used;
};
```

**`struct bonding_req`** (src/device.c):
- Tracks ongoing pairing operation
- References agent for user interaction
- Retry timer for failed attempts

**`struct authentication_req`** (src/device.c):
- Represents authentication request
- Type: PIN, passkey, confirmation
- Agent reference

### 6.4 Storage Structures

**GKeyFile Format:**
BlueZ uses GLib's GKeyFile for persistent storage:

```ini
[General]
Name=Device Name
Alias=My Device
Class=0x240404
Trusted=true
Blocked=false
Services=0000110a-0000-1000-8000-00805f9b34fb;0000110b-...

[LinkKey]
Key=0123456789ABCDEF0123456789ABCDEF
Type=4
PINLength=0

[LongTermKey]
Key=0123456789ABCDEF0123456789ABCDEF
Authenticated=true
EncSize=16
EDiv=12345
Rand=9876543210
```

---

## 7. Algorithms and Design Decisions

### 7.1 Device Database Management

**Device Lookup:**
- Devices stored in `adapter->devices` (GSList)
- Lookup by address: O(n) linear search
- Lookup by D-Bus path: O(n) linear search

**Rationale**: Small number of devices per adapter (typically < 100) makes linear search acceptable. Hash table overhead not justified.

**Address Type Handling:**
- BR/EDR: `BDADDR_BREDR`
- LE Public: `BDADDR_LE_PUBLIC`
- LE Random: `BDADDR_LE_RANDOM`
  - Static Random: Stored and persisted
  - Private Resolvable: Resolved using IRK
  - Private Non-Resolvable: Not stored

**Duplicate Device Merging:**
When a device is discovered on both BR/EDR and LE:
1. Check if device with same address exists
2. If yes, update bearer information
3. Merge UUIDs from both bearers
4. Prefer BR/EDR for name resolution

### 7.2 Connection Management

**Auto-Connect List:**
- Maintained in kernel via `MGMT_OP_ADD_DEVICE`
- Kernel automatically connects when device advertises
- BlueZ adds devices with `auto_connect` flag

**Connection Priority:**
Profiles have priority levels:
- `BTD_PROFILE_PRIORITY_HIGH` (2): Critical profiles (HID)
- `BTD_PROFILE_PRIORITY_MEDIUM` (1): Standard profiles (A2DP)
- `BTD_PROFILE_PRIORITY_LOW` (0): Optional profiles

Profiles connected in priority order.

**Bearer Selection Logic:**
For dual-mode devices, bearer selection based on:
1. **Explicit preference**: `PreferredBearer` property
2. **Bonding status**: Prefer bonded bearer
3. **Last used**: Prefer recently used bearer
4. **Last seen**: Prefer recently seen bearer
5. **Default**: BR/EDR takes precedence

### 7.3 Pairing Algorithms

**Security Level Negotiation:**
- **Low**: No encryption
- **Medium**: Encryption, no MITM protection
- **High**: Encryption + MITM protection
- **FIPS**: FIPS-approved algorithms

**Just Works Re-Pairing:**
Configuration option `JustWorksRepairing`:
- `never`: Reject re-pairing with Just Works
- `confirm`: Require user confirmation
- `always`: Allow automatic re-pairing

**Rationale**: Prevents downgrade attacks where attacker forces Just Works pairing.

**Key Storage:**
- Link keys stored in `/var/lib/bluetooth/<adapter>/<device>/info`
- Keys encrypted at rest (optional, via kernel keyring)
- Blocked keys checked against known compromised keys

### 7.4 Service Discovery Optimization

**GATT Caching Strategy:**
Configuration option `Cache`:
- `always`: Always cache, never re-discover
- `yes`: Cache, re-discover on Service Changed
- `no`: Never cache, always re-discover

**Rationale**: Reduces connection time for known devices, but may miss service changes.

**Service Changed Handling:**
1. Device sends Service Changed indication
2. BlueZ invalidates affected services
3. Re-discovers affected range
4. Updates cached database
5. Reprobes affected profiles

**Incremental Discovery:**
For GATT, services discovered incrementally:
1. Discover primary services
2. Discover characteristics for each service
3. Discover descriptors for each characteristic
4. Read characteristic values as needed

### 7.5 Power Management

**Suspend/Resume Handling:**
1. **On suspend**:
   - Disconnect non-wake devices
   - Pause discovery
   - Save adapter state
2. **On resume**:
   - Restore adapter state
   - Reconnect devices
   - Resume discovery if active

**Wake-on-Bluetooth:**
- Devices with `WakeAllowed` property can wake system
- Requires profile support (e.g., HID)
- Kernel filters wake events

**Low Power Mode:**
- Adjust scan intervals during idle
- Use passive scanning when possible
- Reduce advertisement frequency

---

## 8. Architecture Diagrams

### 8.1 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │bluetooth-│  │  Audio   │  │   File   │  │ Custom  │ │
│  │   ctl    │  │  Player  │  │ Manager  │  │  Apps   │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
└─────────────────────────────────────────────────────────┘
                        ↕ D-Bus (IPC)
┌─────────────────────────────────────────────────────────┐
│                   BlueZ Daemon (bluetoothd)              │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Core Components                        │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │ │
│  │  │ Adapter  │  │  Device  │  │  GATT Database   │ │ │
│  │  │ Manager  │  │ Manager  │  │                  │ │ │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │ │
│  │  │  Agent   │  │ Profile  │  │  Advertisement   │ │ │
│  │  │ Manager  │  │ Manager  │  │    Manager       │ │ │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Profile Implementations                │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐ │ │
│  │  │ A2DP │ │ HFP  │ │ HID  │ │ GATT │ │ Network  │ │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Shared Libraries                       │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │ │
│  │  │   ATT    │ │   GATT   │ │  Crypto  │           │ │
│  │  └──────────┘ └──────────┘ └──────────┘           │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                    ↕ MGMT / HCI Sockets
┌─────────────────────────────────────────────────────────┐
│                  Linux Kernel                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │        Bluetooth Subsystem (net/bluetooth/)         │ │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐ │ │
│  │  │ MGMT │ │ HCI  │ │L2CAP │ │RFCOMM│ │   BNEP   │ │ │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────────┘ │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                    ↕ USB / UART / SDIO
┌─────────────────────────────────────────────────────────┐
│            Bluetooth Hardware Controller                 │
└─────────────────────────────────────────────────────────┘
```

### 8.2 Component Interaction Diagram

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│              │         │              │         │              │
│  Application │◄───────►│   D-Bus      │◄───────►│  bluetoothd  │
│              │  IPC    │   Daemon     │  IPC    │              │
└──────────────┘         └──────────────┘         └──────┬───────┘
                                                          │
                                                          │ MGMT
                                                          │
                                                   ┌──────▼───────┐
                                                   │              │
                                                   │    Kernel    │
                                                   │  Bluetooth   │
                                                   │  Subsystem   │
                                                   │              │
                                                   └──────┬───────┘
                                                          │
                                                          │ HCI
                                                          │
                                                   ┌──────▼───────┐
                                                   │              │
                                                   │  Controller  │
                                                   │              │
                                                   └──────────────┘
```

### 8.3 Adapter State Machine Diagram

```
                    ┌──────────────┐
                    │     OFF      │
                    │   BLOCKED    │
                    └──────────────┘
                           ▲
                           │ rfkill
                           │
    ┌──────────────────────┴──────────────────────┐
    │                                              │
    │                                              │
┌───▼──────┐  Power On   ┌──────────────┐  Power On  ┌──────────┐
│   OFF    │────────────►│     OFF      │───────────►│    ON    │
│          │             │  ENABLING    │            │          │
└───┬──────┘             └──────────────┘            └───┬──────┘
    │                                                     │
    │                                                     │
    │                    ┌──────────────┐                │
    └────────────────────│     ON       │◄───────────────┘
         Power Off       │  DISABLING   │   Power Off
                         └──────────────┘
```

### 8.4 Device Connection Sequence Diagram

```
Application    bluetoothd    Kernel    Controller    Remote Device
    │              │           │            │              │
    │  Connect()   │           │            │              │
    ├─────────────►│           │            │              │
    │              │  MGMT_OP_ │            │              │
    │              │  CONNECT  │            │              │
    │              ├──────────►│            │              │
    │              │           │ HCI_Create │              │
    │              │           │ Connection │              │
    │              │           ├───────────►│              │
    │              │           │            │  Connection  │
    │              │           │            │   Request    │
    │              │           │            ├─────────────►│
    │              │           │            │              │
    │              │           │            │  Connection  │
    │              │           │            │   Complete   │
    │              │           │            │◄─────────────┤
    │              │           │ HCI_Conn   │              │
    │              │           │ Complete   │              │
    │              │           │◄───────────┤              │
    │              │  MGMT_EV_ │            │              │
    │              │  DEVICE_  │            │              │
    │              │  CONNECTED│            │              │
    │              │◄──────────┤            │              │
    │              │           │            │              │
    │              │ Update    │            │              │
    │              │ State     │            │              │
    │              │           │            │              │
    │  Connected   │           │            │              │
    │  Signal      │           │            │              │
    │◄─────────────┤           │            │              │
    │              │           │            │              │
    │  Reply       │           │            │              │
    │◄─────────────┤           │            │              │
    │              │           │            │              │
```

### 8.5 GATT Operation Sequence Diagram

```
Application    bluetoothd    bt_gatt_client    ATT    L2CAP    Remote
    │              │              │              │       │         │
    │ ReadValue()  │              │              │       │         │
    ├─────────────►│              │              │       │         │
    │              │ bt_gatt_     │              │       │         │
    │              │ client_read  │              │       │         │
    │              ├─────────────►│              │       │         │
    │              │              │ ATT Read Req │       │         │
    │              │              ├─────────────►│       │         │
    │              │              │              │ Send  │         │
    │              │              │              ├──────►│         │
    │              │              │              │       │ ATT Req │
    │              │              │              │       ├────────►│
    │              │              │              │       │         │
    │              │              │              │       │ ATT Rsp │
    │              │              │              │       │◄────────┤
    │              │              │              │ Recv  │         │
    │              │              │              │◄──────┤         │
    │              │              │ ATT Read Rsp │       │         │
    │              │              │◄─────────────┤       │         │
    │              │ Callback     │              │       │         │
    │              │◄─────────────┤              │       │         │
    │  Value       │              │              │       │         │
    │◄─────────────┤              │              │       │         │
    │              │              │              │       │         │
```

---

## Conclusion

This document provides a comprehensive overview of the BlueZ architecture, covering its high-level design, core components, interfaces, state machines, control flows, data structures, and key algorithms. BlueZ's modular design, event-driven architecture, and extensive D-Bus API make it a flexible and powerful Bluetooth stack for Linux systems.

**Key Architectural Strengths:**
- **Modular Design**: Clear separation between core daemon, profiles, and shared libraries
- **Event-Driven**: GMainLoop-based architecture for efficient resource usage
- **Extensible**: Plugin system and profile registration for easy extension
- **Standards-Compliant**: Comprehensive support for Bluetooth specifications
- **D-Bus Integration**: Well-defined IPC mechanism for application integration

**Design Trade-offs:**
- **Linear Search**: Device lookup is O(n), acceptable for typical use cases
- **GATT Caching**: Balances performance vs. service change detection
- **Bearer Selection**: Complex logic to handle dual-mode devices optimally
- **Storage Format**: GKeyFile is human-readable but not optimized for large datasets

This architecture has evolved over many years to support the growing complexity of Bluetooth specifications while maintaining backward compatibility and system integration.


