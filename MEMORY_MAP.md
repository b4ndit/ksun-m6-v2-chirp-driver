# KSUN M6 V2 Memory Map Reference

**File Format:** `.v2pp`
**Total Size:** 6656 bytes (0x1A00)
**Structure:**
- Configuration Block: 256 bytes (0x0000 - 0x00FF)
- Channel Data: 6400 bytes (0x0100 - 0x19FF) = 200 channels × 32 bytes

---

## Configuration Block (256 bytes, offset 0x0000-0x00FF)

### Password Section (0x00-0x0B)

| Offset | Size | Field | Description | Values |
|--------|------|-------|-------------|--------|
| 0x00-0x0B | 12 | Password | DES-encrypted password | Encrypted with key "badworks"<br>Master password: "707070"<br>Empty: All 0xFF |

### Core Configuration (0x10-0x45)

| Offset | Size | Field | Type | Description | Values/Range |
|--------|------|-------|------|-------------|--------------|
| 0x10 | 1 | Signature | byte | File signature | 0x50 ('P') |
| 0x11-0x15 | 5 | Startup Text | ASCII | Power-on display text | ASCII, 0xFF padded |
| 0x16-0x17 | 2 | Freq Lower Limit | uint16_le | Lower frequency limit | 400-480 (MHz) |
| 0x18-0x19 | 2 | Freq Upper Limit | uint16_le | Upper frequency limit | 400-480 (MHz) |
| 0x1A | 1 | Channel Limit | byte | Max accessible channels | 1-200 |
| 0x1B | 1 | Current Channel | byte | Active channel on startup | 0-199 (0-based) |
| 0x1C | 1 | Language | byte | UI language | 0=Chinese Simplified<br>1=Chinese Traditional<br>2=English |
| 0x1D | 1 | Beep | bitfield | Beep on/off + volume | Bit 7: On (1) or Off (0)<br>Bits 0-6: Volume (0-127) |
| 0x1E | 1 | Key Lock | bitfield | Key lock + auto-lock timeout | Bit 7: Enabled (1) or Disabled (0)<br>Bits 0-6: Timeout index (0-42) |
| 0x1F | 1 | Backlight | bitfield | Brightness + timeout | Bits 6-7: Level (0=Low, 1=Med, 2=High)<br>Bits 0-5: Timeout index (0-42) |
| 0x20 | 1 | Roger/TOT | bitfield | Roger beep + transmit timeout | Bit 7: Roger beep On (1) or Off (0)<br>Bits 0-6: TOT index (0-42) |
| 0x21 | 1 | Battery Save | byte | Power saving mode | 0=Off, 1=1:1, 2=1:2, 3=1:3, 4=1:4 |
| 0x22 | 1 | Squelch | byte | Squelch level | 0-9 |
| 0x23 | 1 | MIC Gain | byte | Microphone gain | 0-31 |
| 0x24 | 1 | RX Volume | byte | Receive volume | 1-63 |
| 0x25 | 1 | VOX | bitfield | VOX on/off + delay | Bit 7: On (1) or Off (0)<br>Bits 0-6: Delay (0-9) |
| 0x26 | 1 | VOX Threshold | byte | VOX sensitivity | 1-255 |
| 0x27 | 1 | Alarm | byte | Alarm type | 0=Local, 1=Remote, 2=Local+Remote |
| 0x28 | 1 | APO | byte | Auto power off | 0=Off, 1-200=(value×10 minutes) |
| 0x29 | 1 | Channel Display | byte | Channel display mode | 0=CH No., 1=CH Alias |
| 0x2A | 1 | Menu Timeout | byte | Menu auto-exit timeout | Index 0-42 (same pattern as other timeouts) |
| 0x2B | 1 | Key Function | byte | Side key long-press function | 0=CH-, 1=CH+, 2=Monitor, 3=Scan, 4=LED, 5=Battery, 6=Freq Detect, 7=Alarm, 8=Talkaround, 9=VOX |
| 0x2C-0x45 | 26 | Menu Settings | bytes | Menu item order/visibility | Array of 26 bytes |
| 0x46-0xFF | 186 | Reserved | bytes | Unused/padding | All 0xFF |

---

## Channel Data (6400 bytes, offset 0x0100-0x19FF)

### Channel Structure (32 bytes per channel)

**Channel N offset:** `0x0100 + (N - 1) × 0x20` where N is 1-200

| Offset | Size | Field | Type | Description | Values/Encoding |
|--------|------|-------|------|-------------|-----------------|
| +0x00 | 4 | RX Frequency | uint32_le | Receive frequency | Stored in units of 10 Hz<br>Example: 446.00625 MHz = 44600625<br>Empty: 0xFFFFFFFF |
| +0x04 | 2 | RX Tone | uint16_le | RX CTCSS/DCS code | See Tone Encoding below |
| +0x06 | 4 | TX Frequency | uint32_le | Transmit frequency | Same encoding as RX frequency<br>0x00000000 = TX disabled |
| +0x0A | 2 | TX Tone | uint16_le | TX CTCSS/DCS code | See Tone Encoding below |
| +0x0C | 1 | Flags 1 | bitfield | Power, scan, bandwidth, scrambler | Bit 7: Power (0=High, 1=Low)<br>Bit 6: Scan (0=Add, 1=Skip)<br>Bit 5: Bandwidth (0=Wide/FM, 1=Narrow/NFM)<br>Bit 4: Reserved<br>Bits 0-3: Scrambler (0=Off, 1-8=Code) |
| +0x0D | 1 | Flags 2 | bitfield | Compander, busy lock, encryption | Bits 6-7: Reserved<br>Bit 5: Compander (0=Off, 1=On)<br>Bits 3-4: Busy lock (0=Off, 1=Carrier, 2=QT/DQT)<br>Bits 0-2: Encryption (0=Off, 1-5=Various) |
| +0x0E | 1 | Reserved | byte | Unknown | Usually 0x00 |
| +0x0F-0x13 | 5 | Channel Name | ASCII | Channel alias | 5 chars max, 0xFF padded |
| +0x14-0x1F | 12 | ID Data | bytes | Radio ID or encryption key | 12 bytes hex |

### Empty Channel Detection

A channel is empty if RX frequency = 0xFFFFFFFF or 0x00000000

---

## Tone Encoding (16-bit little-endian)

### Format

```
Bits 12-13: Tone Type
  00 = CTCSS
  01 = DCS Normal
  10 = DCS Inverted
  11 = None/Off

Bits 0-11: Tone Value (12-bit)
```

### CTCSS Encoding

**Formula:** `tone_value = ctcss_freq_hz × 10`

**Examples:**
- 67.0 Hz: 670 (0x029E) → bytes: `9E 02`
- 88.5 Hz: 885 (0x0375) → bytes: `75 03`
- 136.5 Hz: 1365 (0x0555) → bytes: `55 05`
- 254.1 Hz: 2541 (0x09ED) → bytes: `ED 09`

### DCS Encoding

DCS codes stored as octal converted to binary.

**Formula:** `tone_value = (d1 × 64) + (d2 × 8) + d3 | polarity_bits`

Where d1, d2, d3 are the three octal digits.

**Examples:**
- D023N: (0×64) + (2×8) + 3 = 19 | 0x1000 = 0x1013 → bytes: `13 10`
- D023I: 19 | 0x2000 = 0x2013 → bytes: `13 20`
- D114N: (1×64) + (1×8) + 4 = 76 | 0x1000 = 0x104C → bytes: `4C 10`
- D631N: (6×64) + (3×8) + 1 = 409 | 0x1000 = 0x1199 → bytes: `99 11`

### None/Off Encoding

No tone: 0x3000 → bytes: `00 30`

### Decode/Encode Functions

```python
def decode_tone(tone_value):
    """
    Decode 16-bit little-endian tone value

    Returns: (mode, value, polarity)
      mode: '', 'Tone', or 'DTCS'
      value: frequency (Hz) or DCS code (decimal)
      polarity: None, 'N', or 'R'
    """
    tone_val = tone_value & 0x0FFF
    tone_type = (tone_value >> 12) & 0x03

    if tone_type == 0x00:  # CTCSS
        if tone_val == 0:
            return ('', 0, None)
        freq = tone_val / 10.0
        return ('Tone', freq, None)
    elif tone_type == 0x01:  # DCS Normal
        if tone_val == 0:
            return ('', 0, None)
        return ('DTCS', tone_val, 'N')
    elif tone_type == 0x02:  # DCS Inverted
        if tone_val == 0:
            return ('', 0, None)
        return ('DTCS', tone_val, 'R')
    else:  # 0x03 = No tone
        return ('', 0, None)


def encode_tone(mode, value, polarity):
    """
    Encode tone to 16-bit little-endian format

    Args:
      mode: '', 'Tone', or 'DTCS'
      value: frequency (Hz) or DCS code (decimal)
      polarity: None, 'N', or 'R'

    Returns: 16-bit tone value
    """
    if mode == 'Tone' and value:
        tone_val = int(value * 10) & 0x0FFF
        return tone_val | 0x0000
    elif mode == 'DTCS' and value:
        tone_val = int(value) & 0x0FFF
        if polarity == 'N':
            return tone_val | 0x1000
        else:  # 'R' or inverted
            return tone_val | 0x2000
    else:
        return 0x3000
```

---

## Frequency Encoding

**Storage format:** Frequency in units of 10 Hz as 32-bit little-endian

**Conversions:**
- Read: `frequency_hz = stored_value × 10`
- Write: `stored_value = frequency_hz ÷ 10`

**Examples:**

1. **446.00625 MHz:**
   - Hz: 446,006,250
   - Stored: 44,600,625 = 0x02A8B6F1
   - Bytes (LE): `F1 B6 A8 02`

2. **462.5625 MHz:**
   - Hz: 462,562,500
   - Stored: 46,256,250 = 0x02C1D13A
   - Bytes (LE): `3A D1 C1 02`

**Valid Range:** 400.000 - 480.000 MHz

---

## Complete Examples

### Example 1: Reading Channel 1 (offset 0x0100)

**Raw bytes:**
```
F1 B6 A8 02  9E 02  F1 B6 A8 02  9E 02  20  00  FF  41 42 43 44 45
```

**Decoded:**
- **RX Freq:** 0x02A8B6F1 = 44,600,625 × 10 = 446.00625 MHz
- **RX Tone:** 0x029E = 670 → CTCSS 67.0 Hz
- **TX Freq:** 0x02A8B6F1 = 446.00625 MHz
- **TX Tone:** 0x029E → CTCSS 67.0 Hz
- **Flags 1:** 0x20 (bit 5=1) → NFM, High power, Add to scan
- **Flags 2:** 0x00 → All off
- **Name:** "ABCDE"

### Example 2: Writing Channel 5 (offset 0x0180)

**Target settings:**
- RX: 462.5625 MHz, TX: 467.5625 MHz
- CTCSS: 123.0 Hz (both)
- High power, Wide FM, Skip scan
- Name: "FRS 1"

**Calculations:**
1. RX: 46,256,250 = 0x02C1D13A → `3A D1 C1 02`
2. TX: 46,756,250 = 0x02C8E81A → `1A E8 C8 02`
3. Tone: 123.0 × 10 = 1230 = 0x04CE → `CE 04`
4. Flags1: High(0) + Skip(1) + Wide(0) = 0x40
5. Name: "FRS 1" = `46 52 53 20 31`

**Bytes to write at 0x0180:**
```
3A D1 C1 02  CE 04  1A E8 C8 02  CE 04  40  00  00  46 52 53 20 31
FF FF FF FF  FF FF FF FF  FF FF FF FF
```

---

## CHIRP Implementation Notes

### Memory Format Definition

```python
MEM_FORMAT = """
#seekto 0x0000;
struct {
  u8 password[12];
  u8 _reserved1[4];
  u8 signature;
  char startup_text[5];
  ul16 freq_lower_limit;
  ul16 freq_upper_limit;
  u8 channel_limit;
  u8 current_channel;
  u8 language;
  u8 beep;
  u8 keylock;
  u8 backlight;
  u8 roger_tot;
  u8 battery_save;
  u8 squelch;
  u8 mic_gain;
  u8 rx_volume;
  u8 vox;
  u8 vox_threshold;
  u8 alarm;
  u8 apo;
  u8 channel_display;
  u8 menu_timeout;
  u8 key_function;
  u8 menu_settings[26];
  u8 _reserved2[186];
} settings;

#seekto 0x0100;
struct {
  ul32 rx_freq;
  ul16 rx_tone;
  ul32 tx_freq;
  ul16 tx_tone;
  u8 power: 1,
     scan: 1,
     nfm: 1,
     _unk1: 1,
     scrambler: 4;
  u8 _unk2: 2,
     compander: 1,
     busy: 2,
     encryption: 3;
  u8 _reserved;
  char name[5];
  u8 id_data[12];
} memory[200];
"""
```

### Validation Checklist

1. File size is exactly 6656 bytes
2. Signature byte at offset 0x10 is 0x50
3. Frequencies are within 400-480 MHz
4. Channel indices: 0-199 internal, 1-200 UI
5. Empty channels: RX frequency = 0xFFFFFFFF
6. Power bit: 0=High, 1=Low
7. Scan bit: 0=Add, 1=Skip
8. Frequency unit: 10 Hz
9. Tone modes: 'Tone' (CTCSS), 'DTCS' (DCS), '' (none)
10. DCS polarity: 'N' (normal), 'R' (inverted)

---

**Version:** 2.0 (Simplified)
**Last Updated:** 2026-01-09
**License:** GPL v3+
