# 🚗 DoCAN ISO-TP (ISO 15765-2)

Welcome to **DoCAN ISO-TP**, a powerful and flexible implementation of the **ISO 15765-2 Transport Protocol** for **Controller Area Network (CAN)**. This project simplifies automotive diagnostics and ECU communication, supporting both **Classical CAN** and **CAN FD**. Whether you're building tools for **Unified Diagnostic Services (UDS)**, ECU flashing, or secure vehicle module interactions, DoCAN ISO-TP provides a robust foundation for handling messages beyond standard CAN frame limits.

## 📑 Table of Contents
1. [📘 Overview](#-overview)  
   - [🧠 OSI Layer Integration](#-osi-layer-integration)   
   - [🛠️ Classical CAN vs. CAN FD](#️-classical-can-vs-can-fd) 
2. [🛠️ Implementation of DoCAN](#-implementation-of-docan)
   - [📤 Transmission Process](#-transmission-process)
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

### 📤 Transmission Process

The **Transmission Process** covers the end-to-end path that a diagnostic request takes—from the Application Layer all the way to the Transport Layer. This ensures that each message is properly prepared, segmented (if needed), and transmitted over the CAN bus.

#### 🔀 Gateway: Application to Network Layer

The **gateway** between the **Application Layer** and the **Network Layer** is the entry point for diagnostic communication in DoCAN ISO-TP. The diagram below illustrates this gateway, where a diagnostic request from the Application Layer is passed to the Network/Transport Layer via the `BaseConnection.sendRequest(Request appLayerRequest): A_Result` method.

![diagram-export-4-21-2025-5_49_02-AM](https://github.com/user-attachments/assets/243d6265-fc18-43d0-9436-976996ccbf23)

The result, `A_Result`, indicates whether the request was **successfully placed on the bus (OK)** or **not (Not OK)**. If the data is not successfully transmitted by the Network Layer (i.e., it wasn’t put on the CAN bus), `A_Result` will reflect a failure, signaling an issue at the network transmission stage.

---

#### 🌐 Network Layer Implementation

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
##### 🛠 Responsibilities of NetworkRequestDecoder

- **Address Extraction**: Extracts source (N_SA) and target (N_TA) addresses.
- **Address Extension**: If the message type is **Remote Diagnostics**, it also extracts the **remote address (N_AE)**, which is used as an extension to support addressing remote ECUs.
- **CAN ID Mapping**: Uses shared mapping utilities to resolve the CAN ID.
- **Frame Format Determination**: Selects 11-bit or 29-bit identifiers and CAN (or CAN FD) format.
- **Target Type Resolution**: Determines whether the address is **Physical** or **Functional**.
- **Remote Addressing Support**: Handles optional N_AE field for remote diagnostics.
- **Message Payload Management**: Transfers message data, length, and frame metadata to the lower layers.
- **Transmission Data Length (TX_DL) Handling**: Utilizes the `TX_DL` value configured by the user to determine the maximum payload length for CAN frames.

**📊 TX_DL Configuration Values:**

| TX_DL | Description |
|-------|-------------|
| <8    | **❌ Invalid**<br>This range of values is invalid. |
| =8    | **🛠️ Configured CAN frame maximum payload length of 8 bytes**<br>For use with ISO 11898-1 CLASSICAL CAN type frames and CAN FD type frames:<br>— Valid DLC value range: 2..8;<br>— Valid CAN_DL value range: 2..8. |
| >8    | **🛠️ Configured CAN frame maximum payload length greater than 8 bytes**<br>For use with ISO 11898-1 CAN FD type frames only:<br>— Valid DLC value range: 2..15;<br>— Valid CAN_DL value range: 2..8, 12, 16, 20, 24, 32, 48, 64;<br>— Valid TX_DL value range: 12, 16, 20, 24, 32, 48, 64;<br>— CAN_DL ≤ TX_DL. |

> ✅ **Note**: **DLC (Data Length Code)** represents the number of bytes of data in a CAN frame, while **CAN_DL (CAN Data Length)** specifies the actual payload length in a CAN FD frame. This distinction allows for extended data lengths beyond the classical 8-byte limit, facilitating more flexible data transmission.

---

#### 🚚 Transport Layer Implementation

Once the `NetworkRequestDecoder` has parsed and validated the incoming diagnostic request, the Transport Layer (`N_USData`) takes over to ensure the message is correctly formatted, fits the CAN specifications, and is transmitted over the bus according to ISO 15765-2 rules.

Here's a breakdown of the core steps performed by the `request(Request A_Request)` method in the Transport Layer:

##### ❗ Error Handling & Validation Checks

1. **TX_DL Validation**  
   Ensures that the configured `TX_DL` (transmission data length) is valid.  
   Allowed values: **[8, 12, 16, 20, 24, 32, 48, 64]**.  
   ❌ If invalid, the request is rejected immediately.

2. **Format Field Validation**  
   Confirms that the selected frame format is recognized and supported.

3. **CAN ID Validation**  
   Checks if a valid CAN ID is set.  
   ❌ Invalid CAN IDs will abort the request.

4. **Addressing Mode Validation**  
   Verifies that the addressing mode (e.g., NORMAL, EXTENDED, MIXED) is supported.  
   ❌ If not, the request fails early.

5. **CAN ID Constraint Check for FIXED_NORMAL**  
   If using the `FIXED_NORMAL` mode, a **29-bit CAN ID** is mandatory.  
   ❌ Using an 11-bit ID here is not permitted.

6. **Message Payload Validation**  
   Ensures the diagnostic message is neither null nor empty.  
   ❌ Empty messages are considered invalid.

7. **Remote Address Validation (for MIXED Mode)**  
   In `MIXED` mode, a valid remote address (N_AE) must be provided.  
   ❌ Missing or default values cause rejection.

---

##### 📦 Frame Type Selection

After all validations, the system determines the optimal frame type:

- **Single Frame (SF)**:  
  If the message length fits within one frame (based on addressing mode and CAN type), it uses:
  ```java
  result = SingleFrameHandler.send(request);
  ```

- **Multi-Frame (FF + CF)**:  
  If the message exceeds a single frame's capacity:
  - First, the system checks that it's **not a functional request**.
  - If valid, it proceeds with:
    ```java
    result = MultiFrameHandler.send(request);
    ```

  ❌ Functional requests **must** use a single frame. Attempting to use multi-frame with functional addressing triggers an error.

---

##### 🔄 Transmission Handlers

Here’s a clear, structured breakdown of the `SingleFrameHandler` class under a refined section, with improved readability and emphasis on **error handling**, organized by **addressing mode**:

### 📦 `SingleFrameHandler` Class Overview

This class handles **Single Frame (SF)** transmission in the ISO 15765-2 protocol over CAN, supporting:
- 🧩 Classical CAN and CAN FD
- 🛠 Multiple addressing modes (Normal, Extended, Mixed)
- ✅ Frame padding, PCI construction, and async transmission via the Data Link Layer

#### 🧾 Communication Flow: Single Frame

Below is a simplified sequence diagram illustrating how a Single Frame is transmitted:
![diagram-export-4-23-2025-8_16_42-AM](https://github.com/user-attachments/assets/5fec26ee-0f1d-4ca2-8201-d53499238082)

---

### 📤 Responsibilities

1. **Construct Single Frame messages** depending on addressing mode and CAN frame type.
2. **Calculate and append PCI (Protocol Control Information)** bytes (1 or 2 bytes depending on CAN format).
3. **Add addressing byte** for Extended or Mixed addressing.
4. **Ensure proper frame size** and padding if needed.
5. **Map frame length to valid CAN DLC** using `DLCMapping`.
6. **Send frame asynchronously** and wait for result using timeout.
7. **Log all details** for debugging and analysis.

---

### 🔄 Addressing Mode Handling

#### 🟩 Normal Addressing (default)
- No additional addressing byte required.
- PCI and data directly included in frame.
  
The figure below shows the exact frame layout for a **Single Frame** using **Normal Addressing** in **TX_DL = 8** mode:

![diagram-export-4-23-2025-8_07_42-AM](https://github.com/user-attachments/assets/43ff956b-c7b6-40fa-bd30-cb0d8db8cb5d)

> 🧠 **Explanation**:  
> - The **first byte** is the **PCI byte**, where the upper nibble (`0x0`) indicates a *Single Frame*, and the lower nibble indicates the *data length*.  
> - The remaining bytes (up to 7) are used for **diagnostic data**.  
> - No addressing bytes are included in Normal Addressing.

When using **CAN FD**, the PCI expands to **2 bytes** to accommodate larger payload sizes. The frame layout is shown below:

![diagram-export-4-23-2025-8_18_57-AM](https://github.com/user-attachments/assets/2773eda2-b64b-484a-b9dc-e2c8d0490191)

> 🧠 **Explanation**:  
> - The **first PCI byte** remains `0x0` indicating a *Single Frame*.  
> - The **second PCI byte** specifies the data length explicitly.  
> - The rest of the frame contains the **diagnostic data payload**, with potential length up to 64 bytes, depending on `TX_DL`.

#### 🟨 Extended Addressing
- Adds a single **Target Address** byte at the beginning.
- Useful in UDS when more than one ECU is on the bus.

#### 🟦 Mixed Addressing
- Adds a **Remote Address** byte (Network Address Extension).
- Often used in diagnostic gateways or load balancers.

---

### ❗ Error Handling / Validation

| Stage | Error Check | Handling |
|-------|-------------|----------|
| Frame Length Mapping | If `DLCMapping.getNextValidCANSize` returns `-1` | Logs error + returns `N_ERROR` |
| PCI Construction | Verifies length depending on CAN type | Uses 1 or 2 byte PCI as required |
| Frame Padding | Ensures padded with `0xCC` if frame is shorter | ✅ Done automatically |
| DLC Mapping | If `DLCMapping.getDLC` returns `INVALID_DLC` | Logs error + returns `N_ERROR` |
| Async Transmission | - `TimeoutException`: logs + cancels + returns `N_TIMEOUT_A`<br>- `ExecutionException` or `InterruptedException`: logs + cancels + returns `N_ERROR` |

---

### 🧪 Robustness Features

- ✅ **Timeout-controlled Future** for transmission
- ✅ **Detailed logging** at every step (`LoggerUtility`, `PrintDetailedLogs`)
- ✅ **GUI-configurable optimization** via `GuiHelper.isDataOptimizationEnabled`

---



