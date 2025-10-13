# URST Protocol Specification

**Version:** 0.3.0  
**Status:** Draft  
**Date:** 2025-10-13

---

## Abstract

This document specifies the **Universal Reliable Serial Transport** (URST) protocol, a lightweight - _acknowledgment before next transmission_ - communication protocol designed for reliable data transmission over serial connections in resource-constrained embedded systems. URST provides automatic retransmission, error detection via CRC-16, frame delimiting through COBS encoding, and support for message fragmentation.

---

## Status of This Memo

This document is a **draft** specification and is subject to change. It is provided for implementation and testing purposes.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Protocol Architecture](#2-protocol-architecture)
3. [Message Format Specification](#3-message-format-specification)
4. [Protocol State Machines](#4-protocol-state-machines)
5. [Protocol Operations](#5-protocol-operations)
6. [Fragmentation Protocol](#6-fragmentation-protocol)
7. [Conformance Requirements](#7-conformance-requirements)
8. [Security Considerations](#8-security-considerations)
9. [IANA Considerations](#9-iana-considerations)
10. [References](#10-references)

Appendices:

- [Appendix A: Example Usage](#appendix-a-example-usage)
- [Appendix B: Test Vectors](#appendix-b-test-vectors)
- [Appendix C: Implementation Flowcharts](#appendix-c-implementation-flowcharts)

---

## 1. Introduction

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### 1.2 Terminology

- **Frame**: A complete unit of transmission including header, payload, CRC, and COBS encoding
- **Payload**: The application data carried within a frame (0-200 bytes)
- **Sequence Number**: An 8-bit counter used to detect duplicates and ordering
- **Fragment**: A portion of a larger message split for transmission
- **Stop-and-Wait**: A flow control mechanism requiring acknowledgment before next transmission

### 1.3 Protocol Overview

URST implements a four-layer architecture providing reliable delivery over unreliable serial connections. The protocol uses stop-and-wait flow control with automatic retransmission, CRC-16 error detection, and COBS encoding for frame delimiting.

**Design Goals:**

- Reliable delivery with automatic retransmission
- Minimal overhead suitable for microcontrollers
- Zero-byte-free encoding for robust frame delimiting
- Support for large message fragmentation and reassembly

---

## 2. Protocol Architecture

### 2.1 Layer Model

URST implements four distinct layers:

```
┌─────────────────────────────────────┐
│      Handler Layer (Application)    │  User API: send(), receive()
├─────────────────────────────────────┤
│       Protocol Layer (Reliable)     │  ACK/NAK, Retransmission
├─────────────────────────────────────┤
│      Transport Layer (Framing)      │  Frame Type, Sequence Num's
├─────────────────────────────────────┤
│     Codec Layer (Encoding/IO)       │  COBS, CRC-16, UART
└─────────────────────────────────────┘
```

### 2.2 Layer Responsibilities

#### 2.2.1 Handler Layer

The Handler Layer MUST provide:

- Simple `send()` and `receive()` interfaces for applications
- Automatic fragmentation of messages exceeding MAX_PAYLOAD_SIZE
- Reassembly of fragmented messages
- Queueing of complete messages for application retrieval

#### 2.2.2 Protocol Layer

The Protocol Layer MUST provide:

- Reliable message delivery with acknowledgments
- Sequence number management for duplicate detection
- Retransmission logic with timeout and retry limits
- ACK and NAK frame generation

#### 2.2.3 Transport Layer

The Transport Layer MUST provide:

- Frame header construction (frame type and sequence number)
- Frame header parsing and validation
- Routing of frames to appropriate protocol handlers

#### 2.2.4 Codec Layer

The Codec Layer MUST provide:

- COBS encoding and decoding
- CRC-16 calculation and verification
- Frame delimiter insertion and detection
- UART read/write operations
- Receive buffer management

---

## 3. Message Format Specification

### 3.1 Frame Structure

A complete URST frame consists of the following components:

```
Logical Frame (before COBS encoding):

 HEADER (2 bytes)    PAYLOAD (0-200 bytes)    CRC (2 bytes)
+--------+--------+-------------------------+--------+------+
| Frame  | Seq    |                         | CRC    | CRC  |
| Type   | Number |    Application Data     | Low    | High |
+--------+--------+-------------------------+--------+------+
  Byte 0   Byte 1        Bytes 2 to N         Byte N+1 Byte N+2

Physical Frame (after COBS encoding):
+-------+---------------------------+-------+
| 0x00  |    COBS Encoded Data      | 0x00  |
+-------+---------------------------+-------+
 Delim        (no 0x00 bytes)        Delim
```

**Important:** The sequence number is part of the frame **header**, not the payload. All frame types (DATA, ACK, NAK) include both Frame Type and Sequence Number in their 2-byte header.

### 3.2 Frame Fields

All URST frames consist of a mandatory 2-byte header, optional payload, and 2-byte CRC:

```
Frame = [Frame Type][Sequence Number][Payload (0-200 bytes)][CRC-16]
        └────────── Header ─────────┘
```

#### 3.2.1 Frame Type (1 byte)

The Frame Type field identifies the purpose of the frame.

| Type     | Value     | Description              | Payload Size | Required |
| -------- | --------- | ------------------------ | ------------ | -------- |
| DATA     | 0x01      | Application data frame   | 0-200 bytes  | MUST     |
| ACK      | 0x02      | Acknowledgment (success) | 0 bytes      | MUST     |
| NAK      | 0x03      | Negative acknowledgment  | 0 bytes      | MUST     |
| Reserved | 0x00      | Invalid (zero byte)      | -            | -        |
| Reserved | 0x04-0x0F | Reserved                 | -            | -        |
| Reserved | 0x10-0xFF | Reserved for future use  | -            | -        |

**Requirements:**

- Implementations MUST support DATA, ACK, and NAK frame types
- Implementations MUST silently discard frames with unknown frame types
- Implementations MUST NOT use frame type 0x00
- Frame types 0x04-0xFF are reserved for future protocol versions

#### 3.2.2 Sequence Number (1 byte)

The Sequence Number field is an 8-bit counter (0-255) used for:

- Duplicate frame detection
- Acknowledgment matching
- Frame ordering verification

**Requirements:**

- Sequence numbers MUST increment by 1 for each new DATA frame transmission
- Sequence numbers MUST wrap from 255 to 0
- Retransmissions MUST use the same sequence number as the original
- ACK and NAK frames MUST use the sequence number of the frame being acknowledged

#### 3.2.3 Payload (0-200 bytes)

The Payload field contains application data or protocol-specific information.

**Requirements:**

- Payload size MUST NOT exceed MAX_PAYLOAD_SIZE (200 bytes)
- ACK and NAK frames MUST have empty payloads (0 bytes after the header)
- DATA frames MAY have empty payloads (header only)

**Note:** ACK and NAK frames echo the sequence number in the header (byte 1), not in the payload. The payload is empty for these frame types.

#### 3.2.4 CRC-16 (2 bytes)

The CRC-16 field provides error detection for the frame.

**Requirements:**

- CRC MUST be calculated over Frame Type + Sequence Number + Payload
- CRC MUST use the algorithm specified in Section 3.4
- CRC MUST be serialized in little-endian byte order

### 3.3 Frame Encoding Process

The complete frame encoding process MUST follow these steps:

1. **Construct Logical Frame:**

   ```
   logical_frame = [frame_type] + [seq_num] + payload
   ```

2. **Calculate CRC-16:**

   ```
   crc16 = calculate_crc16(logical_frame)
   ```

3. **Append CRC (little-endian):**

   ```
   frame_with_crc = logical_frame + [crc16 & 0xFF] + [(crc16 >> 8) & 0xFF]
   ```

4. **Apply COBS Encoding:**

   ```
   encoded_data = cobs_encode(frame_with_crc)
   ```

5. **Add Frame Delimiters:**

   ```
   physical_frame = [0x00] + encoded_data + [0x00]
   ```

6. **Transmit via UART**

**Frame Size Calculations:**

- Minimum logical frame: 2 bytes (header only, no payload)
- Maximum logical frame: 202 bytes (header + 200 byte payload)
- Maximum frame with CRC: 204 bytes
- Maximum COBS overhead: 3 bytes (1 byte per 254 bytes + 1)
- Maximum physical frame: 209 bytes (including delimiters)

### 3.4 CRC-16 Algorithm Specification

#### 3.4.1 Algorithm Parameters

| Parameter     | Value                         |
| ------------- | ----------------------------- |
| Algorithm     | CRC-16-CCITT                  |
| Polynomial    | 0x1021 (x¹⁶+x¹²+x⁵+1)         |
| Initial Value | 0xFFFF                        |
| Final XOR     | 0x0000 (none)                 |
| Bit Order     | MSB first                     |
| Byte Order    | Little-endian (serialization) |

#### 3.4.2 Calculation Procedure

```python
def calculate_crc16(data):
    crc = 0xFFFF
    for byte in data:
        crc = ((crc << 8) ^ CRC_TABLE[(crc >> 8) ^ byte]) & 0xFFFF
    return crc

def serialize_crc(crc):
    return bytes([crc & 0xFF, (crc >> 8) & 0xFF])
```

#### 3.4.3 Lookup Table Generation

```python
def build_crc_table():
    table = []
    polynomial = 0x1021
    for i in range(256):
        crc = i << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = ((crc << 1) ^ polynomial) & 0xFFFF
            else:
                crc = (crc << 1) & 0xFFFF
        table.append(crc)
    return table
```

See Appendix B for test vectors.

### 3.5 COBS Encoding Specification

#### 3.5.1 Overview

Consistent Overhead Byte Stuffing (COBS) is used to eliminate 0x00 bytes from the payload, allowing 0x00 to serve as an unambiguous frame delimiter.

#### 3.5.2 Encoding Algorithm

COBS encoding replaces each zero byte with the distance to the next zero byte:

```python
def cobs_encode(data):
    output = bytearray([0])  # Placeholder for first code
    code_index = 0
    code = 0x01

    for byte in data:
        if byte == 0x00:
            output[code_index] = code
            code_index = len(output)
            output.append(0)
            code = 0x01
        else:
            output.append(byte)
            code += 1
            if code == 0xFF:
                output[code_index] = code
                code_index = len(output)
                output.append(0)
                code = 0x01

    output[code_index] = code
    return bytes(output)
```

#### 3.5.3 Decoding Algorithm

```python
def cobs_decode(data):
    output = bytearray()
    i = 0

    while i < len(data):
        code = data[i]
        if code == 0x00:
            return None  # Invalid COBS data

        # Copy next (code-1) bytes
        for j in range(1, code):
            if i + j >= len(data):
                break
            output.append(data[i + j])

        i += code
        if i < len(data) and code < 0xFF:
            output.append(0x00)

    return bytes(output)
```

#### 3.5.4 Properties

- Encoded data MUST NOT contain 0x00 bytes (except delimiters)
- Maximum overhead: 1 byte per 254 bytes of input + 1 byte
- Minimum overhead: 1 byte (for data with no zeros)
- Invalid COBS data (embedded 0x00) MUST return decoding failure

---

## 4. Protocol State Machines

### 4.1 Sender State Machine

```
        send()
          │
          ↓
    ┌──►IDLE
    │     │
    │     │ New frame to send
    │     ↓
    │   SENDING ──────────────┐
    │     │                   │ send_frame()
    │     │                   │
    │     ↓                   ↓
    │  WAITING_ACK ←────── (transmitted)
    │     │
    │     ├──►[ACK received] ──→ SUCCESS ──┐
    │     │                                 │
    │     ├──►[NAK received] ──→ retry++   │
    │     │         │                       │
    │     │         └─────►[retry < MAX] ──┤
    │     │                       │         │
    │     │                  SENDING        │
    │     │                                 │
    │     ├──►[Timeout] ──────→ retry++    │
    │     │         │                       │
    │     │         └─────►[retry < MAX] ──┤
    │     │                                 │
    │     │                                 │
    │     └──►[retry >= MAX] ──→ FAILED    │
    │                              │        │
    └──────────────────────────────┴────────┘
                                   │
                                   ↓
                              Return to IDLE
```

**State Descriptions:**

- **IDLE**: Waiting for data to send
- **SENDING**: Transmitting frame to UART
- **WAITING_ACK**: Waiting for ACK/NAK or timeout
- **SUCCESS**: Frame acknowledged, operation complete
- **FAILED**: Maximum retries exceeded

### 4.2 Receiver State Machine

```
    LISTENING
        │
        │ Frame received
        ↓
    FRAME_RECEIVED
        │
        ├──►[Invalid COBS] ────────────┐
        │                              │
        ├──►[Invalid CRC] ─────────────┤
        │                              │
        │                              ├──► LISTENING
        ├──►[Unknown Frame Type] ──────┤      (discard)
        │                              │
        │                              │
        ├──►[ACK/NAK Frame] ───────────┴──► LISTENING
        │         │                         (process in protocol)
        │         │
        │    (pass to protocol)
        │
        ├──►[DATA Frame]
        │         │
        │         ├──►[seq == expected] ──→ SEND_ACK ──→ DELIVER ──→ LISTENING
        │         │                                                   (advance seq)
        │         │
        │         ├──►[seq == last_received] ──→ SEND_ACK ──→ LISTENING
        │         │                                          (duplicate, discard)
        │         │
        │         └──►[seq != expected] ──→ SEND_NAK ──→ LISTENING
        │                                                (out of order, discard)
        │
        └──► (continue)
```

**State Descriptions:**

- **LISTENING**: Waiting for incoming frames
- **FRAME_RECEIVED**: Frame extracted from buffer, validating
- **SEND_ACK**: Transmitting acknowledgment
- **SEND_NAK**: Transmitting negative acknowledgment
- **DELIVER**: Passing validated payload to application layer

---

## 5. Protocol Operations

### 5.1 Data Transmission

#### 5.1.1 Sender Requirements

When transmitting a DATA frame, the sender MUST:

1. Assign a sequence number using the next value in sequence (0-255, wrapping)
2. Construct the frame with DATA frame type
3. Encode and transmit the frame
4. Start a timeout timer (DEFAULT_TIMEOUT_MS)
5. Wait for ACK or NAK response
6. On timeout or NAK: increment retry counter and retransmit if retry < MAX_RETRIES
7. On ACK: consider transmission successful
8. After MAX_RETRIES failures: report failure to application layer

#### 5.1.2 Receiver Requirements

When receiving a DATA frame, the receiver MUST:

1. Validate COBS encoding (if invalid, silently discard)
2. Validate CRC-16 (if invalid, silently discard)
3. Check sequence number:
   - If seq_num == expected_seq: Send ACK, deliver payload, advance expected_seq
   - If seq_num == last_received_seq: Send ACK, discard payload (duplicate)
   - Otherwise: Send NAK, discard payload (out of sequence)

### 5.2 Acknowledgment Protocol

#### 5.2.1 ACK Frame

An ACK frame MUST be sent when:

- A DATA frame is received with the expected sequence number
- A DATA frame is received that duplicates the last successfully received frame

ACK frames MUST:

- Use frame type FRAME_ACK (0x02)
- Have an empty payload (0 bytes after the 2-byte header)
- Echo the sequence number of the acknowledged DATA frame in byte 1 of the header

#### 5.2.2 NAK Frame

A NAK frame MUST be sent when:

- A DATA frame is received with an unexpected sequence number (not expected, not duplicate)

NAK frames MUST:

- Use frame type FRAME_NAK (0x03)
- Have an empty payload (0 bytes after the 2-byte header)
- Echo the sequence number of the rejected DATA frame in byte 1 of the header

NAK frames MUST NOT be sent for:

- CRC failures (silent discard)
- COBS decoding failures (silent discard)
- Unknown frame types (silent discard)

### 5.3 Error Handling

#### 5.3.1 CRC Failures

When a receiver detects a CRC mismatch:

- The receiver MUST silently discard the frame
- The receiver MUST NOT send a NAK response
- The receiver MUST NOT process the frame in any way
- The sender will detect the missing ACK via timeout and retransmit

**Rationale:** CRC failures indicate corruption during transmission. The frame header (including sequence number) may be corrupted, making NAK responses unreliable. Silent discard with sender timeout provides robust recovery.

#### 5.3.2 COBS Decoding Failures

When COBS decoding fails (e.g., embedded 0x00 byte in encoded data):

- The receiver MUST silently discard the frame
- The receiver MUST NOT send a NAK response
- The receiver MUST NOT attempt further processing
- The sender will detect the missing ACK via timeout and retransmit

#### 5.3.3 Sequence Number Mismatches

When a DATA frame has an unexpected sequence number:

**Case 1: Duplicate Frame** (seq_num == last_received_seq)

- Receiver MUST send ACK
- Receiver MUST NOT deliver payload again
- Receiver MUST NOT advance expected_seq
- This handles the case where sender retransmitted but receiver's ACK was lost

**Case 2: Out-of-Order Frame** (seq_num != expected_seq AND seq_num != last_received_seq)

- Receiver MUST send NAK
- Receiver MUST NOT deliver payload
- Receiver MUST NOT advance expected_seq
- This indicates protocol desynchronization

#### 5.3.4 Timeout Handling

When a sender does not receive an ACK or NAK within DEFAULT_TIMEOUT_MS:

- The sender MUST increment its retry counter
- If retry_count < MAX_RETRIES: retransmit with the same sequence number
- If retry_count >= MAX_RETRIES: report failure to application layer
- The sender MUST NOT advance its sequence number until successful ACK

#### 5.3.5 Receive Buffer Overflow

When the receive buffer exceeds RX_BUFFER_SIZE:

- Implementations MUST discard data to prevent memory exhaustion
- Implementations SHOULD clear the entire buffer to avoid partial frame corruption
- Implementations MAY implement alternative strategies (e.g., discard oldest complete frame) but MUST document this deviation
- Lost frames will be recovered through sender retransmission after timeout

### 5.4 Flow Control

URST implements stop-and-wait flow control:

- Window size: 1 frame
- Sender MUST wait for ACK before sending next frame
- Receiver processes one frame at a time
- This provides reliable delivery at the cost of throughput

**Rationale:** Stop-and-wait is simple, requires minimal state, and is appropriate for resource-constrained microcontrollers. Future versions MAY define sliding window extensions.

### 5.5 Protocol Constants

| Constant           | Value | Description                   | Configurable |
| ------------------ | ----- | ----------------------------- | ------------ |
| MAX_RETRIES        | 3     | Maximum transmission attempts | SHOULD       |
| DEFAULT_TIMEOUT_MS | 1000  | ACK timeout in milliseconds   | SHOULD       |
| MAX_PAYLOAD_SIZE   | 200   | Maximum payload bytes         | MUST NOT     |
| RX_BUFFER_SIZE     | 512   | Receive buffer size in bytes  | MAY          |

**Requirements:**

- Implementations MUST use MAX_PAYLOAD_SIZE of 200 bytes for interoperability
- Implementations SHOULD support at least 3 retries
- Implementations SHOULD use 1000ms timeout as default but MAY allow configuration
- Implementations MAY configure RX_BUFFER_SIZE based on available memory

---

## 6. Fragmentation Protocol

### 6.1 Overview

When application data exceeds the available payload space (accounting for protocol overhead), the Handler Layer MUST fragment the message for transmission and reassemble it on reception.

### 6.2 Fragment Frame Format

Fragmented messages use DATA frames with a specific payload structure:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+---------------+---------------+
|   Message ID  | Fragment Num  | Total Frags   |  Data Length  |
+---------------+---------------+---------------+---------------+
|                    Fragment Data (0-194 bytes)                |
+---------------------------------------------------------------+
```

**Field Descriptions:**

- **Message ID** (1 byte): Unique identifier for the complete message (0-255, wrapping)
- **Fragment Num** (1 byte): Zero-based index of this fragment (0 to Total-1)
- **Total Frags** (1 byte): Total number of fragments in the complete message
- **Data Length** (1 byte): Number of data bytes in this fragment
- **Fragment Data** (0-194 bytes): Portion of the original message

### 6.3 Fragmentation Rules

#### 6.3.1 Sender Requirements

When fragmenting a message, the sender MUST:

1. Check if message size exceeds (MAX_PAYLOAD_SIZE - 6) bytes
2. Assign a unique Message ID (incrementing counter, wrapping 0-255)
3. Calculate required fragments: `total_frags = ceil(msg_len / 194)`
4. For each fragment (i = 0 to total_frags - 1):
   - Extract fragment data: `data[i * 194 : (i+1) * 194]`
   - Construct fragment header: `[msg_id][i][total_frags][len(fragment_data)]`
   - Send fragment + header using reliable delivery (ACK/NAK)
5. Only proceed to next fragment after successful ACK
6. Report failure to application if any fragment fails after MAX_RETRIES

**Maximum Fragment Data Size:** MAX_PAYLOAD_SIZE - 6 = 194 bytes

#### 6.3.2 Receiver Requirements

When receiving fragments, the receiver MUST:

1. Detect fragment by checking:
   - Payload length >= 4 bytes
   - Total Frags > 1
   - Payload length >= 4 + Data Length
2. Extract fragment header fields
3. Store fragment in reassembly buffer keyed by Message ID
4. Track received fragment count per Message ID
5. When all fragments received (received_count == total_frags):
   - Reassemble message by concatenating fragments in order (0 to total-1)
   - Deliver complete message to application
   - Clear reassembly buffer for this Message ID

#### 6.3.3 Fragment Ordering

- Fragments MUST be transmitted in order (0, 1, 2, ...)
- Receivers MUST NOT assume fragments arrive in order
- Receivers MUST store fragments and reassemble based on Fragment Num
- Missing fragments will be retransmitted by reliable delivery mechanism

#### 6.3.4 Incomplete Message Timeout

Implementations SHOULD implement a timeout mechanism for incomplete fragmented messages:

- If fragments for a Message ID are not completed within a reasonable time, discard them
- Recommended timeout: 10 _ DEFAULT_TIMEOUT_MS _ MAX_RETRIES <span style="color:red; font-weight:bold"><i>TODO: Check this</i></span>
- This prevents memory exhaustion from incomplete messages

### 6.4 Fragment Detection

Single-frame messages (< 194 bytes payload) MUST NOT use fragment headers. Receivers distinguish fragments from regular data by:

1. Checking payload length >= 4
2. Verifying Total Frags field > 1
3. Verifying payload contains expected data length

Non-fragmented data that happens to match this pattern is implementation-defined. Applications SHOULD avoid data patterns that could be misinterpreted as fragment headers.

---

## 7. Conformance Requirements

### 7.1 Minimal Conformant Implementation

A conformant URST implementation MUST:

1. Implement all four protocol layers (Codec, Transport, Protocol, Handler)
2. Support DATA, ACK, and NAK frame types
3. Implement COBS encoding/decoding as specified in Section 3.5
4. Implement CRC-16 calculation as specified in Section 3.4
5. Implement stop-and-wait flow control with retransmission
6. Support MAX_PAYLOAD_SIZE of 200 bytes
7. Implement sequence number management (0-255, wrapping)
8. Support at least MAX_RETRIES (3) transmission attempts
9. Implement timeout-based retransmission
10. Handle CRC failures by silent discard
11. Handle sequence mismatches per Section 5.3.3
12. Implement fragmentation and reassembly per Section 6

### 7.2 Optional Features

Conformant implementations MAY optionally:

1. Implement configurable timeout values
2. Implement configurable retry counts
3. Implement receive buffer sizes larger than 512 bytes
4. Implement incomplete fragment timeout mechanisms
5. Define custom frame types in reserved range (0x10-0xFF) for application-specific purposes (non-interoperable)

### 7.3 Interoperability Requirements

For interoperability between implementations:

- All implementations MUST use MAX_PAYLOAD_SIZE = 200 bytes
- All implementations MUST implement identical COBS encoding
- All implementations MUST implement identical CRC-16 calculation
- All implementations MUST use little-endian byte order for CRC serialization
- All implementations MUST use 0x00 as FRAME_DELIMITER

### 7.4 Non-Conformant Behavior

The following behaviors are explicitly NON-CONFORMANT:

- Sending frames with payload > 200 bytes
- Using sequence numbers > 255
- Sending NAK in response to CRC failures
- Accepting frames with invalid CRC
- Modifying frame type values 0x01-0x03
- Using frame type 0x00
- Sending ACK/NAK frames with non-empty payloads
- Failing to implement COBS encoding
- Implementing different CRC algorithms
- Using big-endian byte order for CRC serialization

---

## 8. Security Considerations

### 8.1 Threat Model

URST is designed for point-to-point serial communication and assumes:

- Physical security of the communication medium
- Trusted endpoints
- No adversarial tampering with serial data

### 8.2 Lack of Encryption

URST does NOT provide:

- Confidentiality: All data is transmitted in plaintext
- Authentication: No verification of sender identity
- Integrity protection: CRC-16 detects accidental corruption, not intentional tampering

**Recommendation:** Applications requiring security MUST implement encryption and authentication at a higher layer.

### 8.3 Denial of Service

Potential DoS vulnerabilities:

- **Retransmission storms:** Faulty sender could continuously retransmit, consuming receiver resources
- **Fragment flooding:** Attacker could send fragments with different Message IDs to exhaust memory
- **Buffer overflow:** Malicious sender could attempt to overflow receive buffers

**Mitigations:**

- Implementations SHOULD implement rate limiting
- Implementations SHOULD timeout and discard incomplete fragments
- Receive buffer overflow protection is REQUIRED (Section 5.3.5)

### 8.4 Frame Injection

An attacker with access to the serial line could:

- Inject arbitrary frames
- Replay captured frames
- Modify frames in transit

**Recommendation:** Use physically secured serial connections or implement cryptographic authentication at a higher layer.

### 8.5 CRC-16 Limitations

CRC-16 provides error detection but NOT cryptographic integrity:

- Detects accidental corruption with high probability
- Does NOT protect against intentional modification
- An attacker can modify data and recalculate valid CRC

### 8.6 Sequence Number Prediction

The 8-bit sequence number is predictable:

- Sequences are sequential and wrap at 255
- An attacker could inject frames with predicted sequence numbers
- This is mitigated by physical security of the serial connection

### 8.7 Recommendations for Secure Applications

Applications requiring security SHOULD:

1. Implement encryption (e.g., AES) at application layer
2. Implement authentication (e.g., HMAC) at application layer
3. Use physically secured serial connections
4. Implement sequence number validation beyond URST's basic duplicate detection
5. Implement application-level timeouts and rate limiting
6. Consider adding timestamps to detect replay attacks
7. Implement message authentication codes (MAC) for integrity

---

## 9. IANA Considerations

This protocol does not require any IANA registrations. Frame type values are defined in Section 3.2.1.

Frame types 0x04-0xFF are reserved for future versions of this specification. Implementations MAY use values 0x10-0xFF for application-specific purposes but such usage is non-standard and will not be interoperable with conformant implementations.

---

## 10. References

### 10.1 Normative References

**[RFC2119]**  
Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.  
https://www.rfc-editor.org/rfc/rfc2119

**[COBS]**  
Cheshire, S. and Baker, M., "Consistent Overhead Byte Stuffing", IEEE/ACM Transactions on Networking, Vol. 7, No. 2, April 1999.  
https://en.wikipedia.org/wiki/Consistent_Overhead_Byte_Stuffing

### 10.2 Informative References

**[CRC]**  
"Cyclic Redundancy Check", Wikipedia.  
https://en.wikipedia.org/wiki/Cyclic_redundancy_check

### 10.2 Informative References

**[CRC]**  
"Cyclic Redundancy Check", Wikipedia.  
https://en.wikipedia.org/wiki/Cyclic_redundancy_check

**[MicroPython]**  
MicroPython - Python for microcontrollers.  
https://micropython.org/

---

## 15. Glossary

| Term            | Definition                                                    |
| --------------- | ------------------------------------------------------------- |
| ACK             | Acknowledgment frame indicating successful reception          |
| COBS            | Consistent Overhead Byte Stuffing encoding algorithm          |
| CRC             | Cyclic Redundancy Check for error detection                   |
| Delimiter       | 0x00 byte marking frame boundaries                            |
| Fragment        | Portion of a larger message split for transmission            |
| Frame           | Complete unit of transmission (header + payload + CRC)        |
| Handler         | High-level API layer providing send/receive interface         |
| NAK             | Negative acknowledgment indicating rejection                  |
| Payload         | Application data carried in frame (0-200 bytes)               |
| Sequence Number | 8-bit counter for duplicate detection (0-255)                 |
| Stop-and-Wait   | Flow control requiring ACK before next transmission           |
| UART            | Universal Asynchronous Receiver-Transmitter (serial hardware) |
| URST            | Universal Reliable Serial Transport                           |

---

**End of Specification**

---

## 16. Document Information

**Document Title:** Universal Reliable Serial Transport (URST) Protocol Specification  
**Version:** 0.2.0  
**Status:** Draft  
**Date:** 2025-10-13  
**Authors:** Simon R. Lincoln  
**Copyright:** © 2025 Simon R. Lincoln  
**License:** Specification is freely implementable; reference code under Sustainable Use License

---

## Appendix A. Specification Checklist

Use this checklist when implementing URST:

### A.1 Codec Layer

- [ ] CRC-16-CCITT implemented with correct polynomial (0x1021)
- [ ] CRC initial value is 0xFFFF
- [ ] CRC serialized as little-endian
- [ ] COBS encoding handles empty data (returns 0x01)
- [ ] COBS encoding handles all-zero data correctly
- [ ] COBS decoding rejects embedded 0x00 bytes
- [ ] Frame delimiters are 0x00
- [ ] Receive buffer implements overflow protection
- [ ] Test vectors from Appendix B pass

### A.2 Transport Layer

- [ ] Frame header is exactly 2 bytes [type][seq]
- [ ] Frame type values 0x01-0x03 implemented
- [ ] Unknown frame types silently discarded
- [ ] Frame type 0x00 never used
- [ ] Sequence numbers wrap correctly (255 → 0)
- [ ] Sequence number management implemented

### A.3 Protocol Layer

- [ ] ACK frames have empty payload
- [ ] NAK frames have empty payload
- [ ] ACK/NAK echo sequence number in header
- [ ] CRC failures result in silent discard (no NAK)
- [ ] COBS failures result in silent discard
- [ ] Expected sequence number tracked
- [ ] Last received sequence number tracked
- [ ] Duplicate frames send ACK but discard payload
- [ ] Out-of-sequence frames send NAK
- [ ] Timeout mechanism implemented (1000ms default)
- [ ] Retry counter implemented (MAX_RETRIES=3)
- [ ] Sender waits for ACK before next frame

### A.4 Handler Layer

- [ ] Fragmentation threshold calculated correctly (194 bytes)
- [ ] Fragment header format: [msg_id][frag_num][total][len]
- [ ] Each fragment sent with reliable delivery
- [ ] Fragment reassembly buffer implemented
- [ ] Complete message detection works
- [ ] Fragment timeout implemented (optional but recommended)
- [ ] Non-fragmented messages detected correctly

### A.5 Interoperability

- [ ] MAX_PAYLOAD_SIZE = 200 bytes
- [ ] Works with reference implementation
- [ ] Cross-platform tested (if applicable)
- [ ] Test vectors produce identical results

---

---

## Appendix B. Future Protocol Extensions

This section describes potential extensions for future versions of URST. These are NOT part of the current specification.

### 13.1 Potential Version 0.3.0 Features

#### 13.1.1 Sliding Window Flow Control

Replace stop-and-wait with selective repeat ARQ:

- Window size negotiation
- Out-of-order delivery support
- Reduced latency for bulk transfers

#### 13.1.2 Connection Establishment

Add handshake frames:

- CONNECT / CONNECT_ACK
- Protocol version negotiation
- Parameter exchange (window size, timeout, etc.)

#### 13.1.3 Compression

Optional payload compression:

- Frame type indicating compressed data
- Negotiated compression algorithm
- Beneficial for repetitive data patterns

#### 13.1.4 Timestamps

Add optional timestamp field:

- Latency measurement
- Replay attack detection
- Synchronization support

### 13.2 Reserved Frame Types for Extensions

Frame types 0x04-0xFF are reserved for future use. Potential allocations:

```
0x04: CONNECT (connection establishment)
0x05: CONNECT_ACK (connection acknowledgment)
0x06: DISCONNECT (graceful disconnect)
0x07: KEEPALIVE (connection keepalive)
0x08: COMPRESSED_DATA (compressed payload)
0x09: ENCRYPTED_DATA (encrypted payload)
0x0A: TIMESTAMPED_DATA (data with timestamp)
0x10-0x1F: Reserved for firmware update protocol
0x20-0xFF: Reserved for future use
```

### 13.3 Backward Compatibility

Future versions MUST maintain backward compatibility:

- Version 0.2.0 implementations MUST interoperate with 0.3.0
- Optional features MUST be negotiated during connection
- Implementations MUST gracefully handle unknown frame types

---

## Appendix C. FAQ (Frequently Asked Questions)

### Q1: Why use COBS instead of byte stuffing?

**A:** COBS has predictable overhead (max 0.4%) compared to byte stuffing which can have variable overhead. COBS also guarantees no 0x00 bytes in output, making frame boundaries unambiguous.

### Q2: Why CRC-16 instead of CRC-32?

**A:** CRC-16 provides excellent error detection (detects all single and double bit errors) while using less bandwidth. For typical serial communication error rates, CRC-16 is sufficient. CRC-32 could be added in future versions for critical applications.

### Q3: Why stop-and-wait instead of sliding window?

**A:** Simplicity and minimal state requirements. Stop-and-wait uses ~1KB RAM while sliding window requires significantly more. For typical microcontroller applications, the throughput is adequate. Sliding window may be added in future versions.

### Q4: Can I use URST over other transports (USB, TCP, etc.)?

**A:** Yes, but it's designed for serial UART. Over TCP, the reliability is redundant. Over USB-CDC, it works but you may want to adjust timeouts.

### Q5: What baud rates are supported?

**A:** URST works at any baud rate. Timeout values should be adjusted based on baud rate and payload size. For 115200 baud, 1000ms is appropriate. For 9600 baud, consider 2000ms.

### Q6: How do I handle a completely desynchronized connection?

**A:** If sequence numbers are out of sync:

1. Clear both transmit and receive buffers
2. Wait for timeout period (no transmissions)
3. Reset sequence numbers to 0
4. Resume communication

In future versions, a RESYNC frame may be added.

### Q7: Can I send binary data including 0x00 bytes?

**A:** Yes! COBS encoding allows any binary data including null bytes. The payload can contain any byte values 0x00-0xFF.

### Q8: What happens if fragment reassembly fails?

**A:** Individual fragments use reliable delivery (ACK/NAK), so fragments won't be lost. However, implementations SHOULD implement a timeout to discard incomplete fragment sets after ~30 seconds to prevent memory exhaustion.

### Q9: Is URST suitable for real-time applications?

**A:** URST has bounded latency in the worst case (3.1 seconds with retries). For soft real-time (100ms-1s deadlines), it's suitable. For hard real-time (<10ms), the retry mechanism may cause deadline misses.

### Q10: How do I detect if the other end has disconnected?

**A:** URST doesn't have built-in keepalive. Applications should:

- Implement periodic heartbeat messages
- Consider lack of response to heartbeat as disconnection
- Future versions may add KEEPALIVE frame type

### Q11: Can multiple devices share a serial bus?

**A:** No, URST is strictly point-to-point. It has no addressing mechanism. For multi-drop serial buses, consider Modbus or implement an addressing layer on top of URST.

### Q12: What's the maximum message size?

**A:** With fragmentation, theoretically unlimited. Practically limited by:

- Available RAM for fragment buffers
- Timeout for fragment reassembly
- Application requirements

Tested up to several megabytes for firmware updates.

### Q13: How do I implement firmware updates over URST?

**A:** Application-specific, but typical approach:

1. Send metadata (file size, CRC, version)
2. Fragment binary data (use URST fragmentation)
3. Each fragment is reliably delivered
4. Verify complete file CRC
5. Trigger bootloader

Reserved frame types 0x10-0x1F may be standardized for this in future versions.
