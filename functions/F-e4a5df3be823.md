<!-- Trusted provenance: F-e4a5df3be823; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4a5df3be823

## Purpose

F-e4a5df3be823 retrieves a byte from a bounded input region after skipping space and horizontal-tab bytes. It also records selected boundary conditions in a persistent state word.

The distinguishing behavior is a positive-state shortcut: if the state word is positive when interpreted as a signed 32-bit integer, the function returns `0x0000003B` without inspecting or consuming input. This applies to every positive state value, not only values established by this function.

When that shortcut does not apply, the function consumes bytes until it encounters something other than `0x20` or `0x09`, or reaches the end of the available region. It returns the resulting byte value, or `0xFFFFFFFF` for exhausted input. Selected results update the state:

- Carriage return or line feed establishes state `2`.
- Semicolon establishes state `1`.
- Byte `0x1A` or exhausted input establishes state `3`.

**Inferred intent:** the state appears to latch a boundary for a higher-level consumer. The meanings of the three state values, and the reason all positive states subsequently produce a semicolon result, are not established by the supplied evidence.

## Interface

### Input record

The function receives one pointer to a mutable record. The supplied representation exposes four consecutive 32-bit slots:

| Byte offset | Role in this function |
|---|---|
| `0` | Current input position, represented as an address. Read and advanced during input consumption. |
| `4` | Not accessed by this function. Its meaning is unknown. |
| `8` | End address used as the exclusive upper bound for consuming a byte. |
| `12` | State word. Read with signed interpretation for the entry shortcut and written for selected results. |

The minimum represented record extent is 16 bytes. The four-slot layout and 32-bit state interpretation are the interface model used here. The original ABI declarations, pointer provenance rules, and exact platform address-comparison semantics are not independently supplied.

### Result

The result is modeled as an unsigned 32-bit value:

- A consumed byte is returned zero-extended, in the range `0x00000000` through `0x000000FF`.
- Exhausted input returns `0xFFFFFFFF`.
- A positive entry state returns `0x0000003B`, independently of the available input.

A consumed byte `0xFF` therefore returns `0x000000FF`, not the exhaustion sentinel.

No separate error result, exception contract, or diagnostic output is established.

### Preconditions

For a defined, ordinary use of the scanning path:

- The record pointer must allow the required field reads and any resulting writes.
- The current and end addresses must support the comparison shown by the supplied representation.
- Every byte selected for consumption must be readable.
- Advancing the current address by one must produce a representable, valid position.
- The record and input must remain stable except for the writes described here.

The function does not validate these conditions.

For the positive-state shortcut, the input addresses are not inspected. Consequently, readable input memory is not required for that path, although the state field must still be readable.

Non-overlap between input storage and the mutable record is a useful precondition for the normal interpretation of “consume the current byte.” Overlap is not rejected. It can make the ordering of record writes and byte reads affect the result.

## Behavior

### 1. Entry-state decision

The first behavioral decision is based on the state field interpreted as a signed 32-bit integer.

If that value is greater than zero:

- The result is `0x0000003B`.
- The current position is not read or changed.
- The end address is not read.
- No input byte is read.
- The state is not changed.

The shortcut covers signed values `1` through `2,147,483,647`. A nonzero state with its high bit set is negative under this interpretation and does not activate the shortcut.

If the state is zero or negative, processing continues into input scanning. The state is not rechecked during that scan.

### 2. Obtaining a candidate

Each scan iteration obtains a candidate using this order:

1. Read the current-position field.
2. Read the end-address field and compare the two addresses.
3. If the current position precedes the end address, store the current position advanced by one into the record.
4. Read the byte at the position captured before that advance, and zero-extend it as the candidate.

The current-position write precedes the input-byte read in the supplied operation order. The byte is read from the captured old position, not from the newly stored position.

If the current position does not precede the end address:

- No input byte is read.
- The current-position field is not written for that iteration.
- The candidate is `0xFFFFFFFF`.

Equality with the end address is therefore exhaustion. A current position beyond the end address also produces exhaustion; the function does not rewind it, clamp it, or report a distinct invalid-range condition.

These are static operation-order findings. No additional guarantees about machine-level ordering, concurrent observation, or volatile memory are supplied.

### 3. Skipping bytes

A candidate equal to either of the following causes another scan iteration:

- `0x20`: space.
- `0x09`: horizontal tab.

Each skipped byte has already advanced the stored current position by one.

Only these two byte values are skipped. In particular, carriage return, line feed, vertical tab, form feed, and zero are not skipped.

The loop terminates on the first other candidate, including the exhaustion sentinel. The function performs no additional input lookahead after that candidate.

### 4. Classifying the terminating candidate

The terminating candidate determines the state update and result:

| Candidate | State update | Returned value |
|---|---:|---:|
| `0x0D` | Store `2` | `0x0000000D` |
| `0x0A` | Store `2` | `0x0000000A` |
| `0x3B` | Store `1` | `0x0000003B` |
| `0x1A` | Store `3` | `0x0000001A` |
| `0xFFFFFFFF` | Store `3` | `0xFFFFFFFF` |
| Any other byte | No state write | That byte, zero-extended |

For an ordinary byte, a preexisting negative state remains negative. The function does not reset it to zero.

The state write, when required, occurs after the candidate has been obtained and classified. A special byte has already been consumed before its state is stored. Exhaustion has no corresponding input-byte consumption.

### 5. Behavior across calls

Every state established by this function is positive. Therefore, after a call stores `1`, `2`, or `3`, later calls return `0x0000003B` without consuming more input unless some external action changes the state.

This has several consequences:

- A carriage return and a following line feed are not consumed as one combined sequence.
- Exhaustion does not repeatedly return `0xFFFFFFFF` while state `3` remains installed.
- Consuming `0x1A` does not discard the remaining buffer.
- The stored positive state is preserved by subsequent shortcut calls.

No state-reset operation is present in this function.

## Side effects

### Record writes

The function can write two fields:

- **Current position:** advanced once per consumed byte, including skipped spaces and tabs and the terminating non-skipped byte.
- **State:** overwritten only for carriage return, line feed, semicolon, byte `0x1A`, or exhaustion.

The slot at offset `4` is not accessed. The end-address field is read but not explicitly written.

With stable, non-overlapping storage, consuming \(n\) bytes advances the current position by \(n\) byte positions. There is no separate cumulative counter or multi-byte arithmetic operation.

### Input storage and aliasing

There is no explicit write through an input-byte address. However, the record writes can also modify input storage if those memory regions overlap.

That distinction matters because the current-position update happens before the selected byte is loaded. If that update overlaps the selected byte, the returned candidate can reflect the modified storage rather than the byte present before the call.

Likewise, a state write can affect overlapping input storage. Exact overlap outcomes depend on placement, address representation, and byte order; they are not generalized here.

### Ownership and external effects

The function performs no visible allocation, deallocation, ownership transfer, or dependency call. It retains the updated position and state through the caller-supplied record.

No global access, file operation, logging operation, or synchronization operation is visible in the supplied evidence. These findings do not establish ownership rules for the record or input buffer elsewhere in the system.

Invalid memory access and concurrent mutation have no defined recovery behavior in the supplied evidence. A fault during a selected byte read could occur after the preceding current-position write; no transactional rollback is shown.

## Dependencies

No calls to other functions are present. There are therefore no opaque function dependencies to enumerate.

The behavior relies on primitive operations whose platform details are not fully supplied:

- Reading and writing record fields.
- Comparing input addresses.
- Advancing an address by one byte.
- Loading an unsigned byte.
- Interpreting the state as signed for the entry comparison.
- Returning a 32-bit unsigned result.

No assumptions about an unknown callee preserving globals, acquiring ownership, or releasing memory are needed.

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not observed runtime results. Unless stated otherwise, assume valid, stable, non-overlapping storage and initial state `0`.

### Positive entry state

Given state `2` and a buffer beginning with `0x41`, the result must be `0x0000003B`. The current position and state remain unchanged, and the byte `0x41` is not read.

The same behavior applies to positive state `99`, even though this function never establishes that value itself.

### Ordinary byte

Given bytes `41 42`, the first call must return `0x00000041`, advance the position by one, and leave state `0`. A second call must return `0x00000042` and advance it again.

### Space and tab skipping

Given bytes `20 09 20 41`, one call must return `0x00000041`, advance the position by four, and leave state unchanged.

Given bytes `0B 41`, one call must return `0x0000000B` and advance only once. Vertical tab is not skipped.

### Carriage return followed by line feed

Given bytes `0D 0A 41`, the first call must return `0x0000000D`, advance once, and store state `2`.

A second call without an external state change must return `0x0000003B` without advancing.

If external code resets the state to zero, the next call must consume `0x0A`, return `0x0000000A`, and store state `2` again.

### Semicolon

Given bytes `3B 41`, the first call must return `0x0000003B`, advance once, and store state `1`.

The next call must return the same numeric result without consuming `0x41`. Thus, the return value alone does not distinguish a consumed semicolon from the positive-state shortcut.

### Byte `0x1A`

Given bytes `1A 41`, the first call must return `0x0000001A`, advance once, and store state `3`.

The following call must return `0x0000003B` without consuming the remaining byte.

### Exhaustion after skipped bytes

Given a region containing only `20 09`, one call must consume both bytes, return `0xFFFFFFFF`, and store state `3`. The stored position must equal the end address.

The next call must return `0x0000003B`, not `0xFFFFFFFF`, unless the state is externally changed.

### Empty or reversed range

If the current position equals the end address, the call must return `0xFFFFFFFF`, leave the position unchanged, and store state `3`.

If the current position compares beyond the end address, the same result and state update apply, conditional on that address comparison being valid in the execution environment.

### Negative state and unsigned bytes

Given initial state `0xFFFFFFFF` and input byte `0xFF`, the entry state is negative rather than positive. The call must consume the byte, return `0x000000FF`, and preserve state `0xFFFFFFFF`.

If the input byte instead is `0x3B`, the call must consume it and replace the negative state with `1`.

## Uncertainties

- **documented_only — ABI:** The interface above uses the supplied four-byte-slot representation. Original type declarations and independent ABI documentation are absent.
- **documented_only — Address semantics:** The exact rules for comparing unrelated addresses, representing pointers, and handling address overflow are not supplied.
- **documented_only — State meanings:** The values written to state are established; their higher-level names and interpretation are not.
- **documented_only — Reset protocol:** An external reset is necessary to resume scanning after a positive state, but the responsible component and reset conditions are unknown.
- **documented_only — Ownership and lifetime:** The function does not establish who owns the record or input region or how long either remains valid.
- **documented_only — Exceptional execution:** Invalid memory, overlapping storage, and concurrent mutation lack a comprehensive platform-level contract.

These unknowns do not justify adding validation, normalization, automatic state resets, or alternate return values.

## Evidence

**Established static evidence:** the signed positive-state shortcut; record-field accesses; byte-consumption order; space and tab skipping; exact classification values; conditional state writes; result values; and absence of visible dependency calls.

**Inferred intent:** the record resembles a bounded byte scanner with a persistent boundary latch. This interpretation is not needed to apply the behavioral rules.

**Verification status:** no runtime tests have been performed. The acceptance examples are static predictions, not execution evidence.
