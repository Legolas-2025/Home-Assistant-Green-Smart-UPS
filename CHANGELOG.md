# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

- Boot loop may occur when mains power restores after battery cutoff
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
| [1.0a](#1.0a---2026-08-21) | Pre-Release Alpha | 2026-08-21 | Current |

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

Copyright (c) 2026 HAG Smart UPS Contributors
