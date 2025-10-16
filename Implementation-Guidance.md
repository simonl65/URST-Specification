# URST Protocol Implementation Guidance

This document should be read alongside the [URST-Specification](URST-Specification.md). It gives high-level guidance for possible implementation, a comparrison with other protocols, and possible future extensions.

---

## 1. Implementation Guidance

### 1.1 Recommended Implementation Order

Implementers SHOULD build the protocol in layers:

1. **Phase 1: Codec Layer**

   - Implement CRC-16/CCITT-FALSE calculation with lookup table
   - Implement COBS encoding and decoding
   - Implement frame delimiting
   - Test with known test vectors (Appendix B)

2. **Phase 2: Transport Layer**

   - Implement frame header construction
   - Implement frame header parsing
   - Implement sequence number management
   - Test frame type handling

3. **Phase 3: Protocol Layer**

   - Implement ACK/NAK generation
   - Implement timeout mechanism
   - Implement retry logic
   - Test reliable delivery scenarios

4. **Phase 4: Handler Layer**
   - Implement fragmentation logic
   - Implement reassembly buffers
   - Implement application API
   - Test end-to-end communication

### 1.2 Testing Strategy

#### 1.2.1 Unit Testing

Each layer SHOULD be tested independently:

```python
# Codec layer tests
def test_crc_calculation():
    assert calculate_crc(b"123456789") == 0x29B1

def test_cobs_encoding():
    assert cobs_encode(b"\x00") == b"\x01\x01"

# Transport layer tests
def test_sequence_wraparound():
    seq = 255
    next_seq = (seq + 1) & 0xFF
    assert next_seq == 0
```

#### 1.2.2 Integration Testing

Test complete send/receive cycles:

- Normal operation (no errors)
- CRC failure recovery
- Timeout and retry
- Duplicate frame handling
- Out-of-sequence frames
- Fragment reassembly

#### 1.2.3 Interoperability Testing

Test between different implementations:

- Cross-platform communication (MicroPython ↔ PC)
- Various baud rates
- Maximum payload sizes
- Fragmented messages
- Error injection and recovery

### 1.3 Debugging Recommendations

#### 1.3.1 Common Debug Points

Add logging at these critical points:

1. Frame transmission (before UART write)
2. Frame reception (after delimiter detection)
3. CRC calculation results
4. Sequence number comparisons
5. ACK/NAK generation
6. Retry attempts
7. Fragment assembly progress

#### 1.3.2 Protocol Analyzer

Implement a protocol analyzer that displays:

```
TX [seq=5]: DATA "Hello" (5 bytes) CRC=0x3054
RX [seq=5]: ACK
TX [seq=6]: DATA (frag 0/3) 194 bytes
RX [seq=6]: ACK
TX [seq=7]: DATA (frag 1/3) 194 bytes
...
```

### 1.4 Platform-Specific Considerations

#### 1.4.1 MicroPython

```python
# Use micropython.const() for constants
from micropython import const
FRAME_DATA = const(0x01)

# Preallocate buffers where possible
rx_buffer = bytearray(512)

# Use @micropython.native for performance-critical functions
@micropython.native
def calculate_crc(data):
    # CRC calculation code
    pass
```

#### 1.4.2 Standard Python

```python
# Use pyserial for UART access
import serial
ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=0.1)

# Can use standard library features
from collections import deque
message_queue = deque()
```

#### 1.4.3 C/C++ Embedded

```c
// Use fixed-size buffers
uint8_t rx_buffer[512];
uint16_t rx_buffer_len = 0;

// Implement CRC with lookup table
static const uint16_t crc_table[256] = { /* ... */ };

// Use state machine for non-blocking operation
typedef enum {
    STATE_IDLE,
    STATE_RECEIVING,
    STATE_PROCESSING
} urst_state_t;
```

### 1.5 Performance Optimization

#### 1.5.1 CRC Calculation

Always use lookup table, never bit-by-bit calculation:

```python
# SLOW (bit-by-bit)
for bit in range(8):
    if crc & 0x8000:
        crc = ((crc << 1) ^ poly) & 0xFFFF

# FAST (table lookup)
crc = ((crc << 8) ^ crc_table[(crc >> 8) ^ byte]) & 0xFFFF
```

#### 1.5.2 Buffer Management

Minimize allocations in receive path:

```python
# SLOW (creates new bytearray each time)
rx_buffer = rx_buffer + new_data

# FAST (extend existing buffer)
rx_buffer.extend(new_data)
```

#### 1.5.3 Frame Detection

Use efficient delimiter search:

```python
# SLOW (Python loop)
for i, byte in enumerate(buffer):
    if byte == 0x00:
        start = i

# FAST (use built-in methods)
start = buffer.find(b'\x00')
```

---

## 2. Comparison with Related Protocols

### 2.1 URST vs SLIP (Serial Line IP)

| Feature         | URST                  | SLIP                        |
| --------------- | --------------------- | --------------------------- |
| Error Detection | CRC-16/CCITT-FALSE    | None                        |
| Reliability     | ACK/NAK with retries  | None                        |
| Frame Delimiter | 0x00 (COBS encoded)   | 0xC0 (with escaping)        |
| Overhead        | ~4-6 bytes + 4%       | 2-3 bytes                   |
| Use Case        | Reliable serial comms | IP over serial (deprecated) |
| Complexity      | Medium                | Low                         |

**When to use URST over SLIP:**

- When reliability is required (no packet loss acceptable)
- When CRC error detection is needed
- When acknowledgments are required

### 2.2 URST vs HDLC (High-Level Data Link Control)

| Feature         | URST                  | HDLC             |
| --------------- | --------------------- | ---------------- |
| Frame Delimiter | 0x00                  | 0x7E             |
| Encoding        | COBS                  | Bit stuffing     |
| Error Detection | CRC-16/CCITT-FALSE    | CRC-16 or CRC-32 |
| Flow Control    | Stop-and-wait         | Sliding window   |
| Addressing      | None (point-to-point) | Supported        |
| Complexity      | Low                   | High             |
| Standards Body  | None                  | ISO/IEC          |

**When to use URST over HDLC:**

- When simplicity is prioritized over features
- When sliding window is not needed
- When no addressing is required (point-to-point only)
- When standard Python/MicroPython implementation is desired

### 2.3 URST vs PPP (Point-to-Point Protocol)

| Feature         | URST               | PPP                    |
| --------------- | ------------------ | ---------------------- |
| Layer           | Data link          | Data link + Network    |
| Error Detection | CRC-16/CCITT-FALSE | FCS (typically CRC-16) |
| Reliability     | Built-in ACK/NAK   | Optional (LCP)         |
| Configuration   | Static             | Negotiated (LCP/NCP)   |
| Authentication  | None               | PAP/CHAP               |
| Compression     | None               | Optional               |
| Complexity      | Low                | Very High              |

**When to use URST over PPP:**

- When full PPP stack is overkill
- When static configuration is acceptable
- When authentication is handled at application layer
- When microcontroller resources are limited

### 2.4 URST vs Modbus RTU

| Feature           | URST                 | Modbus RTU             |
| ----------------- | -------------------- | ---------------------- |
| Error Detection   | CRC-16/CCITT-FALSE   | CRC-16-Modbus          |
| Framing           | COBS + delimiters    | Timing-based           |
| Addressing        | None                 | 1-247 device addresses |
| Reliability       | ACK/NAK automatic    | Application-level      |
| Message Structure | Flexible payload     | Strict PDU format      |
| Use Case          | General serial comms | Industrial automation  |

**When to use URST over Modbus:**

- When master/slave architecture is not needed
- When peer-to-peer communication is required
- When timing-based framing is unreliable (USB-serial, etc.)
- When flexible message formats are needed

---

## 3. Future Protocol Extensions

This section describes potential extensions for future versions of URST. These are NOT part of the current specification.

### 3.1 Potential Version 0.3.0 Features

#### 3.1.1 Sliding Window Flow Control

Replace stop-and-wait with selective repeat ARQ:

- Window size negotiation
- Out-of-order delivery support
- Reduced latency for bulk transfers

#### 3.1.2 Connection Establishment

Add handshake frames:

- CONNECT / CONNECT_ACK
- Protocol version negotiation
- Parameter exchange (window size, timeout, etc.)

#### 3.1.3 Compression

Optional payload compression:

- Frame type indicating compressed data
- Negotiated compression algorithm
- Beneficial for repetitive data patterns

#### 3.1.4 Timestamps

Add optional timestamp field:

- Latency measurement
- Replay attack detection
- Synchronization support

### 3.2 Reserved Frame Types for Extensions

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

### 3.3 Backward Compatibility

Future versions MUST maintain backward compatibility:

- Version 0.2.0 implementations MUST interoperate with 0.3.0
- Optional features MUST be negotiated during connection
- Implementations MUST gracefully handle unknown frame types

---
