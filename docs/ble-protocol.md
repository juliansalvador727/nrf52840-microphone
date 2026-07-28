# BLE Audio Protocol

## 1. Status

**Provisional until the USB milestone is stable.**

The audio frame contract is intended to remain compatible with this protocol so the BLE transport can be added without rewriting capture.

## 2. Goal

Stream live voice audio from the nRF52840 to a host receiver over a custom BLE GATT service.

The BLE device will not appear as a native operating-system microphone. A Python receiver is required.

## 3. Initial audio format

Release A:

```text
Codec: PCM16
Sample rate: 16 kHz
Channels: mono
Frame duration: 10 ms
Samples per frame: 160
Payload per full frame: 320 bytes
```

Release B:

```text
Codec: IMA ADPCM
Approximate payload: 80 bytes per 10 ms frame, plus codec state/header
```

## 4. GATT service

Use one custom 128-bit service.

Characteristics:

### Audio Data

- Notify
- Carries frame fragments
- Highest-throughput path
- No application-level acknowledgment initially

### Control

- Write
- Read if useful
- Starts/stops streaming
- Selects codec
- Requests or resets statistics

### Status

- Read
- Notify
- Reports connection, stream, codec, drops, and error status

UUID values should be generated once and stored here and in code.

```text
Service UUID: TODO
Audio UUID: TODO
Control UUID: TODO
Status UUID: TODO
```

## 5. Packet header

Proposed header:

```c
typedef enum
{
    BLE_AUDIO_CODEC_PCM16 = 0,
    BLE_AUDIO_CODEC_IMA_ADPCM = 1
} ble_audio_codec_t;

typedef enum
{
    BLE_AUDIO_FLAG_NONE          = 0U,
    BLE_AUDIO_FLAG_MUTED         = 1U << 0,
    BLE_AUDIO_FLAG_DISCONTINUITY = 1U << 1,
    BLE_AUDIO_FLAG_STREAM_START  = 1U << 2,
    BLE_AUDIO_FLAG_STREAM_END    = 1U << 3
} ble_audio_flags_t;

typedef struct __attribute__((packed))
{
    uint32_t sequence;
    uint32_t first_sample_index;

    uint16_t payload_length;

    uint8_t fragment_index;
    uint8_t fragment_count;
    uint8_t codec;
    uint8_t flags;
} ble_audio_header_t;
```

All multi-byte fields must have an explicitly documented byte order. Initial recommendation: little-endian, matching the target MCU and PCM encoding.

## 6. Fragmentation

A PCM frame is 320 bytes and may not fit in one notification.

The packetizer must calculate:

```text
notification payload capacity =
negotiated ATT payload - protocol header
```

It must not assume an MTU of 247.

Rules:

- Every fragment repeats the frame sequence and sample index.
- `fragment_index` begins at zero.
- `fragment_count` is identical for every fragment of a frame.
- Fragments contain consecutive audio bytes.
- The receiver may discard an incomplete frame after a timeout.
- A later frame must not wait indefinitely for an incomplete earlier frame.
- Incomplete-frame loss must be counted.

## 7. Sequence handling

The device increments `sequence` once per 10 ms source frame.

The receiver uses unsigned modular arithmetic.

Definitions:

- Expected sequence: next complete frame desired
- Ahead sequence: one or more frames were missed
- Behind sequence: duplicate, delayed, or reordered frame
- Rollover: naturally handled by modulo-2^32 subtraction

The receiver must report:

- Complete frames
- Missing frames
- Duplicate frames
- Old/reordered frames
- Incomplete frames
- Duplicate fragments

## 8. Sample timeline

`first_sample_index` identifies the first sample of the frame on the device timeline.

Expected increment:

```text
160 samples per frame
```

If the increment is different:

- A discontinuity occurred
- A frame was dropped before transmission
- The receiver missed a frame
- A future sample-rate correction changed the count

The receiver should preserve both:

- Device sample index
- Host monotonic receive timestamp

## 9. Control messages

Initial commands:

```text
START_STREAM
STOP_STREAM
SET_CODEC
GET_STATISTICS
RESET_STATISTICS
```

Possible binary command structure:

```c
typedef struct __attribute__((packed))
{
    uint8_t opcode;
    uint8_t version;
    uint16_t payload_length;
    uint8_t payload[];
} ble_control_message_t;
```

The exact opcode values and responses must be frozen before implementation.

## 10. Status data

Suggested status fields:

- Protocol version
- Firmware version
- Current codec
- Streaming state
- Muted state
- Frames produced
- Frames dropped
- BLE notifications sent
- BLE backpressure events
- Last error
- Current negotiated ATT MTU
- Current PHY
- Current data length
- Maximum queue depth

## 11. Backpressure

When notification submission is temporarily unavailable:

- Do not block the audio pipeline.
- Preserve the newest audio.
- Drop the oldest queued transport frame if necessary.
- Mark the next transmitted frame as discontinuous.
- Increment `ble_tx_backpressure` and `ble_frames_dropped`.
- Resume when the stack reports available TX capacity.

## 12. Security and pairing

Initial development may use an unsecured custom service for simplicity.

Before calling the project polished, decide:

- Whether bonding is required
- Whether only one saved host is allowed
- Whether control writes require encryption
- How pairing information is cleared

This is not required for the two-week release.

## 13. Python receiver versions

### Version 1 — BLE to WAV

Responsibilities:

- Discover the device
- Connect
- Subscribe to Audio Data
- Start streaming
- Reassemble fragments
- Validate sequence numbers
- Write PCM to WAV
- Report losses and timing

### Version 2 — Live playback

Add:

- Bounded jitter buffer
- Real-time audio output
- Gap concealment with silence initially
- Latency and underrun statistics

### Version 3 — ADPCM

Add:

- Codec negotiation
- IMA ADPCM decode
- Codec-state validation
- PCM comparison tests

## 14. Protocol tests

Host-side tests should cover:

- Header serialization
- Endianness
- Maximum payload calculation
- Single-fragment frame
- Multi-fragment frame
- Missing first/middle/final fragment
- Duplicate fragment
- Out-of-order fragment
- Sequence rollover
- Truncated packet
- Invalid fragment count
- Unknown codec
- Discontinuity flag propagation
- WAV sample-count correctness

## TODO(USER): Freeze the protocol

Before BLE implementation, complete:

1. Generate the four UUIDs.
2. Choose exact opcode values.
3. Choose byte order.
4. Define incomplete-frame timeout.
5. Define whether fragments may arrive out of order.
6. Define the maximum number of partially assembled frames.
7. Explain why the protocol repeats sequence metadata in every fragment.

The agent may review the result but must not silently choose these values.
