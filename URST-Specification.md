# URST Protocol Specification

**Version:** 0.1.0  
**Status:** Draft  
**Date:** 2025-10-12

## 1. Introduction

**Universal Reliable Serial Transport** (URST) is a lightweight, robust communication protocol, providing reliable data transmission over serial connections. This document specifies the protocol's architecture, message formats, and operational behavior.

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

### 1.2 Protocol Overview

URST implements a four-layer architecture:

1. **Handler Layer**: User-facing API for message handling
2. **Protocol Layer**: Reliable delivery with acknowledgments
3. **Transport Layer**: Frame construction and validation
4. **Codec Layer**: COBS encoding and serial port I/O

## 2. Protocol Architecture

### 2.1 Layer Responsibilities

#### 2.1.1 Handler Layer

- Provides simple `send()` and `receive()` interfaces
- Manages message fragmentation and reassembly
- Handles application-level timeouts

#### 2.1.2 Protocol Layer

- Implements reliable message delivery
- Manages sequence numbers and acknowledgments
- Handles retransmissions on timeout

#### 2.1.3 Transport Layer

- Constructs and parses frame headers
- Calculates and verifies CRCs
- Manages frame boundaries

#### 2.1.4 Codec Layer

- Implements COBS (Consistent Overhead Byte Stuffing)
- Handles UART read/write operations
- Manages frame delimiters

## 3. Message Format

### 3.1 Frame Structure

```
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+---------------+---------------+
|   Frame Type  | Sequence Num  |  [Header: 2 bytes]
+---------------+---------------+
|     Payload (0-200 bytes)     |
+---------------+---------------+
|        CRC-16 (2 bytes)       |  [Not shown in frame]
+---------------+---------------+
```

**Frame Encoding Process**:

1. Header (2 bytes): [Frame Type][Sequence Number]
2. Append payload (0-200 bytes)
3. Calculate CRC-16 and append (2 bytes)
4. COBS encode the entire block
5. Wrap with FRAME_DELIMITER (0x00) bytes

### 3.2 Frame Types

| Type | Value | Description             |
| ---- | ----- | ----------------------- |
| DATA | 0x01  | Data frame              |
| ACK  | 0x02  | Acknowledgment frame    |
| NAK  | 0x03  | Negative acknowledgment |

### 3.3 Protocol Constants

| Constant           | Value | Description                   |
| ------------------ | ----- | ----------------------------- |
| MAX_RETRIES        | 3     | Maximum transmission attempts |
| DEFAULT_TIMEOUT_MS | 1000  | Default ACK timeout in ms     |
| MAX_PAYLOAD_SIZE   | 200   | Maximum payload size in bytes |
| RX_BUFFER_SIZE     | 512   | Receive buffer size in bytes  |

### 3.4 COBS Encoding and Error Detection

All frames MUST be encoded using COBS (Consistent Overhead Byte Stuffing) before transmission. The encoding process:

1. Start with raw frame: `[Frame Type][Sequence Number][Payload]`
2. Calculate CRC-16 of the raw frame (polynomial 0x1021, initial value 0xFFFF)
3. Append CRC-16 (little-endian) to the frame
4. COBS encode the entire block (header + payload + CRC-16)
5. Wrap with FRAME_DELIMITER (0x00) bytes

**Key Properties:**

- No zero bytes in encoded data (except delimiters)
- Frame boundaries marked by FRAME_DELIMITER (0x00)
- Maximum overhead: 1 byte per 254 bytes of data
- Maximum encoded frame size: 207 bytes (200 payload + 2 header + 2 CRC + 3 COBS overhead)

**Error Detection:**

- **CRC-16**: Detects bit errors in transmission
- **Sequence Numbers**: 8-bit (0-255) for detecting out-of-order frames
- **Frame Validation**: Invalid CRCs cause silent discard
- **Sequence Validation**: Out-of-order frames trigger NAK responses
- **Buffer Management**: RX_BUFFER_SIZE limits memory usage

## 4. Protocol Operation

### 4.1 Reliable Transmission

1. The sender transmits a DATA frame with a unique sequence number
2. The receiver acknowledges successful reception with an ACK frame
3. If no ACK is received within the timeout period, the sender retransmits
4. The receiver discards duplicate frames based on sequence numbers

### 4.2 Flow Control

- Window size: 1 (stop-and-wait)
- Maximum retransmission attempts: 3
- Default timeout: 1000ms (configurable)

### 4.3 Error Handling

- CRC failures result in NAK responses
- Invalid frames are silently discarded
- Sequence number mismatches trigger NAK responses

## 5. Implementation Notes

### 5.1 Memory Management

- The implementation is optimized for microcontrollers with limited RAM
- Large messages are automatically fragmented and reassembled
- Fixed-size receive buffers prevent memory exhaustion

### 5.2 Performance Considerations

- COBS encoding adds minimal overhead
- Zero-copy operations where possible
- Efficient CRC16 implementation for low overhead

## 6. Security Considerations

- No built-in encryption (can be layered on top if needed)
- CRC16 provides basic error detection but no integrity protection
- Applications should implement their own authentication if required

## 7. References

- [[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)] Key words for use in RFCs to Indicate Requirement Levels
- [COBS] [Consistent Overhead Byte Stuffing](https://en.wikipedia.org/wiki/Consistent_Overhead_Byte_Stuffing)
- [MicroPython] https://micropython.org/

## Appendix A. Example Usage

```python
from machine import UART
from urst import URSTHandler

# Initialize UART (example settings)
uart = UART(1, baudrate=115200, tx=4, rx=5)

# Create URST handler
urst = URSTHandler(uart)

# Send data
urst.send(b'Hello, World!')
# Since this is reliable data transfer it will wait for ACK/NACK

# Receive data
data = urst.receive()
if data:
    print("Received:", data)
```

## Appendix B. High-level Data Send Flow

### Application Layer

User sends data

`success = comms.send(b"Hello, World!")`

### URST Protocol Stack

**Handler Layer**

- `send()` checks payload size
- Handler routes to protocol
  - `return self.send_with_ack(data)`

**Protocol Layer**

- `send_with_ack()` gets sequence
- `_get_next_seq()` increments counter
- Protocol gets sequence
  - `seq_num = self._get_next_seq()`

**Transport Layer**

- Build frame header [type][seq]
- Protocol sends typed frame
  - `if not self.send_typed_frame(data, frame_type, seq_num):`

**Codec Layer**

- CRC16 calculation
- Append CRC bytes
- COBS encoding
- Add frame delimiters
- Codec writes to UART
  - `return self.uart.write(frame) or 0`

## Appendix C. Frame encoding with COBS and CRC

### Frame Encoding Pipeline

- Calculate CRC16
  - `crc = self._calculate_crc(data)`
  - Append CRC
    - `data_with_crc = data + bytes([crc & 0xFF, (crc >> 8) & 0xFF])`
    - Little-endian byte order
  - COBS encode
    - `cobs_data = self._cobs_encode(data_with_crc)`
    - \_cobs_encode() implementation
  - Add delimiters
    - `frame = bytes([FRAME_DELIMITER]) + cobs_data + bytes([FRAME_DELIMITER]`
    - 0x00 start/end markers
- `uart.write()` final output

## Appendix D. Frame reception and decoding

### Frame Reception and Decoding

- User receives data
  - `data = comms.receive()`
  - receive() method call
    - \_process_incoming()
      - Handler processes frames
      - `frame_data = self.handle_incoming_data()`
      - Protocol receives frame
        - `frame = self.recv_typed_frame()`
          - Transport gets raw frame
            - `raw_frame = self.recv_frame()`
            - \_update_rx_buffer()
            - \_extract_frame_from_buffer()
              - COBS decode
              - `decoded_data = self._cobs_decode(frame_data)`
                - Validate CRC
                  - `if self._calculate_crc(payload) == received_crc:`
                - Return payload or None

### Protocol layer processing

- Sequence number validation
  - `_send_ack()` or `_send_nack()`
- Frame type handling
  - FRAME_DATA processing

## Appendix E. Large data fragmentation and reassembly

### Large Data Fragmentation Flow

#### Handler fragments data

- `return self._send_fragmented(data)`
- \_send_fragmented() calculates fragments
  - Loop through each fragment
    - Build fragment header
    - `header = bytes([msg_id & 0xFF, i & 0xFF, total_fragments & 0xFF, len(fragment_data) & 0xFF])`
    - send_with_ack() for each fragment
- receive() processes incoming frames
  - Detect fragment
  - `if total_fragments > 1 and len(payload) >= 4 + data_len:`
    - \_handlefragment() when detected
      - Store fragment in buffer
      - Check completion
      - `if msg_info["received"] >= msg_info["total"]:`
  - Queue complete message
  - `self._complete_messages.append(bytes(complete_data))`

#### Fragment Reassembly Process

- Fragment buffer management
  - Store fragments by msg_id
- Complete message reconstruction
  - Queue complete message

## Appendix F. Flowcharts

### `handler.py`

See the [handler flowchart](./01_handler-flowchart.md)

### `protocol.py`

See the [protocol flowchart](./02_protocol-flowchart.md)

### `transport.py`

See the [transport flowchart](./03_transport-flowchart.md)

### `codec.py`

See the [codec flowchart](./04_codec-flowchart.md)

## Appendix G. Version History

- 0.1.0 (2025-10-12): Initial specification draft

## Appendix H. Contributors

- Simon R. Lincoln <urst@codeability.co.uk>

## Appendix I. License

This code is released under the **Sustainable Use License** detailed in [LICENSE.md](LICENSE.md).

### TL;DR ( _NOT forming part of the license_ )

In essence the license is one of [Fair-code](https://faircode.io). It effectively prohibits you from making profit from this code if I haven\'t given you a specific license to do so. If I give you such a separate license then I\'m happy for us both to profit off this code.
