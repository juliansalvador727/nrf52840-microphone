# Requirements

## 1. Purpose

Build a modular real-time microphone on the Adafruit Feather nRF52840 Sense.

The same PDM audio capture and processing pipeline must support two mutually exclusive transports:

1. USB Audio Class 1.0
2. Custom Bluetooth Low Energy audio

The wired USB implementation is the first production milestone. The BLE implementation follows without rewriting the microphone capture layer.

## 2. Functional requirements

### 2.1 Audio capture

- Capture from the onboard PDM microphone.
- Use the nRF52840 PDM peripheral and EasyDMA.
- Produce mono, signed 16-bit PCM.
- Nominal sample rate: 16,000 samples per second.
- Audio frame duration: 10 ms.
- Samples per frame: 160.
- Audio bytes per frame: 320.
- Capture must run continuously while the selected transport is streaming.

### 2.2 Frame storage

- Each frame owns its own sample array.
- Frames come from a fixed-size pool.
- Frames are passed between stages by pointer.
- No dynamic allocation is permitted in the audio path.
- Every pointer must have one unambiguous owner.
- Frame drops must be counted.
- Queue capacity must be fixed at compile time.

### 2.3 USB mode

- Cold power-up defaults to USB mode.
- Use USB Audio Class 1.0.
- Expose a microphone input only.
- Format: mono, signed 16-bit little-endian PCM at 16 kHz.
- Support host mute through a Feature Unit.
- Software volume control is not required initially.
- The device should require no custom driver or host application.
- The target is compatibility with computers and compatible USB-host phones.
- USB streaming and BLE audio streaming must not run simultaneously.

### 2.4 BLE mode

- Use a custom GATT service.
- Stream live PCM frames initially.
- Include sequence and sample-position information.
- Fragment based on the actual negotiated ATT payload.
- Provide control and status characteristics.
- The first host receiver is written in Python.
- The first receiver milestone saves a valid WAV file.
- Real-time playback and ADPCM are later milestones.

### 2.5 Mode switching

- One user-button press requests a transport change.
- A software reset performs the change cleanly.
- The selected mode survives only that software reset.
- A true power cycle returns to USB mode.
- USB and BLE stacks are not dynamically rebuilt in place.
- Mode switching must stop capture before resetting.
- A visible LED state should indicate the transition.

### 2.6 LEDs

NeoPixel state colors:

- Blue: looking for a host
- Green: connected and actively streaming
- Red: muted
- Yellow: connected but idle, stalled, or recovering
- Purple: switching modes
- Fast red flash: fatal audio or buffer error

A separate onboard LED may indicate audio level using a rate-limited RMS value.

Silence must not be treated as a transport error.

### 2.7 Diagnostics

At minimum, collect:

- PDM frames completed
- PDM overruns
- Frames processed
- Frames dropped
- Maximum ready-queue depth
- USB packets sent
- USB underruns
- USB sample corrections
- BLE notifications sent
- BLE backpressure events
- BLE frames dropped
- Clipped samples
- Mode switches
- Unexpected errors

Debug logging must be deferred out of interrupt context.

### 2.8 Build and flash

- Build system: CMake.
- Preferred generator: Ninja.
- Firmware language: C11.
- Flash format: UF2.
- Flash through the installed Adafruit UF2 bootloader.
- Do not require SWD/J-Link for normal development.
- Build must produce an ELF and map file for inspection.

## 3. Performance requirements

### 3.1 Real-time behavior

- The audio path must not block.
- Frame processing must complete before the next frame deadline.
- No unbounded queues.
- No silent data loss.
- USB mode should complete long-duration tests with zero PDM overruns and zero USB underruns.
- BLE mode should expose backpressure and packet-loss statistics.

### 3.2 Latency

Initial targets:

- USB device-to-host path: below 30 ms where measurable
- Full application recording path: below 100 ms
- BLE-to-receiver path: below 100 ms initially
- Mute response: below 20 ms after the action is recognized

These are engineering targets, not guaranteed hardware specifications.

### 3.3 Audio quality

Initial target measurements:

- DC offset
- RMS noise floor
- Peak level
- Clipping count
- Frequency response from approximately 100 Hz to 7 kHz
- Sample-count continuity
- Comparison against at least two existing microphones

## 4. Non-functional requirements

- Core pure-C modules must compile and run on a host machine.
- Architecture and packet formats must be documented.
- Errors must be observable through counters, retained fault data, logs, or LED state.
- No code should depend on undocumented pin mappings.
- Vendored nRF5 SDK files must remain unmodified.
- The repository should be understandable by another firmware developer.

## 5. Two-week scope

Included:

- Toolchain and UF2 workflow
- PDM capture
- Frame pool and pointer queues
- WAV validation
- USB Audio Class microphone
- Basic diagnostics
- USB/BLE mode switching
- Basic BLE PCM streaming
- Python BLE-to-WAV receiver
- Host-side unit tests
- Initial benchmarks

Excluded:

- Bluetooth Classic HFP/A2DP
- Native LE Audio
- Noise cancellation
- Echo cancellation
- Automatic gain control
- Production-grade adaptive resampling
- C++ receiver
- Custom virtual audio driver
- USB receiver dongle
- Custom PCB
- Battery optimization
- Enclosure design

## 6. Acceptance criteria

The two-week release is accepted when:

- [ ] It builds reproducibly from documented commands.
- [ ] It flashes through UF2.
- [ ] It captures continuous 16 kHz audio.
- [ ] Exported PCM produces an intelligible WAV file.
- [ ] It enumerates as a USB microphone.
- [ ] USB audio works in at least one real application.
- [ ] Core counters remain visible and meaningful.
- [ ] The same capture API can feed BLE without modification.
- [ ] BLE audio can be received and saved to WAV.
- [ ] Core pure-C behavior is covered by host tests.
- [ ] Known limitations are documented honestly.

## TODO(USER): Prioritize the requirements

For each category below, assign **Must**, **Should**, or **Could**, and explain one trade-off:

- Cross-platform USB compatibility
- USB latency
- BLE latency
- Audio quality
- Long-duration reliability
- Code modularity
- Number of tests
- LED polish
- BLE compression
- Documentation depth

This prioritization should be written by the human before major implementation begins.
