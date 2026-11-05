<!-- Trusted provenance: F-4ff51b2fb22f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ff51b2fb22f

## Purpose

F-4ff51b2fb22f acquires a 64-bit value, selects one of two formatting modes, and transfers a fixed number of temporary bytes into a caller-provided record. A nonzero option reserves an initial temporary byte for a lowercase “d” prefix and increases the transferred character count by one.

The destination record has a two-byte header followed by two-byte character slots. Each slot contains one transferred byte followed by a zero byte. The first header byte records the total record size, including the header; the second header byte is 3.

This establishes a descriptor-like byte layout. Interpreting the record as a particular external descriptor type, or interpreting its payload as a hexadecimal identifier, is inferred intent rather than a fully established contract. The supplied evidence does not include the selected formatting strings, the formatter’s behavior, or the acquisition dependency’s implementation.

## Interface

### Inputs

The function accepts:

- **Destination:** a writable address at which the output record is constructed.
- **Option:** a 32-bit integer tested only for equality with zero. Every nonzero value selects the prefix-enabled behavior; its magnitude and sign otherwise have no effect.

There is no destination-capacity argument and no explicit null-pointer check.

For a valid, ordinary execution, the destination must permit all writes required by the selected mode:

| Selected mode | Option | Payload slots | Required destination bytes |
|---|---:|---:|---:|
| Compact | Zero | 12 | 26 |
| Compact | Nonzero | 13 | 28 |
| Full | Zero | 16 | 34 |
| Full | Nonzero | 17 | 36 |

A destination capacity of 36 bytes covers every normal branch of this function. That capacity does not establish the safety of either dependency.

### Result

The declared result is one character-sized value. After constructing the record, the function reads the destination’s first byte and returns it.

For a valid execution without external interference, that byte is one of:

- 26 for compact, prefix disabled;
- 28 for compact, prefix enabled;
- 34 for full, prefix disabled;
- 36 for full, prefix enabled.

All four values fit in an unsigned or signed 8-bit character representation, so character signedness does not change these normal results.

### Width model

The supplied representation supports this behavioral model:

- character storage units are 8 bits;
- the option and the halves supplied to the formatter are 32 bits;
- the acquired value and the intermediate shifted value are 64 bits;
- the temporary character region contains 20 bytes.

These widths describe the supplied implementation evidence, not a promise of portability to arbitrary language or machine models.

## Behavior

### Acquisition and initial temporary state

The first dependency invocation acquires a 64-bit value and receives no explicit arguments.

Only after that invocation returns does the function process the option for temporary-output placement:

- With a zero option, the formatter’s destination is the start of the 20-byte temporary region.
- With a nonzero option, the function writes a lowercase “d” to the temporary region’s first byte, then gives the formatter an address one byte later.

The final payload-copy operation always starts at the beginning of the temporary region. Thus, in the prefix-enabled case, the intended prefix occupies the first payload slot.

The prefix write occurs before the formatter invocation. Preservation of that byte by the formatter is not independently established.

### Mode selection

The acquired value is considered as an upper 32-bit half and a lower 32-bit half.

Compact mode is selected only when both conditions hold:

1. The upper half equals a separately stored 32-bit comparison value.
2. Bits 31 through 24 of the lower half equal hexadecimal 02.

Equivalently, the second condition requires the lower half, after masking with hexadecimal FF000000, to equal hexadecimal 02000000. Bits 23 through 0 do not affect that condition.

The comparison against the stored value is evaluated before the masked-lower-half condition. The stored comparison value is read after acquisition and after any prefix initialization, but before formatting.

If either condition fails, full mode is selected. A mismatch does not produce an error or an early return.

The value and stability of the stored comparison value are not supplied. In particular, the acquisition dependency must not be assumed to leave it unchanged.

### Formatter inputs

Exactly one formatter invocation is made after mode selection.

In **full mode**:

- The selected format is the full-mode format object.
- The first 32-bit value argument contains the acquired value’s upper half.
- The second 32-bit value argument contains its lower half.
- The base payload count is 16.

In **compact mode**:

- The selected format is the compact-mode format object.
- The first 32-bit value argument contains the original lower half masked to its low 24 bits.
- The second 32-bit value argument is zero.
- The base payload count is 12.

The compact transformation is width-sensitive. The masked 24-bit quantity is promoted into a 64-bit value before being shifted left by 32 bits. Extracting the resulting halves therefore yields the masked quantity as the upper half and zero as the lower half. It is not a 32-bit shift operation, and it does not preserve the original upper half.

Both mode paths supply two 32-bit value arguments to the formatter. The formatter may consume fewer arguments depending on the selected format; that cannot be determined from this evidence.

The function does not use a formatter result to determine the payload count, detect failure, or choose another path.

### Fixed payload count

After formatting returns, the function uses the selected base count:

- 12 in compact mode;
- 16 in full mode.

It adds one when the option is nonzero.

This count is independent of the number of bytes the formatter actually produced. The function does not search for a terminator and does not calculate a string length.

Consequences include:

- A zero byte within the copied range is copied like any other byte.
- Bytes beyond the fixed range are not transferred into the record.
- If the formatter leaves part of the copied range uninitialized, this function still attempts to read that range.
- If the formatter writes too much, the fixed copy count does not prevent the preceding temporary-buffer overrun.

The format contents and formatter behavior are therefore prerequisites for a stronger claim about payload text or memory safety.

### Destination layout and write order

Let the payload count be \(N\). Its possible values are 12, 13, 16, and 17.

After the formatter returns, the function performs its own destination writes in this order:

1. At byte offset 0, write \(2N + 2\).
2. At byte offset 1, write 3.
3. For each payload slot, in increasing slot order:
   - read the corresponding byte from the beginning of the temporary region;
   - write that byte to destination offset \(2 + 2j\);
   - write zero to destination offset \(3 + 2j\).

Here \(j\) ranges from zero through \(N-1\).

The resulting layout is:

| Destination offset | Meaning |
|---|---|
| 0 | Total record size, \(2N + 2\) |
| 1 | Fixed value 3 |
| 2, 4, 6, … | Successive temporary payload bytes |
| 3, 5, 7, … | Zero bytes paired with the payload bytes |

The last destination byte written is at offset \(2N + 1\). No additional terminator slot is written after the payload. The zero paired with the final payload byte is part of that final slot, not a separate string terminator.

The multiplication and addition used for the header cannot overflow for any reachable payload count: the maximum result is 36.

### Completion and failures

After the final slot is written, the function reads and returns the first destination byte.

There is no explicit validation, error return, rollback, retry, or recovery branch. If a dependency does not return, record construction does not proceed past that invocation.

Invalid destination storage or invalid temporary-buffer behavior does not have a specified recoverable result. If execution is interrupted during the destination writes, the destination may contain a partially updated record.

## Side effects

### Effects directly established for this function

The function:

- invokes the acquisition dependency once;
- may write a prefix byte into temporary storage;
- reads one stored comparison value during mode selection;
- invokes F-ce41e65501f5 once;
- writes exactly 26, 28, 34, or 36 destination bytes in a valid normal execution;
- reads the first destination byte for its result.

Its own destination writes begin only after both dependency invocations have returned.

This timing is not a guarantee that the destination remains unchanged during those calls. Either dependency could affect it indirectly through state or addresses not shown in the supplied evidence.

### Storage and ownership

The 20-byte temporary region is invocation-local storage. Its bytes are copied into the destination; the function does not intentionally place a pointer to that region in the output record.

No allocation, deallocation, ownership transfer, or retention of the destination is directly expressed by this function’s own operations.

The formatter receives an address into temporary storage. Whether it retains that address, allocates memory, releases memory, or modifies unrelated state is unknown. Such behavior must not be inferred from its apparent formatting role.

The caller remains responsible for supplying valid destination storage. No capacity or ownership metadata accompanies that pointer.

### Concurrency and aliasing

The function expresses no synchronization around the stored comparison value or destination.

The comparison value is consulted before this function’s own destination writes. If the destination overlaps that stored value, the subsequent writes may change state used by later invocations; no such overlap is explicitly rejected.

Concurrent modification of the destination or comparison value has no established safety guarantee.

## Dependencies

### Acquisition dependency — opaque ID not supplied

The supplied dependency map does not assign an opaque ID to the acquisition operation.

Its established boundary is limited to:

- one invocation;
- no explicit arguments;
- a 64-bit result consumed by this function.

Its meaning, failure behavior, global-state effects, ownership behavior, and repeatability are **documented_only** unknowns.

### F-ce41e65501f5

The established call boundary consists of:

- a destination address within the 20-byte temporary region;
- one of two selected format objects;
- two 32-bit value arguments.

The address is the temporary region’s start with the option disabled, or one byte later with the option enabled.

The dependency’s return value is not used. Its formatting rules, output length, terminator behavior, bounds behavior, and side effects are **documented_only** unknowns.

### Read-only inputs to the decision and formatting process

The function relies on:

- a stored 32-bit comparison value;
- a compact-mode format object;
- a full-mode format object.

Their actual contents are not provided. The apparent naming of an external object is not sufficient evidence of its contents.

## Acceptance examples

These examples are conditional static acceptance cases, not reports of execution. Unless noted otherwise, assume valid destination storage, returning dependencies, no external interference, and formatter behavior that initializes the copied temporary bytes without exceeding the temporary region.

### Compact mode without prefix

Suppose the acquired upper half equals the stored comparison value, and the lower half is hexadecimal 02ABCDEF. The option is zero.

Required behavior:

- Select compact mode.
- Supply hexadecimal 00ABCDEF and zero as the two formatter value arguments.
- Give the formatter the start of the temporary region.
- Transfer exactly 12 temporary bytes into 12 paired slots.
- Write header bytes 26 and 3.
- Return 26.

No particular twelve-character textual value is established without the compact format contents and formatter contract.

### Compact mode with any nonzero option

Use the same acquired value, but let the option be negative one.

Required behavior:

- Select the same compact mode.
- Write the prefix before formatting.
- Give the formatter the address immediately after that prefix.
- Transfer 13 bytes beginning at the temporary region’s start.
- Write header bytes 28 and 3.
- Return 28.

If the formatter preserves the preceding byte, the first payload slot contains lowercase “d” followed by zero. Option values of one or any other nonzero 32-bit value select the same behavior.

### Full mode because the upper half differs

Suppose the upper half differs from the stored comparison value, even though the lower half begins with hexadecimal 02.

Required behavior:

- Select full mode.
- Pass the original upper and lower halves as the formatter value arguments.
- With a zero option, transfer 16 bytes, write header bytes 34 and 3, and return 34.

### Full mode because the masked lower half differs

Suppose the upper half matches, but bits 31 through 24 of the lower half equal hexadecimal 03.

Required behavior:

- Select full mode.
- With a nonzero option, transfer 17 bytes, write header bytes 36 and 3, and return 36.

### Embedded zero in temporary output

Suppose the formatter initializes a copied temporary position to zero.

Required behavior:

- Transfer that zero into its payload-byte position.
- Write the paired zero byte as usual.
- Continue transferring subsequent positions through the fixed count.

The embedded zero must not shorten record construction.

## Uncertainties

The following remain **documented_only**:

- The acquisition operation’s opaque ID, semantics, and side effects.
- The stored comparison value’s actual value and initialization.
- Both format objects’ actual contents.
- Whether either format denotes hexadecimal text, padding, or a minimum width.
- Whether F-ce41e65501f5 initializes every byte subsequently copied.
- Whether its output fits the available 20-byte region, or the 19-byte suffix used with the prefix enabled.
- Whether it writes a terminator and whether that terminator is inside or outside the copied range.
- Whether either dependency modifies global state, retains pointers, or owns or releases memory.
- The intended external interpretation of the two-byte character slots and header value 3.
- Fault, interruption, and concurrent-access behavior.

The record layout is established independently of these unknowns. A guarantee of a specific printable identifier is not.

## Evidence

This contract is based solely on static examination of the supplied function representation and the supplied opaque dependency mapping.

Established evidence covers the branch conditions, integer transformations, explicit dependency arguments, fixed payload counts, destination layout, write order, and returned byte.

Descriptor-like encoding, textual identifier intent, and prefix preservation beyond the explicit prefix write are interpretations or conditional properties, not independently verified dependency behavior.

No runtime tests have been performed.
