# Range Ball iButton Analyzer

A web-based tool for analyzing and modifying DS1971 iButton EEPROM data used in golf range ball dispensers.

## Live Site

Open `index.html` in a browser or serve it locally:
```bash
python3 -m http.server 8000
```

---

## Reverse-Engineered Format

### EEPROM Structure (32 bytes)

| Bytes | Purpose | Notes |
|-------|---------|-------|
| 0-5 | Header/Config | Static values: `0E BF 99 00 01 01` |
| **6-7** | **Credit Counter** | Little-endian, raw value = credits × 20 |
| 8 | Unknown | Always `00` |
| **9-10** | **Session Counter** | Changes each dispense (purpose unclear) |
| 11-14 | Padding | Always `00 00 00 00` |
| **15-16** | **Checksum** | Validates bytes 6-7 and 9-10 |
| 17-31 | Unused | Filled with `FF` |

---

## Credit System

Credits are stored in **bytes 6-7** as a **little-endian 16-bit value**.

### Formula:
```
raw_value = credits × 20
credits = raw_value ÷ 20
```

### Bucket Sizes:
- **Small bucket** = 1 credit
- **Medium bucket** = 2 credits  
- **Large bucket** = 3 credits

### Reference Table:

| Credits | Raw Value | Bytes 6-7 (LE) | Buckets |
|---------|-----------|----------------|---------|
| 30 (Fresh) | 600 (0x0258) | `58 02` | 30S / 15M / 10L |
| 28 | 560 (0x0230) | `30 02` | 28S / 14M / 9L |
| 24 | 480 (0x01E0) | `E0 01` | 24S / 12M / 8L |
| 23 | 460 (0x01CC) | `CC 01` | 23S / 11M / 7L |
| 20 | 400 (0x0190) | `90 01` | 20S / 10M / 6L |
| 10 | 200 (0x00C8) | `C8 00` | 10S / 5M / 3L |

---

## Checksum Algorithm

The checksum bytes (15-16) are calculated using the following formulas:

### Byte 15:
```
byte15 = (-16 - 5×byte6 + 119×byte7 - byte9 - byte10) mod 256
```

### Byte 16:
```
byte16 = (byte6 - 17×byte7 + 2×byte10 + 31) mod 256
```

### JavaScript Implementation:
```javascript
function calculateChecksum(eepromBytes) {
    const byte6 = eepromBytes[6];
    const byte7 = eepromBytes[7];
    const byte9 = eepromBytes[9];
    const byte10 = eepromBytes[10];
    
    const checksum15 = (-16 - 5*byte6 + 119*byte7 - byte9 - byte10) & 0xFF;
    const checksum16 = (byte6 - 17*byte7 + 2*byte10 + 31) & 0xFF;
    
    return { byte15: checksum15, byte16: checksum16 };
}
```

### Python Implementation:
```python
def calculate_checksum(eeprom_bytes):
    b6, b7, b9, b10 = eeprom_bytes[6], eeprom_bytes[7], eeprom_bytes[9], eeprom_bytes[10]
    
    byte15 = (-16 - 5*b6 + 119*b7 - b9 - b10) & 0xFF
    byte16 = (b6 - 17*b7 + 2*b10 + 31) & 0xFF
    
    return byte15, byte16
```

---

## Verified Samples

These samples were used to reverse-engineer the format:

### Sample 1: 28 Credits
```
Eeprom Data: 0E BF 99 00 01 01 30 02 00 24 CE 00 00 00 00 FC C9 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF
```
- Bytes 6-7: `30 02` = 0x0230 = 560 → 28 credits ✓
- Checksum: `FC C9` ✓

### Sample 2: 24 Credits
```
Eeprom Data: 0E BF 99 00 01 01 E0 01 00 A1 D0 00 00 00 00 96 8E FF FF FF FF FF FF FF FF FF FF EF FF FF FF FF
```
- Bytes 6-7: `E0 01` = 0x01E0 = 480 → 24 credits ✓
- Checksum: `96 8E` ✓

### Sample 3: 23 Credits
```
Eeprom Data: 0E BF 99 00 01 01 CC 01 00 22 D1 00 00 00 00 78 7C FF FF FF FF FF FF FF FF FF FF EF FF FF FF FF
```
- Bytes 6-7: `CC 01` = 0x01CC = 460 → 23 credits ✓
- Checksum: `78 7C` ✓

---

## Unknown / Notes

### Bytes 9-10 (Session Counter)
- Changes with each dispense
- Pattern observed: increments by ~129 per use
- May be used for anti-replay protection
- **Not fully understood** - more research needed

### ROM Data
- Unique to each physical iButton
- Format: `14 XX XX 9A 09 00 00 XX`
- Not part of the credit/checksum system

---

## Version History

- **v2.0** - Corrected checksum formulas (includes byte7), accurate credit reading
- **v1.0** - Initial analyzer with basic byte inspection

---

## Disclaimer

This tool is for educational and research purposes only. Use responsibly.
