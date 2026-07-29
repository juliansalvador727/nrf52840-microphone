# Roadmap

## Ground rules

- Target duration: less than two weeks for USB plus basic BLE-to-WAV.
- USB is the priority.
- BLE must not delay completion of a polished wired release.
- Each task should end in a visible or measurable result.
- Do not begin advanced DSP, ADPCM, or C++ host work during the initial sprint.
- Update this file after every work session.

## Day 0 — Decisions and preflight

- [x] Read all design documents.
- [x] Complete the `TODO(USER)` requirements prioritization.
- [x] Complete the architecture ownership diagram.
- [x] Inspect `INFO_UF2.TXT`.
      Bootloader version: 0.8.0
      SoftDevice version: S140 6.1.1
      Board identifier: nRF52840-Feather-Sense
      Bootloader date: Sep 29 2023
      UF2 family identifier: 0xADA52840
- [x] Record installed `arm-none-eabi-gcc` version.
      arm-none-eabi-gcc (15:13.2.rel1-2) 13.2.1 20231009
- [x] Verify the Feather pin map.

      PDM Data: P0.00
      PDM Clock: P0.01
      User Button: P1.02; Active Low, connecting P1.02 to GND
      NeoPixel: P0.16; Serial Data, not a simple active level
      Red LED: P1.09; Active High
      Blue LED: P1.10; Active High
      NFC1: P0.09
      NFC2: P0.10
      Voltage Monitor: P0.29
      SCL P0.11
      SDA P0.12
      IMU INT1: P1.11. polarity is sensor-configurable. if used configure to active low. if mic does not use the imu interrupt, mark reserved.
      APDS IRQ: P1.00

      The QSPI pins are documented in Adafruit's board-support files because they are internal. These are marked as RESERVED for onboard QSPI flash.
      Do not configure them as GPIOs.
      QSPI clock: P0.19
      QSPI CS: P0.20
      QSPI IO0: P0.17
      QSPI IO1: P0.22
      QSPI IO2: P0.23
      QSPI IO3: P0.21

      APDS9960 interrupt output is active-low, open-drain
      `NRF_GPIO_PIN_MAP(0,0)` // data
      `NRF_GPIO_PIN_MAP(0,1)` // clock

      Adafruit Feather Sense pinout (https://learn.adafruit.com/adafruit-feather-sense/pinouts)
      Adafruit board variant.cpp (https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/variants/feather_nrf52840_sense/variant.cpp)
      Adafruit board variant.h (https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/master/variants/feather_nrf52840_sense/variant.h)
      Official Rev C schematic (https://github.com/adafruit/Adafruit-Feather-nRF52840-Sense-PCB/blob/master/Adafruit%20Feather%20nRF52840%20Sense%20Rev%20C.sch)

- [x] Verify LFCLK hardware.
      32.768 kHz crystal present: No
      Chosen LFCLK source: Calibrated LFRC
      Reason: The XL1 and XL2 pins are used by the PDM microphone, and Adafruit’s board configuration explicitly selects LFRC.
- [ ] Decide the initial C test framework.
- [x] Create the repository and initial commit.

**Exit criterion:** no unknown linker origin, pin mapping, or bootloader/SoftDevice version.

## Day 1 — CMake and UF2

- [ ] Add Arm GCC toolchain file.
- [ ] Add startup assembly and linker script.
- [ ] Add nRF5 SDK paths without modifying SDK files.
- [ ] Build a minimal application.
- [ ] Generate ELF, HEX, BIN, MAP, and UF2.
- [ ] Flash a known LED pattern.
- [ ] Confirm software reset.
- [ ] Save a known-good recovery UF2.

**Exit criterion:** one-command build produces a flashable UF2.

## Day 2 — Board services

- [ ] Implement `board.h`.
- [ ] Implement button driver.
- [ ] Implement LED state driver.
- [ ] Implement retained mode request.
- [ ] Verify power-cycle default is USB.
- [ ] Verify a press toggles USB/BLE mode through reset.
- [ ] Add simple host tests for the mode state logic.

**Exit criterion:** mode switching and LED states work without audio.

## Days 3–4 — Frame pool and PDM capture

- [ ] Define `audio_frame_t`.
- [ ] Implement fixed frame pool.
- [ ] Implement pointer queue.
- [ ] Write the first human-owned pool and queue tests.
- [ ] Let the agent add boundary and failure tests.
- [ ] Initialize PDM and clocks.
- [ ] Capture the first buffer.
- [ ] Run continuous ping-pong capture.
- [ ] Track sequence and sample index.
- [ ] Export a finite PCM capture.

**Exit criterion:** continuous samples are captured with explicit ownership and no overrun.

## Day 5 — WAV validation and processing

- [ ] Write a host tool to convert PCM to WAV.
- [ ] Record spoken audio.
- [ ] Confirm correct duration and sample rate.
- [ ] Implement bypass processing mode.
- [ ] Implement fixed gain and saturation.
- [ ] Add RMS, peak, and clipping metrics.
- [ ] Add optional DC blocker.
- [ ] Write human-owned saturation tests.
- [ ] Let the agent expand processing tests.

**Exit criterion:** the WAV is intelligible and metrics are trustworthy.

## Days 6–7 — USB enumeration

- [ ] Study the Nordic UAC example.
- [ ] Complete the descriptor learning checkpoint.
- [ ] Define UAC 1.0 descriptors.
- [ ] Enumerate as an input-only microphone.
- [ ] Handle alternate streaming setting.
- [ ] Handle host mute.
- [ ] Expose debug counters.
- [ ] Record descriptor dumps or host observations.

**Exit criterion:** host lists the device as a 16 kHz mono microphone.

## Days 8–9 — Continuous USB audio

- [ ] Feed PDM frames into USB packets.
- [ ] Verify 32-byte packets per USB frame.
- [ ] Implement underrun silence.
- [ ] Measure queue depth.
- [ ] Test record/start/stop cycles.
- [ ] Test unplug/replug.
- [ ] Test one real calling application.
- [ ] Run one-hour reliability test.
- [ ] Investigate clock drift only if measured.

**Exit criterion:** stable wired microphone with no hidden data loss.

## Day 10 — USB quality and cleanup

- [ ] Record comparison samples.
- [ ] Measure noise, peak, clipping, and DC.
- [ ] Measure approximate latency.
- [ ] Fix major audio artifacts.
- [ ] Run all host tests.
- [ ] Clean warnings.
- [ ] Update README with actual status.
- [ ] Tag the wired release.

**Exit criterion:** polished and demoable USB project even if BLE is unfinished.

## Day 11 — BLE service skeleton

- [ ] Complete BLE protocol TODOs.
- [ ] Generate UUIDs.
- [ ] Configure S140 memory.
- [ ] Add one peripheral connection.
- [ ] Advertise custom service.
- [ ] Add control and status characteristics.
- [ ] Send generated counter data.
- [ ] Expose negotiated MTU and connection parameters.

**Exit criterion:** Python can connect and receive deterministic notifications.

## Day 12 — BLE PCM packetization

- [ ] Implement MTU-dependent fragmentation.
- [ ] Add sequence/sample metadata.
- [ ] Implement backpressure handling.
- [ ] Write first human-owned fragmentation tests.
- [ ] Let the agent add invalid and reordered cases.
- [ ] Stream generated sine-wave PCM.
- [ ] Verify exact reconstruction.

**Exit criterion:** deterministic PCM frames survive BLE fragmentation.

## Day 13 — Python BLE-to-WAV

- [ ] Discover and connect.
- [ ] Start/stop stream.
- [ ] Reassemble frames.
- [ ] Detect loss and duplicates.
- [ ] Save valid WAV.
- [ ] Stream live PDM audio.
- [ ] Compare duration and sample count.
- [ ] Run a basic range test.

**Exit criterion:** wireless microphone audio is saved correctly to WAV.

## Day 14 — Finalization

- [ ] Fix the highest-impact bugs.
- [ ] Run complete host tests.
- [ ] Run USB reliability test.
- [ ] Run BLE short reliability test.
- [ ] Save benchmark results.
- [ ] Update architecture and limitations.
- [ ] Record a demo.
- [ ] Tag the two-week release.
- [ ] Move ADPCM/live playback/C++ tasks to backlog.

## Backlog

### BLE improvements

- [ ] IMA ADPCM
- [ ] Live Python playback
- [ ] Jitter buffer
- [ ] Packet-loss concealment
- [ ] Bonding and encrypted controls
- [ ] Power measurements

### Host improvements

- [ ] C++ receiver
- [ ] Virtual microphone routing
- [ ] GUI
- [ ] Automated benchmark dashboard

### Hardware/product improvements

- [ ] USB receiver dongle
- [ ] Battery optimization
- [ ] Enclosure
- [ ] Acoustic port experiments
- [ ] Custom PCB exploration

## Risk register

| Risk                           | Impact                     | Mitigation                                          |
| ------------------------------ | -------------------------- | --------------------------------------------------- |
| Wrong SoftDevice memory layout | Firmware does not boot     | Inspect UF2 metadata and map file before BLE        |
| No RTT probe                   | Harder debugging           | USB CDC, counters, retained fault record            |
| USB descriptor errors          | Host rejects device        | Start from minimal topology and inspect descriptors |
| PDM/USB clock drift            | Long-run overrun/underrun  | Instrument queue depth before correction            |
| BLE throughput/backpressure    | Drops and latency          | Fragment by negotiated payload; oldest-drop policy  |
| Scope expansion                | Missed two-week goal       | Tag USB release before advanced BLE work            |
| Agent over-implementation      | Reduced learning           | Enforce `AGENTS.md` and protected TODOs             |
| Hardware recovery failure      | Board temporarily unusable | Preserve bootloader and known-good UF2              |
