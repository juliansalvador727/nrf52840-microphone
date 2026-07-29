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

sequenceDiagram
autonumber

      participant Host
      participant App as Main Loop / App
      participant State as Audio Pipeline State
      participant Pool as Free Frame Pool
      participant Capture as PDM Capture Callback
      participant PDM as PDM Driver + EasyDMA
      participant Ready as READY Queue
      participant Process as Processing
      participant USB as USB Transport
      participant Stats as Diagnostics

      Note over Pool: Owns all 8 frames in FREE state

      %% Start USB streaming
      Host->>USB: Select active streaming alternate setting
      USB->>App: Streaming active
      App->>PDM: Start capture

      %% Prime current and next DMA buffers
      loop Prime current and next DMA buffers
          PDM-->>Capture: buffer_requested
          Capture->>Pool: Acquire FREE frame
          Pool-->>Capture: Frame, FREE to CAPTURE
          Note over Capture: Capture owns frame
          Capture->>PDM: Supply frame sample storage
          Note over Capture,PDM: Capture stage owns frame<br/>EasyDMA is the only writer
      end

      %% Continuous PDM callback
      PDM-->>Capture: buffer_requested and buffer_released

      Note over Capture: Handle buffer request first

      Capture->>Pool: Acquire next FREE frame

      alt FREE frame available
          Pool-->>Capture: Next frame, FREE to CAPTURE
          Capture->>PDM: Supply next frame sample storage

          alt PDM accepts buffer
              Note over Capture,PDM: Next frame remains in CAPTURE
          else PDM rejects buffer
              Note over Capture: Capture still owns unused frame
              Capture->>Pool: Release unused frame
              Capture->>Stats: Increment unexpected_errors
              Capture->>State: Set discontinuity_pending
              Capture->>App: Request RECOVERING
          end

      else Free pool exhausted
          Capture->>Stats: Increment pool_exhaustions

          Note over Capture,Ready: Enter short critical section
          Capture->>Ready: Pop oldest READY frame

          alt Oldest READY frame available
              Ready-->>Capture: Oldest frame, READY to CAPTURE
              Capture->>Stats: Increment frames_dropped
              Capture->>State: Set discontinuity_pending
              Capture->>PDM: Reuse frame as next DMA buffer

              alt PDM accepts reclaimed buffer
                  Note over Capture,PDM: Capture continues without stopping
              else PDM rejects reclaimed buffer
                  Note over Capture: Capture still owns reclaimed frame
                  Capture->>Pool: Release reclaimed frame
                  Capture->>Stats: Increment unexpected_errors
                  Capture->>App: Request RECOVERING
              end

          else No READY frame can be reclaimed
              Capture->>State: Set discontinuity_pending
              Capture->>App: Request RECOVERING
          end

          Note over Capture,Ready: Exit short critical section
      end

      %% Handle completed DMA frame
      Note over Capture: Handle released buffer second
      Note over Capture,PDM: EasyDMA no longer accesses released frame

      Capture->>Stats: Increment pdm_frames_completed
      Capture->>Capture: Set valid_samples
      Capture->>Capture: Set sequence
      Capture->>Capture: Set first_sample_index

      Capture->>State: Atomically consume discontinuity_pending

      alt Discontinuity was pending
          State-->>Capture: Pending
          Capture->>Capture: Set DISCONTINUITY flag
      else No discontinuity
          State-->>Capture: Clear
      end

      %% Transfer completed frame to READY
      Note over Capture,Ready: Enter short critical section
      Capture->>Ready: Try push completed frame

      alt READY push succeeds
          Note over Ready: READY queue owns frame
          Capture->>Stats: Update maximum READY depth

      else READY queue is full
          Note over Capture: Capture still owns incoming frame
          Capture->>Ready: Pop oldest frame
          Ready-->>Capture: Oldest frame, READY to CAPTURE
          Capture->>Stats: Increment frames_dropped
          Capture->>Capture: Set incoming DISCONTINUITY flag
          Capture->>Pool: Release oldest frame
          Note over Pool: Pool owns released oldest frame
          Capture->>Ready: Push incoming frame
          Note over Ready: READY queue owns incoming frame
          Capture->>Stats: Update maximum READY depth
      end

      Note over Capture,Ready: Exit short critical section

      %% PDM hardware error
      opt PDM driver reports underflow
          PDM-->>Capture: Error event
          Capture->>Stats: Increment pdm_overruns
          Capture->>State: Set discontinuity_pending
          Capture->>App: Request RECOVERING
      end

      %% Deferred processing
      App->>Process: Process one READY frame
      Note over Process,Ready: Enter short critical section
      Process->>Ready: Pop frame
      Note over Process,Ready: Exit short critical section

      alt READY frame available
          Ready-->>Process: Frame, READY to PROCESSING
          Note over Process: Processing owns frame

          Process->>Process: Validate frame metadata

          alt Metadata is valid
              Process->>Process: Gain, saturation, RMS, peak, mute ramp

              opt Samples clipped
                  Process->>Process: Saturate clipped samples
                  Process->>Process: Set CLIPPED flag
                  Process->>Stats: Increment clipped_samples
              end

              Process->>Stats: Increment frames_processed
              Note over Process: Processing still owns frame

              %% USB submission
              Process->>USB: Submit frame

              alt USB accepts frame
                  Note over USB: Ownership transfers to USB
                  Note over USB: Frame state is TRANSPORT

                  loop Ten 1 ms USB packets
                      Host->>USB: Request isochronous IN packet
                      USB-->>Host: 32 bytes PCM
                      USB->>Stats: Increment usb_packets_sent
                  end

                  USB->>USB: Wait for final SDK completion
                  Note over USB: SDK no longer accesses frame
                  USB->>Pool: Release frame
                  Note over Pool: Frame returns to FREE

              else USB rejects frame
                  Note over Process: Processing still owns frame
                  Process->>Stats: Increment frames_dropped
                  Process->>State: Set discontinuity_pending
                  Process->>Pool: Release frame
                  Note over Pool: Frame returns to FREE
              end

          else Impossible metadata
              Process->>Stats: Increment unexpected_errors
              Process->>App: Enter FATAL_ERROR
              Note over Process: Do not process invalid memory range
          end

      else READY queue empty
          Ready-->>Process: No frame
          Process-->>App: Continue main loop or low-power wait
      end

      %% USB underrun
      opt Host requests audio but USB has no samples
          Host->>USB: Request isochronous IN packet
          USB-->>Host: 32 bytes zero-valued PCM
          USB->>Stats: Increment usb_underruns
          USB->>State: Set discontinuity_pending
      end

      %% Host mute
      opt Host changes mute control
          Host->>USB: Set mute
          USB->>App: Update mute state
          Note over App,Process: Processing ramps samples toward zero<br/>USB streaming continues
      end

      %% Clean stop
      Host->>USB: Select alternate setting 0 or disconnect
      USB->>App: Streaming inactive
      App->>PDM: Stop capture

      PDM-->>Capture: Release all DMA buffers
      Note over Capture,PDM: SDK no longer accesses returned buffers

      loop Every returned CAPTURE frame
          Capture->>Pool: Release frame
      end

      loop Until READY queue is empty
          App->>Ready: Pop frame
          Ready-->>App: READY frame
          App->>Pool: Release frame
      end

      opt Processing owns a frame
          Process->>Pool: Release frame
      end

      App->>USB: Cancel pending USB transfer
      USB-->>App: Transfer stopped and memory released

      opt USB owns a frame
          USB->>Pool: Release frame
      end

      App->>Pool: Verify 8 unique FREE frames

      alt All ownership invariants pass
          Note over App,Pool: Clean stop complete
      else Missing or duplicate ownership
          App->>Stats: Increment unexpected_errors
          App->>App: Enter FATAL_ERROR
      end

      %% Deferred recovery
      opt recovery_requested
          Note over App: Run the same clean-stop sequence
          App->>Pool: Verify all 8 frames are uniquely owned

          alt Ownership is consistent and host is streaming
              App->>PDM: Restart capture
          else Ownership remains inconsistent
              App->>App: Enter FATAL_ERROR
          end
      end
