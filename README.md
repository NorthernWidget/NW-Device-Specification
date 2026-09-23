# NW-Device-Specification

**Author:** Andy Wickert ([ORCID 0000-0002-9545-3365](https://orcid.org/0000-0002-9545-3365)), Northern Widget LLC
**License:** [CC BY-SA 4.0](LICENSE)

**Key words.** "Must", "must not", "should", and "may" carry the meanings of [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119): a "must" is a requirement without which a device or controller is not Schema 1; a "should" is the expected practice, departed from only for a reason the appendix records; a "may" is an option. Descriptive sentences ("the device clears ready") state what a conforming implementation does and are requirements all the same. The key words mark the places where the difference between a requirement and a recommendation matters.

**Status.** A section is in one of three states: **normative** (decided; a change is a revision recorded in [CHANGELOG.md](CHANGELOG.md)), **proposed** (drafted for a decision, and dated), or **hypothetical** (a placeholder with no device behind it). A proposed or hypothetical section says so in its heading or its first line; every other section is normative. As of 2026-09-23 the address space, the EEPROM layout, Page 0, Page 2 Block 0 with the Report kinds, and the Apis, Haar, Walrus, Libelle, and Margay appendices are normative; Tally is proposed; Okapi's Page 2, Liasis, and the controllers' addresses in the registry are hypothetical. The [conformance checks](#conformance-checks) name the harness case that exercises each rule.

This is a transport-agnostic specification for embedded device identity, communication, and data exchange on shared buses. Its target is low-power environmental sensors and data loggers, but any device that can expose a byte-addressable address space can implement it. The specification defines how a device presents itself – what it is, what version it runs, what it measures – and thereby lets a host (logger, microcontroller, or any I²C controller) discover it and interact with it automatically, without prior knowledge of the specific device.

---

## Background and history

This specification grew out of a decade of practice building open-source environmental sensors and data loggers at [Northern Widget](https://northernwidget.com). Three generations of design appear below. Each builds on the last.

### Schema 0: Margay serial number block (c. 2015, deployed)

The [Margay](https://github.com/NorthernWidget/Project-Margay) data logger was the first Northern Widget hardware to carry a structured identity block. A setup script writes that block at manufacture. It is an 8-byte serial number in the last 8 bytes of the ATmega1284p's EEPROM:

```
Offset  Field       Size  Notes
  0     Board type  2 B   High byte = ASCII initial of board name;
                          low byte = hardware revision index
                          (e.g. 0x4D03 = Margay v3.0)
  2     Group ID    2 B   Batch or deployment group
  4     Unique ID   2 B   Individual unit within the group
  6     FirmwareID  2 B   Always written as 0x0000; reserved
```

A programmer or setup jig reads this block; it is not exposed over I2C. It is a manufacturing-time convention, not a bus-communication protocol. Its significance here is twofold:

1. It established the serial number vocabulary (board type / group / unique ID) that Schema 1 inherits.
2. The `FirmwareID` field, always `0x0000`, means that every Margay ever programmed already carries `Schema = 0x00` in that slot: an accidental deployment of schema numbering before the concept was formalized.

Schema 0 is a Margay-only artifact. Sensors produced before Schema 1 typically have blank EEPROM (`0xFF`) and no structured identity block.

---

### Schema 1 prototype: Apis I2C register map (c. 2019, deployed)

The [Apis](https://github.com/NorthernWidget/Project-Apis) LiDAR rangefinder board was the first Northern Widget sensor to expose a structured I2C register map. The ATtiny1634 firmware populates a 32-byte array (`Reg[32]`) in SRAM and serves it over I2C via the WireS library. The 32-byte size is deliberate: it fits within the Arduino Wire library's default transaction buffer.

The register map as currently deployed:

```
Address  Field             Type    Notes
  0x00   Status            uint8   Bit 0 = ready (0 = initialising, 1 = ready)
  0x01   'A'               ASCII   ─┐
  0x02   'p'               ASCII    │ Device name, 4 bytes
  0x03   'i'               ASCII    │
  0x04   's'               ASCII   ─┘
  0x05   HW version major  uint8
  0x06   HW version minor  uint8
  0x07   FW patch version  uint8
  0x08   Range low byte    uint8   ─┐ int16 [cm], little-endian
  0x09   Range high byte   uint8   ─┘
  0x0A   Signal strength   uint8   From LiDAR Lite register 0x0E
  0x0B   Config            uint8   Sensitivity [bits 1:0], writable
  0x0C   I2C address       uint8   Writable; persisted to EEPROM
  0x0D   Reserved
  0x0E   Reserved
  0x0F   Reserved
  0x10   Accel X low       uint8   ─┐ int16, little-endian
  0x11   Accel X high      uint8   ─┘
  0x12   Accel Y low       uint8   ─┐
  0x13   Accel Y high      uint8   ─┘
  0x14   Accel Z low       uint8   ─┐
  0x15   Accel Z high      uint8   ─┘
  0x16   Reserved
  0x17   Reserved
  0x18   Offset X low      uint8   ─┐ int16, little-endian; from EEPROM
  0x19   Offset X high     uint8   ─┘
  0x1A   Offset Y low      uint8   ─┐
  0x1B   Offset Y high     uint8   ─┘
  0x1C   Offset Z low      uint8   ─┐
  0x1D   Offset Z high     uint8   ─┘
  0x1E   Reserved
  0x1F   Reserved
```

This prototype introduced the key concepts that Schema 1 formalises: a fixed device name at known addresses, hardware and firmware version bytes, a ready flag, and a writable I2C address register. Its limitations motivated the design below. Status at 0x00 left no room for a schema byte, identity and sensor data shared a single page, and the map carried no serial number and no integrity check.

---

### Schema 1: NW-Device-Specification v1 (2026)

The full specification, described in detail in the sections that follow.

---

## Design principles

- **Transport-agnostic.** The specification defines a virtual byte-addressable address space. I2C, RS-485, SPI, or any byte-serial transport may carry it.
- **Fixed 32-byte pages.** Pages are the atomic read unit. 32 bytes is the lowest-common-denominator single-transaction read on stock Arduino hardware (Wire buffer limit). Fixed page size means fixed offsets, no parser, and no seek: a corrupt byte damages only its own field.
- **Auto-discovery.** Your controller scans addresses, reads 8 bytes (Block 0 of Page 0), and immediately knows whether a device is NW-schema-compliant and what it is. It needs no prior knowledge.
- **Layered.** A controller that only needs identity reads Page 0. A controller that needs measurements reads Page 2. Future schemas add pages, and old controllers ignore pages they don't know.
- **No central registrar.** A 7-byte ASCII name at a fixed address establishes device identity. You need no central manufacturer ID registry.

---

## Address space

Your device exposes a flat, byte-addressable virtual address space. The controller writes a starting address, then reads N bytes (maximum 32 per transaction). The device increments the address pointer with each byte returned.

Pages are 32-byte aligned:

| Page   | Addresses   | Contents                  | Backing store |
|--------|-------------|---------------------------|---------------|
| Page 0 | 0x00–0x1F   | Identity (this document)  | Persistent (EEPROM or equivalent) |
| Page 1 | 0x20–0x3F   | Calibration               | Persistent (EEPROM or equivalent) |
| Page 2 | 0x40–0x5F   | Sensor data: Block 0 (status and control), then data | SRAM (runtime) |
| Pages 3–5 | 0x60–0xBF | Sensor data, continued (devices with more than 24 bytes of data): up to 120 data bytes in all | SRAM (runtime) |
| Pages 6–7 | 0xC0–0xFF | Reserved, universal: undefined until a need has a device (2026-09-23) | – |

One rule fixes the backing store: **0x00–0x3F is stored, 0x40 and above is served.** The stored half is a single 64-byte image, byte for byte the top 64 bytes of the device's EEPROM in the same order (see [Physical EEPROM layout](#physical-eeprom-layout)); the served half is the firmware's register array in SRAM, rewritten at every reading and gone at power-off. The schema byte (Page 0, address 0x00) declares which pages a device exposes.

> Renumbered on 2026-09-23, before any release: calibration moved from Page 2 to Page 1 and sensor data from Pages 1 and 3 to Pages 2 and 3, so that the stored pages are contiguous on the bus as they are in EEPROM, and data continues upward without a gap. Data owns Pages 2–5, 120 bytes, four times the largest device on the list (Libelle with a bridged accelerometer, 28 bytes); a device that outgrows them is a case for a two-byte register pointer in a later schema, not for the last two pages, which stay reserved for whatever universal need first has a device. Nothing provisioned or deployed carried the earlier numbering; the schema byte stays 0x01.

---

## Physical EEPROM layout

The two stored pages are one 64-byte image at the **top of EEPROM**: Page 0 at `EEPROM[length−64]` through `EEPROM[length−33]`, Page 1 directly above it at `EEPROM[length−32]` through `EEPROM[length−1]`. The bus address and the EEPROM address differ by one constant:

```cpp
#define STORED_BASE  (EEPROM.length() - 64)

// Read the byte at bus address i (0x00–0x3F: Page 0 identity, Page 1 calibration):
EEPROM.read(STORED_BASE + i)

// Examples:
EEPROM.read(STORED_BASE + 0x00)  // Schema byte
EEPROM.read(STORED_BASE + 0x1E)  // CRC-8 of Page 0
EEPROM.read(STORED_BASE + 0x1F)  // I²C address
EEPROM.read(STORED_BASE + 0x20)  // first calibration byte (Page 1, Block 0)
```

The top of EEPROM protects identity and calibration from any code that writes EEPROM sequentially from address 0, a pattern common in user sketches on loggers and in firmware bugs. A dedicated programmer sketch, or [NW-Provision](https://github.com/NorthernWidget/NW-Provision), writes Page 0 once at manufacture; your production firmware must never write to Page 0 except the I²C address at 0x1F. Page 1 must be written by the firmware only when a calibration is stored (Apis's zero), and by nothing else. Keep all other device-specific EEPROM usage **below** `EEPROM[length−64]`. Devices with no calibration data still reserve Page 1 (`0xFF` unprogrammed or `0x00` by convention), so every device maps the same way. The CRC-8 at Page 0 offset 0x1E must be checked at every boot.

---

## Device provisioning reference

The table below lists the MCU, EEPROM size, and avrdude part identifier for each NW device. EEPROM size determines where the stored image lives (`EEPROM[length−64]`: Page 0, then Page 1). [NW-Provision](https://github.com/NorthernWidget/NW-Provision) uses the avrdude part, which also serves as a hardware safety check: avrdude verifies the chip's signature bytes against the specified part and refuses to program a mismatch.

| Device  | MCU          | EEPROM | Stored image offset | avrdude part |
|---------|-------------|--------|--------------|-------------|
| Margay  | ATmega1284P | 4096 B | 0x0FC0       | `m1284p`    |
| Okapi   | ATmega1284P | 4096 B | 0x0FC0       | `m1284p`    |
| Apis    | ATtiny1634  | 256 B  | 0x00C0       | `t1634`     |
| Haar    | ATtiny1634  | 256 B  | 0x00C0       | `t1634`     |
| Walrus  | ATtiny1634  | 256 B  | 0x00C0       | `t1634`     |
| Libelle | ATtiny841   | 512 B  | 0x01C0       | `t841`      |
| Liasis  | – (no MCU) | – | – | – |

---

## Transport

This specification is transport-agnostic. The 32-byte page layout is a data structure. The physical layer is separate.

### I²C (primary transport)

Your controller performs a register-addressed read: write the starting address (e.g., `0x00` for Page 0, `0x40` for Page 2), then clock out up to 32 bytes. The device increments its address pointer with each byte returned. This is the standard NW sensor peripheral interface.

### UART

UART has no inherent framing: a bare address byte will be misinterpreted if the line carries other traffic. You therefore need a framing layer. Two options are supported:

**COBS (Consistent Overhead Byte Stuffing)**: a formal standard that encodes data such that `0x00` never appears in the payload, which makes `0x00` an unambiguous frame boundary. Arduino libraries are available. It is recommended when interoperability or formal correctness matters.

**Magic Preamble**: a two-byte sync sequence (`0xAA 0x55`) that cannot appear in normal ASCII traffic, followed by a page-address byte. It is simple to implement by hand.

Magic Preamble frame format:

```
Request  (controller → device):  [0xAA][0x55][page_addr]           3 bytes
Response (device → controller):  [0xAA][0x55][page_addr][32 bytes][CRC-8]  36 bytes
```

The preamble in the response lets the controller re-synchronise if it misses the start. Both COBS and Magic Preamble framing are acceptable NW UART conventions. The choice is per-device, and you should document it in the device's own specification.

---

## Page 0: Identity

32 bytes, persistent. Written at manufacture and rarely changed. Organised as four 8-byte blocks.

### Block 0 (0x00–0x07): Core identity

Read this block first. 8 bytes, one transaction. Sufficient to identify any compliant device.

```
Address  Field   Size  Contents
  0x00   Schema  1 B   NW schema version (0x01 for this specification)
  0x01   Name    7 B   ASCII device name, null-padded if shorter than 7 characters
  –0x07
```

The schema byte at 0x00 is the first thing your controller reads. Its decision tree:

| Value | Meaning | Action |
|-------|---------|--------|
| `0x01` | Schema 1 (this document) | Parse Pages 0–N per spec |
| `0x00` | Schema 0: Margay legacy SN convention | Not I²C auto-detectable; skip |
| `0xFF` | Unprogrammed EEPROM | Not auto-detectable; skip |
| `0x02`–`0xFE` | Future or unknown schema | Skip, or parse if the controller knows that version |

**`0x00` and `0xFF` are permanently reserved and will never be assigned to a valid schema.** `0x00` is the erased-then-overwritten default of the Margay serial number block (Schema 0), and `0xFF` is the hardware default of blank EEPROM. Both are unambiguously "not auto-detectable"; your controller need not distinguish between them. Valid schemas therefore occupy `0x01`–`0xFE`: 254 values, sufficient for any conceivable rate of fundamental protocol change.

The 7-byte name field accommodates all current Northern Widget device names without truncation. A fixed-position name at a fixed address gives negligible collision probability with non-compliant devices, and no manufacturer prefix is required. Reserved bytes must be written as `0x00` at manufacture. `0xFF` indicates unprogrammed EEPROM.

### Block 1 (0x08–0x0F): Version

```
Address  Field     Size  Contents
  0x08   HW major  1 B   Hardware version major
  0x09   HW minor  1 B   Hardware version minor
  0x0A   FW/HW patch 1B  Combined-repo devices: firmware patch version (NW convention).
                         Separate-repo devices: hardware patch version.
  0x0B   FW major  1 B   Firmware major (0x00 if combined repo)
  0x0C   FW minor  1 B   Firmware minor (0x00 if combined repo)
  0x0D   FW patch  1 B   Firmware patch (0x00 if combined repo)
  0x0E   Reserved  1 B   0x00
  0x0F   Reserved  1 B   0x00
```

**Combined-repo convention (Northern Widget):** Hardware and firmware share a single repository and a single version tag (M.m.F, where M.m = hardware version and F = firmware patch). Write the firmware patch to 0x0A. Leave 0x0B–0x0D as 0x00.

**Separate-repo convention:** Write full SemVer for hardware (0x08–0x0A) and firmware (0x0B–0x0D) independently.

### Block 2 (0x10–0x17): Serial number

```
Address  Field        Size  Contents
  0x10   Board type   2 B   High byte: ASCII initial of device name.
                            Low byte: hardware revision index (0, 1, 2, …).
  0x12   Group ID     2 B   Batch or deployment group identifier.
  0x14   Unique ID    2 B   Individual unit identifier within the group.
  0x16   FirmwareID   2 B   Legacy field; write as 0x0000. Reserved for future use.
```

The serial number block follows the convention that the Margay data logger established (Schema 0). Preserving this layout keeps it consistent with existing NW manufacturing records. Group ID and unique ID assignment are the device manufacturer's responsibility: no central registry is required, and uniqueness within a deployment is the practical requirement.

### Block 3 (0x18–0x1F): Integrity and administration

```
Address  Field       Size  Contents
  0x18   Build       4 B   The first four bytes of the git commit the firmware was
  –0x1B  commit            built from (the first eight hex digits of its SHA-1),
                           or 0x00 when the build recorded none.
  0x1C   Build flags 1 B   bit 0: the working tree held uncommitted changes when
                           the firmware was built (a "dirty" build). Bits 1–7: 0x00.
  0x1D   Magic byte  1 B   Fixed NW marker: always 0x4E (ASCII 'N'). A controller
                           may check this byte as a quick sanity test before
                           computing the full CRC-8.
  0x1E   CRC         1 B   CRC-8/SMBUS over bytes 0x00–0x1D (polynomial 0x07,
                           init 0x00, no reflection, no final XOR). Standard variant
                           used by common I²C sensors (SHT3x, HDC1080, etc.);
                           no lookup table required for a 32-byte input.
  0x1F   I2C address 1 B   Writable. The device's current I2C address, persisted
                           to EEPROM. Takes effect on next boot. Falls back to
                           the device's default address if 0xFF (unprogrammed).
```

Reference implementation (C):

```c
uint8_t crc8_smbus(const uint8_t *data, uint8_t len) {
    uint8_t crc = 0x00;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t b = 0; b < 8; b++)
            crc = (crc & 0x80) ? (crc << 1) ^ 0x07 : (crc << 1);
    }
    return crc;
}
/* Usage: crc8_smbus(page0, 0x1E) should equal page0[0x1E] */
```

**The served copy of Page 0.** Your device serves Page 0 from SRAM, not from EEPROM. At boot the firmware copies the 64-byte stored image into its register array, checks the CRC of the copy, and then rewrites the three things that belong to the firmware rather than to the board: the firmware patch at 0x0A, the build commit at 0x18–0x1B, and the build flags at 0x1C. It recomputes the CRC at 0x1E over the served bytes, and a controller that checks the CRC therefore checks the copy it reads. EEPROM keeps zeros in those six bytes (NW-Provision writes them as 0x00), and the firmware never writes them back. The build wrapper ([NW-Build](https://github.com/NorthernWidget-Skunkworks/NW-Build)) defines `FW_COMMIT` from git at compile time, with a trailing `+` when the tree is dirty; an Arduino IDE build leaves it blank, the bytes stay zero, and a library prints an empty FWCommit column. Furthermore, the same wrapper defines `<LIB>_LIBRARY_COMMIT` for every library it compiles, which is how the Lib and LibCommit columns of a status file are filled (see [LIBRARY-DESIGN.md](LIBRARY-DESIGN.md)).

---

## Page 1: Calibration

32 bytes, persistent (EEPROM-backed). Written at calibration time, and read by your controller when it wishes to verify or update calibration state. Device-specific and defined per device type.

---

## Page 2: Sensor data

32 bytes, SRAM-backed, rewritten by the device on every reading. Block 0 is universal – identical in meaning on every NW device – and Blocks 1–3 (0x48–0x5F, 24 bytes) carry the device's data, defined per device type in its appendix. A device with more than 24 bytes of data continues on Page 3 (see [Address space](#address-space)).

A *reading* is one acquisition of every measurement the device reports, at one moment. A *chip* is one sensing IC on the board. Each appendix numbers its chips in a fixed order, and the status, control, and fault bytes below use that index.

### Block 0 (0x40–0x47): Status and control

```
Address  Field         Access      Contents
  0x40   Status        read-only   bit 0    ready:      1 = data registers hold a complete reading
                       (live)      bits 1–6 chip fault: bit n set = chip n−1 is faulted now
                                   bit 7    pan-fault:  OR of bits 1–6
  0x41   Control       writable    bit 0    trigger:     controller writes 1 to request a reading;
                                                         the device clears it when the reading starts
                                   bits 1–6 chip select: bit n set = measure chip n−1 on the next
                                                         reading; power-up value = every chip present
                                   bit 7    sleep:       device enters its lowest-power state after
                                                         this transaction, wakes on its next I²C
                                                         address match, and clears the bit
  0x42   Reading       read-only   uint16, little-endian. Incremented by one when ready is set.
  0x43   counter                   0 after power-up. Wraps at 65535.
  0x44   Readings      writable    uint16, little-endian. How many readings the controller will
  0x45   requested                 trigger with the device's chips held powered: the device powers
                                   the selected chips up at the first trigger after this write and
                                   down once the reading counter has advanced by this amount.
                                   0 (the power-up value) = one reading per trigger, powered down
                                   after each. A new write replaces the remainder.
  0x46   Config        writable    Device-specific configuration, 8 bits defined by the appendix.
                                   0x00 = the device's defaults. Volatile: the controller sets it
                                   after every power-up. A device needing more than 8 bits of
                                   configuration places the rest in its own data area, never here.
  0x47   Report        read-only   Latched code for the device's most recent report, good or
                       (latched)   bad, held until the controller acknowledges it; 0x00 = none.
                                   bits 7–5 chip index (0–6), or 7 = the unit itself
                                   bits 4–0 kind (table below)
```

**Report kinds** (bits 4–0 of 0x47). A report is a *fault* when the device also sets the chip's Status bit for that reading (the data are not to be trusted); a report that sets no Status bit is a *notice*, news the controller should record but the data stand.

| Kind | Meaning | Note word | Printed as |
|------|---------|-----------|------------|
| 0 | No report | None | none |
| 1 | Chip did not acknowledge on its bus | NotAnswering | not answering |
| 2 | Conversion timeout | Timeout | timed out |
| 3 | Checksum or CRC failure reported by the chip; on the unit, Page 0 failed its check | ChecksumFailed; on the unit, Page0Invalid | checksum failed; Page 0 invalid |
| 4 | Value outside the chip's valid range | OutOfRange | out of range |
| 5 | Chip not initialised or failed self-test | SelfTestFailed | self-test failed |
| 6 | Device reset since the controller last wrote Control (configuration lost); a notice | Restarted | restarted since configured |
| 7 | Configuration write rejected | ConfigRejected | configuration rejected |
| 8 | Supply or power-good fault | PowerFault | power fault |
| 9 | Calibration stored (the device wrote Page 1; a notice) | CalibrationStored | calibration stored |
| 10 | Batch abandoned: the controller stopped triggering and the device powered its chips down (a notice) | BatchAbandoned | batch abandoned |
| 11–15 | Reserved, universal | | |
| 16–31 | Device-specific, defined in the appendix, which says whether each is a fault or a notice, and gives its note word | | |

A library writes a report into a logger's note column as one token, the chip name followed by the note word: `LiDARNotAnswering`, `SHT31ChecksumFailed`, `AccelCalibrationStored`, `UnitRestarted`. Chip names come from the appendix's chip table, and the table names a chip by its function when the device has one of its kind (Clock, Battery, SDCard, LiDAR, Accel) and by its part number when only the part tells two apart (SHT31 and LPS35HW, MS5803 and MCP9808, VEML6075 and VEML6030).

**Rules**

- **Writable bytes.** Your device must accept writes only at 0x41, 0x44–0x45, and 0x46 on Page 2, and must ignore writes elsewhere: the firmware checks the register address in its receive handler.
- **Readings requested (0x44–0x45).** Your controller writes the number of readings it is about to trigger, then triggers them one by one through the normal handshake. The device pays its chips' power-up and initialisation once, at the first trigger, and powers them down when the counter has advanced by the requested amount, and a single reading (0, or 1) is powered up and down around that one reading. The device keeps no idle timer: the count is the whole contract, and a controller that stops mid-batch therefore cannot leave a chip powered beyond the device's own fault fallback (a batch that does not complete within a device-defined time is abandoned, the chips powered down, and a fault latched). The count and the reading counter both count unit readings and move at the same moment. A single reading is a batch of one, and is the default. A chip selected at any trigger of a batch stays powered until the batch ends, the selection may change from trigger to trigger, and unselected chips keep their previous data.
- **Ready and the counter.** The device clears ready the moment a reading begins, whether the controller triggered it or the device's own timer started it, and sets it when the data registers are complete. The reading counter increments at that same moment, after the data is in place, and your controller can therefore read the counter and the data in one transaction and never see a new count with old data. When the data take more than one transaction (a second page, or a read longer than the Wire buffer), or when the device starts readings on its own timer, your controller must read the counter again after the data: if it still holds the value captured before, every byte belongs to that reading; if it moved, capture and read again. The retries should be bounded, and the controller then gives up (NW_Core `readData()` allows two).
- **Atomic rewrite.** Your device must rewrite the data registers and increment the counter with interrupts disabled, so that a page read never straddles a rewrite.
- **Trigger.** A trigger written while a reading is in progress stays set, and the device honours it when the current reading completes. A trigger written while ready is set starts a new reading and clears ready: the controller never confuses the previous reading with the one it requested. A device may also start readings on its own schedule, and the trigger then adds one immediate reading without changing that schedule.
- **Chip count.** Block 0 addresses up to six chip groups: six select bits, six fault bits, and a three-bit chip field with 7 meaning the unit. Chips that are always read together share an index. If your device has more than six independently selectable groups, define a second select byte and a second fault byte in its own data area and describe them in its appendix. Block 0 covers the first six and never changes.
- **Faults and reports.** Status bits 1–6 show which chips are faulted *now* and clear when the chip next succeeds. The Report register at 0x47 holds the device's most recent report until the controller acknowledges it, which keeps a fault that cleared itself between readings visible and carries notices that no live bit could. Any write to Control acknowledges: it clears 0x47 to 0x00. If your controller triggers readings, it therefore acknowledges on every request. If it only reads a free-running device, it acknowledges whenever it sets chip select or sleep. Your controller must read Block 0 once before its first write, so that the reports the device made at boot (a reset, an invalid Page 0) reach it before the first acknowledgement.
- **One report at a time.** The register holds one code. A fault overwrites a notice; a notice must never overwrite an unacknowledged fault, and the device repeats the notice at its next reading if it still applies. Whatever a notice announces (a stored calibration, an abandoned batch) is also visible in the device's data or Page 2, so a lost notice loses nothing but the moment.
- **Sleep.** After a transaction that sets bit 7, the device completes the transaction, then enters its lowest-power state. The ATtiny TWI slave wakes on address match, and no timer is needed, but the first transaction after waking may see a delayed acknowledge.
- **Power-up state.** Status 0x00 (not ready), Control with every present chip selected, counter 0, Config 0x00, Report 0xE6 (the unit reset since the controller last configured it), or 0xE3 if Page 0 failed its check.

Your controller reads all 32 bytes of Page 2 in one transaction, checks ready first, then the pan-fault bit, and only then uses the data. Bit 7 gives a fault summary without your controller knowing the device's chip assignments. The Report register gives the detail, and the news, when it wants them.

### Blocks 1–3 (0x48–0x5F): Device data

The appendices define these per device. Values are little-endian, in the types and scaled units the appendix states. Each appendix also provides a numbered **chip table**, which fixes the index used by the status fault bits, the control chip-select bits, and the Report register.

---

## Prior art

The design took the following existing standards into account:

- **IEEE 1451 / TEDS** is the closest philosophical precedent: smart-transducer self-identification, an EEPROM-resident identity block, and media independence. This specification differs in three ways: it is open and unencumbered (IEEE 1451 is paywalled), it uses a flat byte map rather than bit-packed templates, and it co-locates runtime data with identity rather than separating them entirely.
- **I2C Device ID** (reserved address 0xF8) is the closest base-I2C primitive: a 3-byte identifier (12-bit manufacturer / 9-bit part / 3-bit revision). This specification extends that concept to a full identity page.
- **SMBus ARP and UDID** provide bus-native device discovery and dynamic address assignment, analogous to the writable address register at 0x1F. They are widely considered heavyweight, and this specification offers a lighter scan-and-read alternative.
- **IPMI FRU** is the structural reference for area-based versioning and extensibility. This specification borrows the extensibility philosophy (schema versioning allows old controllers to skip unknown pages) but rejects FRU's variable-length offset-chained block structure in favour of fixed-position fields.
- **JEDEC SPD**, a fixed byte map in EEPROM over SMBus, is the closest historical precedent for Page 0.
- **Adafruit STEMMA QT / SparkFun Qwiic**: these ecosystems standardised I2C connectors and voltage levels but not device identity or discovery. Devices in these ecosystems rely on fixed I2C address, chip-specific WHO_AM_I registers, and human-maintained conflict lists for identification. This specification provides the missing auto-discovery layer.

---

## Per-device appendices

### Apis (LiDAR rangefinder + accelerometer)

#### Page 0

```
Block 0:  Schema=0x01, Name='A','p','i','s',0x00,0x00,0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x4100 ('A'=0x41, rev 0), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=0x41
```

Legacy deployed units carry board type `0x6C00` and I²C address `0x50` (pre-Schema-1).

#### Page 1 (0x20–0x3F): Calibration

```
Block 0 (0x20–0x27)   Current zero: Offset X, Y, Z int16 LE; accel temperature word int16   (as today)
Block 1 (0x28–0x2F)   Previous zero, same form
Block 2 (0x30–0x37)   The zero before that, same form
Block 3 (0x38–0x3F)   0x38–0x39 zero generation, uint16 LE: zeros stored since manufacture (0 = never);
                      0x3A–0x3F reserved
```

The current zero in full (firmware patch 3): Offset X at 0x20–0x21, Y at 0x22–0x23, Z at 0x24–0x25, each little-endian int16, and at 0x26–0x27 the accelerometer temperature word when the offsets were taken. Storing a zero (firmware patch 5) shifts Block 1 to Block 2 and Block 0 to Block 1, writes the new zero into Block 0, and adds one to the generation (a blank 0xFFFF reads as 0), byte by byte with compare-before-write; the served page and the mirror at 0x58 follow at once, and notice 0x29 (calibration stored) is latched as before. A blank word anywhere on the page reads as 0.

---

#### Page 2 (0x40–0x5F): Sensor data

Chip table (index used by status bits 1–6, control chip-select bits 1–6, and the Report register):

| Index | Chip | Measurements |
|-------|------|--------------|
| 0 | LiDAR Lite v3HP | range, signal strength |
| 1 | LIS3DH accelerometer | X, Y, Z |

Block 0 (0x40–0x47) is the universal block. Config (0x46): bits 1:0 = LiDAR sensitivity (as the former register 0x45); bits 7:2 reserved. Firmware patch 1 clears the sleep bit without sleeping (implementation deferred) and free-runs every 100 ms in addition to answering triggers.

**Run model (firmware patch 2, Project-Apis #23).** The unit is on-demand: it idles until a trigger, and there is no free-running cycle. The firmware powers the LiDAR through the board's 5 V switch and its enable pin only while readings are being taken, per the readings-requested word (0x44–0x45): powered up at the first trigger, powered down when the requested count is done. Power-up sequence: (1) 5 V switch on, (2) a short wait for the rail (680 µF through the MIC2544 at its ~227 mA limit), (3) enable high, and (4) a poll of the LiDAR for an I²C acknowledge and the health flag in its STATUS register (0x01 bit 5) rather than a fixed delay. On timeout the firmware toggles the enable once more, and a second failure powers the LiDAR down, latches fault chip 0 kind 1 (no acknowledge) or 5 (not initialised), and completes the reading with range −9999. Each acquisition writes ACQ_COMMAND (0x00, where any non-zero value starts a measurement on the v3HP) and polls STATUS bit 0 (busy) until clear before reading the distance registers. The LiDAR's mode pin is not used (on this board it is held high through a 1 kΩ resistor and cannot indicate busy). The firmware reads the accelerometer on every reading in which it is selected. Serial output exists only in debug builds.

**Inclination (firmware patch 3, 2026-09-23).** The LIS3DH runs at 10 Hz (5 Hz bandwidth, about 0.5 mg rms per sample) and each reading waits for a fresh sample, so consecutive readings are independent. Its temperature sensor is on, and the OUT_ADC3 word is served beside the axes at 0x56 for a per-unit drift correction; a magnet at the Hall switch, at boot or later, starts a zero that averages samples until the standard error of every axis mean is below 0.25 counts (at least 32, at most 1000 samples) and stores its temperature at 0x26; the magnet need not stay. Patch 4 reports through the Report register: kind 9 on chip 1 after a zero is stored (Page 2 holds it), kind 10 on chip 0 when a batch is abandoned (the LiDAR powered down because the controller stopped triggering); a notice never overwrites an unacknowledged fault. Patch 5 keeps a record of the zeros: Page 1 holds the current zero and the two before it with a generation count, and the generation is mirrored into Page 2 Block 3 so that a logger sees a new zero in the reading itself; the library requires patch 5.

```
Block 1 (0x48–0x4F)   LiDAR Lite
  0x48–0x49   Range [cm], little-endian int16
  0x4A        Signal strength, uint8
  0x4B–0x4F   Reserved

Block 2 (0x50–0x57)   Accelerometer
  0x50–0x51   Accel X, little-endian int16
  0x52–0x53   Accel Y, little-endian int16
  0x54–0x55   Accel Z, little-endian int16
  0x56–0x57   Accel temperature, the LIS3DH OUT_ADC3 word as read (low byte first): relative,
              1 digit per °C in the high byte (firmware patch 3; the reference for a drift correction)

Block 3 (0x58–0x5F)   0x58–0x59 zero generation, uint16 LE, the same value, served with every reading
  0x5A–0x5F   Reserved
```

### Haar (temperature, pressure, relative humidity)

#### Page 0

```
Block 0:  Schema=0x01, Name='H','a','a','r',0x00,0x00,0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x4801 ('H'=0x48, rev 1), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=0x48
```

#### Page 2 (0x40–0x5F): Sensor data

Chip table:

| Index | Chip | Measurements |
|-------|------|--------------|
| 0 | SHT31 | temperature, relative humidity |
| 1 | LPS35HW | pressure, temperature |

Block 0 (0x40–0x47) is the universal block. Config (0x46): no bits defined. Write 0x00.

```
Block 1 (0x48–0x4F)   SHT31: temperature and humidity
  0x48–0x49   Temp SHT31, int16, 0.01 °C, little-endian
  0x4A–0x4B   Humidity, uint16, 0.01 % RH, little-endian
  0x4C–0x4F   Reserved

Block 2 (0x50–0x57)   LPS35HW: pressure and temperature
  0x50–0x53   Pressure, uint32, 0.01 hPa, little-endian
  0x54–0x55   Temp LPS35HW, int16, 0.01 °C, little-endian
  0x56–0x57   Reserved

Block 3 (0x58–0x5F)   Reserved
```

> **Migration note:** the firmware on `master` has implemented this map since 2026-09-23 (patch 1): Page 0 from EEPROM with CRC check, the writable-register rule with the address persisted, 32-byte page reads, and the Block 0 handshake (on-demand trigger only, chip select, reading counter, live status bits, latched fault byte: SHT31 no-acknowledge or checksum, LPS35HW no-acknowledge or timeout, unit reset at boot). Data moved from the legacy 0x02–0x0A raw counts to 0x28–0x35 in the units above (0x48–0x55 since the 2026-09-23 renumbering), and the default address moved from 0x42 to 0x48. The firmware accepts the readings-requested word and the sleep bit without effect. It is unreleased and not yet validated on hardware.

No Page 1. Both sensors are factory-calibrated. There is no user calibration step.

> **I²C address note:** `0x48` (`'H'`) is also a common address for the ADS1115 ADC. There is no conflict among NW devices. If your system independently uses an ADS1115 on the same bus, configure the ADS1115 to a different address (ADDR pin to GND = `0x48`, VDD = `0x49`, SDA = `0x4A`, SCL = `0x4B`; avoid `0x48`).

**LPS35HW pressure sensor characteristics:**

| Parameter | Value |
|-----------|-------|
| Sensor range | 260–1260 hPa |
| Raw LSB | 1/4096 hPa ≈ 0.000244 hPa (0.024 Pa) |
| Stored unit | 0.01 hPa (1 Pa) |
| Stored range | 26,000–126,000 |
| Effective noise floor | ~0.002 hPa RMS @ 1 Hz |

Unit rationale: 0.01 hPa is the natural meteorological unit, and it aligns with the LPS35HW's native hPa output. The Walrus pressure register uses µBar (see below). The difference is intentional: each unit matches its sensor's native output format.

---

### Walrus (submersible pressure + temperature)

#### Page 0

```
Block 0:  Schema=0x01, Name='W','a','l','r','u','s',0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x5702 ('W'=0x57, rev 2), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=0x57
```

#### Page 2 (0x40–0x5F): Sensor data

Chip table:

| Index | Chip | Measurements |
|-------|------|--------------|
| 0 | MS5803 | pressure, temperature |
| 1 | MCP9808 | external (water) temperature |

Block 0 (0x40–0x47) is the universal block. Config (0x46): bits 1:0 = free-running update period, 0 = 5 s, 1 = 10 s, 2 = 60 s, 3 = 300 s (as the former control register 0x00); bits 7:2 reserved.

```
Block 1 (0x48–0x4F)   MS5803: pressure and temperature
  0x48–0x4B   Pressure, int32, µBar, little-endian
  0x4C–0x4D   Temp MS5803, int16, 0.01 °C, little-endian
  0x4E–0x4F   Reserved

Block 2 (0x50–0x57)   MCP9808: external temperature
  0x50–0x51   Temp ext, int16, 0.01 °C, little-endian
  0x52–0x57   Reserved

Block 3 (0x58–0x5F)   Reserved
```

> **Migration note:** firmware and library on `master` moved this data from 0x22 to 0x28, and the update-period configuration from 0x00 to 0x26, on 2026-09-21 (unreleased; 0x48 and 0x46 since the 2026-09-23 renumbering). On 2026-09-23 the firmware (patch 1) gained Page 0 from EEPROM with CRC check, the writable-register rule with the address persisted to EEPROM, 32-byte page reads, and the Block 0 handshake: trigger or free-running timer, chip select, reading counter, live status bits and the latched fault byte (kind 1 per chip, unit kind 6 at boot, kind 3 for an invalid Page 0); later that day the data registers began to update all at once with the counter (the atomic-rewrite rule) instead of field by field. The firmware accepts the readings-requested word and the sleep bit without effect. Walrus_Library moved onto NW_Core the same day: begin() gates on Page 0 (minimum patch 1), one triggered reading per getString() through the counter, chip faults to -9999. Hardware validation is Project-Walrus #18.

No Page 1. MS5803 calibration coefficients are read from its internal PROM at startup. The MCP9808 is factory-calibrated.

**MS5803 pressure sensor characteristics by variant:**

| Variant | Application | Range | Resolution @ OSR=4096 | Stored range (µBar) |
|---------|-------------|-------|-----------------------|---------------------|
| MS5803-01BA | Barometer | 10–1300 mbar | ~0.012 mbar (12 µBar) | 10,000–1,300,000 |
| MS5803-02BA | Shallow water | 0–2 bar | ~0.020 mbar (20 µBar) | 0–2,000,000 |
| MS5803-05BA | Medium depth | 0–6 bar | ~0.050 mbar (50 µBar) | 0–6,000,000 |
| MS5803-14BA | Deep water | 0–14 bar | ~0.200 mbar (200 µBar) | 0–14,000,000 |

All variants fit within int32 (~2.1 billion µBar max). The -01BA installed in a Walrus makes it a high-resolution barometer. Installed -02BA/-05BA variants measure water column depth (1 mbar ≈ 1 cm water).

Unit rationale: µBar is the MS5803 library's internal `_pressure_actual` unit, and it requires no conversion in firmware. The Haar pressure register uses 0.01 hPa. The difference is intentional: each unit matches its sensor's native output format.

---

### Libelle (multi-spectral shortwave pyranometer)

#### Page 0

```
Block 0:  Schema=0x01, Name='L','i','b','e','l','l','e'  (exact 7-byte fit)
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x4C01 ('L'=0x4C, rev 1), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=0x4C (UP) or 0x0C (DOWN)
          [DOWN = 'L' (0x4C) XOR 0x40; see I²C address registry for secondary address scheme]
```

#### Page 2 (0x40–0x5F): Sensor data

Chip table:

| Index | Chip | Measurements |
|-------|------|--------------|
| 0 | VEML6075 | UVA, UVB |
| 1 | VEML6030 | ambient light, white |
| 2 | ADS1115 | IR short, IR mid, thermistor temperature |
| 3 | ADXL343 | X, Y, Z (hardware v2 only) |

Block 0 (0x40–0x47) is the universal block. Config (0x46): bits 1:0 = free-running update period (0 = 5 s, 1 = 10 s, 2 = 60 s, 3 = 300 s); bit 2 = auto-range disable (0 = auto-range each reading, as the former control register); bit 3 = run auto-range once now (self-clearing); bits 7:4 reserved.

```
Block 1 (0x48–0x4F)   VEML6030: visible light
  0x48–0x49   ALS, uint16, raw VEML6030 counts, little-endian
  0x4A–0x4B   White, uint16, raw VEML6030 counts, little-endian
  0x4C–0x4D   Lux mult, uint16, auto-range scaler (ALS × mult × 0.0036 → lux)
  0x4E–0x4F   Reserved

Block 2 (0x50–0x57)   VEML6075: UV
  0x50–0x53   UVA, int32, compensated counts, little-endian
  0x54–0x57   UVB, int32, compensated counts, little-endian

Block 3 (0x58–0x5F)   ADS1115: IR and temperature
  0x58–0x59   IR Short, uint16, raw ADC counts (×1.25e-4 → V)
  0x5A–0x5B   IR Mid, uint16, raw ADC counts (×1.25e-4 → V)
  0x5C–0x5D   Temperature, uint16, raw ADC counts (Steinhart-Hart → °C)
  0x5E–0x5F   Reserved

Page 3, Block 0 (0x60–0x67)   ADXL343: accelerometer (hardware v2 only)
  0x60–0x61   Accel X, int16, little-endian
  0x62–0x63   Accel Y, int16, little-endian
  0x64–0x65   Accel Z, int16, little-endian
  0x66–0x67   Reserved
Page 3, Blocks 1–3 (0x68–0x7F)   Reserved
```

No Page 1. Calibration constants (Steinhart-Hart coefficients, UV cross-talk compensation) are hardcoded in the library. If you add per-unit calibration, Page 1 is the natural home.

> **Hardware v1 note:** The ADXL343 accelerometer is wired to the controller I2C bus (addresses 0x1D / 0x53), not bridged through the ATtiny register map. v1 hardware does not serve Page 3. Hardware v2 will move the ADXL343 to the ATtiny software I2C bus, which enables single-address access. See [Project-Libelle issue #19](https://github.com/NorthernWidget-Skunkworks/Project-Libelle/issues/19).

> **Planned MCU migration:** The proposal for hardware v2 is to migrate from the ATtiny841 (512 B EEPROM, stored image at `0x01C0`) to the ATtiny1634 (256 B EEPROM, stored image at `0x00C0`), which aligns Libelle with Apis, Haar, and Walrus. The provisioning table and avrdude part will update accordingly. See [Project-Libelle issue #20](https://github.com/NorthernWidget-Skunkworks/Project-Libelle/issues/20).

> **Migration note:** the firmware on `master` (`Libelle_Driver_ShortWave`) implements this map since 2026-09-23 (patch 1): Page 0 from EEPROM (0x1E0–0x1FF on the ATtiny841) with CRC check, the writable-register rule with the address persisted, 32-byte page reads, data at 0x28–0x3D (0x48–0x5D since the 2026-09-23 renumbering), and the Block 0 handshake (trigger or the Config free-running timer, chip select, reading counter, unit faults at boot, and since the same day a chip that does not acknowledge its address gets its status bit and the latched code kind 1, and the data registers update all at once with the counter rather than field by field during the 0.8–2 s reading; no data check per chip yet). The default address moved from 0x40/0x41 to 0x4C UP and 0x0C DOWN by the jumper. Hardware v1 still serves no Page 3. The library is on NW_Core since 2026-09-23 (data from 0x28-0x3D in one read; the accelerometer read on the controller bus; the library judges the accelerometer fault itself). Unreleased and not yet validated on hardware.

> **Known bug in deployed firmware:** The firmware writes UVB starting at register 0x07, but the library reads it from 0x06. `getUVB()` therefore returns approximately true_UVB × 256. All historical UVB data is affected. See [Project-Libelle issue #18](https://github.com/NorthernWidget-Skunkworks/Project-Libelle/issues/18).

### Liasis (longwave pyrgeometer) – hypothetical until the board carries an MCU

#### Page 0

```
Block 0:  Schema=0x01, Name='L','i','a','s','i','s',0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x6C01 ('l'=0x6C, rev 1), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=TBD
```

Note: `0x6C00` is reserved. It was assigned to Apis before the ASCII-initial naming convention was established. Legacy deployed units carry board types `0x2400`/`0x2401` (formerly Dyson LW, Monarch LW).

#### Page 2 (0x40–0x5F): Sensor data

Page 2 layout TBD. Liasis does not currently have an onboard MCU: it communicates via the host controller's I²C bus rather than exposing its own register map. Page 2 and the I²C address will be defined once Liasis is updated to carry its own MCU. At that point Block 0 follows the universal layout, and the chip table will be: 0 = ADS1115 (thermopile and thermistor channels). Config (0x46) would then carry the ADS1115 gain and data-rate selections.

---

### Tally (event counter) – proposed 2026-09-23, not yet decided

Tally counts pulses from a reed switch, a tipping-bucket gauge or an anemometer on its own supercapacitor, so it keeps counting while the logger sleeps. Its ATtiny841 reads a 16-bit hardware counter (LOAD, CLK, DATA pins), clears it, and adds the ticks to a running total that it serves over I²C. The controller latches that total and takes the difference from the total it logged last time, so a failed read loses nothing and no clear command exists on the bus.

#### Page 0

```
Block 0:  Schema=0x01, Name='T','a','l','l','y',0x00,0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x5401 ('T'=0x54, rev 1), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=[build], flags=[build], Magic=0x4E, CRC=[computed], I2C address=0x54
```

#### Page 2 (0x40–0x5F): Counter data

Chip table:

| Index | Chip | Measurements |
|-------|------|--------------|
| 0 | Counter logic (16-bit hardware counter behind LOAD/CLK/DATA) | event count |
| 1 | ATtiny841 ADC (the unit's own) | supercapacitor voltage |

Block 0 (0x40–0x47) is the universal block. A trigger *latches*: the firmware reads and clears the hardware counter, adds the ticks to the running total, writes the total, and increments the reading counter. Config (0x46): bit 0 = capacitor disconnected from the charger (the legacy NOCAP command; 0 = charging, the power-on state); bits 7:1 reserved. There is no free-running cycle: the hardware counts continuously, and the firmware sleeps between transactions.

```
Block 1 (0x48–0x4F)   Counter
  0x48–0x4B   Event count, uint32, little-endian: ticks since power-up (wraps at 2^32)
  0x4C–0x4D   Supercapacitor voltage, uint16, raw ADC counts (× 3.3 / 1024 → V), read at each latch
  0x4E–0x4F   Reserved

Blocks 2–3 (0x50–0x5F)   Reserved
```

No Page 1.

Faults: chip 0 kind 4 (out of range) if the hardware counter reads all ones at a latch, which the logic cannot produce in one logging interval; unit kinds 6 and 3 at boot as for every device.

> **What changes from firmware 0.2.0 (2019):** the five-byte map at 0x00 (command bits at 0x00: SAMPLE, CLEAR, RESET, PEEK, SLEEP, GET_VOLTAGE, NOCAP; ticks at 0x01–0x02; capacitor voltage at 0x03–0x04) becomes Schema 1. SAMPLE is the Block 0 trigger; PEEK is the only behaviour (the count is monotonic, so every read is a peek); CLEAR and RESET disappear from the bus (the controller differences totals); SLEEP is Control bit 7; GET_VOLTAGE is folded into the latch; NOCAP is Config bit 0. The address moves from the hard-coded 0x33 to 0x54 ('T'), with Page 0 byte 0x1F overriding it. The 16-bit hardware counter still wraps at 65 536 ticks between latches (an anemometer at 100 pulses/s wraps in 11 minutes), so the firmware must be latched, by the controller or by its own timer, more often than that; a device-side latch timer is the open question.

### Okapi (data logger with solar charging and telemetry)

Okapi is an I²C controller that communicates with a Particle Boron telemetry board via UART, and it has no current peripheral interface. Schema 1 formalizes its EEPROM serial number as Page 0 and reserves a hypothetical Page 2 for status reporting to the Boron or any higher-level device. The UART transport requires a framing layer (Magic Preamble or COBS, see [Transport](#transport)).

#### Page 0

```
Block 0:  Schema=0x01, Name='O','k','a','p','i',0x00,0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=[mfr], 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x4F01 ('O'=0x4F, rev 1), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=0x00000000, flags=0x00 (a logger's build is its library's, reported in the status file), Magic=0x4E, CRC=[computed], Peripheral address=0x00 (unassigned)
```

Pre-production prototype units ("Resnik") carry board type `0x9950` and are not field-upgradeable to Schema 1.

#### Page 2 (0x40–0x5F): Logger status (hypothetical)

Subsystem table (a logger's "chips" are its subsystems, with the same index rules):

| Index | Subsystem |
|-------|-----------|
| 0 | SD card |
| 1 | DS3231M RTC |
| 2 | BME280 onboard environment |
| 3 | Sensor bus |
| 4 | LiPo / solar charger |
| 5 | AA backup rail |

Block 0 (0x40–0x47) is the universal block. Config (0x46): reserved. A logger's data exceeds 24 bytes, and logger state therefore continues on Page 3.

```
Block 1 (0x48–0x4F)   Power
  0x48        LiPo %, uint8, 0–100
  0x49–0x4A   LiPo voltage, uint16, 0.01 V
  0x4B–0x4C   Solar input, uint16, TBD (pending GetPowerStats() implementation)
  0x4D        Backup voltage, uint8, 0.1 V (AA backup rail; 0–25.5 V range)
  0x4E–0x4F   Reserved

Block 2 (0x50–0x57)   BME280: onboard environment
  0x50–0x51   Temperature, int16, 0.01 °C
  0x52–0x53   Humidity, uint16, 0.01 %RH
  0x54–0x57   Pressure, uint32, 0.01 hPa

Block 3 (0x58–0x5F)   DS3231M: RTC
  0x58–0x5B   Timestamp, uint32, Unix time (seconds since 1970-01-01 UTC)
  0x5C–0x5D   Temperature, int16, 0.01 °C
  0x5E–0x5F   Reserved

Page 3, Block 0 (0x60–0x67)   Logger state
  0x60–0x61   External interrupt count, uint16, accumulated
  0x62–0x63   Log file number, uint16
  0x64–0x67   Log interval, uint32, seconds
Page 3, Blocks 1–3 (0x68–0x7F)   Reserved
```

No Page 1. Calibration constants are hardcoded in the library.

---

### Margay (data logger)

Margay is an I²C controller: it queries the sensors on its bus, and no I²C peripheral reads it. It is a Schema 1 device all the same (decided 2026-09-23): it carries Page 0 in EEPROM, keeps Page 1 there for its calibration, and maintains Pages 2 and 3 in SRAM as a reading of itself, with a Report register of its own, so that its rows in a status file decode like any sensor's and a gateway can read it over UART (the Magic Preamble transport) the way a logger reads a sensor. The UART serving is Okapi's to build first.

#### Page 0

```
Block 0:  Schema=0x01, Name='M','a','r','g','a','y',0x00
Block 1:  HW major=[mfr], HW minor=[mfr], FW patch=0x00 (a logger's firmware is its library, reported by the library), 0x00,0x00,0x00, Reserved
Block 2:  Board type=0x4D03 ('M'=0x4D, rev 3), Group ID=[mfr], Unique ID=[mfr], FirmwareID=0x0000
Block 3:  Build commit=0x00000000, flags=0x00 (a logger's build is its library's, reported in the status file), Magic=0x4E, CRC=[computed], I²C address=0x00 (unassigned)
```

Block 2 keeps the format of the existing 8-byte Schema 0 EEPROM serial number (board type, group ID, unique ID, FirmwareID) with no data loss, but not its location: Schema 0 wrote those 8 bytes at the very end of EEPROM, which under Schema 1 is the last block of Page 1, while Block 2 sits at Page 0 offset 0x10–0x17. A logger library that reads its serial number from the last 8 bytes therefore reads calibration once the board is provisioned. Your library must instead read Page 0 (schema byte 0x01, magic, CRC) and take the serial number from Block 2, falling back to the old location when the schema byte is not 0x01. The board type encoding (`'M'` = 0x4D high byte, revision index low byte) already followed the Schema 1 convention before the spec was written.

In the status file the logger's own row fills FW and FWCommit with the library's version and build commit (`MARGAY_LIBRARY_VERSION`, `MARGAY_LIBRARY_COMMIT`), leaves Lib blank (a sketch has no version), and fills LibCommit with `SKETCH_COMMIT`, the sketch's build commit. Both commits are blank in an IDE build.

#### Page 1 (0x20–0x3F): Calibration

Written once by NW-Provision per board, from the hardware model; the library reads it at boot and falls back to its built-in constants when the page is blank (0xFF).

```
  0x20–0x21   Battery divider × 1000, uint16 (2000 for models 1.0–3.x; 9000 for the v0.0 prototype only)
  0x22–0x31   Thermistor Steinhart–Hart A, B, C, D, float32 each, little-endian
  0x32–0x33   Battery low threshold, uint16, 0.01 V (a fault below it)
  0x34        Battery warning, uint8, percent (a notice below it)
  0x35–0x3F   Reserved
```

#### Page 2 (0x40–0x5F): The logger's reading

Chip table:

| Index | Chip | The library detects |
|-------|------|---------------------|
| 0 | SDCard | no card (card-detect), the boot write-and-read-back failed |
| 1 | Clock (DS3231M) | not answering on the internal bus |
| 2 | BME280 (onboard environment, models 2.0 and up) | not answering |
| 3 | SensorBus (the switched external I²C rail) | an address the sketch declared that does not answer |
| 4 | Battery (thermistor and divider through the MCP3421 or the ADC) | below the low threshold; below the warning percentage |

Block 0 (0x40–0x47) is the universal block. The reading counter counts rows written to the current data file. Config (0x46): reserved, 0x00.

```
Block 1 (0x48–0x4F)   Battery
  0x48        Battery, uint8, percent
  0x49–0x4A   Battery voltage, uint16, 0.01 V
  0x4B–0x4C   Thermistor temperature, int16, 0.01 °C
  0x4D–0x4F   Reserved

Block 2 (0x50–0x57)   BME280: onboard environment (0x00 on models without it)
  0x50–0x51   Temperature, int16, 0.01 °C
  0x52–0x53   Humidity, uint16, 0.01 %RH
  0x54–0x57   Pressure, uint32, 0.01 hPa

Block 3 (0x58–0x5F)   Clock
  0x58–0x5B   Time, uint32, Unix seconds UTC
  0x5C–0x5D   Clock temperature, int16, 0.01 °C
  0x5E–0x5F   Reserved

Page 3, Block 0 (0x60–0x67)   Logger state
  0x60–0x61   Log file number, uint16
  0x62–0x65   Log interval, uint32, seconds
  0x66–0x67   External interrupt count, uint16, since power-up
Page 3, Blocks 1–3 (0x68–0x7F)   Reserved
```

Reports. The universal kinds apply to the chips (SDCardMissing is chip 0 kind 1, SDCardSelfTestFailed chip 0 kind 5, ClockNotAnswering chip 1 kind 1, BME280NotAnswering chip 2 kind 1, BatteryLow chip 4 kind 4, all faults; UnitRestarted at boot). Device-specific kinds:

| Code | Chip, kind | Note word | Fault or notice | When |
|------|-----------|-----------|-----------------|------|
| 0x90 | Battery, 16 | BatteryWarning | notice | below the warning percentage |
| 0x30 | Clock, 16 | ClockSet | notice | the time was set from serial |
| 0xF0 | unit, 16 | LoggingStarted | notice | logging began, at power-up or by the button |
| 0xF1 | unit, 17 | NewLogFile | notice | a new data and status file pair was opened |
| 0xF2 | unit, 18 | RowNotWritten | notice | a data row could not be written to the card |

A logger writes its own reports into its status file the way it writes a sensor's: it watches itself.

## I²C address registry

All NW device addresses, current and proposed. Proposed addresses follow an ASCII-mnemonic scheme:

- **Primary address** = ASCII code of the device's initial letter (uppercase), which makes addresses human-readable in a hex dump.
- **Secondary address** applies only to devices with an on-board hardware jumper that selects between two fixed addresses (e.g. Libelle JP1 for UP/DOWN net-radiation pairs). The secondary address = primary address XOR `0x40`, which clears bit 6 and maps the letter to the control character at the analogous position in the ASCII table (zones 000/001 mirror zones 100/101). Secondary addresses therefore occupy the range `0x08`–`0x3F`, entirely outside the letter space reserved for primary addresses. Devices whose address is software-configurable via the EEPROM register at `0x1F` do not need a hardware secondary address.

Controller-only devices (Margay, Okapi) have no current peripheral address, and their entries are reserved for hypothetical future peripheral interfaces.

| Device | Type | Current address(es) | Proposed (Schema 1) | Mnemonic | Notes |
|--------|------|---------------------|---------------------|----------|-------|
| Apis | Peripheral | `0x50` (primary) | `0x41` | `'A'` | Legacy board type `0x6C00`; Schema 1 board type `0x4100` |
| Liasis | Peripheral | – | TBD | `'l'` | `0x6C00` reserved (legacy Apis); Schema 1 board type `0x6C01`; lowercase initial; the secondary address scheme does not apply |
| Haar | Peripheral | `0x42` (primary) | `0x48` | `'H'` | `0x48` is common for ADS1115; no NW-device conflict |
| Libelle | Peripheral | `0x40` UP, `0x41` DOWN | `0x4C` UP, `0x0C` DOWN | `'L'` / FF | Hardware solder jumper JP1; DOWN = `'L'` XOR `0x40` = `0x0C` (ASCII Form Feed) |
| Margay | Controller (hypothetical) | – | `0x4D` | `'M'` | **Reserved.** Controller only; no peripheral interface yet |
| Okapi | Controller (hypothetical) | – | `0x4F` | `'O'` | **Reserved.** Controller only; no peripheral interface yet |
| Walrus | Peripheral | `0x4D` primary, `0x41` alt | `0x57` | `'W'` | ⚠ Current `0x4D` must change; this clashes with the proposed Margay design. The address is software-configurable via EEPROM |

### Bus occupancy

The registry above lists NW devices. A sensor shares the bus with the logger's on-board chips as well, and the table below therefore lists every address in use or reserved across the family, from the device appendices, the logger READMEs, the Okapi v1.0 schematic, and the sensor libraries. Chips whose address is set by pin strapping are marked, but their actual strapping has not yet been read from the schematic (see the to-do below).

| Address | NW device (Schema 1) | Logger on-board chips | Chips driven by a sensor library on the controller bus |
|---------|----------------------|-----------------------|--------------------------------------------------------|
| `0x0C` | Libelle DOWN | Okapi: MLX90393 magnetometer (`0x0C`–`0x0F`, by strapping) | |
| `0x10`–`0x1F` | | Okapi: PAC1934 (by strapping); VEML6030 if strapped to `0x10` | Libelle v1: ADXL343 at `0x1D` |
| `0x18`/`0x19` | | Okapi: BMA456 accelerometer (by strapping) | |
| `0x20` | | Okapi: MCP23018 I/O expander | |
| `0x28` | | | T9602 |
| `0x41` | **Apis** | | |
| `0x48` | **Haar** | **Okapi: ADS1115 (on-board); VEML6030 if strapped to `0x48`** | |
| `0x49` | | Okapi: ADS1115 (off-board channels) | |
| `0x4A` | | | Liasis: ADS1115 |
| `0x4C` | Libelle UP | | |
| `0x4D` | Margay (reserved, hypothetical) | | |
| `0x4F` | Okapi (reserved, hypothetical) | | |
| `0x50`–`0x57` | **Walrus `0x57`**; **Tally `0x54`** (proposed); Apis legacy `0x50` | **Okapi: MB85RC FRAM (`0x50`–`0x57`, by strapping)** | |
| `0x62` | | Okapi: MCP4725 DAC | |
| `0x68` | | Margay, Okapi: DS3231 RTC | |
| `0x69`–`0x6B` | | Margay: MCP3421 ADC (by model) | |
| `0x76`/`0x77` | | Margay, Okapi: BME280 | |

**Potential clashes, pending the Okapi strapping:** Haar `0x48` against Okapi's on-board ADS1115 (and VEML6030 if strapped high), and Walrus `0x57` against the FRAM's range. Whether either bites also depends on whether Okapi's on-board I²C segment is electrically joined to the sensor segment when the external bus is switched on. Okapi is a prototype under revision, and restrapping a chip is therefore cheap. Moving Haar or Walrus is not.

**To do:** (1) read the ADDR strapping of the ADS1115, VEML6030, PAC1934, MB85RC, and BMA456 from the Okapi v1.0 schematic and the segment-switch topology, (2) fill in this table, and (3) resolve the two potential clashes. Tracked in [Project-Okapi issue #29](https://github.com/NorthernWidget-Skunkworks/Project-Okapi/issues/29).

### Clashes requiring resolution

1. **Walrus `0x4D` → Margay `'M'` = `0x4D`:** Walrus must migrate to `0x57` (`'W'`) in Schema 1. This is a breaking change to existing deployments.

2. **Libelle DOWN `0x41` → Apis `'A'` = `0x41`:** Resolved in Schema 1. Libelle moves to `0x4C` (UP) and `0x0C` (DOWN), and Apis takes `0x41`.

---

## Controller-side library design

[LIBRARY-DESIGN.md](LIBRARY-DESIGN.md) describes how a controller's Arduino library reads a Schema 1 device: the layered library shape, the one-sample raw-reading primitive, the proposed universal control register, and the naming conventions. Its handshake proposal is now the Page 1 Block 0 definition above. The rest is the library-side design and its work plan.

---

## Schema 1 implementation status

Tracks whether each device's library/firmware and hardware have been updated for Schema 1, and whether the combination has been verified.

| Device | Library / firmware | Hardware | Integration check | Physical test |
|--------|--------------------|----------|-------------------|---------------|
| Apis | ✅ firmware patch 5 and library on NW_Core (master, unreleased) | – | ✅ 2026-09-23 (host harness) | – |
| Haar | ✅ firmware patch 1 and library on NW_Core (master, unreleased) | – | ✅ 2026-09-23 (host harness) | – |
| Walrus | ✅ firmware patch 1 and library on NW_Core (master, unreleased) | – | ✅ 2026-09-23 (host harness) | – |
| Libelle | ✅ firmware patch 1 and library on NW_Core (master, unreleased) | – | ✅ 2026-09-23 (host harness) | – |
| Liasis | – | – | – | – |
| Margay | – | – | – | – |
| Okapi | – | – | – | – |

**Library / firmware:** library updated to build and serve Schema 1 Page 0, and `begin()` validates the schema byte.
**Hardware:** new board revision tagged (if required), and Page 0 EEPROM provisioned via nw-provision.
**Integration check:** register map verified against this spec, library compiles against it, and data types confirmed.
**Physical test:** library and hardware tested together on real hardware, and data confirmed correct end-to-end.

Note: Liasis shows `–` across all columns because it does not yet have an onboard MCU. All columns will remain not applicable until the hardware is updated. See [Project-Liasis issue #2](https://github.com/NorthernWidget-Skunkworks/Project-Liasis/issues/2).

### Conformance checks

Each rule below is exercised by a case in a host-side harness: NW_Core's `extras/test/test_output.cpp` for the universal rules, and each library's own `extras/test/` for its register map and status row. A case is a line of recorded output compared byte for byte with `baseline.txt`; `run.sh --record` re-records after an intended change, and the diff is reviewed in the commit. The case names are the bracketed labels in the harness output.

| Rule | Harness | Case |
|------|---------|------|
| `begin()` rejects a schema byte other than 0x01 | NW_Core | `[begin] schema 0x00` |
| `begin()` rejects a wrong name, and a prefix of the right one | NW_Core | `[begin] wrong name`, `[begin] name 'Api' vs 'Apis'` |
| `begin()` rejects a firmware patch below the library minimum | NW_Core | `[begin] patch 1 < min 2` |
| `begin()` waits a bounded time for a device still booting | NW_Core | `[begin] boots after 30 ms`, `[begin] boots after 300 ms`, `[begin] absent` |
| Boot reports are read before the first acknowledgement | NW_Core | `[begin] boot report`, `[begin] boot status line` |
| Trigger, chip select, ready, and the counter | NW_Core | `[handshake] request`, `[handshake] takeReading`, `[handshake] newReading` |
| The counter is re-read after a multi-transaction read, with bounded retries | NW_Core | `[readData] one commit during the read`, `[readData] a commit on every transaction`, `[readData] after the runaway stops`, `[readData] quiet device` |
| A free-running device is read without a request | NW_Core | `[free-running] waitReading without request` |
| A fault overwrites a notice; a notice never overwrites a fault | NW_Core | `[pages] notice then fault`, `[pages] after a reading` |
| Page 0 validity and a blank Page 1 on the device side | NW_Core | `[pages] page0Valid` |
| Report text and note tokens, universal and device-specific | NW_Core | `[report text]`, `[report] device-specific kind` |
| The status row's fourteen columns | NW_Core | `[snapshot]`, `[pages] snapshot` |
| Readings requested, batches, and dead chips in a batch | Apis_Library | `[request]`, `[batch stats]`, `[faults]` |
| The zero ring and generation (Apis patch 5) | Apis_Library | `[zero generation]`, `[dumpZeros]` |
| Each library's register map, faults, and status row | Apis, Walrus, Haar, Libelle, T9602 libraries | `[status]` and the library's own cases |
| Every library compiles on Margay and on Okapi | NW-Tests `compile.py` | one sketch per library and logger |

---

## License

This specification is released under the [Creative Commons Attribution-ShareAlike 4.0 International License](LICENSE) (CC BY-SA 4.0).

You are free to implement, adapt, and redistribute this specification, including for commercial purposes, provided you give appropriate credit and share any adaptations under the same license.

&copy; 2026 Andy Wickert, Northern Widget LLC
