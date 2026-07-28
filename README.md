# Modular USB/BLE Microphone — nRF52840

A real-time embedded audio project built in C for the Adafruit Feather nRF52840 Sense.

The device captures mono audio from the onboard PDM microphone and sends the same transport-independent audio frames through one of two mutually exclusive transports:

1. **USB Audio Class 1.0** — the first release, exposing a wired, driverless microphone to computers and compatible phones.
2. **Custom BLE audio** — a later release, sending live PCM or compressed voice frames to a Python receiver.

The project is intended to demonstrate firmware architecture, DMA-backed audio capture, fixed-memory ownership, USB device development, BLE protocol design, testing, and measurable real-time behavior.

## Current target

The two-week target is complete when:

- [ ] The application builds with CMake.
- [ ] The application is converted to UF2 and flashed through the Adafruit bootloader.
- [ ] The onboard PDM microphone captures continuous 16 kHz, mono, signed 16-bit PCM.
- [ ] Audio is organized into transport-independent 10 ms frames.
- [ ] The device enumerates as a USB Audio Class 1.0 microphone.
- [ ] The microphone works in at least one real calling or recording application.
- [ ] A button-triggered software reset switches between USB and BLE modes.
- [ ] BLE mode streams PCM to a Python receiver.
- [ ] The Python receiver reconstructs and saves a valid WAV file.
- [ ] Core platform-independent modules have host-side tests.
- [ ] Runtime counters expose dropped frames, overruns, underruns, and queue depth.

## Planned releases

### Release 0 — Platform bring-up

- CMake and Arm GCC toolchain
- UF2 generation
- Button and LED validation
- Bootloader and SoftDevice memory verification

### Release 1 — Audio capture

- PDM and EasyDMA
- Fixed frame pool
- Pointer queues and ownership transfer
- WAV export for validation
- Basic audio statistics

### Release 2 — Wired USB microphone

- USB Audio Class 1.0
- Mono, 16 kHz, 16-bit PCM
- Host mute handling
- Continuous streaming
- Compatibility and long-run tests

### Release 3 — BLE-to-WAV

- Custom GATT service
- Sequence numbers and sample indices
- MTU-independent fragmentation
- Python receiver
- WAV reconstruction and packet-loss statistics

### Later releases

- IMA ADPCM
- Real-time Python playback
- C++ receiver
- Virtual microphone routing
- Optional nRF52840 USB receiver dongle
- Advanced audio processing

## Repository layout

```text
.
├── README.md
├── AGENTS.md
├── CMakeLists.txt
├── cmake/
├── firmware/
│   ├── app/
│   ├── audio/
│   ├── diagnostics/
│   ├── drivers/
│   ├── platform/
│   └── transports/
│       ├── usb/
│       └── ble/
├── host/
│   ├── receiver/
│   └── analysis/
├── tests/
├── tools/
└── docs/
    ├── requirements.md
    ├── architecture.md
    ├── platform.md
    ├── usb-audio.md
    ├── ble-protocol.md
    ├── test-plan.md
    └── roadmap.md
```

## Build and flash

The final commands will be documented after the toolchain is validated.

```bash
cmake -S . -B build -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake \
  -DCMAKE_BUILD_TYPE=Debug

cmake --build build
```

The build must generate:

```text
microphone.elf
microphone.hex
microphone.bin
microphone.uf2
microphone.map
```

Flash by double-resetting the Feather and copying `microphone.uf2` to the mounted UF2 drive.

## Important constraints

- No dynamic allocation in the real-time audio path.
- No blocking operations in PDM, USB, BLE, timer, or GPIO callbacks.
- No formatted logging inside interrupt context.
- USB audio and BLE audio are never active simultaneously.
- Cold power-up defaults to USB mode.
- Audio capture code must not depend on the selected transport.
- Frames are allocated from a fixed pool and transferred by pointer.
- Queue-full behavior must be explicit and measurable.
- Hardware pin mappings must come from the board schematic or datasheet.
- The existing Adafruit bootloader and SoftDevice must not be replaced without SWD recovery hardware.

## Personal learning checkpoint

Complete this before asking an agent to implement the core pipeline.

### TODO(USER): Explain the project in your own words

Write 5–10 sentences answering:

- What problem does the project solve?
- Why are USB and BLE separate transports?
- Why can the nRF52840 not become a normal Bluetooth Classic headset?
- What are the three hardest firmware problems you expect?
- What would make the project flagship-level rather than a simple SDK integration?

Do not let an agent write this section for you.

## Status

**Design phase.** See [docs/roadmap.md](docs/roadmap.md) for the current task list.
