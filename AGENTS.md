# AGENTS.md

This file defines how coding agents must work in this repository.

The human developer is intentionally writing most architecture and core firmware code for learning. The agent should accelerate repetitive work, testing, review, and debugging without silently taking ownership of the design.

## Primary objective

Help deliver a working nRF52840 USB/BLE microphone while preserving the human developer's understanding of:

- PDM and EasyDMA
- Fixed-memory ownership
- Interrupt-safe queues
- USB Audio Class
- BLE GATT and packetization
- Real-time backpressure
- Testing and measurement

## Read-first order

Before changing code, read:

1. `docs/requirements.md`
2. `docs/architecture.md`
3. `docs/platform.md`
4. The transport-specific document for the requested task
5. `docs/test-plan.md`
6. `docs/roadmap.md`

Treat those files as the source of truth. Report contradictions instead of choosing silently.

## Human-owned decisions

The agent must not independently redefine:

- Audio frame size or sample format
- Frame ownership semantics
- Queue-full policy
- Interrupt responsibilities
- Transport interface
- Mode-switch behavior
- SoftDevice or bootloader memory layout
- USB descriptors
- BLE packet format
- Clock-source configuration
- Fatal-error policy
- Public APIs already marked stable

Changes to these require an explicit request or a written proposal that the human approves.

## Areas the agent may handle freely

The agent may normally modify:

- `tests/`
- `host/`
- `tools/`
- Documentation proofreading
- CMake helper modules
- UF2 post-build tooling
- Repetitive nRF5 SDK source lists
- Test mocks and fakes
- Benchmark scripts
- Static-analysis configuration
- Formatting and lint fixes that do not change behavior

## Areas requiring a focused request

The agent may modify these only when the task explicitly targets them:

- `firmware/audio/`
- `firmware/transports/usb/`
- `firmware/transports/ble/`
- `firmware/platform/`
- `firmware/drivers/`
- Linker scripts
- Startup code
- USB descriptors
- BLE service definitions

## Protected learning placeholders

Sections marked `TODO(USER)` or `LEARNING CHECKPOINT` must not be completed by the agent unless the human explicitly asks for feedback after writing a first attempt.

Allowed behavior:

- Point out missing topics
- Ask the human to clarify an invariant
- Proofread the human's completed text
- Suggest questions the human should answer
- Compare the human's answer with the implementation

Disallowed behavior:

- Replacing the placeholder with a complete answer
- Generating the human's explanation before the human attempts it
- Hiding a design decision inside generated code

## Coding rules

- Use C11.
- Use fixed-width integer types for serialized or hardware-facing data.
- Do not allocate memory dynamically in the audio path.
- Avoid dynamic allocation elsewhere unless documented and justified.
- Do not block inside callbacks or interrupt handlers.
- Do not call `printf`, RTT logging, USB logging, or string formatting from interrupt context.
- Keep interrupt handlers minimal: acknowledge hardware, transfer ownership, update counters, and schedule deferred work.
- Check every SDK return value unless explicitly documented as infallible.
- Use named constants instead of unexplained literals.
- Use `static` for file-local functions and objects.
- Avoid global mutable state unless it is part of a clearly owned module.
- Public functions must document ownership, thread/interrupt context, and failure behavior.
- Serialized structs must not rely accidentally on compiler padding.
- Never assume a BLE MTU, data length, or PHY without checking the negotiated value.
- Never assume a host application consumes audio at exactly the nominal rate without instrumentation.
- Never modify vendored nRF5 SDK source files directly.

## Frame ownership rules

The intended lifecycle is:

```text
FREE -> CAPTURE -> READY -> PROCESSING -> TRANSPORT -> FREE
```

Only the current owner may write to a frame.

Pointer queues transfer ownership. A successful queue push transfers ownership to the receiving stage. A failed push leaves ownership with the caller unless the API explicitly documents otherwise.

Any ownership ambiguity is a correctness bug.

## Queue-full policy

For live audio, preserve low latency.

When a ready queue is full:

1. Remove the oldest queued frame.
2. Increment the relevant dropped-frame counter.
3. Mark a discontinuity.
4. Return the old frame to the free pool.
5. Enqueue the newest completed frame.

USB mode should normally never drop frames. Any USB drop must be visible in diagnostics and treated as a failed reliability test.

## Testing responsibilities

For each important module:

1. The human defines the API and invariants.
2. The human writes the first essential tests.
3. The human implements or leads the implementation.
4. The agent reviews the code.
5. The agent adds boundary, failure-injection, parameterized, and regression tests.
6. The human reviews every agent-written test.

The agent may write most test lines, but it must not decide the contract by copying implementation behavior.

## Required checks before completing a task

Run or document:

```bash
cmake --build build
ctest --test-dir build --output-on-failure
```

Where hardware is required, provide:

- Exact flash command or UF2 output
- Expected LED/log behavior
- A short manual verification procedure
- Counters that should remain zero
- Failure symptoms and likely causes

Never claim hardware behavior was verified when it was not run on hardware.

## Change discipline

Keep changes small and reviewable.

For every nontrivial change, summarize:

- What changed
- Why it changed
- Which documented requirement it implements
- Which tests were added or updated
- Which hardware assumptions remain unverified
- Any risks or follow-up work

Do not perform broad rewrites merely to match personal style.

## Debugging approach

Prefer evidence in this order:

1. Compiler and linker errors
2. Unit tests
3. Map file and memory layout
4. USB descriptors or BLE packet captures
5. Runtime counters
6. Compact deferred logs
7. Manual listening tests

Do not guess when a counter, descriptor dump, or minimal reproduction can answer the question.

## Documentation behavior

The agent may proofread and improve clarity after the human has completed `TODO(USER)` sections.

The agent must preserve:

- Technical meaning
- Open questions
- Explicit uncertainty
- Learning notes
- Decisions that are intentionally provisional

## Your collaboration rules

- Be straight to the point I have ADHD.
- No em dashes.
- Start all messages with "Penguin \n".
- Before coding, explain step by step all decisions.
- A patch must be limited to a singular commit.
- Tests will be proposed after implementation.
- Never implement ANY code without explicit allowance.
- When the design is wrong, immediately cite the exact lines of code or concept and offer up alternative means. Use Socratic Reasoning to explain any wrong concepts.
