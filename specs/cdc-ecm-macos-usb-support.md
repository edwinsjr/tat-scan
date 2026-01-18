# CDC-ECM Implementation Specification for macOS USB Support

**Version**: 1.1
**Date**: January 2026
**Status**: Draft (Updated with research findings)
**Author**: Engineering

---

## 1. Executive Summary

> **IMPORTANT**: This specification has been updated based on research into CDC-ECM implementations.
> Key finding: The [D21ecm project](https://github.com/majbthrd/D21ecm) is a verified working implementation
> on macOS High Sierra. A related project (stm32ecm) does NOT work on macOS 10.11+, demonstrating that
> implementation details significantly affect macOS compatibility.

This specification describes the implementation of CDC-ECM (Communications Device Class - Ethernet Control Model) USB networking support for the Coral Dev Board Micro firmware. This addresses the current limitation where macOS cannot communicate with the device over USB because macOS does not support the existing CDC-EEM protocol.

### 1.1 Problem Statement

The Coral Dev Board Micro currently uses CDC-EEM (Ethernet Emulation Model) for USB networking, exposing a virtual network interface at IP address `10.10.10.1`. While this works on Linux, **macOS has deliberately disabled CDC-EEM support** in its kernel, making USB-based camera access impossible on Mac systems.

### 1.2 Proposed Solution

Implement CDC-ECM as an alternative USB networking protocol. CDC-ECM is natively supported by macOS, Linux, and (with limitations) Windows, providing cross-platform USB networking capability.

---

## 2. Background

### 2.1 Current Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Coral Dev Board Micro                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Application Layer (Camera Streaming, RPC Server)        │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  LwIP TCP/IP Stack                                       │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  CDC-EEM Network Interface (netif)                       │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  USB Device Stack (NXP RT1176 SDK)                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │ USB                                 │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │     Host Computer       │
              │  ┌───────────────────┐  │
              │  │ CDC-EEM Driver    │  │ ← Works on Linux
              │  │ (Not on macOS)    │  │ ← FAILS on macOS
              │  └───────────────────┘  │
              └─────────────────────────┘
```

### 2.2 Why CDC-EEM Fails on macOS

Apple's macOS kernel includes a CDC-EEM driver (`IOUSBCDCEthernetCommDevice.kext`), but it is **deliberately disabled**. According to Apple engineering comments in the driver source:

> "The kext loads but is deliberately disabled as we've never seen a valid EEM device."

This is a policy decision by Apple, not a technical limitation. The driver exists but refuses to attach to any CDC-EEM device.

### 2.3 CDC Protocol Comparison

| Feature | CDC-EEM | CDC-ECM | CDC-NCM | RNDIS |
|---------|---------|---------|---------|-------|
| USB Class | 0x02 | 0x02 | 0x02 | 0x02 |
| USB Subclass | 0x0C | 0x06 | 0x0D | 0x02 |
| macOS Support | ❌ Disabled | ✅ Native | ✅ Native | ❌ None |
| Linux Support | ✅ Native | ✅ Native | ✅ Native | ✅ Native |
| Windows Support | ⚠️ Limited | ⚠️ Driver needed | ✅ Win 11+ | ✅ Native |
| Packet Format | EEM Header | Raw Ethernet | NCM Header | RNDIS Msg |
| Complexity | Medium | Low | Medium | High |
| Throughput | ~26 MB/s | ~11 MB/s | ~26 MB/s | ~11 MB/s |
| Endpoints Required | 2 | 3 | 3 | 3 |
| Multi-frame/transfer | Yes | No | Yes | No |

**Recommendation**: CDC-ECM provides the best cross-platform compatibility with the simplest implementation.

> **Research Note**: Per [Belcarra's protocol comparison](https://usblan.belcarra.com/2011/02/cdc-eem-vs-cdc-ecm-protocols.html),
> CDC-EEM achieves higher throughput through multi-frame bundling, but CDC-ECM's ~11 MB/s (88 Mbps) is
> sufficient for camera streaming at 1080p30 (~5-15 Mbps typical).

### 2.4 NXP RT1176 SDK Limitations

> **WARNING**: The NXP RT1176 SDK does **NOT** include a CDC-ECM implementation.
> Per [NXP Community discussions](https://community.nxp.com/t5/i-MX-RT-Crossover-MCUs/USB-CDC-ECM-Support/m-p/808594/1000):
> "We haven't provided an ECM demo and can't guarantee third-party implementations will be fully compatible."

This means CDC-ECM must be implemented from scratch using:
- [NXP Application Note AN14169](https://www.nxp.com/docs/en/application-note/AN14169.pdf) for custom USB device classes
- The [D21ecm project](https://github.com/majbthrd/D21ecm) as a reference implementation
- The existing `usb_device_cdc_vcom` demo as a starting point

---

## 3. Technical Specification

### 3.1 USB Descriptors

#### 3.1.1 Interface Association Descriptor (IAD)

CDC-ECM requires two interfaces (Control + Data), so an IAD is needed for composite device support.

```c
typedef struct {
    uint8_t bLength;            // 8
    uint8_t bDescriptorType;    // 0x0B (IAD)
    uint8_t bFirstInterface;    // First interface number
    uint8_t bInterfaceCount;    // 2 (Control + Data)
    uint8_t bFunctionClass;     // 0x02 (CDC)
    uint8_t bFunctionSubClass;  // 0x06 (ECM)
    uint8_t bFunctionProtocol;  // 0x00
    uint8_t iFunction;          // String index
} usb_iad_descriptor_t;
```

#### 3.1.2 Control Interface Descriptor

```c
typedef struct {
    uint8_t bLength;            // 9
    uint8_t bDescriptorType;    // 0x04 (Interface)
    uint8_t bInterfaceNumber;   // Control interface number
    uint8_t bAlternateSetting;  // 0
    uint8_t bNumEndpoints;      // 1 (Notification endpoint)
    uint8_t bInterfaceClass;    // 0x02 (CDC)
    uint8_t bInterfaceSubClass; // 0x06 (ECM)
    uint8_t bInterfaceProtocol; // 0x00
    uint8_t iInterface;         // String index
} usb_control_interface_descriptor_t;
```

#### 3.1.3 CDC Functional Descriptors

**Header Functional Descriptor**:
```c
typedef struct {
    uint8_t bFunctionLength;    // 5
    uint8_t bDescriptorType;    // 0x24 (CS_INTERFACE)
    uint8_t bDescriptorSubtype; // 0x00 (Header)
    uint16_t bcdCDC;            // 0x0120 (CDC 1.2)
} cdc_header_functional_descriptor_t;
```

**Union Functional Descriptor**:
```c
typedef struct {
    uint8_t bFunctionLength;    // 5
    uint8_t bDescriptorType;    // 0x24 (CS_INTERFACE)
    uint8_t bDescriptorSubtype; // 0x06 (Union)
    uint8_t bControlInterface;  // Control interface number
    uint8_t bSubordinateInterface0; // Data interface number
} cdc_union_functional_descriptor_t;
```

**Ethernet Networking Functional Descriptor** (ECM-specific):
```c
typedef struct {
    uint8_t bFunctionLength;    // 13
    uint8_t bDescriptorType;    // 0x24 (CS_INTERFACE)
    uint8_t bDescriptorSubtype; // 0x0F (Ethernet Networking)
    uint8_t iMACAddress;        // String index for MAC address
    uint32_t bmEthernetStatistics; // Supported statistics bitmap
    uint16_t wMaxSegmentSize;   // 1514 (standard Ethernet MTU)
    uint16_t wNumberMCFilters;  // 0 (no multicast filtering)
    uint8_t bNumberPowerFilters; // 0 (no power filtering)
} cdc_ethernet_functional_descriptor_t;
```

#### 3.1.4 Notification Endpoint Descriptor

```c
typedef struct {
    uint8_t bLength;            // 7
    uint8_t bDescriptorType;    // 0x05 (Endpoint)
    uint8_t bEndpointAddress;   // 0x8X (IN)
    uint8_t bmAttributes;       // 0x03 (Interrupt)
    uint16_t wMaxPacketSize;    // 16
    uint8_t bInterval;          // 32 (polling interval)
} usb_notification_endpoint_descriptor_t;
```

#### 3.1.5 Data Interface Descriptor

> **CRITICAL for macOS 10.15 Compatibility**: The data interface `bInterfaceSubClass` MUST be `0x00`.
> Some implementations use `0x06`, which causes detection failures on macOS Catalina (10.15).
> See [ev3dev issue #471](https://github.com/ev3dev/ev3dev/issues/471) for details.

**Alternate Setting 0** (No endpoints - idle state):
```c
typedef struct {
    uint8_t bLength;            // 9
    uint8_t bDescriptorType;    // 0x04 (Interface)
    uint8_t bInterfaceNumber;   // Data interface number
    uint8_t bAlternateSetting;  // 0
    uint8_t bNumEndpoints;      // 0 (no endpoints in alt 0)
    uint8_t bInterfaceClass;    // 0x0A (CDC Data)
    uint8_t bInterfaceSubClass; // 0x00  // MUST be 0x00 for macOS 10.15!
    uint8_t bInterfaceProtocol; // 0x00
    uint8_t iInterface;         // String index
} usb_data_interface_alt0_descriptor_t;
```

**Alternate Setting 1** (Active with endpoints):
```c
typedef struct {
    uint8_t bLength;            // 9
    uint8_t bDescriptorType;    // 0x04 (Interface)
    uint8_t bInterfaceNumber;   // Data interface number
    uint8_t bAlternateSetting;  // 1
    uint8_t bNumEndpoints;      // 2 (Bulk IN + Bulk OUT)
    uint8_t bInterfaceClass;    // 0x0A (CDC Data)
    uint8_t bInterfaceSubClass; // 0x00  // MUST be 0x00 for macOS 10.15!
    uint8_t bInterfaceProtocol; // 0x00
    uint8_t iInterface;         // String index
} usb_data_interface_alt1_descriptor_t;
```

> **Implementation Note**: macOS switches between Alt 0 and Alt 1 during suspend/resume cycles.
> The firmware MUST handle `SET_INTERFACE` requests properly for both alternate settings.

#### 3.1.6 Bulk Endpoint Descriptors

```c
// Bulk IN Endpoint
typedef struct {
    uint8_t bLength;            // 7
    uint8_t bDescriptorType;    // 0x05 (Endpoint)
    uint8_t bEndpointAddress;   // 0x8X (IN)
    uint8_t bmAttributes;       // 0x02 (Bulk)
    uint16_t wMaxPacketSize;    // 512 (High-Speed)
    uint8_t bInterval;          // 0
} usb_bulk_in_endpoint_descriptor_t;

// Bulk OUT Endpoint
typedef struct {
    uint8_t bLength;            // 7
    uint8_t bDescriptorType;    // 0x05 (Endpoint)
    uint8_t bEndpointAddress;   // 0x0X (OUT)
    uint8_t bmAttributes;       // 0x02 (Bulk)
    uint16_t wMaxPacketSize;    // 512 (High-Speed)
    uint8_t bInterval;          // 0
} usb_bulk_out_endpoint_descriptor_t;
```

### 3.2 Notification Messages

CDC-ECM uses the notification endpoint to inform the host of network state changes.

> **🚨 CRITICAL FOR macOS**: NetworkConnection notification is **MANDATORY** for macOS/iOS/iPadOS.
> Per [Apple Developer Forums](https://developer.apple.com/forums/thread/769793) and
> [Zephyr issue #69209](https://github.com/zephyrproject-rtos/zephyr/issues/69209):
>
> "Platforms such as macOS, iOS, and iPadOS will NOT perform DHCP negotiation with autoconf fallback
> without the interface being 'connected'. The ECM device MUST support NetworkConnection notification."
>
> **Without this notification:**
> - macOS will show the interface as "Not Connected" in Network Preferences
> - DHCP will NOT start
> - No IP address will be assigned
> - The device will be unusable

#### 3.2.1 Network Connection Notification

Sent when the network link state changes. **MUST be sent immediately when the interface is set to alternate setting 1.**

```c
typedef struct {
    uint8_t bmRequestType;      // 0xA1 (Device-to-host, Class, Interface)
    uint8_t bNotificationCode;  // 0x00 (NETWORK_CONNECTION)
    uint16_t wValue;            // 0 = Disconnected, 1 = Connected
    uint16_t wIndex;            // Interface number (Control interface)
    uint16_t wLength;           // 0
} cdc_network_connection_notification_t;
```

**Reference implementation from [D21ecm](https://github.com/majbthrd/D21ecm) (`usb/usb_ecm.c`):**
```c
static alignas(4) usb_request_t notify_nc =
{
  .bmRequestType = 0xA1,
  .bRequest = 0,           // NETWORK_CONNECTION
  .wValue = 1,             // Connected
  .wIndex = USB_ECM_NOTIFY_ITF,
  .wLength = 0,
};

// Send via ecm_report(true) when interface activates
```

#### 3.2.2 Connection Speed Change Notification

Sent to indicate network speed capabilities. Should be sent immediately after Network Connection notification.

```c
typedef struct {
    uint8_t bmRequestType;      // 0xA1
    uint8_t bNotificationCode;  // 0x2A (CONNECTION_SPEED_CHANGE)
    uint16_t wValue;            // 0
    uint16_t wIndex;            // Interface number
    uint16_t wLength;           // 8
    uint32_t DLBitRate;         // Downstream bit rate (e.g., 100000000)
    uint32_t ULBitRate;         // Upstream bit rate (e.g., 100000000)
} cdc_connection_speed_notification_t;
```

#### 3.2.3 Notification Timing Requirements

The following sequence MUST be followed when the host sets the data interface to alternate setting 1:

1. **Immediately** send `NETWORK_CONNECTION` notification with `wValue = 1` (Connected)
2. **After step 1 completes**, send `CONNECTION_SPEED_CHANGE` notification
3. Queue the first Bulk OUT receive to be ready for incoming packets

On USB disconnect or interface set to alternate setting 0:
1. Send `NETWORK_CONNECTION` notification with `wValue = 0` (Disconnected)
2. Reset internal network state

### 3.3 Data Transfer

Unlike CDC-EEM, CDC-ECM transfers **raw Ethernet frames** without any additional headers or trailers.

#### 3.3.1 Transmit Path (Device → Host)

```
┌─────────────────────────────────────────────────────────────────┐
│  LwIP pbuf (Ethernet Frame)                                     │
│  ┌────────────┬───────────────────────────────┬───────────────┐ │
│  │ Dst MAC (6)│ Src MAC (6) │ EthType (2)     │ Payload       │ │
│  └────────────┴───────────────────────────────┴───────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ USB_DeviceCdcEcmSend()
┌─────────────────────────────────────────────────────────────────┐
│  USB Bulk IN Transfer (Raw frame, no modification)              │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 Receive Path (Host → Device)

```
┌─────────────────────────────────────────────────────────────────┐
│  USB Bulk OUT Transfer (Raw Ethernet Frame)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ USB_DeviceCdcEcmRecv()
┌─────────────────────────────────────────────────────────────────┐
│  Pass directly to LwIP netif->input()                           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Packet Format Comparison

**CDC-EEM** (Current):
```
┌──────────────┬─────────────────────────┬──────────────┐
│ EEM Header   │ Ethernet Frame          │ CRC-32       │
│ (2 bytes)    │ (14-1500 bytes)         │ (4 bytes)    │
└──────────────┴─────────────────────────┴──────────────┘
```

**CDC-ECM** (Proposed):
```
┌─────────────────────────────────────────┐
│ Ethernet Frame (14-1514 bytes)          │
│ [No additional headers or trailers]     │
└─────────────────────────────────────────┘
```

This simplification means:
- No header parsing on receive
- No header construction on transmit
- No CRC calculation
- No endianness detection logic

### 3.5 Zero-Length Packet (ZLP) Handling

> **IMPORTANT**: USB Bulk transfers that are exact multiples of the endpoint max packet size (512 bytes
> for High-Speed USB) require a Zero-Length Packet (ZLP) to signal transfer completion.
> Many CDC implementations get this wrong, causing data to hang.
> See [STM32 CDC issue](https://community.st.com/t5/stm32-mcus-embedded-software/bug-cdc-does-not-transmit-packets-which-length-is-an-exact/td-p/257267).

```c
// After sending a Bulk IN transfer
if (transfer_length > 0 && (transfer_length % 512) == 0) {
    // MUST send a Zero-Length Packet to terminate the transfer
    USB_DeviceCdcEcmSend(handle, bulk_in_ep, NULL, 0);
}
```

This applies to transfers of exactly 512, 1024, 1536, 2048... bytes.

---

## 4. Implementation Design

### 4.1 File Structure

```
coralmicro/
├── libs/
│   ├── cdc_ecm/                           # NEW DIRECTORY
│   │   ├── CMakeLists.txt                 # Build configuration
│   │   ├── cdc_ecm.h                      # C++ class header
│   │   └── cdc_ecm.cc                     # C++ implementation
│   ├── nxp/
│   │   └── rt1176-sdk/
│   │       ├── usb_device_cdc_ecm.h       # NEW: Low-level driver header
│   │       └── usb_device_cdc_ecm.c       # NEW: Low-level driver impl
│   ├── usb/
│   │   ├── descriptors.h                  # MODIFY: Add ECM descriptors
│   │   └── usb_device_task.cc             # No changes needed
│   └── base/
│       └── main_freertos_m7.cc            # MODIFY: Add ECM init
└── third_party/
    ├── modified/
    │   └── nxp/
    │       └── rt1176-sdk/
    │           └── usb_device_config.h    # MODIFY: Enable ECM
    └── nxp/
        └── rt1176-sdk/
            └── middleware/usb/.../
                └── usb_device_class.h     # MODIFY: Add ECM class type
```

### 4.2 Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CdcEcm (C++ Class)                      │
├─────────────────────────────────────────────────────────────────┤
│ - bulk_in_ep_: uint8_t                                          │
│ - bulk_out_ep_: uint8_t                                         │
│ - notify_ep_: uint8_t                                           │
│ - class_handle_: class_handle_t                                 │
│ - netif_: struct netif                                          │
│ - netif_ipaddr_: ip4_addr_t                                     │
│ - tx_buffer_[1520]: uint8_t                                     │
│ - rx_buffer_[1520]: uint8_t                                     │
│ - tx_queue_: QueueHandle_t                                      │
│ - connected_: bool                                              │
├─────────────────────────────────────────────────────────────────┤
│ + Init(bulk_in, bulk_out, notify, ctrl_iface, data_iface): void │
│ + SetClassHandle(handle): void                                  │
│ + HandleEvent(event, param): bool                               │
│ + config_data(): usb_device_class_config_struct_t*              │
│ + descriptor_data(): void*                                      │
│ + descriptor_data_size(): size_t                                │
├─────────────────────────────────────────────────────────────────┤
│ - StaticNetifInit(netif): err_t                                 │
│ - NetifInit(netif): err_t                                       │
│ - StaticTxFunc(netif, pbuf): err_t                              │
│ - TxFunc(netif, pbuf): err_t                                    │
│ - TransmitFrame(buffer, length): err_t                          │
│ - ReceiveFrame(buffer, length): err_t                           │
│ - Handler(event, param): usb_status_t                           │
│ - SendNotification(type, value): usb_status_t                   │
│ - TaskFunction(param): void                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               USB Device CDC-ECM Driver (C API)                 │
├─────────────────────────────────────────────────────────────────┤
│ USB_DeviceCdcEcmInit(controllerId, config, handle)              │
│ USB_DeviceCdcEcmDeinit(handle)                                  │
│ USB_DeviceCdcEcmEvent(handle, event, param)                     │
│ USB_DeviceCdcEcmSend(handle, ep, buffer, length)                │
│ USB_DeviceCdcEcmRecv(handle, ep, buffer, length)                │
│ USB_DeviceCdcEcmNotify(handle, ep, notification, length)        │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Initialization Sequence

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   main()    │     │  CdcEcm     │     │ USB Task    │     │   LwIP      │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ InitializeCDCECM()│                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ Init()            │                   │
       │                   │───────────────────┼──────────────────>│
       │                   │                   │      netifapi_    │
       │                   │                   │      netif_add()  │
       │                   │                   │<──────────────────│
       │                   │                   │                   │
       │                   │ AddDevice()       │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ Register class    │
       │                   │                   │ with USB stack    │
       │                   │<──────────────────│                   │
       │                   │                   │                   │
       │                   │ start_dhcp_server │                   │
       │                   │───────────────────┼──────────────────>│
       │                   │                   │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
```

### 4.4 Data Flow: Host → Device

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Host     │     │  USB HW     │     │  CdcEcm     │     │   LwIP      │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ Ethernet Frame    │                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ Bulk OUT complete │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ ReceiveFrame()    │
       │                   │                   │──────────────────>│
       │                   │                   │                   │
       │                   │                   │ netif->input()    │
       │                   │                   │<──────────────────│
       │                   │                   │                   │
       │                   │ Queue next recv   │                   │
       │                   │<──────────────────│                   │
       │                   │                   │                   │
```

### 4.5 Data Flow: Device → Host

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   LwIP      │     │  CdcEcm     │     │  USB HW     │     │    Host     │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ netif->linkoutput │                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ Queue packet      │                   │
       │                   │────────┐          │                   │
       │                   │        │          │                   │
       │                   │<───────┘          │                   │
       │                   │                   │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
       │                   │ [TX Task]         │                   │
       │                   │ TransmitFrame()   │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ Bulk IN           │
       │                   │                   │──────────────────>│
       │                   │                   │                   │
```

---

## 5. API Specification

### 5.1 Low-Level C API

#### 5.1.1 USB_DeviceCdcEcmInit

```c
/**
 * @brief Initialize CDC-ECM device class
 * @param controllerId USB controller ID
 * @param config Class configuration structure
 * @param handle Output parameter for class handle
 * @return USB status code
 */
usb_status_t USB_DeviceCdcEcmInit(
    uint8_t controllerId,
    usb_device_class_config_struct_t *config,
    class_handle_t *handle);
```

#### 5.1.2 USB_DeviceCdcEcmSend

```c
/**
 * @brief Send data over Bulk IN endpoint
 * @param handle Class handle from init
 * @param ep Endpoint address
 * @param buffer Data buffer (raw Ethernet frame)
 * @param length Data length (14-1514 bytes)
 * @return USB status code
 */
usb_status_t USB_DeviceCdcEcmSend(
    class_handle_t handle,
    uint8_t ep,
    uint8_t *buffer,
    uint32_t length);
```

#### 5.1.3 USB_DeviceCdcEcmRecv

```c
/**
 * @brief Queue receive on Bulk OUT endpoint
 * @param handle Class handle from init
 * @param ep Endpoint address
 * @param buffer Receive buffer
 * @param length Buffer size (should be >= 1514)
 * @return USB status code
 */
usb_status_t USB_DeviceCdcEcmRecv(
    class_handle_t handle,
    uint8_t ep,
    uint8_t *buffer,
    uint32_t length);
```

### 5.2 High-Level C++ API

#### 5.2.1 CdcEcm::Init

```cpp
/**
 * @brief Initialize CDC-ECM network interface
 * @param bulk_in_ep Bulk IN endpoint address
 * @param bulk_out_ep Bulk OUT endpoint address
 * @param notify_ep Interrupt endpoint address
 * @param ctrl_iface Control interface number
 * @param data_iface Data interface number
 */
void CdcEcm::Init(
    uint8_t bulk_in_ep,
    uint8_t bulk_out_ep,
    uint8_t notify_ep,
    uint8_t ctrl_iface,
    uint8_t data_iface);
```

---

## 6. Configuration

### 6.1 Build Configuration

Add to `usb_device_config.h`:

```c
/* CDC-ECM Configuration */
#define USB_DEVICE_CONFIG_CDC_ECM              (1U)
#define USB_DEVICE_CONFIG_CDC_ECM_MAX_MTU      (1514U)
```

### 6.2 Network Configuration

Default network settings (matching current CDC-EEM):

| Parameter | Value |
|-----------|-------|
| Device IP | 10.10.10.1 |
| Netmask | 255.255.255.0 |
| Gateway | 0.0.0.0 |
| DHCP Server | Enabled (assigns 10.10.10.2-254 to host) |
| MAC Address | 00:1A:11:BA:DF:AD |

### 6.3 Compile-Time Selection

To support both protocols, use conditional compilation:

```c
#if defined(USB_DEVICE_CONFIG_CDC_ECM) && (USB_DEVICE_CONFIG_CDC_ECM > 0)
    InitializeCDCECM();
#elif defined(USB_DEVICE_CONFIG_CDC_EEM) && (USB_DEVICE_CONFIG_CDC_EEM > 0)
    InitializeCDCEEM();
#endif
```

---

## 7. Testing Plan

### 7.1 macOS Version Testing Matrix

> **REQUIRED**: Test on multiple macOS versions due to driver differences across versions.
> See [Apple Developer Forums](https://developer.apple.com/forums/thread/769793) for known issues.

| macOS Version | Driver | Priority | Known Quirks |
|---------------|--------|----------|--------------|
| 10.15 Catalina | AppleUSBECM.kext | **High** | Subclass 0x06 detection fails |
| 11 Big Sur | kext + dext | **High** | Both drivers present |
| 12 Monterey | AppleUSERECM.dext | Medium | dext preferred |
| 13 Ventura | AppleUSERECM.dext | Medium | - |
| 14 Sonoma | AppleUSERECM.dext | Medium | - |
| 15 Sequoia | AppleUSERECM.dext | Low | Latest, likely stable |

### 7.2 Unit Tests

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| ECM-UT-001 | Initialize CDC-ECM class | No errors, handle valid |
| ECM-UT-002 | Send 64-byte frame | USB_Success |
| ECM-UT-003 | Send 1514-byte frame | USB_Success |
| ECM-UT-004 | Send oversized frame (1515+) | Error or truncation |
| ECM-UT-005 | Receive valid frame | Delivered to LwIP |
| ECM-UT-006 | Send 512-byte frame (ZLP test) | USB_Success with ZLP |
| ECM-UT-007 | Send 1024-byte frame (ZLP test) | USB_Success with ZLP |

### 7.3 Integration Tests

| Test ID | Description | Platform | Expected Result |
|---------|-------------|----------|-----------------|
| ECM-IT-001 | Device enumeration | macOS | Network interface appears |
| ECM-IT-002 | Device enumeration | Linux | Network interface appears |
| ECM-IT-003 | DHCP assignment | macOS | Host gets 10.10.10.x IP |
| ECM-IT-004 | Ping device | macOS | `ping 10.10.10.1` succeeds |
| ECM-IT-005 | HTTP access | macOS | Camera stream accessible |
| ECM-IT-006 | Sustained transfer | macOS | 1000 frames without error |

### 7.4 macOS-Specific Tests (CRITICAL)

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| ECM-MAC-001 | Connect, check Network Preferences | Shows "Connected", NOT "Not Connected" |
| ECM-MAC-002 | DHCP without manual config | Host auto-gets 10.10.10.x IP |
| ECM-MAC-003 | USB disconnect/reconnect | Re-establishes connection within 5s |
| ECM-MAC-004 | Mac sleep/wake cycle | Connection survives, no manual intervention |
| ECM-MAC-005 | 512-byte aligned transfer | Completes without hang |
| ECM-MAC-006 | Test on macOS 10.15 specifically | Enumerates and works (subclass test) |

### 7.5 Verification Commands

**macOS**:
```bash
# Before connecting
ifconfig | grep -c "^en"

# After connecting (should show one more interface)
ifconfig | grep -c "^en"

# Check for CDC-ECM interface
system_profiler SPUSBDataType | grep -A 20 "Coral"

# Test connectivity
ping 10.10.10.1

# Test camera
curl http://10.10.10.1/camera_stream --output /dev/null -w "%{http_code}"
```

**Linux**:
```bash
# Check for new interface
ip link show | grep usb

# Test connectivity
ping 10.10.10.1
```

---

## 8. Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **No NXP SDK CDC-ECM support** | High | **Confirmed** | Use D21ecm as reference, NXP AN14169 for custom class |
| **NetworkConnection notification missing** | **Critical** | Medium | Implement as Day 1 requirement, test on macOS first |
| macOS 10.15 subclass bug | Medium | Low | Use subclass 0x00 for data interface |
| ZLP handling bugs | Medium | Medium | Explicit ZLP logic, test 512-aligned transfers |
| Endpoint allocation conflict | Medium | Low | Audit current composite device endpoint usage |
| stm32ecm-style macOS failure | High | Medium | Follow D21ecm structure exactly |
| macOS version quirks | Medium | Medium | Test on 10.15, 11, 12+ |
| Performance regression vs EEM | Low | Low | ~11 MB/s sufficient for camera streaming |
| Apple Silicon CPU overhead | Low | Low | CDC-ECM uses Efficiency cores on M1/M2/M3 |
| Breaking Linux compatibility | High | Low | Keep EEM as compile-time fallback |

### 8.1 D21ecm vs stm32ecm Compatibility Warning

> **CRITICAL**: Two CDC-ECM implementations by the same author have different macOS compatibility:
>
> | Project | macOS Lion 10.7 | macOS El Capitan 10.11 | macOS High Sierra 10.13 |
> |---------|-----------------|------------------------|-------------------------|
> | [D21ecm](https://github.com/majbthrd/D21ecm) (SAMD21) | ✅ Works | (not tested) | ✅ Works |
> | [stm32ecm](https://github.com/majbthrd/stm32ecm) (STM32F072) | ✅ Works | ❌ Fails | ❌ Fails |
>
> This demonstrates that **subtle implementation differences** significantly affect macOS compatibility.
> **Recommendation**: Study D21ecm's `usb/usb_ecm.c` and replicate its structure and timing exactly.

---

## 9. Schedule Estimate

| Phase | Tasks | Duration |
|-------|-------|----------|
| 1 | Low-level C driver | 2-3 days |
| 2 | C++ wrapper class | 1-2 days |
| 3 | USB descriptor setup | 1 day |
| 4 | Integration & debugging | 2-3 days |
| 5 | Testing on macOS/Linux | 1-2 days |
| **Total** | | **7-11 days** |

---

## 10. References

### USB Specifications
1. USB CDC Specification 1.2 (CDC120-20101103-track.pdf)
2. USB ECM Subclass Specification (ECM120.pdf)

### NXP Documentation
3. NXP RT1170 SDK USB Device Stack Documentation
4. [NXP AN14169: User-Defined USB Device Class](https://www.nxp.com/docs/en/application-note/AN14169.pdf)
5. [MCUXpresso USB Composite Device Guide](https://mcuxpresso.nxp.com/mcuxsdk/latest/html/middleware/usb/docs/MCUXSDKUSBCOMDUG/ugindex.html)
6. [NXP Community: CDC-ECM Support Discussion](https://community.nxp.com/t5/i-MX-RT-Crossover-MCUs/USB-CDC-ECM-Support/m-p/808594/1000)

### Reference Implementations (CRITICAL)
7. **[D21ecm](https://github.com/majbthrd/D21ecm)** - Working CDC-ECM for SAMD21, tested on macOS High Sierra ✅
8. [stm32ecm](https://github.com/majbthrd/stm32ecm) - CDC-ECM for STM32F072 (⚠️ does NOT work on macOS 10.11+)
9. [TinyUSB CDC-NCM](https://github.com/hathach/tinyusb/pull/550) - Alternative protocol implementation

### macOS-Specific
10. [Apple Developer Forums: CDC ECM Requirements](https://developer.apple.com/forums/thread/769793)
11. [Zephyr: CDC ECM NetworkConnection Issue](https://github.com/zephyrproject-rtos/zephyr/issues/69209)
12. [Zephyr: NetworkConnection Discussion](https://github.com/zephyrproject-rtos/zephyr/discussions/69211)
13. Apple macOS USB CDC Driver Source (IOUSBFamily)

### Protocol Comparisons
14. [Belcarra: CDC-EEM vs CDC-ECM Protocols](https://usblan.belcarra.com/2011/02/cdc-eem-vs-cdc-ecm-protocols.html)
15. [Fobnail: Ethernet over USB Research](https://fobnail.3mdeb.com/eth-over-usb-research/)
16. [Dante: USB Ethernet Adaptor Choice for macOS](https://www.getdante.com/support/faq/dvs-usb-ethernet-adaptor-choice-for-macos-systems/)

### Windows Support (Future)
17. [Microsoft NCM Driver for Windows](https://github.com/microsoft/NCM-Driver-for-Windows)
18. LwIP Network Interface Documentation

---

## 11. Appendix

### A. Complete Descriptor Structure

```c
typedef struct {
    // Interface Association Descriptor
    usb_iad_descriptor_t iad;

    // Control Interface
    usb_interface_descriptor_t control_interface;
    cdc_header_functional_descriptor_t cdc_header;
    cdc_union_functional_descriptor_t cdc_union;
    cdc_ethernet_functional_descriptor_t cdc_ethernet;
    usb_endpoint_descriptor_t notify_endpoint;

    // Data Interface (Alt 0 - No endpoints)
    usb_interface_descriptor_t data_interface_alt0;

    // Data Interface (Alt 1 - Active)
    usb_interface_descriptor_t data_interface_alt1;
    usb_endpoint_descriptor_t bulk_in_endpoint;
    usb_endpoint_descriptor_t bulk_out_endpoint;
} cdc_ecm_descriptors_t;
```

### B. MAC Address String Descriptor

CDC-ECM requires the MAC address as a USB string descriptor in hex format:

```c
// String descriptor for MAC "00:1A:11:BA:DF:AD"
// Format: "001A11BADFAD" (12 Unicode characters)
const uint8_t mac_address_string[] = {
    26,                         // bLength (2 + 12*2)
    USB_DESCRIPTOR_TYPE_STRING, // bDescriptorType
    '0', 0, '0', 0, '1', 0, 'A', 0,
    '1', 0, '1', 0, 'B', 0, 'A', 0,
    'D', 0, 'F', 0, 'A', 0, 'D', 0
};
```

### C. D21ecm Reference Implementation Details

The [D21ecm project](https://github.com/majbthrd/D21ecm) is the **recommended reference** for this implementation.

**Project Structure:**
```
D21ecm/
├── usb/
│   └── usb_ecm.c          # Core CDC-ECM implementation (KEY FILE)
├── lwip-2.1.2/            # TCP/IP stack
├── dhcp-server/           # DHCP server
├── dns-server/            # DNS server
├── www/                   # Web server content
└── project/
    └── app.c              # Application code
```

**Key Implementation Patterns from `usb/usb_ecm.c`:**

1. **NetworkConnection Notification Structure:**
```c
static alignas(4) usb_request_t notify_nc =
{
  .bmRequestType = 0xA1,
  .bRequest = 0,           // NETWORK_CONNECTION
  .wValue = 1,             // Connected
  .wIndex = USB_ECM_NOTIFY_ITF,
  .wLength = 0,
};
```

2. **Link State Management:**
```c
static bool can_xmit;  // Transmission readiness flag
// Set to true during configuration
// Toggled by endpoint callbacks
```

3. **Notification Timing:**
   - Notification sent via `ecm_report(true)` when interface activates
   - Callback triggers speed-change reporting after network connection notification completes

**Why D21ecm Works on macOS (vs stm32ecm):**

The exact difference is not documented, but likely factors:
1. **Timing/ordering of notifications** - D21ecm sends NetworkConnection immediately on interface activation
2. **Descriptor structure** - May affect AppleUSBECM driver matching
3. **USB stack differences** - SAMD21 vs STM32 USB peripheral behaviors
4. **lwIP version** - D21ecm uses 2.1.2, stm32ecm uses older 1.4.1

### D. CDC-NCM as Future Alternative

If Windows 11 native support becomes a requirement, consider CDC-NCM for a future revision:

| Feature | CDC-ECM (Current) | CDC-NCM (Future) |
|---------|-------------------|------------------|
| Windows 11 | ❌ No native driver | ✅ Native (UsbNcm.sys) |
| macOS Big Sur+ | ✅ Yes | ✅ Yes (better perf) |
| Linux | ✅ Yes | ✅ Yes |
| Throughput | ~11 MB/s | ~26 MB/s |
| Multi-frame | No | Yes |
| Complexity | Medium | Higher |

Reference: [TinyUSB CDC-NCM implementation](https://github.com/hathach/tinyusb/pull/550)

### E. Apple Silicon Considerations

Per [Dante's USB Ethernet guide](https://www.getdante.com/support/faq/dvs-usb-ethernet-adaptor-choice-for-macos-systems/):

> "CDC-ECM is more inefficient than CDC-NCM and offloads quite a bit of processing to the CPU.
> On Apple Silicon where we have Performance and Efficiency cores, it appears that the
> Efficiency cores are heavily utilized—intentionally—by the CDC-ECM driver."

This is acceptable for camera streaming workloads but worth noting for performance-sensitive applications.
