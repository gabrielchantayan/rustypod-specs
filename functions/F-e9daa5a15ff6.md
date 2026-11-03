<!-- Trusted provenance: F-e9daa5a15ff6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e9daa5a15ff6

## Purpose

F-e9daa5a15ff6 advances a cursor through a bounded byte region until it encounters a carriage return, line feed, substitute byte, or the region boundary. It records one of two stopping states and returns a value derived from the cursor captured before scanning.

An unconditional call to F-e4a5df3be823 precedes all visible inspection of the supplied record. After that call, a signed state value greater than one suppresses scanning and causes a zero return.

The operation resembles a text-scanning state transition, but that interpretation is inferred intent. The established behavior is byte-oriented: it recognizes individual byte values, advances the cursor before examining each available byte, and does not combine a carriage-return/line-feed pair.

## Interface

### Input record

The sole explicit input is a pointer to a mutable record. The visible operation accesses its first, third, and fourth integer-sized fields. Its second field is not directly accessed.

The represented interface uses the following 32-bit layout:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Current cursor, stored as an integer representation of a byte address |
| 4 | 32 bits | Unused by this function’s visible body |
| 8 | 32 bits | Exclusive end address of the byte region |
| 12 | 32 bits | Signed state value |

This layout follows the apparent 32-bit integer/address model of the supplied representation. An independently verified ABI or original type declaration was not supplied.

The record must be readable wherever the visible path inspects it. Scanning requires its cursor field to be writable; a completed scan also requires its state field to be writable. Any byte selected for examination must be readable.

No null-pointer, alignment, record-size, or buffer-validity checks are visible. Preconditions imposed by F-e4a5df3be823 are additional unknowns.

### Return value

The visible operation returns a signed integer-sized value:

- If the state inspected after the dependency call is greater than one, the return is zero.
- Otherwise, after a stopping condition is reached, the return is the initially captured cursor representation minus one.

The second return is not a byte count, stopping-byte address, or final cursor. Its input is the cursor captured after the state gate and before the scan.

Subtraction is shown as signed integer arithmetic. If the captured value is the minimum representable signed 32-bit integer, subtracting one is outside that signed range. The supplied representation does not establish whether the executable wraps, traps, or has some other consequence in that case. Portable acceptance requirements must exclude that case unless target arithmetic is independently established.

### Ownership

The function accepts an existing record and accesses an existing byte region. Its visible body neither allocates nor releases storage, returns a newly owned object, nor directly writes into the byte region.

Those observations do not establish the ownership or lifetime effects of F-e4a5df3be823. In particular, its call must not be assumed to preserve storage validity.

## Behavior

### Entry and state gate

F-e4a5df3be823 is called first, without explicit arguments in the supplied representation. No returned value from that call is used by the visible body.

The function then reads the record’s fourth field as a signed integer. A value greater than one causes an immediate zero return. On this path, the visible body does not read the cursor or end fields and does not write the record.

This is a post-dependency gate. Changes made by the dependency can affect whether scanning occurs. “No writes on the gated path” applies only to the visible body, not to the complete invocation including the dependency.

The gate is not a test for a single expected starting state. Zero, one, and negative signed state values all permit scanning. Values two and above suppress it.

### Initial cursor capture

If scanning is permitted, the current cursor representation is read and retained for the eventual return calculation.

The cursor is then read again when the first scanning iteration begins. Thus the return baseline is a separate read from the cursor used for the first bounds comparison. They normally agree when storage is stable and non-overlapping, but the representation does not make them a single snapshot.

### Available-byte processing

Each iteration obtains the current cursor and compares it with the current end address. The end field is consulted on every iteration; it is not captured once at scan entry.

If the cursor is strictly below the end address:

1. The cursor field is updated to the selected byte address plus one.
2. The byte at the previously selected address is read.
3. That byte is interpreted as an unsigned value from 0 through 255.
4. The byte is classified as a stopping byte or an ordinary byte.

The cursor write precedes the selected-byte read. This order matters if the byte region overlaps the record or if reads and writes have externally observable effects.

The recognized stopping bytes are:

| Byte value | Classification | State written |
|---|---|---:|
| 0x0D | Carriage return | 2 |
| 0x0A | Line feed | 2 |
| 0x1A | Substitute | 3 |

Every other byte value continues the scan. In particular, a zero byte is not a terminator, and 0xFF is an ordinary byte.

A stopping byte is consumed: the cursor has already advanced past it before the final state is written. No visible step examines or consumes the byte following a stopping byte.

### Region exhaustion

If the current cursor is not strictly below the end address, no byte is read and the cursor is not advanced in that iteration. The function instead treats the iteration as exhaustion.

Exhaustion writes state 3 and returns the captured initial cursor representation minus one.

This same path applies when the cursor equals the end address and when it compares above the end address. There is no separate visible error for a reversed or already-exceeded range.

The exhaustion indicator is distinct from all possible byte values: byte loads are unsigned and cannot produce it. In particular, an actual byte of 0xFF does not indicate exhaustion.

### Completion

After either a recognized stopping byte or exhaustion, the visible body writes the selected state and returns the value derived from the initial cursor.

State 2 therefore identifies a carriage-return or line-feed stop. State 3 does not distinguish substitute-byte termination from exhaustion.

For a stable, ordinary byte region, the final cursor is:

- One byte beyond the first recognized stopping byte, if one is encountered before the end.
- Equal to the end after all available ordinary bytes have been consumed.
- Unchanged if scanning begins at or above the end.

The return does not vary with how many bytes were consumed.

## Side effects

The visible body has two kinds of writes:

- A cursor write for each byte selected for examination, including a stopping byte.
- One state write when scanning completes.

It does not reset the state before scanning. The preexisting state remains until the final state write, although cursor updates may already have occurred.

There is no visible rollback. If a memory access fails after one or more cursor updates, no restoration of the previous cursor is established. No exception handling or fault-to-status conversion is shown.

For an available stopping byte, the observable order is: cursor advancement, byte read, final state write, return. For immediate exhaustion, the visible body makes no cursor write before the final state write.

The second record field receives no direct access from this function. This is not a guarantee that it remains unchanged: the dependency, overlapping storage, or external mutation could affect it.

No global accesses, I/O operations, allocation, or release are explicit in the scanning body. Their absence from that body does not constrain the dependency.

## Dependencies

### F-e4a5df3be823

Established call-site properties:

- Called exactly once on every visible invocation path.
- Called before the state gate or any visible cursor/end inspection.
- Receives no explicit arguments in the supplied representation.
- Has no return value consumed by this function.

Documented-only unknowns:

- Whether it reads or writes global state.
- Whether it can access or change the supplied record indirectly.
- Whether it allocates, releases, or invalidates memory.
- Whether it performs I/O, synchronizes execution, or invokes other code.
- Whether it always returns normally.

Consequently, all scanning descriptions are conditional on the dependency returning and the required record and byte storage being valid afterward. The dependency must not be modeled as a no-op merely because its return value is unused.

## Acceptance examples

These examples assume F-e4a5df3be823 returns normally and leaves the described post-call setup intact. They also assume ordinary, stable, non-overlapping storage, valid address comparisons and increments, and a representable return subtraction.

### State prevents scanning

After the dependency returns, the state is 2. The function returns zero without directly reading the cursor or end fields and without directly writing the record.

The same result applies to any signed state greater than one. A negative state does not satisfy this gate.

### Ordinary bytes followed by line feed

The post-call state is 0. The cursor addresses three available bytes: 0x41, 0x42, and 0x0A.

The function consumes all three bytes and writes state 2. The final cursor is three bytes beyond its initial address. The return is the initial cursor representation minus one, not three and not the final cursor minus one.

### Carriage return followed by line feed

The available bytes begin with 0x0D followed by 0x0A, and the state permits scanning.

Only the carriage return is consumed. State becomes 2, and the cursor addresses the line feed.

If a subsequent invocation sees state 2 after its dependency call, it returns zero without consuming the line feed. Continued scanning would require the gate to permit it; this function does not visibly reset that state.

### Substitute byte

The available region begins with 0x1A and contains further bytes.

The substitute byte is consumed, state becomes 3, and the following bytes are not examined by the visible scan. The return remains the initial cursor representation minus one.

### Empty region

The cursor equals the end address and the state permits scanning.

No byte is read and the cursor remains unchanged. State becomes 3. The return is still the initial cursor representation minus one; immediate exhaustion does not itself require a zero return.

### Cursor beyond the end

The cursor compares above the end address.

The visible body does not attempt a byte read or move the cursor backward. It writes state 3 and returns the captured cursor representation minus one.

### Zero and 0xFF are ordinary bytes

The complete available region consists of 0x00 followed by 0xFF.

Both bytes are consumed. On the next iteration, the cursor equals the end address, so the function writes state 3 without another byte read. Neither byte independently terminates scanning.

### Dependency changes the gate

The state is 0 before invocation, but the dependency changes it to 3.

The subsequent gate returns zero. Conversely, if the dependency changes a previously greater-than-one state to 1 and leaves valid storage, scanning proceeds from the post-call record contents.

These are conditional outcomes, not claims that the dependency actually performs either mutation.

## Uncertainties

### Established static evidence

The supplied representation establishes the dependency-before-gate order, signed state test, separate initial cursor capture, per-iteration cursor and end inspection, cursor-write-before-byte-read order, recognized byte values, final state assignments, and return calculation.

It also establishes that no explicit validation, allocation, release, or recovery mechanism appears in the visible body.

### Inferred intent

States 2 and 3 appear to represent distinct terminal conditions: line-ending recognition versus substitute/exhaustion. Their application-level meanings are not supplied.

The operation appears intended for bounded text traversal. It does not establish an encoding, record format, or policy for paired line endings.

### Documented-only unknowns

- The independently verified ABI, original types, and address model.
- The exact machine behavior of signed return underflow.
- Address increment behavior at the address-space boundary.
- The validity and ordering semantics of comparisons for unrelated addresses.
- The dependency’s full behavior and memory effects.
- Whether overlapping record/buffer storage or concurrent mutation is supported.
- Failure behavior for invalid, unmapped, misaligned, or non-writable storage.

The sequential operation order is described, but no atomicity or synchronization guarantee is established. Stable-region examples must not be generalized to concurrently changing or overlapping storage. Because the end is reread each iteration, termination also cannot be guaranteed if external changes continually extend the region or alter the cursor.

## Evidence

This contract is based solely on the supplied static representation of F-e9daa5a15ff6 and the provided opaque identity for its direct dependency.

No implementation access, runtime execution, or tests were used. Acceptance examples are conditional consequences of the visible control flow and data accesses, not observed test results. Dependency internals and target-specific arithmetic or memory behavior remain unknown where identified above.
