# ESP32 RFMON

**8-channel RF Power Monitor and SNMP Agent for Amateur Radio**

Created with assistance from Grok (xAI) and implemented by [TMDrake](https://github.com/tmdrake).

The 1-port NodeMCU build is [ESP8266-RFMON](https://github.com/tmdrake/ESP8266-RFMON). This repository is the ESP32-DevKitC build: eight PTT inputs, eight RF taps, and one temperature sensor on a single board.

![License](https://img.shields.io/github/license/tmdrake/ESP32-RFMON)
![Language](https://img.shields.io/badge/language-C%2B%2B-blue)
![Platform](https://img.shields.io/badge/platform-ESP32-orange)

**Rev 2.0.0**

## About

This sketch turns an ESP32-DevKitC into a network RF power monitor. Each channel watches an active-low PTT line. While that line is active, the matching ADC pin is scaled to a percent and published over SNMP. A DS18B20 on GPIO4 reports temperature. A serial menu stores Wi-Fi, trap destination, community strings, and contact in EEPROM.

Use it to watch several radios or amplifier samples from Zabbix, LibreNMS, PRTG, Observium, or `snmpget`. Traps are in the sketch and are off until you turn them on.

`sysDescr` reports `ESP32-DevKitC RF Power Monitor by Drakes Computers, Rev 2.0.0`.

## Features

- Eight active-low PTT inputs with internal pull-ups.
- Eight RF percent readings, sampled only while that channel's PTT is active.
- Last RF percent kept after PTT releases. The live percent returns to 0 while idle.
- DS18B20 temperature, 9-bit, read every 5 seconds. Value is degrees Celsius times 100.
- Wi-Fi RSSI.
- SNMPv1 and SNMPv2c agent, RFC1213 system group plus enterprise OIDs under `1.3.6.1.4.1.63637`.
- Optional SNMPv2c Inform on each PTT press, sent to a configured trap host.
- Serial menu at 115200 baud. Option 8 writes EEPROM and reboots.
- DHCP or static IPv4. Factory default is DHCP.
- Debug dump every 50 ms, off until menu item 9.

## Hardware

### Parts

- ESP32-DevKitC, or another ESP32 module that breaks out the pins below.
- One DS18B20 and a 4.7 kΩ pull-up from the data pin to 3.3 V.
- Eight RF taps that produce about 0–1 V DC into the ADC pins. A 10 kΩ pull-down on each tap is a good idea.
- Eight active-low PTT lines. Isolate them. A transistor or optocoupler is safer than wiring a radio accessory pin straight to the ESP32.

### Wiring

| Channel | RF ADC | PTT (active low) | Notes |
|---------|--------|------------------|-------|
| 1 | GPIO32 | GPIO12 | GPIO12 is a boot strap pin. See below. |
| 2 | GPIO33 | GPIO13 | |
| 3 | GPIO34 | GPIO14 | GPIO34 is input-only. |
| 4 | GPIO35 | GPIO15 | GPIO35 is input-only. |
| 5 | GPIO36 | GPIO16 | GPIO36 (SENSOR_VP) is input-only. |
| 6 | GPIO25 | GPIO17 | GPIO25 is ADC2. Unreliable while Wi-Fi is up. |
| 7 | GPIO26 | GPIO18 | GPIO26 is ADC2. Unreliable while Wi-Fi is up. |
| 8 | GPIO39 | GPIO19 | GPIO39 (SENSOR_VN) is input-only. |
| Temperature | GPIO4 | — | DS18B20 data. 4.7 kΩ to 3.3 V. |

GND is common. Power the ESP32 from USB or a clean 5 V pin on the DevKit. Power the DS18B20 from 3.3 V.

**RF safety.** Do not connect transmitter output, a high-voltage sampler, or a negative voltage to an ESP32 pin. The ADC wants a ground-referenced DC sample inside 0–3.3 V. The percent math treats 1088 counts as 100%, which is under 1 V on the default 12-bit scale. Anything above that clamps at 100%.

### GPIO12 at boot

GPIO12 (MTDI) selects the flash voltage. If it is high when the chip comes out of reset, many modules try to use 1.8 V flash and never boot. This sketch turns on the internal pull-up, so the pin sits high unless the PTT circuit pulls it down. If a freshly flashed board is silent, GPIO12 is the first place to look. An external pull-down that wins over the internal pull-up, or a PTT driver that holds the pin low at reset, keeps the strap in the normal 3.3 V flash state.

### Channels 6 and 7

GPIO25 and GPIO26 are ADC2. On the ESP32, ADC2 and Wi-Fi share hardware, and `analogRead` on ADC2 is not reliable while the radio is associated. Channels 1–5 and 8 use ADC1 (GPIO32, 33, 34, 35, 36, 39) and are the ones to trust for RF percent. Channels 6 and 7 are wired in this revision and will often read 0, or jump, after Wi-Fi connects.

## How a reading is made

PTT is `digitalRead` with `INPUT_PULLUP`. `0` is active (transmitting). `1` is idle.

While PTT is active:

- ADC counts below 10 become 0%.
- Otherwise the percent is `map(raw, 0, 1088, 0, 100)`, limited to 0–100.
- That percent is also stored as the last-PTT value.

While PTT is idle, the live percent is forced to 0. The last-PTT value stays where it was.

This is a scaled ADC count, not watts. 1088 counts is 100% in this firmware. Change the `map` call in `loop` if your tap uses a different full-scale count.

Temperature is Celsius times 100, so 2530 means 25.30 °C. If the sensor is missing, the OID is `-9999`.

## Software setup

### What you need

- [Arduino IDE](https://www.arduino.cc/en/software) 2.x, or PlatformIO with the Arduino-ESP32 core.
- ESP32 board package. In Arduino IDE Preferences, add:

  `https://espressif.github.io/arduino-esp32/package_esp32_index.json`

  Then install **esp32** from Boards Manager.
- Libraries, from Library Manager unless noted:
  - `WiFi` and `WiFiUdp` — part of the ESP32 core
  - [SNMP_Agent](https://github.com/0neblock/SNMP_Agent) by 0neblock, which also provides `SNMPTrap.h`. Install it as an Arduino library (the GitHub repo is the library).
  - `OneWire` by Paul Stoffregen
  - `DallasTemperature` by Miles Burton
  - `EEPROM` — part of the ESP32 core

### Install

1. Clone this repository. The folder name and the sketch name both have to be `ESP32-RFMON`.

   ```bash
   git clone https://github.com/tmdrake/ESP32-RFMON.git
   ```

2. Open `ESP32-RFMON.ino`.
3. Board: **ESP32 Dev Module**. Leave the default partition. Upload speed 115200 or 921600, whichever your cable accepts.
4. Upload. Open Serial Monitor at **115200**, newline ending.
5. The menu prints before Wi-Fi starts. Association waits up to 30 seconds, and the menu does not answer during that wait. After `setup` finishes, any line you send reprints the menu.

Factory defaults, used until EEPROM has been saved once:

| Setting | Default |
|---------|---------|
| SSID | `deamonnet` |
| Password | `<hidden>` (placeholder, not a real key) |
| Address mode | DHCP |
| Static IP, if you turn DHCP off | `192.168.254.247` / `255.255.255.0` / gateway `192.168.254.1` |
| Trap destination | `192.168.254.10` |
| Read community | `public` |
| Write community | `private` |
| sysContact | `Your Name <your.email@example.com>` |
| Traps | off (`enableTraps = false` in the sketch) |

Put your own SSID and password in with the menu, then choose **8** so they survive reboot. Do not commit a real password back to this repository.

### Serial menu

| Key | Action | Takes effect |
|-----|--------|----------------|
| 1 | SSID, 31 characters max | Next boot, after you save |
| 2 | Wi-Fi password, 63 characters max | Next boot, after you save |
| 3 | Trap IP | Next boot, after you save. Used only when traps are compiled on |
| 4 | sysContact, 63 characters max | SNMP at once. EEPROM only if you save |
| 5 | Read community, then write community | Agent restarts at once. EEPROM only if you save |
| 6 | DHCP, or static IP, mask, and gateway | Next boot, after you save |
| 7 | Print the current settings | Now. This prints the Wi-Fi password on the serial port |
| 8 | Write EEPROM and restart | Now |
| 9 | Toggle the 50 ms debug listing | Now. Not stored |

EEPROM is 512 bytes. Saved data loads only when byte 241 is `0xAA`, which option 8 writes.

| Offset | Length | Contents |
|--------|--------|----------|
| 0 | 32 | SSID |
| 32 | 64 | Password |
| 96 | 4 | Trap IP |
| 100 | 64 | sysContact |
| 164 | 32 | Read community |
| 196 | 32 | Write community |
| 228 | 4 | Static IP |
| 232 | 4 | Subnet mask |
| 236 | 4 | Gateway |
| 240 | 1 | DHCP flag, nonzero means DHCP |
| 241 | 1 | Init flag `0xAA` |

`sysName` (`ESP32-RF-Monitor`), `sysLocation` (`Lab`), and `sysDescr` are compiled in. The menu does not change them.

## SNMP

Port 161, UDP. Read community `public` and write community `private` until you change them. The agent answers SNMPv1 and SNMPv2c.

Replace `<ip>` with the address printed at boot.

```bash
snmpget -v2c -c public <ip> 1.3.6.1.2.1.1.1.0
snmpget -v2c -c public <ip> 1.3.6.1.4.1.63637.1.0.1
snmpget -v2c -c public <ip> 1.3.6.1.4.1.63637.1.1.1
snmpget -v2c -c public <ip> 1.3.6.1.4.1.63637.1.4
snmpwalk -v2c -c public <ip> 1.3.6.1.4.1.63637
```

The firmware OID strings include a leading dot (`.1.3.6.1...`). Net-SNMP accepts them with or without that dot.

### System group

| OID | Name | Access | Value |
|-----|------|--------|-------|
| `1.3.6.1.2.1.1.1.0` | sysDescr | read | ESP32-DevKitC RF Power Monitor by Drakes Computers, Rev 2.0.0 |
| `1.3.6.1.2.1.1.2.0` | sysObjectID | read | `1.3.6.1.4.1.63637` |
| `1.3.6.1.2.1.1.3.0` | sysUpTime | read | TimeTicks, hundredths of a second |
| `1.3.6.1.2.1.1.4.0` | sysContact | read | Serial menu item 4 |
| `1.3.6.1.2.1.1.5.0` | sysName | read | `ESP32-RF-Monitor` |
| `1.3.6.1.2.1.1.6.0` | sysLocation | read | `Lab` |
| `1.3.6.1.2.1.1.7.0` | sysServices | read | `72` |

### Enterprise objects

Channel `N` is 1 through 8. The same enterprise number is `sysObjectID`.

| OID | Meaning |
|-----|---------|
| `1.3.6.1.4.1.63637.1.0.N` | Live RF percent, 0–100. 0 while PTT is idle. |
| `1.3.6.1.4.1.63637.1.1.N` | PTT. `0` active, `1` idle. |
| `1.3.6.1.4.1.63637.1.2.N` | Last RF percent captured while PTT was active. |
| `1.3.6.1.4.1.63637.1.3` | Wi-Fi RSSI in dBm, or 0 if Wi-Fi is down. |
| `1.3.6.1.4.1.63637.1.4` | Temperature, Celsius times 100. `-9999` if the DS18B20 is missing. |

These integers are registered as writable because that is how `addIntegerHandler` is called. `loop` overwrites live RF, PTT, RSSI, and temperature, so a SET on those OIDs does not stick. Last-PTT percent is refreshed on the next active sample.

### Traps

`enableTraps` defaults to `false`. Nothing is sent until you set that flag to `true` and flash the board again. With the flag on, and Wi-Fi up:

- A PTT edge from idle to active sends an SNMPv2c Inform to the trap IP.
- The trap OID is `1.3.6.1.4.1.63637.2.N` for channel `N`.
- The varbind is that channel's last RF percent.
- Delivery uses 2 tries and a 5 second timeout. The serial port prints success or failure.
- A SET of the demo integer `1.3.6.1.4.1.5.1` also sends a trap. Even values go out as an SNMPv2c Inform. Odd values go out as an SNMPv1 trap. That trap OID is `1.3.6.1.2.1.33.2`.

The trap PDU uses the read community. Menu item 3 sets the destination. Menu item 8 stores it.

A host running `snmptrapd` on UDP 162 is enough to see them:

```bash
sudo snmptrapd -f -Lo -c /dev/null
```

### Demo OIDs from the library example

These come from the SNMP_Agent example and are not part of the radio tree. They are still served.

| OID | What it is |
|-----|------------|
| `1.3.6.1.4.1.5.0` | Integer that increments about every 50 ms. |
| `1.3.6.1.4.1.5.1` | Writable integer. A SET can fire the demo trap when traps are enabled. |
| `1.3.6.1.4.1.4.0` | Same incrementing integer as `.5.0`. |
| `1.3.6.1.4.1.5.9` | 4-byte opaque value. |
| `1.3.6.1.4.1.5.11` | Read-only string `This value will never change`. |
| `1.3.6.1.4.1.5.12` | Writable string, 25 bytes, default `This is changeable`. |

Enterprise numbers 4 and 5 belong to other organizations in the IANA PEN registry. Poll `1.3.6.1.4.1.63637` for this device.

## First check

1. Flash the board and open 115200 serial.
2. Menu item 1, then 2, then 8. Wait for the reboot and note the IP address.
3. `snmpget -v2c -c public <ip> 1.3.6.1.2.1.1.1.0` should return the Rev 2.0.0 sysDescr.
4. Ground a PTT pin (channel 1 is GPIO12). `1.3.6.1.4.1.63637.1.1.1` should read `0`.
5. With PTT held, put a small DC voltage on that channel's ADC pin. `1.3.6.1.4.1.63637.1.0.1` should rise. Release PTT. The live percent goes to 0 and `.1.2.1` keeps the last value.
6. `1.3.6.1.4.1.63637.1.4` should be about the room temperature times 100 once the DS18B20 has answered. The first polls can be `-9999`.

Menu item 9 prints every channel, the temperature, and free heap twice a second or so. Turn it off when you are done. It is noisy and it is not saved.

## License

[GNU GPL v3.0](LICENSE). Copyright (C) 2026 Drake Dragon, Drakes Computers.

You can share and change this program. If you distribute a modified version, it stays under GPL-3.0.
