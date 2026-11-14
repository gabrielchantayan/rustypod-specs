<!-- Trusted provenance: F-7dbf55efb6cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7dbf55efb6cc

## Purpose

F-7dbf55efb6cc interprets a bounded byte stream, constructs a shared coordinate table, optionally processes auxiliary data, and dispatches decoded coordinate pairs to point and curve dependencies.

Processing is incremental, not transactional. Allocation results, table entries, contour-state changes, diagnostics, and dependency effects can remain observable after failure. The function does not check coordinate-table indices against the counts declared by the stream.

Glyph-outline loading is inferred intent, supported by the operation structure and diagnostic labels. The precise geometric representation and the semantics implemented by dependencies are not established.

## Interface

### Arguments and result

The function accepts three 32-bit address arguments:

1. **Context:** writable state containing the fields below.
2. **Start:** address of the first input byte.
3. **Limit:** exclusive upper bound for direct input reads.

The representation uses 8-bit bytes, 16-bit signed quantities, and 32-bit addresses and integer fields.

The result is a 32-bit status, initially zero:

- A directly detected insufficient input span produces status **8**.
- Allocation, auxiliary processing, capacity expansion, point emission, and curve emission can supply other statuses.
- A terminating instruction returns the current status without assigning the return values of its termination dependencies.

There is no returned cursor or consumed-byte count. Termination need not consume all input.

### Context and associated-object layout

Offsets are byte offsets. These tables describe accessed fields, not complete object definitions.

| Context offset | Width | Established use |
|---|---:|---|
| `+0x0c` | 32 bits | Coordinate-table capacity, compared as unsigned |
| `+0x10` | 32 bits | Coordinate-table address |
| `+0x14` | 32 bits | Y-table base |
| `+0x24` | 32 bits | Associated-object address |
| `+0x28` | 8 bits | Contour-related state flag |

Coordinate entries occupy four bytes each. X entries precede Y entries. The Y-base field is assigned the table address plus four times the declared X count.

| Associated-object offset | Width | Established use |
|---|---:|---|
| `+0x00` | 32 bits | Value captured at entry for allocation |
| `+0x04` | 32 bits, signed | First capacity |
| `+0x08` | 32 bits, signed | Second capacity |
| `+0x14`, `+0x16` | 16 bits, signed | Quantities used in capacity checks |
| `+0x34`, `+0x36` | 16 bits, signed | Additional quantities used in those checks |

Instruction tracing uses a separately retained trace-state address. Its field at `+0xe0` is read as a signed 32-bit trace level.

### Preconditions

The context, associated object, reached table entries, trace-state object, descriptor storage, and other accessed storage must remain valid when accessed. Even empty input requires readable context and associated-object storage: the allocation-related value is read before the first input-bound check.

Start and Limit must form a coherent interval. Direct checks compare Limit with the cursor plus one or two; address wraparound is not separately checked. Safe use requires nonwrapping address calculations, including table-address calculations.

Input and mutable state must not be concurrently modified. Aliasing between input and storage modified by this function or its dependencies is not prohibited by an explicit check; its consequences are not independently established.

## Behavior

### Header and coordinate counts

Before checking input availability, the function reads the associated-object address from context `+0x24` and captures that object’s 32-bit value at `+0x00`.

It then requires and reads one header byte. Header bit `0x80` invokes F-540af0155702. No ordinary status is assigned from this call. If it returns, decoding continues.

Counts are determined as follows:

- With bit `0x04` set, one additional byte supplies both counts: its low nibble is the X count and its high nibble is the Y count. Bits `0x02` and `0x01` do not request further count bytes.
- Otherwise, bit `0x02` requests an unsigned one-byte X count, followed by an unsigned one-byte Y count if bit `0x01` is set.
- An omitted count is zero.

Separate counts are each at most 255, giving a total at most 510. Packed counts are each at most 15. Bit `0x08` controls auxiliary processing. Other header bits have no direct effect here.

### Coordinate-table preparation

If the total count exceeds the unsigned capacity at context `+0x0c`, the requested capacity is the total rounded upward to a multiple of eight. Count limits make this rounding safe within 32 bits.

F-6dd003795db7 receives the captured allocation-related value, element width four, existing capacity, requested capacity, and existing table address. A writable status destination is also exposed in the visible call setup; its exact calling convention is unresolved.

The replacement table value is stored at context `+0x10` before the status is checked. A nonzero allocation status returns immediately. On zero status, the requested capacity is stored at `+0x0c`.

Next, context `+0x14` is assigned the current table address plus four times the X count, even when the total count is zero.

All X entries are decoded before all Y entries, with a single accumulator initially zero:

- One format byte is consumed before each group of up to eight entries.
- Its bits are used least-significant first.
- A clear bit selects an unsigned one-byte increment added to the accumulator.
- A set bit selects a two-byte, big-endian absolute value replacing the accumulator.
- Each completed value is immediately stored as a four-byte entry through the table base read from context `+0x10`.

The absolute construction yields values from 0 through 65535; it does not sign-extend a 16-bit value. The accumulator is not reset at the X/Y boundary. At most 510 entries, with increments at most 255, cannot overflow a signed 32-bit accumulator under these rules.

A zero total consumes neither format bytes nor coordinate bytes.

### Auxiliary processing and instruction initialization

If header bit `0x08` is set, F-129dd09ab861 receives a writable cursor address and Limit. A nonzero returned status is returned immediately. Otherwise processing uses the cursor left by that dependency. No separate validation of that cursor occurs beyond subsequent ordinary read checks.

After successful table and optional auxiliary processing, initialization occurs in this order:

1. Context byte `+0x28` is cleared.
2. The function reads the global trace-state base once and retains that address.
3. Current X and Y coordinates and the initial pair-storage values are initialized to zero.

Throughout instruction decoding, threshold checks read the signed field at the **retained address plus `0xe0`**. They do not reload the global base.

### Instruction classification

Each instruction consumes one byte. Its high nibble selects the operation:

| High nibble | Operation |
|---:|---|
| `0` | Terminate |
| `1` | General line |
| `2` | Horizontal line |
| `3` | Vertical line |
| `4` | Move described as inside |
| `5` | Move described as outside |
| `6` | Fixed-descriptor curve, first class |
| `7` | Fixed-descriptor curve, second class |
| `8`–`15` | General curve |

There is no invalid-opcode class.

For horizontal lines, the low nibble selects an X-table entry; current Y is retained. For vertical lines, it selects a Y-table entry; current X is retained. These indices are not checked against declared counts.

General lines and both move classes use the low nibble as a coordinate-pair descriptor. Fixed-curve classes obtain their descriptor from separate external words and do not use the instruction’s low nibble for component selection.

### Coordinate-pair decoding

A descriptor nibble selects X with its low two bits and Y with its next two bits:

| Selector | Input bytes | Component value |
|---:|---:|---|
| `0` | 1 | Four-byte table entry selected by an unsigned byte index |
| `1` | 2 | Big-endian absolute value, 0 through 65535 |
| `2` | 1 | Retained coordinate plus a signed 8-bit displacement |
| `3` | 0 | Retained coordinate unchanged |

X uses context `+0x10`; Y uses context `+0x14`. Each component table base is read when that component is decoded. Indices receive no count or allocation-bound check.

X is decoded before Y. Both relative or unchanged components refer to the preceding completed pair, not to a partially decoded new pair. After a pair is completed and any required descriptor-byte processing succeeds, it becomes the retained pair for subsequent decoding.

Relative addition uses 32-bit arithmetic without an overflow check or error translation. Arbitrarily long instruction streams can exceed the signed range; an unbounded mathematical interpretation is not established.

### Curves

Every curve supplies three coordinate pairs to F-314d752856b0.

For each fixed-curve class, descriptor acquisition has observable ordering:

1. Read that class’s external 32-bit descriptor word.
2. Read the instruction trace level from the retained trace-state object.
3. If the level exceeds 5, call F-4457a9a44a68.
4. If that call returns, read the descriptor word again.
5. Decode using the later read when the diagnostic occurred; otherwise use the first read.

Thus a diagnostic can change the descriptor that controls decoding. Neither descriptor value nor preservation of descriptor storage by diagnostics is established. Three successive low nibbles of the selected word describe the three pairs.

For a general curve:

- The first pair uses the instruction’s low nibble.
- After both components of that pair have been decoded, one additional descriptor byte is required and consumed.
- That byte’s low nibble describes the second pair; its high nibble describes the third.
- The first pair becomes the retained coordinate pair only after the additional descriptor byte has been obtained.

The additional byte is required even when the first pair consumes no component bytes.

### Diagnostic ordering during instruction decoding

Instruction diagnostics are conditional on a freshly read signed trace level greater than 5. General-line, move, and general-curve instruction diagnostics precede component decoding. The fixed-curve ordering is specified above.

Horizontal and vertical instruction diagnostics occur **before** their indexed coordinate-table loads. If the diagnostic changes the relevant context base or table contents, the later load can observe that change.

Component diagnostics use a fresh trace-level read with threshold greater than 6:

- For selectors `0`, `1`, and `2`, input decoding and storage of the component precede its diagnostic.
- For selector `3`, the diagnostic precedes copying the retained coordinate into component storage.

X’s diagnostic, if enabled, occurs before Y decoding. Accordingly, dependency changes can affect later input or state reads; no preservation guarantee is assumed.

After all pair decoding—or immediately after classification and direct coordinate selection for operations without pair decoding—the function performs another trace-level check with threshold greater than 6. Any resulting separator diagnostic precedes operation dispatch, including termination.

### Dispatch, moves, and termination

General, horizontal, and vertical lines call F-5b74492c5bab with the context and one decoded pair. Curves call F-314d752856b0 with the context and three pairs. A zero status continues instruction decoding; a nonzero status is returned without reading another instruction.

Both move variants use the same direct path:

1. After decoding and the separator diagnostic, capture the associated-object address from context `+0x24`.
2. Call F-1911a8836dad with the context.
3. Write one to context byte `+0x28`.
4. Check capacities through the captured object.
5. If necessary, call F-5fbfabc4db1c with that object and counts `1, 1`.
6. On zero capacity status, emit the decoded pair through F-5b74492c5bab.

The first requirement is the signed 16-bit values at object `+0x16` and `+0x36`, added and increased by one, compared with signed capacity `+0x04`. Only if that requirement fits is the second checked: signed 16-bit values at `+0x14` and `+0x34`, added and increased by one, compared with signed capacity `+0x08`.

Equality fits. Either excess requests capacity expansion. Each requirement lies between −65535 and 65535, so its sum fits signed 32 bits. No nonnegativity validation is performed.

The captured object is not reloaded after F-1911a8836dad. Its continuing validity is required even if that dependency replaces context `+0x24`.

Termination, after its diagnostics, calls F-1911a8836dad, reloads context `+0x24`, and passes that address to F-3c4d89a1b06c. Neither call’s return value replaces the status. Trailing input is ignored.

### Failures

Every direct one-byte or two-byte input read is preceded by a check that the entire span fits. Failure assigns status 8, invokes F-4457a9a44a68 for an invalid-input diagnostic, and returns if that dependency returns normally.

There is no rollback or common termination cleanup on truncation or dependency-status failure. Already completed writes and dependency effects remain.

## Side effects

Direct effects include replacement of the table address, capacity, and Y-base; incremental table writes; and contour-flag writes. The table address can change even when allocation reports failure.

Diagnostics are observable dependencies, not assumed inert. Changing the retained trace object’s `+0xe0` field can affect later checks during this invocation. Replacing the global trace-state base alone does not redirect those checks: the previously captured address remains in use. If a dependency invalidates that retained object, continued tracing accesses are not made safe by the base replacement.

Diagnostic effects can also influence later descriptor reads, coordinate-table loads, component decoding, and dispatch-time context reads according to the ordering above.

The function contains no explicit release of input or coordinate storage and establishes no input-retention mechanism. Allocation, release, retention, or ownership transfer inside dependencies remains unknown. No dependency is assumed to preserve globals, context fields, descriptor storage, table contents, or memory lifetimes.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-6dd003795db7 | Resize/allocation operation returning a table value, with an apparent status-output mechanism |
| F-129dd09ab861 | Optional auxiliary processing through a writable cursor |
| F-1911a8836dad | Called before move capacity checks and during termination; return value ignored |
| F-5fbfabc4db1c | Capacity request for one item in each of two categories |
| F-5b74492c5bab | One-pair emission; returned status controls continuation |
| F-314d752856b0 | Three-pair emission; returned status controls continuation |
| F-3c4d89a1b06c | Termination operation on the reloaded associated object; return value ignored |
| F-4457a9a44a68 | Invalid-input and conditional trace diagnostics |
| F-540af0155702 | Assertion-related handling for header bit `0x80` |

These roles do not establish complete callee contracts, ownership, preservation guarantees, or abnormal-control-flow behavior.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Normal-return examples assume valid reached storage and returning dependencies.

1. **Empty description.** Header `0x00` followed by instruction `0x00` declares no coordinates. The Y-base is assigned, the contour flag is cleared, and the trace base is captured. After applicable diagnostics, termination dependencies run in order. Ordinary status is zero; trailing bytes are uninterpreted.

2. **Missing header.** With Start equal to Limit, no input byte is read. The associated-object allocation value has already been read. The invalid-input diagnostic occurs and status 8 is returned.

3. **Packed counts.** Header `0x07` followed by count byte `0x21` declares one X and two Y entries. No separate count bytes are requested.

4. **Shared accumulator.** With one X and one Y entry, a format byte whose first two bits are clear and increments 10 and 5 yields X equal to 10 and Y equal to 15.

5. **Absolute and relative values.** Absolute bytes `0xff, 0xff` yield 65535, not −1. From current pair `(0, 0)`, line instruction `0x1e` followed by `0xff` supplies `(−1, 0)`.

6. **General curve without component input.** Instruction `0x8f` followed by descriptor `0xff` supplies three unchanged pairs. From initial coordinates, all are `(0, 0)`. Omitting the descriptor byte instead produces status 8.

7. **Truncated absolute component.** With only one byte remaining for a two-byte absolute component, neither component byte is read. Earlier writes remain; the invalid-input diagnostic precedes return of status 8.

8. **Unchecked index.** An index exceeding its declared coordinate count does not itself produce status 8. The function attempts the table load; safe acceptance requires separately valid referenced storage.

9. **Retained trace base.** If a dependency replaces the global trace base but leaves the previously captured object valid, later checks still read the old object’s `+0xe0`. Changing that old field can change later diagnostic decisions.

10. **Fixed-descriptor mutation.** If instruction tracing is enabled and its diagnostic changes the relevant fixed-curve descriptor word, the post-diagnostic word controls decoding. Without that diagnostic, the initial word controls decoding.

11. **Horizontal load timing.** If an enabled horizontal instruction diagnostic changes context `+0x10` to another valid table, the indexed load uses the later base, not an entry-time snapshot.

12. **Component diagnostic placement.** For a relative X component, its result is stored before its component diagnostic. Any Y decoding follows that diagnostic. For an unchanged component, the diagnostic instead occurs before the retained value is copied.

13. **Emitter failure.** A nonzero point or curve status returns directly without consuming the next instruction or automatically running termination dependencies.

## Uncertainties

The following remain **documented_only**:

- Values and intended meanings of the two fixed-curve descriptor words.
- Complete dependency contracts, including allocation-failure ownership and the status-output calling convention.
- Whether assertion handling or diagnostics return, terminate, or otherwise transfer control.
- Meanings of nonzero dependency statuses.
- Semantic differences, if any, between the two move variants beyond their diagnostic selection.
- Coordinate units and the semantic categories represented by capacity fields.
- Relative-coordinate behavior at signed-overflow boundaries beyond the visible 32-bit operations.
- Dependency changes to globals, passed or retained storage, context, and memory lifetimes.
- Exact diagnostic text and external trace configuration.

## Evidence

Established behavior derives from the supplied static representation: access widths, masks, bounds checks, arithmetic, direct storage operations, and call ordering. In particular, it establishes the one-time trace-base capture, repeated signed trace-field reads, conditional fixed-descriptor rereads, and diagnostic placement relative to coordinate reads and stores.

Glyph, contour, allocator, and builder terminology expresses inferred intent where direct operations do not establish the full semantics.

No runtime tests or observations have been performed. The acceptance examples are conditional requirements derived from static evidence, not execution reports.
