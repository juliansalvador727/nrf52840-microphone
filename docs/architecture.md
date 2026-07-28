# Architecture

## 1. System overview

```text
Onboard PDM microphone
          |
          v
PDM peripheral + EasyDMA
          |
          v
Fixed audio-frame pool
          |
          v
Ready pointer queue
          |
          v
Audio processing
          |
          v
Transport interface
       /       \
      v         v
USB UAC       BLE GATT
                |
                v
         Python receiver
```

USB and BLE audio transports are mutually exclusive.

The capture and processing layers must not know which transport is selected.

## 2. Module boundaries

Proposed firmware layout:

```text
firmware/
├── app/
│   ├── app.c
│   ├── app.h
│   ├── app_state.c
│   └── app_state.h
├── audio/
│   ├── audio_capture.c
│   ├── audio_capture.h
│   ├── audio_frame.h
│   ├── audio_pool.c
│   ├── audio_pool.h
│   ├── audio_queue.c
│   ├── audio_queue.h
│   ├── audio_process.c
│   ├── audio_process.h
│   └── audio_stats.c
├── diagnostics/
│   ├── diagnostics.c
│   ├── diagnostics.h
│   ├── event_log.c
│   ├── event_log.h
│   ├── fault_record.c
│   └── fault_record.h
├── drivers/
│   ├── board_button.c
│   ├── board_button.h
│   ├── board_led.c
│   └── board_led.h
├── platform/
│   ├── board.h
│   ├── clock.c
│   ├── clock.h
│   ├── reset_mode.c
│   └── reset_mode.h
└── transports/
    ├── audio_transport.h
    ├── usb/
    │   ├── usb_audio.c
    │   ├── usb_audio.h
    │   └── usb_descriptors.c
    └── ble/
        ├── ble_audio.c
        ├── ble_audio.h
        ├── ble_audio_service.c
        ├── ble_audio_service.h
        ├── ble_packetizer.c
        └── ble_packetizer.h
```

## 3. Audio frame contract

```c
#define AUDIO_SAMPLE_RATE_HZ       16000U
#define AUDIO_FRAME_DURATION_MS    10U
#define AUDIO_SAMPLES_PER_FRAME    160U
#define AUDIO_BYTES_PER_FRAME      320U

typedef enum
{
    AUDIO_FRAME_FLAG_NONE          = 0U,
    AUDIO_FRAME_FLAG_MUTED         = 1U << 0,
    AUDIO_FRAME_FLAG_DISCONTINUITY = 1U << 1,
    AUDIO_FRAME_FLAG_CLIPPED       = 1U << 2
} audio_frame_flags_t;

typedef struct
{
    int16_t samples[AUDIO_SAMPLES_PER_FRAME];

    uint64_t first_sample_index;
    uint32_t sequence;

    uint16_t valid_samples;
    uint8_t flags;
    uint8_t reserved;
} audio_frame_t;
```

`first_sample_index` is the authoritative position of the first sample in the device audio timeline.

The PC may record its own receive timestamp, but host arrival time is not the source of truth for audio ordering.

## 4. Fixed frame pool

Initial pool size:

```text
8 frames
```

The exact size may be tuned after measuring queue occupancy.

A frame is never allocated dynamically. Frames move through a fixed lifecycle:

```text
FREE -> CAPTURE -> READY -> PROCESSING -> TRANSPORT -> FREE
```

Only the current owner may modify a frame.

### Ownership rules

- `audio_pool_acquire()` transfers a frame from the pool to the caller.
- A successful queue push transfers ownership to the queue's consumer.
- A failed queue push leaves ownership with the caller unless documented otherwise.
- A transport returns the frame only after it no longer needs the sample data.
- Every error path must either transfer or release ownership.
- A frame must never appear in two queues simultaneously.
- A frame must never be returned twice.

## 5. Queues

Queues contain `audio_frame_t *`.

Required queues:

- Free-frame queue
- Ready-frame queue
- Optional transport queue if the transport cannot consume immediately

Queue operations used by interrupt context must be bounded and nonblocking.

### Queue-full behavior

For live audio, low latency is more important than preserving stale frames.

When the ready queue is full:

1. Pop the oldest ready frame.
2. Increment `frames_dropped`.
3. Return the old frame to the free pool.
4. Mark the incoming or next transmitted frame with `DISCONTINUITY`.
5. Push the newly completed frame.

A drop in USB mode is considered a reliability failure even though the firmware recovers.

## 6. PDM capture

PDM capture uses EasyDMA and continuous buffer replacement.

The callback or interrupt must do only bounded work:

- Handle the completed buffer
- Provide the next buffer
- Update counters
- Transfer frame ownership
- Signal deferred processing

It must not:

- Perform DSP loops
- Serialize BLE packets
- Submit complex USB descriptors
- Format strings
- Write logs
- Sleep or block

The design may use frame-owned sample storage directly as DMA buffers if alignment and SDK behavior permit. Otherwise, it may use dedicated DMA ping-pong buffers and copy into frames. This decision must be measured rather than assumed.

## 7. Processing stage

Initial processing sequence:

```text
Raw PCM
  -> optional DC-blocking high-pass filter
  -> fixed gain
  -> saturating conversion
  -> RMS / peak / clipping metrics
  -> mute ramp
  -> transport
```

Version 0 should support bypassing all processing for raw-path validation.

Excluded initially:

- Automatic gain control
- Noise suppression
- Echo cancellation
- Beamforming

## 8. Transport interface

Proposed interface:

```c
typedef enum
{
    AUDIO_TRANSPORT_USB = 0,
    AUDIO_TRANSPORT_BLE = 1
} audio_transport_kind_t;

typedef struct
{
    bool (*init)(void);
    bool (*start)(void);
    bool (*submit)(audio_frame_t *frame);
    void (*stop)(void);
    void (*process)(void);
    bool (*is_connected)(void);
    bool (*is_streaming)(void);
} audio_transport_ops_t;
```

Important contract:

- `submit(frame)` must document whether ownership transfers on success.
- On failure, the caller retains ownership unless explicitly stated.
- `process()` performs deferred, non-interrupt work.
- The transport must eventually return every accepted frame to the pool.
- USB and BLE implementations must conform to the same ownership behavior.

## 9. Device state machine

Proposed top-level states:

```text
BOOT
INITIALIZING
WAITING_FOR_HOST
STREAM_IDLE
STREAM_ACTIVE
SWITCHING_MODE
RECOVERING
FATAL_ERROR
```

Transport is a separate property:

```text
USB
BLE
```

Mute is an orthogonal property:

```text
UNMUTED
MUTED
```

Do not create a separate top-level state for every combination.

## 10. Mode switching

Cold power-on:

```text
USB
```

Button press:

1. Debounce and create a mode-switch event.
2. Set LED to purple.
3. Stop capture.
4. Stop the active transport.
5. Write a retained next-mode marker.
6. Trigger a software reset.
7. Read and clear the marker at startup.
8. Initialize only the selected transport.

The retained marker must not persist through a true power loss.

The exact retention mechanism must be checked against the Adafruit bootloader before implementation.

## 11. Sequence numbers

Use a `uint32_t` sequence number incremented once per 10 ms frame.

Do not special-case rollover with a guessed numerical threshold. Use unsigned modular arithmetic.

Example host-side interpretation:

```c
uint32_t delta = received - expected;

if (delta == 0U) {
    // Expected frame.
} else if (delta < 0x80000000U) {
    // Received frame is ahead; delta frames are missing.
} else {
    // Received frame is old, duplicate, or reordered.
}
```

At 100 frames per second, rollover occurs after approximately 497 days.

## 12. Concurrency model

Initial firmware should use the nRF5 SDK event-driven model without adding an RTOS.

Contexts:

- Hardware interrupt/callback context
- SoftDevice/USB event callback context
- Main-loop deferred processing

Shared data must be:

- Single-owner where possible
- Protected by critical sections where necessary
- Accessed atomically for simple counters where appropriate

Avoid long critical sections because they can disrupt radio and USB timing.

## 13. Error handling

Recoverable examples:

- BLE notification temporarily unavailable
- Host not connected
- Ready queue full
- USB stream idle
- Mode switch requested

Fatal or near-fatal examples:

- Frame ownership corruption
- Pool double-free
- Impossible queue state
- Repeated PDM initialization failure
- Linker or SoftDevice RAM mismatch
- Unrecoverable USB initialization failure

Fatal errors should:

1. Save a compact retained fault record.
2. Set a visible LED code.
3. Reset or halt according to build configuration.
4. Never pretend streaming is still valid.

## TODO(USER): Draw the ownership sequence

Create your own sequence diagram for one frame moving from PDM capture to USB and back to the free pool.

Include:

- Which context performs each action
- Exactly when ownership changes
- What happens if a queue operation fails
- What happens if USB is not ready
- Which counters change

The agent may proofread the diagram but should not create the first version.
