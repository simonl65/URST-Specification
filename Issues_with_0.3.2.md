# URST Specification 0.3.2 Issues

In v0.3.2, several issues and edge cases have been identified that could affect robustness:

## Critical Issues

### 1. Device Restart/Sequence Number Rollback

If a sender restarts and resets its sequence number to 0, the receiver will:

- See seq=0 when expecting seq=N
- Send NAK (out of sequence)
- The sender will retry with seq=0
- This creates a deadlock that can only resolve by chance when sequence numbers wrap around

**Recommendation**: Add a connection establishment handshake (`CONNECT`/`CONNECT_ACK` frames) that resets both sides' sequence numbers. Alternatively, detect large sequence number jumps backwards as potential restarts.

### 2. Fragment Reassembly Memory Exhaustion

The spec says receivers SHOULD timeout incomplete fragments, but there's no MUST requirement for tracking multiple concurrent message IDs. An attacker or buggy sender could:

- Start transmitting message ID 0
- Switch to message ID 1 before completing 0
- Continue cycling through all 256 message IDs
- Force receiver to buffer 256 incomplete message sets

**Recommendation**: Make fragment timeout MUST instead of SHOULD, and specify that concurrent messages are NOT allowed.

### 3. Fragment Interleaving Not Addressed

What happens if fragments from different messages interleave?

```
TX: FRAG(msg_id=5, frag=0)
TX: FRAG(msg_id=6, frag=0)  // Started new message!
TX: FRAG(msg_id=5, frag=1)  // Back to first message
```

The spec doesn't explicitly forbid this, but the reassembly logic assumes completion before starting a new message.

**Recommendation:** Explicitly require sequential fragment transmission (current message MUST complete before starting next).

## Significant Issues

### 4. **No Maximum Fragment Total**

`Total Frags` is 1 byte (0-255), meaning a message could theoretically require 255 fragments × 194 bytes = 49,470 bytes of reassembly buffer. Most embedded systems can't allocate this.

**Recommendation:** Require implementations to reject fragment sets exceeding their capabilities with a new `ERROR` frame type. This frame type MUST let the other end know it's maximum fragment size in the `ERROR` frame's payload.

### 5. **Lost Final Fragment Detection**

If the last fragment is lost and the sender gives up after MAX_RETRIES:

- Receiver has fragments 0-(N-2) buffered indefinitely
- No signal to discard the incomplete message
- Only the fragment timeout saves you

**Recommendation:** Add an explicit "abort transmission" mechanism or require senders to send all fragments to completion even after application-layer failure.

**_<span style="background-color:yellowgreen; color:black;padding:0 4px 2px 3px;border-radius:4px;">TODO:</span> Determine what would be sensible._** Is the fragment timeout good enough? Does the message buffer get cleared in such a case?

## Minor Issues

### 6. **ACK/NAK Ambiguity During Sequence Wrap**

When sequence wraps 255→0:

```
Receiver expects seq=255, receives seq=0
Receiver last_received=254
```

Is seq=0 a duplicate of a very old frame, or the next expected frame? The current logic would send NAK, creating unnecessary retransmissions.

**Recommendation**: Define a "sequence number window" (e.g., ±16) to distinguish old duplicates from valid wraps.

**_<span style="background-color:yellowgreen; color:black;padding:0 4px 2px 3px;border-radius:4px;">TODO:</span> Determine what would be sensible._** In theory this will never happen because the protocol is stop-and-wait?

### 7. No Receiver-Not-Ready Signal

If the receiver's application layer is slow to process messages (e.g. during garbage collection, application processing, etc.), the receiver has no way to signal "stop sending, I'm busy." The sender will timeout and retry, wasting bandwidth.

**Recommendation**: Add a `BUSY`/`NOT_READY` frame type for flow control.

### 8. Fragment Timeout Calculation Assumes Serial Transmission

The spec calculates timeout as max_frags × (MAX_RETRIES+1) × TIMEOUT, assuming fragments are sent sequentially. But what if the sender has a large buffer and transmits fragments in bursts?

**Recommendation**: Clarify that the timeout assumes sequential, stop-and-wait transmission of fragments.

### 9. No Version Field

The protocol has no version negotiation. If a version is released with breaking changes, devices can't detect version mismatches.

**Recommendation**: Use a reserved frame type (0x05) for `VERSION` or add version to a future `CONNECT` handshake.

### 10. CRC-16 Coverage Ambiguity

Section 3.2.4 says "CRC MUST be calculated over Frame Type + Sequence Number + Payload" but doesn't explicitly state whether this is the pre-COBS or post-COBS data. (It's clearly pre-COBS from context, but should be explicit.)

### 11. Partial Frame in Buffer at Startup

What if device resets mid-frame and the buffer contains a partial frame?

**Recommendation:** Clear buffers on init.

## Edge Cases Not Addressed

12. **Simultaneous Transmission** - What if both sides send DATA frames simultaneously? Both will receive unexpected frames and NAK each other, potentially creating a livelock.

13. **Baud Rate Mismatch** - Receiver will see garbage. Should probably result in consistent COBS decode failures, but worth mentioning.
