# Releasing NW Arduino libraries, firmware, and hardware

Moved here from `NorthernWidget/.github` on 2026-09-22 so that it lives beside the specification it depends on and in a visible repository. The design the libraries and firmware converge on is [LIBRARY-DESIGN.md](LIBRARY-DESIGN.md) (section 0 first); register-level rules are in [README.md](README.md).

# Releasing an NW Arduino Library

This checklist applies to every versioned release of a NorthernWidget Arduino sensor library. Work through the items in order; make one commit per item.

**Physical hardware testing is required before any release.** A complete checklist does not mean the library is ready to release.

## Required files

Every library must have:

| File | Requirements |
|------|-------------|
| `library.properties` | `version=` at release semver; `paragraph=` filled in |
| `LICENSE` | GPL-3.0 |
| `README.md` | Zenodo DOI badge (concept DOI); basic usage example |
| `CITATION.cff` | cff-version 1.2.0; all authors with ORCIDs; concept DOI; `date-released`; `license: GPL-3.0` |
| `.zenodo.json` | `upload_type: software`; `license: {id: GPL-3.0-only}`; same authors and keywords |
| `keywords.txt` | KEYWORD1 = class/enum types; KEYWORD2 = public methods; LITERAL1 = constants/enum values |
| `doxygen_NW.cfg` | `OPTIMIZE_OUTPUT_FOR_C = NO`; no `MDFILE_AS_MAINPAGE`; `EXCLUDE = examples extras` |
| `src/` | Source in `src/`, not flat layout |
| `examples/LibraryName_Demo/` | Minimal sketch: `begin()` error check that prints `getFirmwareVersion()` on refusal; `getHeader()` in setup; `getString()` in loop with `delay(1000)` |
| `extras/test/` | Host-side output-regression harness (stub `Arduino.h`/`Wire.h`, `run.sh`, recorded `baseline.txt`); see Apis_Library. Required once a library has been refactored to the common interface |
| `.github/workflows/docs.yml` | Thin wrapper: `permissions: contents: write` then a job with `uses: NorthernWidget/.github/.github/workflows/deploy-docs.yml@main` (the shared workflow pushes the site; without the explicit permission a repository whose default is read-only fails at startup) |
| `_docs/` | Jekyll site config the shared docs workflow copies: `_config.yml` (title, description, `baseurl: /<Repo>/`, `url: https://docs.northernwidget.com`), `Gemfile`, `_data/navigation.yml` (Overview, the class, Classes, Files) — without it the workflow fails at the copy step |
| `.doxybook/config.json` | `baseUrl: /<Repo>/`; the workflow runs doxybook2 with it |
| GitHub Pages enabled | One-time repository setting, not a file: source = branch `gh-pages`, path `/`. The workflow pushes the built site to that branch; until Pages is enabled the site returns 404. `gh api -X POST repos/<owner>/<repo>/pages -f 'source[branch]=gh-pages' -f 'source[path]=/'` |

## Pre-release checklist

One commit per item.

### Schema 0 baseline (all releases)

Complete these before any versioned release, regardless of schema:

1. **Version** — assess git history since last tag; choose semver bump; `0.x` → `1.0.0` for first stable release
2. **`library.properties`** — bump `version=`; fill `paragraph=` if empty
3. **DOI badge** — replace any deprecated `latestdoi`/`GITHUB_REPO_ID` format; use concept DOI (ask maintainer; never guess)
4. **`CITATION.cff`** — create or update; cff-version 1.2.0; all authors with ORCIDs
5. **`.zenodo.json`** — create or update; `upload_type: software`; `license: {id: GPL-3.0-only}`
6. **`keywords.txt`** — create or update
7. **`examples/LibraryName_Demo`** — create or update minimal demo sketch
8. **`doxygen_NW.cfg`** — fix known bad settings (`OPTIMIZE_OUTPUT_FOR_C = NO`; remove `MDFILE_AS_MAINPAGE`; `EXCLUDE = examples`)
9. **Return types** — `begin()` must return `bool`; update Doxygen tags and examples
10. **`.github/workflows/docs.yml`** — add if missing

Tag a Schema 0 snapshot release once items 1–10 are complete and the library has been tested on physical hardware. This snapshot preserves the last known-good register map before any breaking Schema 1 changes.

### Schema 1 migration (sensor libraries only)

Do not begin until the spec is stable and a Schema 0 snapshot tag exists.

11. **Schema 1 compliance** — implement the full Schema 1 register map per the device appendix in [NW-Device-Specification](https://github.com/NorthernWidget/NW-Device-Specification):
    - **Page 0 (0x00–0x1F, EEPROM-backed identity):** schema byte `0x01` at `0x00`; 7-byte name at `0x01–0x07`; HW/FW version at `0x08–0x0A`; serial number block at `0x10–0x17`; magic byte `0x4E` at `0x1D`; CRC-8/SMBUS at `0x1E`; I²C address at `0x1F`
    - **Page 1 (0x20–0x3F, SRAM):** the universal Block 0 — status `0x20` (ready, per-chip fault bits, pan-fault), control `0x21` (trigger, chip select, sleep), reading counter `0x22–0x23`, readings requested `0x24–0x25` (writable count for batches), device config `0x26`, latched fault `0x27` — then device data from `0x28` per the appendix (Page 3 continues data past 24 bytes)
    - **Page 2 (0x40–0x5F, calibration, if applicable):** per device appendix
    - Update default I²C address to the Schema 1 value from the address registry, and check the bus-occupancy table there for clashes with logger on-board chips
    - `begin()` reads Page 0 Blocks 0–1 and rejects: schema byte ≠ `0x01`; wrong name; firmware patch below the library's `<LIB>_FW_MIN_PATCH`; exposes `getHardwareMajor()`, `getHardwareMinor()`, `getFirmwareVersion()`
    - The appendix's chip table fixes the index used by status, control and fault bytes; the library's `Component` enum (`Lib::ALL`, one value per chip) follows it

12. **Common interface** — per [LIBRARY-DESIGN.md](LIBRARY-DESIGN.md) §0: `updateMeasurements(component)` as the only bus code for data, requested and waited for through the reading counter; per-measurement static reading arrays with `<LIB>_<FIELD>_CAPACITY`; `set<Field>Readings()`, `get<Field>Mean/Std/Sterr/Median/Count()`; the reading interface `beginReadings`/`printReading`/`logReading`/`endReadings` + `printHeader` (required on every sensor library); `ready()`, `newReading()`, `requestReading()`; the faults API. Refactor first with byte-identical `getHeader()`/`getString()` output against the harness baseline, then change behaviour in separate commits.

13. **Names** — Arduino style; camelCase; class-scoped selectors; register constants file-local; no `NW_` prefix. Casing migrates in two steps (camelCase with `[[deprecated]]` aliases, then removal). A name already removed is never re-added: migrate its callers forward.

14. **Version** — a Schema 1 conversion is a breaking change: major version, after the hardware test.

Controllers (Margay, Okapi) are not sensors and do not implement the sensor register map; their Schema 1 entries are identity-only.

## Code conventions

- `begin()` returns `bool`; checks I²C ACK; stubs return `false`
- No build artifacts committed (no generated `_site/` or Doxygen `xml/`, no downloaded binaries; `_docs/` is site *configuration* and is required)
- DOI badge uses concept DOI (permanent, not per-version)

## Authors

Known authors for citation files:

| Name | ORCID | Affiliation |
|------|-------|-------------|
| Andrew D. Wickert | 0000-0002-9545-3365 | University of Minnesota, Department of Earth & Environmental Sciences |
| Bobby Schulz | 0000-0002-9272-4756 | Northern Widget LLC |

Confirm full author list from `library.properties` and `git log` before writing `CITATION.cff` or `.zenodo.json`.

## Reference libraries

- **[Apis_Library](https://github.com/NorthernWidget/Apis_Library)** — reference for `CITATION.cff`, `keywords.txt`, `.github/workflows/docs.yml`, code conventions, the common interface, and the test harness
- **[Walrus_Library](https://github.com/NorthernWidget-Skunkworks/Walrus_Library)** — reference for `.zenodo.json`
- **[NW_BME280](https://github.com/NorthernWidget/NW_BME280)** — alternate reference for `CITATION.cff`

---

# Releasing NW sensor firmware (in a Project-\* repo)

The firmware on a sensor's MCU is released with the hardware repo (`HWmajor.HWminor.FWversion`, below). Before a tag that includes a firmware change:

1. **Patch constant equals the tag** — the firmware's compiled patch (e.g. `FW_FW_PATCH`) must equal the `FWversion` digit of the tag; it is what the firmware writes to Page 0 byte `0x0A` and what the library checks. Bump it on any behavioural change visible to the library.
2. **Names** — camelCase functions and globals, as for libraries; macros upper case.
3. **Register map** — Page 0 served from the top of EEPROM (CRC checked, patch substituted); Page 1 universal Block 0 with the rules in the spec (only `0x21`, `0x24–0x25`, and `0x26` writable; atomic rewrite; ready cleared at reading start, set with the counter increment; latched fault cleared by a control write); data from `0x28`; Page 2 for calibration.
4. **Provisioning** — Page 0 written with [NW-Provision](https://github.com/NorthernWidget/NW-Provision) and the unit recorded in [NW-Registry](https://github.com/NorthernWidget/NW-Registry); the ATTinyCore "EEPROM retained" fuse keeps it across reflashes.
5. **Compile for the target** with `arduino-cli` before every commit; **bench test** the firmware + library pair on a provisioned board before the tag.
6. **README** — the register map section describes the firmware on `master`, with the previous map kept for unreflashed boards.

---

# Releasing NW Hardware (Project-\*)

This checklist applies to every versioned release of a NorthernWidget hardware design repo (`Project-*`). Work through the items in order; make one commit per item.

**Physical build and test is required before any release.** Verified electrical function and known errata must be documented before tagging.

## Required files

Every hardware repo must have:

| File | Requirements |
|------|-------------|
| `README.md` | Zenodo DOI badge (concept DOI); board overview; version history with errata |
| `LICENSE` | CERN-OHL-S-2.0 or CC-BY-SA-4.0 |
| `CITATION.cff` | cff-version 1.2.0; all authors with ORCIDs; concept DOI; `date-released`; license matching `LICENSE` |
| `.zenodo.json` | `upload_type: other`; license matching `LICENSE`; same authors and keywords |
| Fabrication outputs | Gerbers, drill file, BOM — committed under `fab/` or equivalent; regenerated fresh for each release |
| Schematic PDF | Exported and committed alongside source files |

## Pre-release checklist

One commit per item:

1. **Version** — bump version in README and schematic title block using the `HWmajor.HWminor.FWversion` convention (see below)
2. **Firmware version in the patch slot** — for every hardware release, with or without a firmware change: the tag's `FWversion` digit equals the firmware's compiled patch constant (e.g. `FW_FW_PATCH`), which the firmware writes to Page 0 byte `0x0A` and the library checks in `begin()`. If they differ, fix the constant (and reflash/bench) before tagging; never tag a firmware that reports a different patch than its tag.
3. **Fabrication outputs** — regenerate Gerbers, drill file, and BOM from the release-tagged source; commit under `fab/`
4. **Schematic PDF** — export and commit
5. **Errata and version notes** — update README with any known issues on this board revision
6. **NW-Registry** — add a new row to [`NW-Registry/board_types.csv`](https://github.com/NorthernWidget/NW-Registry) for the new board type if this is a new hardware version (see [address registry](https://github.com/NorthernWidget/NW-Device-Specification) for Schema 1 board type assignment)
7. **DOI badge** — replace any deprecated `latestdoi`/`GITHUB_REPO_ID` format; use concept DOI (ask maintainer; never guess)
8. **`CITATION.cff`** — create or update
9. **`.zenodo.json`** — create or update; `upload_type: other`

## Version numbering

NorthernWidget hardware repos use a `HWmajor.HWminor.FWversion` scheme rather than standard semver:

| Field | Meaning |
|-------|---------|
| `HWmajor` | Major hardware revision — significant layout or functional change |
| `HWminor` | Minor hardware revision — component substitution, silkscreen fix, small layout tweak |
| `FWversion` | Version of the firmware burned directly to the sensor's onboard MCU (e.g. ATtiny on Haar or Libelle) |

`FWversion` refers to the embedded firmware on the sensor itself, **not** the Arduino library that runs on the controller (Margay, Okapi). Library versioning is tracked separately in the corresponding `*_Library` repo.

See [version-numbering-standards](https://github.com/NorthernWidget/version-numbering-standards) for the full NorthernWidget versioning scheme.

## Notes

- The `Project-` prefix is a NorthernWidget convention for hardware design repos; see [CONTRIBUTING.md](CONTRIBUTING.md).
- Controllers (Margay, Okapi) follow this same checklist. They do not require Schema 1 sensor register map compliance, but their board type should appear in NW-Registry.
