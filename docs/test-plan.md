# Test Plan

## 1. Testing philosophy

Tests must establish contracts, not merely reproduce the implementation.

The human developer writes the first essential tests for important modules. The agent expands coverage with edge cases, parameterized cases, mocks, and regressions.

Tests are divided into:

1. Host-side unit tests
2. Firmware hardware tests
3. System and audio benchmarks

## 2. Host test environment

Platform-independent C modules should compile with the host compiler.

Selected test setup:

```text
Unity test framework driven through CTest
```

Unity is a host-test dependency only. Production firmware modules must not
include Unity headers or depend on Unity behavior. The Unity revision must be
pinned when the dependency is added to the build.

Potential structure:

```text
tests/
├── CMakeLists.txt
├── test_audio_pool.c
├── test_audio_queue.c
├── test_audio_process.c
├── test_sequence.c
├── test_ble_packetizer.c
├── test_ble_reassembly.py
└── fakes/
    ├── fake_transport.c
    └── fake_platform.c
```

A lightweight C test framework may be used, or a minimal local assertion harness may be written.

The test framework must not force production code to depend on host-only features.

## 3. Human-written contract tests

The human should write the first tests for:

- Frame pool acquisition and release
- Double release detection
- Queue ownership transfer
- Queue-full oldest-drop policy
- Sequence rollover
- Saturating gain
- Mute ramp endpoints
- Transport submit ownership behavior

These tests define architectural behavior and must not be outsourced initially.

## 4. Agent-expanded tests

The agent may add:

- Boundary values
- Parameterized queue capacities
- Random operation sequences
- Failure injection
- Invalid packet tests
- Fragment permutations
- Long sequence rollover simulations
- Regression tests after understood bugs
- Python WAV and receiver tests
- Coverage reports

Every generated test must be reviewed by the human.

## 5. Unit-test matrix

### Audio frame pool

- Acquire until empty
- Release and reacquire
- Detect duplicate release
- Detect foreign pointer
- Reset behavior
- Pool size of one
- Pool exhaustion counter
- No frame returned twice

### Pointer queue

- Empty pop
- Single push/pop
- FIFO order
- Full queue
- Oldest-drop helper
- Wraparound of head and tail
- Capacity one
- Repeated fill/drain
- Pointer identity preservation

### Audio processing

- Unity gain
- Positive gain
- Negative and positive saturation
- Zero input
- DC blocker stability
- Mute ramp start/end
- RMS calculation
- Peak calculation
- Clipping counter
- Bypass path bit-exactness

### Sequence handling

- Expected frame
- One missing frame
- Multiple missing frames
- Duplicate frame
- Old frame
- `UINT32_MAX` rollover
- Large ambiguous delta

### BLE packetizer/reassembly

- See `ble-protocol.md`
- Verify serialized length
- Verify byte order
- Verify MTU-dependent fragment size
- Verify complete PCM reconstruction
- Verify incomplete frame timeout

## 6. Firmware bring-up tests

### Build and flash

- Clean configure
- Clean build
- UF2 produced
- Map file produced
- UF2 accepted by bootloader
- Known LED pattern runs
- Software reset works
- Power cycle returns to USB default

### PDM

- Clock starts
- First buffer completes
- Continuous buffers complete
- Correct sample count
- No overruns during one minute
- No overruns during one hour
- Captured PCM is not constant
- Spoken audio is intelligible
- Silence level is plausible
- Sample timeline increments correctly

### Button and LEDs

- Debounce
- Exactly one event per normal press
- Mode switch requests reset
- LED states match transport lifecycle
- Level indicator update is rate-limited
- Button press does not interrupt audio unexpectedly

## 7. USB system tests

### Enumeration

- Device descriptor accepted
- AudioControl interface visible
- AudioStreaming interface visible
- Correct alternate settings
- Correct sample format
- Host lists one mono microphone
- Host mute request is received

### Streaming

- Correct WAV duration
- Correct sample rate
- Correct channel count
- No repeated stale frame
- No audible periodic gap
- No clipped normal speech
- Unplug/replug recovery
- Suspend/resume recovery
- Host starts/stops stream repeatedly

### Long run

Target:

```text
8 hours on the primary host
```

Required counters:

```text
pdm_overruns == 0
usb_underruns == 0
frames_dropped == 0
unexpected_errors == 0
```

Also record:

- Maximum queue depth
- Sample corrections
- USB resets
- Stream starts/stops
- Final sample count

## 8. BLE system tests

### Connection

- Advertising visible
- Connect/disconnect
- Service and characteristics discovered
- Notifications enabled
- Stream start/stop command
- Reconnection

### Audio

- Generated counter pattern first
- Generated sine wave second
- Live microphone third
- PCM frame reconstruction
- Correct WAV duration
- Sequence gaps counted
- Fragment loss counted
- Backpressure recovers
- Queue depth remains bounded

### Range scenarios

- 1 m line of sight
- 5 m line of sight
- Human body between board and host
- Board in a pocket
- Busy Wi-Fi/Bluetooth environment

## 9. Audio benchmarks

### Reference devices

Compare with at least two:

- Laptop microphone
- Phone microphone
- Existing headset or USB microphone

### Test conditions

- Quiet room, 30 cm
- Quiet room, 1 m
- Typing nearby
- Laptop fan running
- Loud speech at 10 cm
- Normal speech while turning head

### Measurements

- RMS level
- Peak level
- DC offset
- Clipping percentage
- Noise floor
- Spectrum
- Frequency response
- Sample-rate accuracy
- End-to-end latency
- Listening comparison

Keep geometry and gain fixed for comparative tests.

## 10. Latency tests

Possible methods:

- Record a physical click simultaneously through a reference path and the device.
- Use an LED or electrical trigger correlated with a known audio event.
- Compare device sample index with host receive time for BLE transport analysis.

Report separately:

- Firmware buffering
- USB/BLE transport
- Host audio stack
- Application buffering

Do not present one total number as entirely caused by firmware.

## 11. Test artifacts

Store:

```text
benchmarks/
├── raw/
│   ├── usb/
│   ├── ble/
│   └── reference/
├── results/
│   ├── csv/
│   └── plots/
└── scripts/
```

Do not commit very large recordings without a clear reason. Small representative fixtures are sufficient for automated tests.

## 12. Definition of a passing release

### USB release

- [ ] All host tests pass.
- [ ] USB enumerates on the primary computer.
- [ ] Audio works in one call or recording application.
- [ ] One-hour test has zero capture/USB failures.
- [ ] Eight-hour test is scheduled or completed before flagship claims.
- [ ] WAV and audio metrics are recorded.

### BLE-to-WAV release

- [ ] All packet tests pass.
- [ ] Receiver reconstructs a valid WAV.
- [ ] Missing frames are detected.
- [ ] Backpressure does not deadlock.
- [ ] Queue depth remains bounded.
- [ ] Basic range test is documented.

## TODO(USER): Write the first contract tests

Before asking the agent to expand the suite, write pseudocode or real tests for:

1. Frame pool acquisition and release
2. Queue-full oldest-drop behavior
3. Sequence rollover
4. Saturating gain
5. Successful versus failed `transport_submit()` ownership

For each test, state the invariant it protects.
