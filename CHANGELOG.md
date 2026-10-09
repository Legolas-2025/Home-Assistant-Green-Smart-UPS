# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.2.0-alpha] - 2026-10-10

### Pre-Release Alpha — Firmware Correctness & Documentation

> **Warning**: This is a pre-release alpha. The configuration was validated against ESPHome **2026.9.1**
> (`esphome config` → `Configuration is valid!`), but hardware behaviour has not been exhaustively tested.
> Use in production systems at your own risk.

> ⚠️ **Upgrade notice**: v1.1a recorded mains-event timestamps against the **wrong edge** of the detection
> signal. Historical `Last Power Outage Time` / `Last Mains Restore Time` values stored before this release
> are swapped and cannot be corrected retroactively.

### Fixed

- **Mains detection triggers were inverted (`binary_sensor.on_press` / `on_release`)**
  - The PC817 optocoupler pulls GPIO4 LOW while mains is present. With `inverted: true`, LOW maps to a
    logical `true`, so `on_press` fires on **mains restored** and `on_release` on **mains lost**.
  - The handlers were the other way round: the outage timestamp was written when power *arrived*, and the
    restore timestamp when power *left*.
  - Both handlers are now swapped. Confirmed against `esphome/core/automation.h`, where `on_press` is bound
    to `TriggerOnTrueForwarder` and `on_release` to `TriggerOnFalseForwarder`.
  - Side effect fixed: `low_bat_logged` is now reset when mains is restored rather than when it is lost, so
    the 3 % critical-battery event logs correctly on each subsequent outage.
- **Mains-restored timestamp could be recorded as `1970-01-01`**
  - The restore handler called `strftime()` without first checking `current_time.now().is_valid()`.
  - On an unsynced clock it stored a bogus epoch value as a real event time.
  - It now writes `Time Sync Pending`, matching the behaviour already present in the outage handler.
- **`sensor:` update-interval number was inert**
  - `update_interval_slider` was declared and advertised in the README but referenced by no component.
  - See *Added* below for the wiring.

### Added

- **Runtime update-interval control**
  - Added `id: ups_fuel_gauge` to the MAX17043 platform so the polling loop can be retuned at runtime.
  - Wired `set_action` on the *Sensor Update Interval* template number to `set_update_interval()`.
  - Added an `on_boot` hook (priority `-100`) so the NVS-restored value is re-applied after every reboot.
  - Previously the entity changed nothing at runtime.
- **`safe_mode` boot-loop guard**
  - `boot_is_good_after: 2min`, `num_attempts: 10`, `reboot_timeout: 5min`.
  - Addresses the long-standing "Boot Loop on Power Restore" known issue by falling back to the last
    known-good firmware after repeated failed boots.
  - Added as a **top-level component** — `ota: - platform: safe_mode` is *not* valid; `safe_mode` is a
    standalone component that `ota:` auto-loads.
- **Documentation**
  - Added a minimum-version requirement table explaining why each floor matters:
    - ESPHome **2026.9.1** (validated against) — `openthread_info` sensor/text_sensor platforms
    - ESPHome **2025.11.0** — `dallas_temp` regained the `index:` option (removed in 2024.6, restored 2025.11)
    - ESPHome **2025.6.0** — the `openthread` component was introduced
  - Documented that `on_press` / `on_release` are *logical state* triggers, not physical pin edges.
  - Documented that ESPHome's built-in `SDA`/`SCL` aliases resolve to GPIO21/GPIO22 on every ESP32
    variant — including the H2, where GPIO21 is a SPI-flash pin. Explicit `sda:`/`scl:` are mandatory.
  - Added a DS18B20 addressing section noting `index:` is 0-based and order-dependent.
  - Added a "What's New in v1.2.0-alpha" section to the README.

### Changed

- **Documented `openthread: tlv:` as correct, with an explicit warning.**
  - A third-party review recommended renaming `tlv:` to `network_dataset:`. That key does not exist;
    applying it breaks the build with
    `[network_dataset] is an invalid option for [openthread]`.
  - The real key is `CONF_TLV = "tlv"` (`esphome/components/openthread/const.py`). The value is a
    hex-encoded operational dataset TLV, **not** a file path.
  - Added inline comments in the YAML plus README warnings so this is not "corrected" again.
- Version scheme standardised to `1.2.0-alpha` (was `1.1a` / `1.0a`) so the tag parses as valid SemVer and
  GitHub's pre-release UI recognises it.

### Documentation Fixes

- **Thread dataset example in `secrets.yaml` was invalid.** The sample had an odd number of hex characters
  and embedded prose, so copying it verbatim failed validation with
  `TLV must have an even number of hex characters`. Replaced with a single quoted line of valid hex plus
  an explicit warning and a key-generation command.
- **Runtime claim corrected.** The title said "Up To 6 Hours Runtime" while the calculation and changelog
  both conclude 5–5.5 hours. Title now reads **5.5 Hours**.
- **Power-pin locations corrected.** The README stated 5V/GND were on the top *left* and 3V3 on the top
  *right*. Board photo inspection confirms **5V, GND and 3V3 are all on the top right**, contiguous as one
  3-pin header, with no power pin on the left column. Corrected in the pin tables, wiring steps 4.4 / 8.2
  and the ASCII pinout diagram; added a note about the unused `BAT` pad.
- **Broken filename reference.** `hag_smart_ups_for_esphome.yaml` → `hag_smart_ups.yaml`.
- **Non-existent protection removed.** The features table advertised OVP for the DD4012SA; its actual
  protections are OTP, OCP, SCP, UVLO and BS-voltage. Corrected to UVLO/OCP/SCP/OTP.
- **False persistence claim removed.** The *Time Sync* text sensor was documented as "Survives reboot",
  but `saved_time_sync_status` has `restore_value: no`. Now reads "Resets on reboot".
- **Dead entity documented honestly.** The *Sensor Update Interval* control is no longer listed as
  "Fixed"-interval elsewhere in the doc.
- **Garbled trailing text removed** — a stray `stems at your own risk. Contributions and testing feedback
  are welcome.` line at the end of the README.
- **Copyright attribution unified.** The changelog footer said "HAG Smart UPS Contributors" while `LICENSE`
  said "Legolas-2025". The changelog now matches the LICENSE.
- Softened the "100% interchangeable" claim for PC817/EL817 to a drop-in replacement with a CTR caveat.
- Replaced the non-portable CHANGELOG anchor links with plain-text version names.

### Known Issues

- DS18B20 sensors are addressed by `index:` (probe order), which can change if sensors are swapped.
  **Workaround**: record the hex addresses ESPHome prints on first boot and switch both entries to `address:`.
- Boot loop may still occur when mains power restores after battery cutoff.
  **Workaround**: fit a 100 µF – 220 µF electrolytic across the DD4012SA 5V output (pin 3) and GND (pin 2).
  The new `safe_mode` block limits the damage but does not prevent the underlying brown-out.

### Security Considerations

- API encryption key must be generated per device and stored in `secrets.yaml`; do not commit it.
- Thread network datasets contain the network key and must be kept out of public repositories.
- The README ships an **illustrative placeholder** dataset and key — generate your own before deploying.

---

## [1.1a] - 2026-09-04

### Pre-Release Alpha — DD4012SA Migration

> **Warning**: This is an alpha release. The software is functional but has not been extensively tested in all scenarios. Use in production systems at your own risk.

> ⚠️ **Superseded**: see [1.2.0-alpha](#120-alpha---2026-10-10) — the mains-detection triggers in this
> release are inverted.

### Changed

- **Power Stage: Mini360 → DD4012SA**
  - Replaced the Mini360 adjustable buck converter with the **DD4012SA** factory-set buck converter (5V variant)
  - **Removed** all Mini360 references, including potentiometer adjustment procedures and "adjust-while-measuring" steps
  - The 5V rail is now generated by a TO-220-compatible DD4012SA module with breadboard-friendly 2.54 mm pin pitch
  - The DD4012SA accepts the 12V HW-465C rail directly (6.5V – 40V input range) and outputs a fixed 5.0V with no user adjustment
- **Bill of Materials**
  - Replaced the Mini360 entry with **DD4012SA Buck Converter (5V variant)** — `DD4012SA_5V`
  - Updated all related component notes to reference the DD4012SA
- **Wiring and Schematics**
  - Updated the System Architecture Overview, Detailed Wiring Schematic, ESP32-H2 Wiring Detail, and Sensor Wiring Detail to show the DD4012SA in place of the Mini360
  - DD4012SA pin labels (`IN+`, `GND`, `OUT+`) are now used consistently across all diagrams
  - The wiring guide no longer requires any potentiometer-tuning step
- **Estimated Runtime**
  - DD4012SA efficiency assumed at **~85%** (typical for ~100 mA load from 12V input)
  - Combined system efficiency updated to **~77%**
  - Runtime estimate revised to **5 – 5.5 hours** (up from the Mini360-based figure)
- **Troubleshooting**
  - Added a new **"DD4012SA-Specific Issues"** section covering UVLO, OCP, OTP, and wrong-SKU symptoms
  - Updated the **"Components Getting Hot"** table to describe DD4012SA thermal behavior
  - Updated the **"Boot Loop on Power Restore"** workaround to reference the DD4012SA 5V output instead of the Mini360
- **Documentation**
  - Added a new **"DD4012SA Buck Converter - Detailed Specifications"** section with full electrical, mechanical, and thermal specs
  - Added a new **"Why DD4012SA Instead of Mini360?"** section comparing the two regulators across nine key concerns
  - Updated the README version line from `1.0a` to **`1.1a (Pre-Release Alpha) — DD4012SA migration`**
  - Updated **"Last Updated"** to September 2026

### Added

- **DD4012SA Reference Material**
  - DD4012SA configuration table (5V variant specs)
  - Built-in protections table: OTP, OCP, SCP, UVLO, BS-voltage
  - DD4012SA pinout diagram (3 pins: `IN+`, `GND`, `OUT+`)
  - Pin connections table mapping DD4012SA pins to HW-465C and ESP32-H2
  - Optional 100 µF – 220 µF output-stabilization capacitor recommendation
- **Bench-Test Procedure (Optional)**
  - Step-by-step instructions to verify the 5V output with a multimeter before connecting to the ESP32-H2
- **Phase 2: Prepare the DD4012SA Buck Converter**
  - SKU verification checklist (3.3V, 5V, 12V variants)
  - Removed the "adjust potentiometer" step that existed for the Mini360

### Removed

- All references to the **Mini360** buck converter
- All references to the Mini360 **potentiometer** and its adjustment procedure
- "Adjust potentiometer while measuring with multimeter" assembly step
- Mini360-specific troubleshooting entries

### Known Issues

- Boot loop may still occur when mains power restores after battery cutoff
  - **Workaround**: Add a 100 µF – 220 µF electrolytic capacitor across the **DD4012SA** 5V output (Pin 3) and GND (Pin 2)
- DS18B20 sensor addressing requires manual configuration after initial boot
  - **Workaround**: Use index-based addressing initially, then replace with address-based configuration after identifying sensor IDs

### Security Considerations

- API encryption key must be configured in `secrets.yaml` before deployment
- Thread network dataset should be kept secure and not shared publicly
- Default configurations are not suitable for production without proper key management

---

## [1.0a] - 2026-08-21

### Pre-Release Alpha

> **Warning**: This is an alpha release. The software is functional but has not been extensively tested in all scenarios. Use in production systems at your own risk.

### Added

- **Core UPS Monitoring**
  - Real-time battery level monitoring via MAX17043 I2C fuel gauge
  - Battery voltage sensing with configurable thresholds
  - Dual DS18B20 temperature sensors for battery cell monitoring
  - Mains power status detection using PC817 optocoupler
- **Thread Network Integration**
  - OpenThread protocol support for wireless connectivity
  - ESP32-H2 as Minimal Thread Device (MTD)
  - Thread network diagnostics (signal strength, device role, IP address, channel)
  - Seamless integration with SLZB-MR5U and compatible OTBR hardware
- **Smart Features**
  - Persistent event logging with NVS-backed timestamps
  - Automatic power outage detection and timestamp recording
  - Low battery event logging (3% threshold during outage)
  - Intelligent time synchronization with auto-retry mechanism (up to 4.5 minutes)
  - Configurable sensor update intervals (10s - 120s)
- **Home Assistant Integration**
  - Native ESPHome API with encrypted communication
  - Push-based entity updates for instant state changes
  - Remote device restart capability
  - Comprehensive set of exposed entities:
    - 5 sensors (battery level, voltage, 2x temperature, signal strength)
    - 1 binary sensor (mains power status)
    - 5 text sensors (time sync, outage logs, reboot time)
    - 1 button (restart)
    - 1 number control (update interval slider)
- **Hardware Protection**
  - Hardware cutoff design for automatic recovery
  - Deep discharge prevention via HW-465C module
  - Battery cell temperature monitoring for thermal protection
- **Documentation**
  - Comprehensive step-by-step assembly guide
  - Detailed wiring schematics and pinout references
  - Troubleshooting section for common issues
  - Estimated runtime calculations for various battery configurations

### Known Issues

- Boot loop may still occur when mains power restores after battery cutoff
  - **Workaround**: Add 100µF - 220µF electrolytic capacitor across Mini360 5V and GND terminals
- DS18B20 sensor addressing requires manual configuration after initial boot
  - **Workaround**: Use index-based addressing initially, then replace with address-based configuration after identifying sensor IDs

### Security Considerations

- API encryption key must be configured in secrets.yaml before deployment
- Thread network dataset should be kept secure and not shared publicly
- Default configurations are not suitable for production without proper key management

---

## [Unreleased] - Future Releases

### Potential Future Features

- [ ] Support for additional battery configurations (2S, 3S)
- [ ] Configurable low battery threshold
- [ ] Energy consumption statistics
- [ ] Integration with Home Assistant Energy Dashboard
- [ ] MQTT support for non-Thread deployments
- [ ] Over-the-air (OTA) firmware update improvements
- [ ] Web UI for configuration without Home Assistant
- [ ] Persist `low_bat_logged` across reboots so a mid-outage reboot cannot re-fire the 3 % event

### Potential Enhancements

- [ ] Email/push notifications for critical events
- [ ] Historical data logging and visualization
- [ ] Support for multiple battery packs
- [ ] Solar panel input monitoring
- [ ] Expansion GPIO pins for additional sensors

---

## Version History

| Version | Type | Date | Status |
|---------|------|------|--------|
| 1.2.0-alpha | Pre-Release Alpha | 2026-10-10 | **Current** |
| 1.1a | Pre-Release Alpha | 2026-09-04 | Superseded — mains triggers inverted |
| 1.0a | Pre-Release Alpha | 2026-08-21 | Superseded |

---

## Contributing

If you find bugs or have suggestions for improvements, please:

1. Open an issue on GitHub with detailed reproduction steps
2. Submit pull requests with clear descriptions of changes
3. Test on various hardware configurations when possible

For major changes, please open an issue first to discuss the proposed modifications.

---

## License

This project is released under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 Legolas-2025
