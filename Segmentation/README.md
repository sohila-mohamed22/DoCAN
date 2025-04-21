# 🚗 DoCAN ISO-TP (ISO 15765-2)

Welcome to **DoCAN ISO-TP**, a powerful and flexible implementation of the **ISO 15765-2 Transport Protocol** for **Controller Area Network (CAN)**. This project simplifies automotive diagnostics and ECU communication, supporting both **Classical CAN** and **CAN FD**. Whether you're building tools for **Unified Diagnostic Services (UDS)**, ECU flashing, or secure vehicle module interactions, DoCAN ISO-TP provides a robust foundation for handling messages beyond standard CAN frame limits.

## 📑 Table of Contents
1. [📘 Overview](#-overview)  
   - [🧠 OSI Layer Integration](#-osi-layer-integration)   
   - [🛠️ Classical CAN vs. CAN FD](#️-classical-can-vs-can-fd) 
2. [🛠️ Implementation of DoCAN](#-implementation-of-docan)
   - [🔀 Gateway: Application to Network Layer](#-gateway-application-to-network-layer)
   - [🌐 Network Layer Implementation](#-network-layer-implementation)
   - [🚚 Transport Layer Implementation](#-transport-layer-implementation)

---
## 📘 Overview

ISO-TP (ISO 15765-2), or **ISO Transport Protocol**, is a transport layer protocol used over **CAN (Controller Area Network)** to support message sizes beyond the 8-byte limit of classic CAN frames. It is a critical component of **UDS (Unified Diagnostic Services)** and **automotive diagnostics**, providing segmentation and reassembly of messages.

ISO-TP aligns with the **Network Layer and Transport Layer (Layer 4)** of the OSI model and plays a vital role in ECU flashing, diagnostics, and secure communication between vehicle modules.

### 🧠 OSI Layer Integration

The following diagram illustrates how a diagnostic message originates at the application layer and is broken down as it flows through the OSI layers, ultimately transmitted over the physical layer. ISO-TP sits at the **Transport Layer (Layer 4)** and ensures proper segmentation, flow control, and reassembly of multi-frame messages.

![Picture11](https://github.com/user-attachments/assets/0e7ebac9-b638-4eb5-aa1a-c954c73e4fd1)

When a diagnostic message is initiated by a user (e.g., an ECU tester or diagnostic tool), the **Application Layer** generates a message that may exceed the size limit of a CAN frame. This is where **ISO-TP** comes in:

- ISO-TP **splits the message** into smaller segments depending on the **underlying CAN frame capacity**.
- Each segment is wrapped in a specific **frame format** (Single Frame, First Frame, Consecutive Frame).
- These frames are sent over CAN and reassembled on the receiving end to restore the original message.

### 🛠️ Classical CAN vs. CAN FD

| Feature                                           | Classical CAN | CAN FD |
|--------------------------------------------------|---------------|--------|
| Payload length 0–8 bytes                         | ✅            | ✅     |
| Payload fixed at 8 bytes (DLC 9–15 interpreted)  | ✅ (Max 8)    | ❌     |
| Payload length 12–64 bytes                       | ❌            | ✅     |
| Separate bit rates for arbitration and data      | ❌            | ✅     |
| Remote transmission request (RTR) supported      | ✅            | ❌     |

**Classical CAN** only allows **8 bytes** of data per frame. Any message larger than 8 bytes must be segmented using ISO-TP. This is the most common use case for ISO-TP in diagnostics.

**CAN FD (Flexible Data-rate)**, a newer extension of CAN, allows data payloads of up to **64 bytes** per frame. With CAN FD, smaller diagnostic messages may fit entirely in one frame, reducing the need for segmentation. However, ISO-TP is still used when transmitting messages that exceed the 64-byte limit or when **backward compatibility** with classical CAN tools is needed.

> ✅ **In both cases**, ISO-TP abstracts away the complexities of CAN frame sizes, ensuring a seamless diagnostic message exchange between ECUs and testing tools.

---

## 🛠️ Implementation of DoCAN

This section explores the core implementation of **DoCAN ISO-TP**, detailing how it enables efficient diagnostic communication across the **Application**, **Network**, and **Transport Layers**. The implementation supports all addressing modes and is optimized for reliability and flexibility in automotive systems.

### 🔀 Gateway: Application to Network Layer

The **gateway** between the **Application Layer** and the **Network Layer** is the entry point for diagnostic communication in DoCAN ISO-TP. The diagram below illustrates this gateway, where a diagnostic request from the Application Layer is passed to the Network/Transport Layer via the `BaseConnection.sendRequest(Request appLayerRequest): A_Result` method.

![diagram-export-4-21-2025-5_49_02-AM](https://github.com/user-attachments/assets/243d6265-fc18-43d0-9436-976996ccbf23)

The result, `A_Result`, indicates whether the request was **successfully placed on the bus (OK)** or **not (Not OK)**. If the data is not successfully transmitted by the Network Layer (i.e., it wasn’t put on the CAN bus), `A_Result` will reflect a failure, signaling an issue at the network transmission stage.

---

### 🌐 Network Layer Implementation

When this API is called, it triggers a sequence of operations within the Network Layer. This includes processing the request, preparing the message according to the ISO-TP (ISO 15765-2) protocol, fragmenting the payload if needed, and initiating transmission onto the CAN bus. The Network Layer also provides feedback on whether the transmission was successful, which is essential for the Application Layer to confirm message delivery.

The flow begins when the Application Layer invokes:

```java
BaseConnection.sendRequest(appLayerRequest)
```

This call results in the creation of a network service handler and delegates the request to the transport layer for processing:

```java
// Create an instance of the network service data handler to interact with the transport layer
N_USData networkLayer = new N_USData();

// Send the request using the transport layer's request method and capture the result
N_Result result = networkLayer.request(appLayerRequest);
```

Internally, the `request(...)` method of the `N_USData` class handles the core functionality of the Network Layer. The first step involves decoding the incoming request and extracting relevant network-layer parameters based on the ISO 15765-2:2016 standard:

```java
public N_Result request(Request A_Request) {
    try {
        N_Result result; 
        // Decode the request and extract network parameters (ISO 15765-2:2016)
        NetworkRequestDecoder request = new NetworkRequestDecoder(A_Request);
        ...
```
---

### 🛠 Responsibilities of `NetworkRequestDecoder`

- **Address Extraction**: Extracts source (`N_SA`) and target (`N_TA`) addresses.
- **Address Extension**: If the message type is **Remote Diagnostics**, it also extracts the **remote address (`N_AE`)**, which is used as an extension to support addressing remote ECUs.

![diagram-export-4-21-2025-6_38_58-AM](https://github.com/user-attachments/assets/a1babfc7-a641-41ed-b688-6debe0916014)

- **CAN ID Mapping**: Uses shared mapping utilities to resolve the CAN ID.
- **Frame Format Determination**: Selects 11-bit or 29-bit identifiers and CAN (or CAN FD) format.
- **Target Type Resolution**: Determines whether the address is **Physical** or **Functional**.
- **Remote Addressing Support**: Handles optional `N_AE` field for remote diagnostics.
- **Message Payload Management**: Transfers message data, length, and frame metadata to the lower layers.

---
