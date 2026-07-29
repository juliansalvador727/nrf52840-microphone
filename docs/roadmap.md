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
- [ ] Complete the `TODO(USER)` requirements prioritization.
- [ ] Complete the architecture ownership diagram.
- [ ] Inspect `INFO_UF2.TXT`.
- [ ] Record installed `arm-none-eabi-gcc` version.
- [ ] Verify the Feather pin map.
- [ ] Verify LFCLK hardware.
- [ ] Decide the initial C test framework.
- [ ] Create the repository and initial commit.

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
