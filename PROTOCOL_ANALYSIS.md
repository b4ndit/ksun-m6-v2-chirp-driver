# KSUN M6 V2 Serial Protocol

Complete protocol specification for serial communication with the KSUN M6 V2 radio.

---

## Protocol Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Baud Rate | 38400 | Changed from 4800 in original M6 |
| Data Bits | 8 | |
| Parity | None | |
| Stop Bits | 1 | |
| Flow Control | None | DTR must be enabled |
| Block Size | 128 bytes (0x80) | Changed from 16 bytes in original M6 |
| Start Address | 0x0300 | Changed from 0x0050 in original M6 |
| Total Memory | 6656 bytes | 256 config + 6400 channels |
| Command Format | Big-endian | |
| Data Format | Little-endian | Frequencies and tones |

---

## Checksum Algorithm

**Formula:** `(86 + sum_of_bytes) % 256`

**Changed from original M6:** Base changed from 2 to 86

```python
def _checksum(data):
    """Calculate checksum for radio protocol

    Args:
        data: bytes to checksum

    Returns:
        int: checksum value (0-255)
    """
    cs = 86
    for byte in data:
        cs += byte
    return cs % 256
```

**Verified Examples:**
- Init command `32 31 05 10` → 0xCE
- Exit command `32 31 05 EE` → 0xAC
- Read 0x0300 `52 03 00` → 0xAB
- Read 0x0380 `52 03 80` → 0x2B

---

## Command Set

### Enter Programming Mode

**Command:** `32 31 05 10 CE`
- Bytes 0-3: `32 31 05 10` (command)
- Byte 4: `CE` (checksum)

**Response:** `06` (ACK) or `FF` (NACK)

**Important:** Radio requires **multiple attempts** - implement retry logic!

**Recommended implementation:**
```python
def enter_programming_mode(radio):
    serial = radio.pipe
    cmd = b"\x32\x31\x05\x10"
    req = cmd + bytes([_checksum(cmd)])

    for attempt in range(5):
        try:
            serial.write(req)
            time.sleep(0.1)
            res = serial.read(1)
            if res == b"\x06":
                return  # Success
        except Exception:
            pass
        time.sleep(0.1)

    raise errors.RadioError(
        "Radio refused to enter programming mode after 5 attempts")
```

### Exit Programming Mode

**Command:** `32 31 05 EE AC`
- Bytes 0-3: `32 31 05 EE` (command)
- Byte 4: `AC` (checksum)

**Response:** None (radio resets immediately)

```python
def exit_programming_mode(radio):
    serial = radio.pipe
    cmd = b"\x32\x31\x05\xee"
    req = cmd + bytes([_checksum(cmd)])

    try:
        serial.write(req)
    except Exception:
        raise errors.RadioError("Radio refused to exit programming mode")
```

### Read Block

**Command:** `52 [addr_h] [addr_l] [checksum]`
- Byte 0: `52` ('R' - Read command)
- Bytes 1-2: Address (big-endian, 16-bit)
- Byte 3: Checksum

**Response:** 132 bytes total
- Bytes 0-2: Echo of command (3 bytes)
- Bytes 3-130: Data (128 bytes)
- Byte 131: Checksum of bytes 0-130

**Example - Read address 0x0300:**
```
TX: 52 03 00 AB
RX: 52 03 00 [128 data bytes] [checksum]
```

**Implementation:**
```python
def _read_block(radio, block_addr):
    serial = radio.pipe

    cmd = struct.pack(">cH", b"R", block_addr)
    req = cmd + bytes([_checksum(cmd)])

    try:
        serial.write(req)
        time.sleep(0.05)  # 50ms delay
        res_len = len(cmd) + radio.BLOCK_SIZE + 1
        res = serial.read(res_len)

        if len(res) != res_len or res[:len(cmd)] != cmd:
            raise Exception("unexpected reply!")
        if res[-1] != _checksum(res[:-1]):
            raise Exception("block failed checksum!")

        block_data = res[len(cmd):-1]
    except Exception as e:
        msg = "Failed to read block at %04x: %s" % (block_addr, str(e))
        raise errors.RadioError(msg)

    return block_data
```

### Write Block

**Command:** `57 [addr_h] [addr_l] [128 data bytes] [checksum]`
- Byte 0: `57` ('W' - Write command)
- Bytes 1-2: Address (big-endian, 16-bit)
- Bytes 3-130: Data (128 bytes)
- Byte 131: Checksum of bytes 0-130

**Response:** `06` (ACK) or error

**Implementation:**
```python
def _write_block(radio, block_addr, block_data):
    serial = radio.pipe

    cmd = struct.pack(">cH", b"W", block_addr) + block_data
    req = cmd + bytes([_checksum(cmd)])

    try:
        serial.write(req)
        res = serial.read(1)
        if res != b"\x06":
            raise Exception("unexpected reply!")
    except Exception as e:
        msg = "Failed to write block at %04x: %s" % (block_addr, str(e))
        raise errors.RadioError(msg)
```

---

## Communication Sequence

### Download Sequence

1. Enter programming mode (with retries)
2. For each block from 0x0300 to 0x1D80 (step 0x80):
   - Send read command
   - Wait 50ms
   - Read 132-byte response
   - Verify checksum
   - Extract 128 data bytes
3. Exit programming mode

**Total blocks:** 52 blocks (6656 ÷ 128)
**Expected time:** ~9 seconds

### Upload Sequence

1. Enter programming mode (with retries)
2. Verify model (read first block, check signature 0x50 at offset 0x10)
3. For each block from 0x0300 to 0x1D80 (step 0x80):
   - Send write command with 128 bytes
   - Wait for ACK (0x06)
4. Exit programming mode

---

## Memory Address Map

### Configuration Blocks

| Address | Size | Content | Block Number |
|---------|------|---------|--------------|
| 0x0300 | 128 | Config part 1 (bytes 0-127) | 1 |
| 0x0380 | 128 | Config part 2 (bytes 128-255) | 2 |

**File offsets:** Configuration starts at byte 0 in `.v2pp` file

### Channel Data Blocks

| Address | Size | Content | Block Number |
|---------|------|---------|--------------|
| 0x0400 | 128 | Channels 1-4 (32 bytes each) | 3 |
| 0x0480 | 128 | Channels 5-8 | 4 |
| 0x0500 | 128 | Channels 9-12 | 5 |
| ... | ... | ... | ... |
| 0x1D80 | 128 | Channels 197-200 | 52 |

**File offsets:** Channel data starts at byte 256 in `.v2pp` file

**Channel address formula:**
```
radio_address = 0x0400 + (channel_number - 1) × 0x20
block_address = (radio_address ÷ 0x80) × 0x80
```

---

## Reliability Issues

### Observed Behavior

The radio is **not 100% reliable** and exhibits:

1. **Intermittent NACK responses** - Sometimes responds with 0xFF instead of 0x06
2. **No response** - Occasionally doesn't respond at all
3. **Works on retry** - Usually succeeds on subsequent attempts

This is observed in the **official manufacturer software**, which handles it by:
- Displaying error messages to user
- Requiring manual retry

### Solution: Automatic Retry Logic

Implement automatic retries to improve user experience:

```python
# For enter_programming_mode: 5 attempts with 0.1s delays
# For read_block: Retry on checksum failure
# For write_block: Retry on NACK or timeout
```

### Timing Requirements

**Between blocks:** 50ms delay recommended (manufacturer uses ~17-18ms)

**After commands:**
- Enter programming: 100ms
- Read block: 50ms
- Write block: no delay (waits for ACK)
- Exit programming: no delay (radio resets)

---

## Verification Steps

When implementing the protocol, verify:

1. **Connection:**
   - Command bytes: `32 31 05 10 CE`
   - Response: `06` (ACK)
   - Retry logic works (handles NACK/no response)

2. **Read operation:**
   - Block size: 128 bytes data
   - Response includes command echo + data + checksum
   - Addresses increment by 0x80

3. **Memory content:**
   - First read (0x0300): Contains config data
   - Signature byte 0x50 at offset 0x10 (byte 16 in first block)
   - Channel data starts at 0x0400

4. **Checksums:**
   - All commands use base 86
   - Verify calculated vs received checksums match

5. **Write operation:**
   - ACK byte received (0x06)
   - Data persists after exit programming mode

6. **Exit:**
   - Command: `32 31 05 EE AC`
   - No response expected (radio resets)

---

## CHIRP Driver Constants

```python
BAUD_RATE = 38400
BLOCK_SIZE = 0x80
START_ADDR = 0x0300
CHANNELS = 200
_memsize = 6656  # 256 + (200 × 32)
```

---

## Protocol Comparison: M6 vs M6 V2

| Parameter | Original M6 | M6 V2 | Change |
|-----------|-------------|-------|--------|
| Baud Rate | 4800 | 38400 | 8x faster |
| Checksum Base | 2 | 86 | Different |
| Block Size | 16 bytes | 128 bytes | 8x larger |
| Start Address | 0x0050 | 0x0300 | Different |
| Channels | 80 | 200 | 2.5x more |
| Channel Size | 10 bytes | 32 bytes | 3.2x larger |
| Total Memory | ~810 bytes | 6656 bytes | 8x larger |
| Commands | Same format | Same format | Compatible |
| Reliability | Unknown | Requires retries | Issue confirmed |

**Compatibility:** None - completely different protocol parameters

---

## Serial Capture Analysis

Based on captured communication from manufacturer software:

### Successful Connection (Attempt 2)

```
TX: 32 31 05 10 CE
RX: 06 (ACK)

TX: 52 03 00 AB (Read 0x0300)
RX: 52 03 00 [128 bytes] CS
     ^^^^^^^^ Echo
```

### Failed Connection (Attempt 1)

```
TX: 32 31 05 10 CE
RX: FF (NACK)

TX: 32 31 05 10 CE
RX: FF (NACK)

User sees: "Communication Error"
```

### Failed Connection (Attempt 3)

```
TX: 32 31 05 10 CE
RX: (no response - timeout)

TX: 32 31 05 10 CE
RX: (no response)

TX: 32 31 05 10 CE
RX: (no response)

User sees: "Communication Error"
```

**Conclusion:** Even manufacturer software cannot guarantee first-time success

---

## Error Handling

### Connection Errors

**Symptom:** No ACK or NACK response
**Solution:** Retry up to 5 times with delays
**User message:** "Radio refused to enter programming mode after 5 attempts"

### Checksum Errors

**Symptom:** Received checksum doesn't match calculated
**Solution:** Retry read operation
**User message:** "Failed to read block at [address]: block failed checksum!"

### Write Errors

**Symptom:** No ACK after write
**Solution:** Retry write operation
**User message:** "Failed to write block at [address]: unexpected reply!"

### Model Mismatch

**Symptom:** Signature byte is not 0x50
**Solution:** Abort operation
**User message:** "Invalid radio model (expected M6 V2 signature 0x50)"

---

**Version:** 2.0 (Simplified)
**Last Updated:** 2026-01-09
**License:** GPL v3+
