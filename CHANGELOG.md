# Changelog

Every change to what a device serves, what a controller must do, or what an appendix defines is recorded here by date, newest first, with the commit that made it. The repository has no release tag: the first commit called itself v1.0.0 (2026-05-31), and no tag was ever created, so the whole history stands under Unreleased until the first tagged release. Dated entries are the unit of change, since the schema byte (0x01) has not moved.

## [Unreleased]

### 2026-09-23

- Block 3 of Page 0 holds the build commit (0x18–0x1B) and build flags (0x1C); the served copy of Page 0 is defined, with the firmware patch, commit, and flags rewritten and the CRC recomputed over it (7a0b5b5).
- Key words (RFC 2119), section status (normative, proposed, hypothetical), and must/should on the load-bearing rules of Page 0 and Block 0 (a63247d).
- Apis appendix: firmware patch 5 keeps a ring of two zeros and a generation count on Page 1, mirrored into Page 2 Block 3; Page 1 EEPROM bytes are in bus order (0074e97).
- Margay is a Schema 1 device: Page 0, Page 1 calibration, Pages 2 and 3 as a reading of itself, and its own report kinds; note words for every report kind; the chip-naming rule (026fb26). The 9:1 battery divider belongs to the v0.0 prototype (79fa657).
- The address space renumbered before any release: 0x00–0x3F stored (Page 0 identity, Page 1 calibration), 0x40 and above served; data owns Pages 2–5, Pages 6–7 stay reserved (4c3b18d, 65a8f68).
- 0x47 is the Report register: faults and notices, kinds 9 (calibration stored) and 10 (batch abandoned), one report at a time (0b2af0d, 0c3036b).
- A controller re-reads the counter after a multi-transaction read, a bounded number of times (a8cbd56).
- Status file: fourteen columns, and the logger's own row (9164500, aafb9b6).
- Libelle appendix: Schema 1 firmware (patch 1) with per-chip faults and atomic data updates (add36bf, 37cc4b5, c8f84cc, 9227cff). Walrus and Haar migration notes (bf59b1b, 673691c).
- Tally appendix proposed, not decided (a50ec5e).
- Prose in Andy's voice, en-dashes, headings with colons (2867ddf, 40b3fb8, 4e8c369, 20aca5b).

### 2026-09-22

- Block 0: 0x24–0x25 (now 0x44–0x45) is the writable "readings requested" word; "batch" names a declared run of readings, with the chip-selection and chip-count rules (27ee86f, 53908c1).
- Margay appendix: the Schema 0 serial number keeps its format, not its location (81a666d).
- RELEASING.md moved here from NorthernWidget/.github; the firmware patch constant must match the tag's FWversion (22344dd, 76b5d3a).
- Apis appendix: the on-demand run model for firmware patch 2 (b410cf7); the LiDAR named as the v3HP (8765fd9).

### 2026-09-21

- Universal Block 0 on the data page (then Page 1, now Page 2): status, control, reading counter, config, latched fault code (d959f0c); every appendix moved its data under it and gained a chip table (069f7f1, acf7455, 8493f27, 0e8e830, 903af1c, e2cbdc1, 1ab760d).
- The magic byte is 0x4E in every appendix (e2fdafc); bus-occupancy table across NW devices and on-board chips (3078e96); the Apis accelerometer is the LIS3DH (a1ec46c).

### 2026-09-20

- LIBRARY-DESIGN.md added: the controller-side sensor library and firmware design, with the pilot order Apis, Walrus, Haar and Tally in the campaign (48340e5, 3917a81, 3f2125e, d96281d).

### 2026-06-02

- Page 0 Block 3: the magic byte is 0x4E (f387f66); CRC-8/SMBUS with a reference implementation (a2ae514).
- Physical EEPROM layout: the stored page at EEPROM[length−64] (dd13939).
- Schema 1 implementation-status table (d911b5d); appendix corrections for Haar, Walrus, Libelle and Liasis (96ce834, fe5439e, 116396b, 2c884d8, fc512c5, b62fa89).

### 2026-06-01

- Fault signalling standardised: pan-fault bit and an extended fault byte, moved to 0x21 (23fa434, 64583ab).
- Address registry: the secondary address scheme (XOR 0x40, hardware-jumper devices only); Libelle DOWN is 0x0C (fe44e03).
- Device provisioning reference table (e062628); pressure ranges and resolution documented (be9d48b); Okapi Resnik prototype board type 0x9950 (49094bb); Liasis appendix added and the Apis/Libelle clash resolved (ea7c1c1).

### 2026-05-31

- The specification: Page 0 identity, transport (I²C and UART framing), the I²C address registry, the physical EEPROM layout, reserved schema values 0x00 and 0xFF, and appendices for Haar, Walrus, Libelle, Margay and Okapi (0d8eeb3 through a233a17). "Controller" replaces "master" throughout (62c12cf).
