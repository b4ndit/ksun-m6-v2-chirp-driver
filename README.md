# KSUN M6 V2 CHIRP Driver

CHIRP driver for the KSUN M6 V2 (2024+ hardware revision) UHF handheld transceiver. This model shows channels as "CH001" instead of "CH-01" on the radio display.

## Overview

The KSUN M6 V2 is a compact UHF handheld radio operating in the 400-480 MHz range with 200 memory channels. This driver provides full support for programming the radio using CHIRP software.

**Supported Model:** KSUN M6 V2 (2024+ hardware revision)
**Frequency Range:** 400-480 MHz
**Channels:** 200
**Power Output:** High (2.0W) / Low (0.5W)

## Features

### Memory Channels
- **200 programmable channels** with the following per-channel settings:
  - RX/TX frequencies (10 Hz precision)
  - Channel names (5 characters max)
  - CTCSS/DCS tones (separate RX/TX)
  - Power level (High/Low)
  - Bandwidth (Wide 25kHz / Narrow 12.5kHz)
  - Scan add/skip
  - Scrambler (8 codes)
  - Compander
  - Busy lock
  - Encryption settings

### Tone Squelch
- **CTCSS:** Full support for standard CTCSS tones (67.0-254.1 Hz)
- **DCS:** Full support for standard DCS codes (D023N-D754N)
- **Cross-tone modes:** All CHIRP cross-tone combinations supported
- **Separate RX/TX tones:** Independent tone configuration for receive and transmit

### Radio Settings
- **Basic Settings:**
  - Startup text (5 characters)
  - Language selection (Chinese Simplified/Traditional, English)
  - Beep on/off and volume
  - Squelch level (0-9)
  - RX volume (0-63)
  - Backlight level (Low/Medium/High) and timeout
  - Channel display mode
  - Startup channel

- **Advanced Settings:**
  - Microphone gain (0-31)
  - VOX with threshold and delay
  - Battery save modes (Off, 1:1, 1:2, 1:3, 1:4)
  - Key lock with auto-lock timeout
  - Time-out timer (TOT)
  - Roger beep
  - Frequency band limits (400-480 MHz)
  - Channel limit (max accessible channels 1-200)
  - Alarm mode
  - Auto power off (APO)
  - Menu timeout
  - Side key function

## Technical Details

### Communication Protocol
- **Baud Rate:** 38400
- **Protocol:** Custom binary protocol with checksums
- **Block Size:** 128 bytes (0x80)
- **Memory Size:** 6656 bytes (6.5 KB)
  - Configuration: 256 bytes
  - Channel Data: 6400 bytes (200 channels × 32 bytes)

### Memory Map
- **Configuration Block:** 0x0000-0x00FF (256 bytes)
  - Password protection (DES encrypted)
  - Radio settings
  - Menu customization
- **Channel Data:** 0x0100-0x19FF (6400 bytes)
  - 200 channels × 32 bytes per channel
  - RX/TX frequencies stored in 10 Hz units (little-endian)
  - Tone encoding: 16-bit values with polarity bits

For complete memory map details, see [MEMORY_MAP.md](MEMORY_MAP.md).

For serial protocol and communication details, see [PROTOCOL_ANALYSIS.md](PROTOCOL_ANALYSIS.md).

### Frequency Storage
- Stored in units of 10 Hz (0.01 MHz precision)
- Example: 446.00625 MHz = 44,600,625 units
- Little-endian 32-bit unsigned integer

### Tone Encoding
16-bit values with polarity bits:
- **Bits 12-13:** Polarity (00=CTCSS, 01=DCS Normal, 10=DCS Inverted, 11=None)
- **Bits 0-11:** Tone value
  - CTCSS: frequency × 10 (e.g., 67.0 Hz = 670)
  - DCS: binary encoding of octal code

## Credits

**Based on CHIRP framework:**
- CHIRP Project: https://chirp.danplanet.com
- CHIRP License: GNU GPL v3

## Support

For issues specific to this driver:
- Open an issue on GitHub

For general CHIRP questions:
- CHIRP Mailing List: https://chirp.danplanet.com/projects/chirp/wiki/MailingList
- CHIRP Documentation: https://chirp.danplanet.com

## Disclaimer

This driver is provided as-is for educational and interoperability purposes. Use at your own risk. Always maintain backups of your radio configuration.

**Not affiliated with or endorsed by KSUN or Quzhou KSUN Electronic Technology Co., Ltd.**
