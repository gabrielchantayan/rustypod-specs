<!-- Trusted provenance: F-2dcc32bca856; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2dcc32bca856

## Purpose

F-2dcc32bca856 decodes a sequence of compressed literals and length/distance matches into caller-provided output storage. It uses table-driven, least-significant-bit-first decoding and can obtain match bytes from both previously produced output and a circular history buffer.

The function is a fast decoding path, not a complete stream decoder. It begins decoding without checking input or output capacity. After each completed literal or match, it continues only while sufficient input and output margins remain. It also stops on an end-of-block entry or a recognized decoding error.

Its direct return carries no value. Results are observable through output bytes, updated stream cursors and remaining counts, saved bit-buffer state, and, on selected exits, a state code and diagnostic pointer.

The apparent relationship to a particular compression format is inferred intent. The table interpretation and state transitions below are established by the supplied static evidence.

## Interface

### Arguments and representation

The first argument addresses a mutable stream record containing at least eight 32-bit slots. Relevant fields are:

| Byte offset | Meaning |
|---|---|
| 0 | Address of the next input byte |
| 4 | Number of available input bytes |
| 12 | Address of the next output byte |
| 16 | Number of available output bytes |
| 24 | Diagnostic-message address |
| 28 | Address of the decoding-state record |

Slots at offsets 8 and 20 are not accessed.

The second argument is a 32-bit integer used to establish how much output precedes the current output cursor within the current decoding region. Its apparent intended meaning is the output capacity recorded at the beginning of a surrounding decoding operation. This function does not check that interpretation.

For ordinary, nonwrapping arithmetic, let:

- **Initial output availability** be the stream count on entry.
- **Prior output** be the second argument minus initial output availability.
- **Produced output** be the number of bytes emitted during this call.

A decoded distance can be satisfied entirely from the current output region when it is no greater than prior output plus produced output.

The implementation uses 32-bit addresses and arithmetic quantities, 16-bit table words, and 8-bit data bytes. It must not be interpreted as a native-pointer-width interface on an arbitrary host.

### Decoding-state record

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 8 bits written | Decoder mode or status |
| 40 | 32 bits | History-buffer capacity |
| 44 | 32 bits | Valid history-byte count |
| 48 | 32 bits | History write position |
| 52 | 32 bits | History-buffer address |
| 56 | 32 bits | Buffered input bits |
| 60 | 32 bits | Number of buffered bits |
| 76 | 32 bits | Literal/length table address |
| 80 | 32 bits | Distance table address |
| 84 | 32 bits | Literal/length root-index width |
| 88 | 32 bits | Distance root-index width |

The history metadata and decoding tables are read but not updated. Only the low byte at state offset 0 is written when reporting end-of-block or an error.

### Table layout

Each table entry occupies four bytes, interpreted as two 16-bit words:

- The first word’s low eight bits are an operation descriptor.
- The first word’s high eight bits are the number of bits to consume.
- The second word is a 16-bit value: a literal, a base length, a base distance, or a subtable offset.

The descriptor is interpreted numerically as described below. The supplied representation does not independently establish a portable external byte serialization for these words.

### Preconditions

This function does not validate its pointers, tables, capacities, or bit-buffer invariants. Safe use requires:

- Readable input sufficient for the unconditional first decoding iteration and its possible refills.
- Writable output sufficient for that iteration’s complete result.
- Valid table addresses and every reachable subtable entry.
- Valid history metadata and readable history storage whenever a match needs history.
- An output-region baseline consistent with the second argument.
- Buffered bits and their count in a representation compatible with the refill operations.
- Table-controlled widths, bit consumption, lengths, and distances that avoid invalid shifts, count underflow, unintended arithmetic wrap, and out-of-range memory access.

The continuation margins are six input bytes and 258 output bytes. These are not checked before the first iteration and do not validate malformed tables.

## Behavior

### Entry snapshot and decoding boundaries

The function obtains the decoding-state address and snapshots the stream cursors, capacities, history metadata, table addresses, root widths, and bit-buffer state.

It maintains local cursors representing the last input byte fetched and the last output byte written. Initially, these are one byte before the corresponding stream cursors. This bookkeeping does not itself read those preceding bytes.

Two boundaries are derived from the original stream ranges:

- An input boundary six bytes before the input end.
- An output boundary 258 bytes before the output end.

The output baseline used for distance comparisons incorporates the second argument. It can therefore include bytes produced before this call.

The function enters the decoding loop unconditionally.

### Bit-buffer operation

Decoding proceeds from the least significant end of a 32-bit bit accumulator.

Before a root literal/length lookup, if fewer than 15 bits are buffered, the function fetches two successive input bytes. The earlier byte contributes at the current bit count; the later byte contributes eight positions above it. The buffered-bit count increases by 16.

Refills add shifted byte values to the accumulator. They do not first clear the destination bit positions. Correct operation therefore depends on the unused accumulator region being clear.

Each selected entry consumes its specified bit count by shifting the accumulator right and subtracting that count from the buffered-bit count. The function does not verify that this consumption is valid.

### Literal/length decoding

The root index is formed from the accumulator’s low bits, using the configured literal/length root width.

For every selected entry, the function consumes the entry’s bit count before interpreting its descriptor. Descriptor precedence is:

1. **Descriptor zero:** emit the low eight bits of the entry value as one literal byte.
2. **Descriptor bit 4 set:** interpret the entry as a length.
3. **Descriptor bit 6 set:** terminate with end-of-block or an invalid literal/length error.
4. **Otherwise:** follow a subtable entry.

For a length entry, the low four descriptor bits give the extra-bit count. If nonzero and not enough bits remain, one input byte is fetched. The length is the entry’s 16-bit base plus the selected low extra bits. Those extra bits are then consumed.

Before decoding the corresponding distance, fewer than 15 buffered bits trigger another two-byte refill.

For a subtable entry, the descriptor specifies the index width. The next entry is found in the same literal/length table at the entry value plus the accumulator’s low index bits. The index bits are not separately consumed during this navigation; the subsequently selected entry controls consumption. Subtable traversal has no independent refill or depth limit.

For an entry with bit 6 set:

- If bit 5 is also set, the function writes mode value **11** and exits.
- Otherwise, it reports **“invalid literal/length code”**, writes mode value **27**, and exits.

Bit 4 takes precedence over bit 6, so an entry carrying both is treated as a length.

### Distance decoding

The root distance index uses the accumulator’s low bits and the configured distance root width.

Each selected entry’s bit count is consumed before descriptor interpretation:

- If descriptor bit 4 is set, the entry supplies a distance.
- Otherwise, if bit 6 is set, the function reports **“invalid distance code”**, writes mode 27, and exits.
- Otherwise, the descriptor gives a subtable index width, and the entry value gives the offset within the distance table.

For a distance-producing entry, the low four descriptor bits specify the extra-bit count. The function fetches one byte if required, then a second if the first was insufficient. It consumes the extra bits and adds their unsigned value to the 16-bit base distance.

There is no separate check against a configured maximum distance and no explicit rejection of distance zero. Safe behavior for such cases depends on valid tables and caller state.

### Match source selection

Let the decoded length be **L**, distance be **D**, and available current-region output be prior output plus produced output.

If D does not exceed available current-region output, the match source is D bytes behind the next output position.

Otherwise, the function computes the history deficit:

**History deficit = D − available current-region output.**

If this deficit exceeds the valid history-byte count, it reports **“invalid distance too far back”**, writes mode 27, and exits without emitting any bytes of that match. Its length and distance bits have already been consumed.

When sufficient history exists, source selection depends on the circular-buffer write position:

- At write position zero, the source begins history-deficit bytes before the end of the history capacity.
- When the write position is smaller than the deficit, the source begins near the end of the history buffer and may wrap to its beginning.
- Otherwise, the source begins history-deficit bytes before the write position.

A match may use a tail segment of history, then a head segment, then bytes from current output. The function switches to current output once the required preceding history bytes have been exhausted.

Comparisons deciding whether to switch segments are strict. If a match ends exactly at a segment boundary, it completes there without taking the next-segment path.

### Match-copy ordering

Copies proceed forward, one byte at a time, even where processing is grouped into triples. Within each triple, each source byte is read and written before the next source byte is read.

Consequently, overlapping current-output matches can read bytes written earlier in the same match. For example, a distance-one match propagates the immediately preceding byte through the match.

The current-output-only path copies at least three bytes before testing the remainder. Valid length entries must therefore supply a length of at least three for that path. No short-length guard exists.

History prefixes are copied byte by byte. Remaining bytes are processed in triples and then a one- or two-byte tail.

### Continuation and exit

After completing a literal or an entire match, the function continues only when both margins hold:

- At least six input bytes remain relative to the current fetched-byte position.
- At least 258 output bytes remain.

It does not stop partway through a literal or match merely because a margin is crossed.

Every explicit exit—capacity margin, end-of-block, or recognized error—performs the same finalization:

1. Return all whole buffered bytes to the input range by backing up the local input cursor.
2. Reduce the buffered-bit count to its remainder modulo eight.
3. Write the resulting next-input pointer.
4. Write the resulting next-output pointer.
5. Write the remaining input count.
6. Write the remaining output count.
7. Save only the low residual bits of the accumulator.
8. Save the residual bit count.

Thus, the saved bit count is between zero and seven. Bytes fetched speculatively can remain logically unconsumed after finalization.

## Side effects

Output bytes are written immediately during decoding. Successful earlier output is retained if a later symbol fails; there is no rollback.

On a recognized error, the diagnostic pointer is written before mode 27. Final cursor, count, and bit-state updates follow those writes.

On end-of-block, mode 11 is written without replacing the diagnostic pointer. On a capacity-margin exit, neither the diagnostic pointer nor mode is written. Existing values can therefore remain unchanged and potentially stale.

No allocation, release, external call, or history-buffer write is visible. The function does not maintain a cumulative output counter or insert newly produced bytes into the circular history buffer.

Input and table reads can occur before any corresponding output is produced. Callers must not rely on arbitrary aliasing among input, output, tables, stream metadata, and state. Such aliasing can make writes affect subsequent reads.

The explicit final stream writes precede the saved bit-buffer writes. No atomicity, synchronization, or concurrent-access guarantee is established.

## Dependencies

No external function dependency appears in the supplied body. No additional opaque function IDs are available or required.

The function depends on caller-supplied data contracts:

- Stream-record layout.
- Decoding-state layout.
- Literal/length and distance table construction.
- Circular-history metadata.
- Output-baseline convention for the second argument.

It also selects three fixed diagnostic messages. Their storage ownership, lifetime guarantees, and mutability are not independently established. This function merely stores their addresses; it does not release or modify them.

## Acceptance examples

These examples are conditional static acceptance cases, not executed tests.

### Literal with a nonzero high value byte

Given a valid literal entry whose value is 0x0141, the emitted byte is 0x41. Only the value’s low eight bits are used.

If continuation margins still hold, decoding proceeds to another symbol. Otherwise, the common finalization occurs.

### Overlapping match in current output

Given a valid length of six, distance one, and a preceding output byte of 0x5A, six bytes of 0x5A are appended. Forward copy ordering permits later reads to observe earlier writes from the same match.

This example assumes the distance is classified as available within the current output region.

### History wrap followed by current output

Suppose the history capacity is eight, write position is two, history deficit is four, valid history is at least four bytes, and match length is six.

The first two bytes come from history positions six and seven. The next two come from positions zero and one. The final two come from current output at the decoded distance behind the advancing output position.

### History exactly sufficient

If the history deficit equals the valid history-byte count, the “too far back” check accepts it. All other pointer and metadata preconditions still apply.

If the deficit exceeds the valid count by one, the function reports “invalid distance too far back,” sets mode 27, emits no bytes for that match, and finalizes consumed input and residual bits.

### Terminal descriptor precedence

A literal/length entry with descriptor bits 6 and 5 set, but bit 4 clear, sets mode 11 without replacing the diagnostic pointer.

If bit 4 is also set, length processing takes precedence; it is not treated as end-of-block.

For a distance entry with bit 6 set and bit 4 clear, the result is “invalid distance code,” regardless of bit 5.

### Exact continuation margins

After a completed symbol, exactly six input bytes and 258 output bytes satisfy the continuation test.

Five input bytes or 257 output bytes cause exit after that completed symbol. These thresholds do not protect an initial call made with insufficient space.

### Returning speculative input

If an exit is reached with 19 buffered bits, the function returns two whole bytes to the available input range and retains three bits. The stored accumulator contains only those low three residual bits.

## Uncertainties

The following are **documented_only**:

- The surrounding decoder’s meaning for modes 11 and 27 beyond their observed use here.
- The complete external stream and state structures.
- The precise caller contract for the second argument.
- Table construction rules guaranteeing valid widths, lengths, distances, and finite subtable traversal.
- Target-specific behavior for out-of-range shifts, address overflow, and malformed count arithmetic.
- Portable byte order for externally constructed table records.
- Diagnostic-storage ownership and lifetime.
- Whether any caller intentionally permits particular memory aliases.

DEFLATE-style fast decoding is an inferred intent, not a replacement for the observed contract. In particular, conventional format limits must not be mistaken for checks performed by this function.

Malformed tables or inconsistent state can cause unchecked reads, writes, arithmetic underflow, or nontermination. The three diagnostic paths do not constitute comprehensive input validation.

## Evidence

Established static evidence includes the record offsets, four-byte table stride, descriptor precedence, low-bit decoding, refill sizes, history-source selection, forward overlapping-copy order, three diagnostic selections, mode writes, continuation thresholds, and common exit-state normalization.

The 32-bit interface model follows the supplied address and scalar representation. Arithmetic above is expressed without wrapping where caller invariants make that valid; malformed-state behavior is not assigned a portable interpretation.

No runtime tests have been performed. The acceptance examples are deductions from the supplied static evidence, not observed executions.
