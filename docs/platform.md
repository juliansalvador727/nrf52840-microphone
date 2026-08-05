# Platform and Toolchain

## 1. Hardware

Target board:

```text
Adafruit Feather nRF52840 Sense
```

Important onboard resources:

- Nordic nRF52840
- Onboard digital PDM microphone
- USB connector and native nRF52840 USB device support
- User button
- NeoPixel
- Additional controllable LEDs
- Adafruit UF2 bootloader
- Existing SoftDevice supplied with the bootloader image

Exact pin mappings must be copied from the official schematic, board definition, or datasheet and recorded in `firmware/platform/board.h`.

Do not infer pins from Arduino labels alone.

## 2. SDK

Preferred SDK:

```text
nRF5 SDK 17.1.0
```

This is chosen because:

- It is the latest nRF5 SDK release.
- It supports the nRF52840, S140, USB, PDM, and BLE.
- It provides a direct C/event-driven environment.
- The project is educational and not a new commercial product.

The nRF5 SDK is in maintenance mode. This project intentionally uses it for low-level learning.

## 3. Compiler

Preferred reference compiler:

```text
GNU Arm Embedded 9-2020-q2-major
```

Before changing the installed compiler, record:

```bash
arm-none-eabi-gcc --version
```

A newer compiler may work, but any warning, ABI, linker, or startup issue must be resolved rather than ignored.

## 4. Build system

Use:

```text
CMake + Ninja
```

The nRF5 SDK is not CMake-native. The project must explicitly configure:

- CPU flags
- ABI flags
- SDK include directories
- SDK source files
- Startup assembly
- CMSIS
- nrfx drivers
- USB libraries
- BLE libraries
- SoftDevice headers
- Linker script
- Post-build HEX/BIN/UF2 conversion

Expected compile flags include the correct Cortex-M4F and floating-point ABI settings. These must match all linked objects.

## 5. UF2 flashing

The normal workflow is:

1. Build an application HEX file at the correct linked addresses.
2. Convert the HEX file to UF2 using the nRF52840 family identifier.
3. Double-reset the Feather.
4. Copy the UF2 file to the mounted bootloader drive.

The build should generate:

```text
microphone.elf
microphone.hex
microphone.bin
microphone.uf2
microphone.map
```

The bootloader must be treated as protected because no SWD/J-Link recovery hardware is available.

## 6. Bootloader inspection

Before linking the BLE application, double-reset the board and inspect:

```text
INFO_UF2.TXT
```

Record:

```text
Bootloader version:
SoftDevice version:
Board identifier:
UF2 family:
```

### Bootloader information

```text
Bootloader version: 0.8.0
SoftDevice version: S140 6.1.1
Board identifier: nRF52840-Feather-Sense
Bootloader date: Sep 29 2023
UF2 family identifier: 0xADA52840
```

Do not let an agent guess these values.

## 7. Flash layout

The application must not overlap:

- MBR
- SoftDevice
- Adafruit bootloader
- Bootloader configuration
- MBR parameter page
- Bootloader settings

Verified layout for the installed S140 6.1.1 and Adafruit bootloader 0.8.0:

```text
0x00000-0x25FFF   MBR and S140 6.1.1
0x26000-0xECFFF   Application
0xED000-0xF3FFF   Reserved application/DFU space
0xF4000-0xFD7FF   Bootloader code
0xFD800-0xFDFFF   Bootloader configuration
0xFE000-0xFEFFF   MBR parameter page
0xFF000-0xFFFFF   Bootloader settings
```

The application linker values are:

```text
NRF_FLASH_ORIGIN = 0x26000
NRF_FLASH_LENGTH = 0xC7000
Application end, exclusive = 0xED000
```

These values are verified against:

- [Adafruit nRF52 Bootloader 0.8.0 `linker/nrf52840.ld`](https://github.com/adafruit/Adafruit_nRF52_Bootloader/blob/0.8.0/linker/nrf52840.ld)
- [Adafruit nRF52 Arduino `nrf52840_s140_v6.ld`](https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/cores/nRF5/linker/nrf52840_s140_v6.ld)
- [Adafruit nRF52 Arduino Feather Sense board configuration for S140 6.1.1](https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/boards.txt)

The application ends at `0xED000`, not at the bootloader start. The region from
`0xED000` through `0xF3FFF` must remain reserved for the Adafruit application and
DFU layout.

### Linker variables

The linker setup should define values such as:

```cmake
NRF_FLASH_ORIGIN
NRF_FLASH_LENGTH
NRF_RAM_ORIGIN
NRF_RAM_LENGTH
```

The application flash end must remain below the bootloader start.

## 8. SoftDevice RAM layout

The RAM origin depends on the BLE configuration.

Factors include:

- Number of peripheral and central links
- ATT MTU
- Attribute table size
- Data Length Extension
- Vendor-specific UUID count
- GATT queues
- Security configuration

Use a conservative initial BLE configuration:

```text
Peripheral links: 1
Central links: 0
ATT MTU: begin at default, then raise deliberately
Vendor UUID count: minimum needed
Attribute table: sized to the custom service
```

The application should call the SoftDevice enable/configuration path and capture the required RAM start reported by the SDK. Update the linker script only from measured/configured output.

### TODO(USER): Explain the SoftDevice RAM process

In your own words, explain:

- Why the BLE application cannot simply use RAM from `0x20000000`
- How BLE configuration changes SoftDevice RAM use
- How you will determine the correct application RAM origin
- What failure you expect if the linker RAM origin is wrong

## 9. Clock configuration

### High-frequency clock

Do not deliberately use the internal HFINT oscillator as the final audio clock source.

Use the external high-frequency crystal oscillator where required for stable PDM timing and radio operation.

The intended PDM configuration is:

```text
PDM clock: 1.280 MHz
Decimation ratio: 80
PCM sample rate: 16 kHz
```

USB mode should explicitly ensure the required high-frequency clock is running before capture begins.

BLE mode must respect SoftDevice ownership of clock and radio resources.

### Low-frequency clock

The LFCLK choice depends on the board hardware.

Possible choices:

- LFXO if the board includes a 32.768 kHz crystal
- Calibrated LFRC otherwise

### TODO(USER): Verify the LFCLK hardware

Using the Feather Sense schematic, record:

```text
32.768 kHz crystal present: No
Chosen LFCLK source: Calibrated LFRC
Reason: The XL1 and XL2 pins are used by the PDM microphone, and Adafruit’s board configuration explicitly selects LFRC.
```

## 10. Logging without SWD

RTT cannot be read without a debug probe.

Use:

- USB CDC logging in debug builds
- A fixed in-memory event log
- Retained fault records
- Runtime counters
- LED error codes

Build variants:

```text
usb-release
usb-debug
ble-release
ble-debug
host-tests
```

A BLE debug build may use USB CDC for diagnostics while BLE carries audio, but it must not expose USB Audio simultaneously.

## 11. Pin mapping

Create `firmware/platform/board.h` with:

- PDM CLK
- PDM DATA
- User button
- NeoPixel
- Red LED
- Blue LED
- Active-high/active-low definitions
- Pins reserved by QSPI, NFC, sensors, USB, or bootloader behavior

### Board mapping

```c
#define BOARD_PDM_CLK_PIN       NRF_GPIO_PIN_MAP(0, 1)
#define BOARD_PDM_DATA_PIN      NRF_GPIO_PIN_MAP(0, 0)
#define BOARD_BUTTON_PIN        NRF_GPIO_PIN_MAP(1, 2)
#define BOARD_NEOPIXEL_PIN      NRF_GPIO_PIN_MAP(0, 16)
#define BOARD_RED_LED_PIN       NRF_GPIO_PIN_MAP(1, 9)
#define BOARD_BLUE_LED_PIN      NRF_GPIO_PIN_MAP(1, 10)
```

Electrical behavior:

- User button: active-low; a press connects P1.02 to ground
- Red LED: active-high
- Blue LED: active-high
- NeoPixel: serialized data signal, not a simple active level

Reserved onboard pins:

- NFC1: P0.09
- NFC2: P0.10
- Voltage monitor: P0.29
- I2C SCL: P0.11
- I2C SDA: P0.12
- IMU INT1: P1.11
- APDS9960 interrupt: P1.00, active-low open-drain
- QSPI clock: P0.19
- QSPI chip select: P0.20
- QSPI IO0: P0.17
- QSPI IO1: P0.22
- QSPI IO2: P0.23
- QSPI IO3: P0.21

Verification sources:

- [Adafruit Feather Sense pinout](https://learn.adafruit.com/adafruit-feather-sense/pinouts)
- [Adafruit Feather nRF52840 Sense `variant.cpp`](https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/variants/feather_nrf52840_sense/variant.cpp)
- [Adafruit Feather nRF52840 Sense `variant.h`](https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/variants/feather_nrf52840_sense/variant.h)
- [Official Adafruit Feather nRF52840 Sense Rev C schematic](https://github.com/adafruit/Adafruit-Feather-nRF52840-Sense-PCB/blob/master/Adafruit%20Feather%20nRF52840%20Sense%20Rev%20C.sch)

## 12. Recovery limitations

Because there is no SWD/J-Link hardware:

- Do not overwrite the bootloader.
- Do not change UICR casually.
- Do not disable the reset path required by the bootloader.
- Do not rely on firmware that can only be recovered through SWD.
- Test linker addresses before copying UF2.
- Keep a known-good LED application UF2 available.
