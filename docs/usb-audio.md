# USB Audio Design

## 1. Goal

Expose the Feather as a class-compliant wired microphone that can be selected by a host without a custom driver or application.

## 2. USB class

Initial specification:

```text
USB Audio Class: 1.0
Direction: input only
Channels: 1
Sample format: signed 16-bit little-endian PCM
Sample rate: 16,000 Hz
Feature control: mute
Software volume: not included initially
```

UAC 1.0 is selected for broad compatibility and reduced implementation complexity.

## 3. Audio topology

The intended descriptor topology is:

```text
Microphone Input Terminal
          |
          v
Feature Unit
  - mute control
          |
          v
USB Streaming Output Terminal
          |
          v
Isochronous IN endpoint
```

The exact terminal types, interface numbers, endpoint numbers, string indices, and control bitmaps must be documented alongside the descriptor code.

## 4. Packet timing

At 16 kHz, mono, 16-bit PCM:

```text
16,000 samples/s
16 samples/ms
2 bytes/sample
32 bytes per 1 ms USB frame
```

The normal USB packet contains 32 bytes.

The USB transport consumes data continuously once the host activates the streaming alternate setting.

## 5. Buffering

The audio capture layer produces one 10 ms frame:

```text
160 samples
320 bytes
```

The USB transport divides that frame into ten 1 ms packets:

```text
10 packets x 16 samples
```

The transport must retain ownership of a frame until all samples have been submitted or copied into a transport-owned buffer.

The ownership strategy must be explicit:

Option A:

- USB retains the frame pointer for 10 ms and releases it after the final packet.

Option B:

- USB copies the frame into a fixed USB-side ring and releases the frame immediately.

The first implementation should prefer the simplest approach that preserves bounded memory and clear ownership.

## 6. USB lifecycle

States:

```text
DISCONNECTED
ENUMERATING
CONFIGURED
STREAM_IDLE
STREAM_ACTIVE
SUSPENDED
ERROR
```

Events to handle:

- USB power detected
- Device reset
- Set configuration
- Set interface/alternate setting
- Start of Frame
- Suspend
- Resume
- Disconnect
- Host mute request

PDM capture may remain stopped until the streaming interface is active.

## 7. Underrun behavior

If USB requires a packet and no sample is available:

1. Send silence.
2. Increment `usb_underruns`.
3. Mark the stream discontinuity in diagnostics.
4. Continue streaming without resetting the USB device.

An underrun is recoverable at runtime but fails the long-duration reliability target.

## 8. Overrun and drift

The PDM clock and USB host timing may drift over long periods.

Initial policy:

- Send exactly 16 samples per USB frame.
- Track source queue depth.
- Run extended tests before adding correction.
- Count all corrections.

If queue depth trends continuously upward or downward, add a minimal correction policy:

- Remove one sample when above a high watermark.
- Duplicate or interpolate one sample when below a low watermark.
- Prefer a zero crossing or low-amplitude location.
- Never hide how frequently corrections occur.

Do not implement a complex resampler before measurement proves it is needed.

## 9. Mute behavior

Host mute should not stop the USB stream.

When muted:

- Continue sending clocked audio packets.
- Ramp gain smoothly toward zero.
- Send zero-valued samples after the ramp.
- Keep capture and diagnostics active.
- Set the NeoPixel to red.

On unmute:

- Ramp gain back to the configured value.
- Avoid a sudden DC or waveform discontinuity.

The physical button initially switches transport mode rather than mute.

## 10. Descriptor development rules

- Keep descriptor definitions in one transport-specific module.
- Add static assertions where possible for total length fields.
- Do not copy unexplained descriptor byte arrays.
- Comment each interface, terminal, feature unit, endpoint, and class-specific field.
- Store VID/PID and strings in one configuration location.
- Use a development VID/PID only with clear documentation and no implication of ownership.
- Validate descriptors on each target host.

## 11. Compatibility matrix

Record at least:

| Host              | OS version | Enumerates | Records audio | Calling app | Notes |
| ----------------- | ---------- | ---------: | ------------: | ----------- | ----- |
| Primary laptop    | TODO       |       TODO |          TODO | TODO        | TODO  |
| Second computer   | TODO       |       TODO |          TODO | TODO        | TODO  |
| Android phone     | TODO       |       TODO |          TODO | TODO        | TODO  |
| USB-C iPhone/iPad | TODO       |       TODO |          TODO | TODO        | TODO  |

Do not claim universal phone compatibility from one device.

## 12. USB diagnostics

Track:

- USB resets
- Configurations
- Streaming starts
- Streaming stops
- Packets submitted
- Bytes submitted
- Underruns
- Sample corrections
- Suspends
- Resumes
- Unexpected requests or errors

## 13. USB acceptance tests

- [ ] Device enumerates without a custom driver.
- [ ] Host shows one mono input.
- [ ] Host activates the streaming interface.
- [ ] Recorded WAV has the correct duration and sample rate.
- [ ] Audio is intelligible.
- [ ] Host mute produces silence without stream interruption.
- [ ] Unplug/replug works repeatedly.
- [ ] A long run produces zero overruns and underruns.
- [ ] Queue depth remains bounded.
- [ ] No memory usage grows over time.

## User Descriptor walkthrough

Before asking an agent to write the final descriptors, explain:

- What an Input Terminal represents
  The mic is represented by an Input Terminal. Microphone audio enters the USB audio function at the physical mic. Audio leaves the audio processing graph toward USB through an Output terminal.
- Why a Feature Unit is needed
  Allow host to mute mic.
- What the Streaming Output Terminal connects
  Audio leaving the AudioControl topology for USB AudioStreaming interface.
- Why the endpoint is isochronous
  In order to stream real-time time-sensitive data It requires fixed bandwidth, no retransmission, and scheduled bounded latency delivery.
- Why the host uses an alternate interface setting
  To give the host the flexibility to change device configurations when required without consuming USB bandwidth. Alternate setting 0 has no streaming endpoint, alternate setting 1 enables isochronous endpoint. Host switches these settings to start/stop streaming.
- How 16 kHz becomes 32 bytes per USB frame
  A full speed USB communications sends packets every 1 ms, and 16-bit audio streams at 16,000 samples per second produces 16 samples per ms, which equals 32 bytes of data.

You may use the USB Audio Class specification and Nordic example as references, but write the explanation yourself.
