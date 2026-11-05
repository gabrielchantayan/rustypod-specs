<!-- Trusted provenance: F-b5b672be5c1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5b672be5c1a

## Purpose

F-b5b672be5c1a performs table-driven positional accumulation from a byte sequence and optionally reports the address of the first byte rejected by the selected membership table.

Established behavior includes:

- Selecting an accumulation multiplier of 8, 16, or 10.
- Recognizing an initial `0x` or `0X` byte pair and overriding the multiplier to 16.
- Accumulating values obtained from a separate byte-indexed value table.
- Returning zero immediately for a null input address or an initially zero input byte.
- Leaving the optional output location unchanged on either immediate-return path.

The apparent intent is integer parsing with octal, hexadecimal, and decimal modes. That interpretation is inferred from the multipliers and prefix recognition. The supplied evidence does not include the tables, so conventional digit membership and digit values are not established.

## Interface

The function has four inputs and one return value.

| Input | Representation | Contract |
|---|---|---|
| Input address | Pointer to bytes | May be null. Otherwise, the first byte must be readable. Subsequent readability requirements depend on prefix recognition and membership-table results. |
| Stop-address destination | Optional pointer to a four-byte output slot | May be null. On completion of the scanning path, receives the address of the first rejected byte. Not written on either immediate-return path. |
| Requested mode | Signed 32-bit integer | Exactly 8 selects multiplier 8; exactly 16 selects multiplier 16; every other value selects multiplier 10. An initial hexadecimal prefix overrides this selection. |
| Auxiliary input | Unsigned 32-bit integer | Has no effect on observable behavior. With nonnull input, its working value is replaced by the first input byte. With null input, the function returns without using it. |

The return value is represented as a signed 32-bit integer. Before any accumulation, its value is zero.

The output slot is four bytes wide in the supplied representation and stores an input address. This is consistent with a 32-bit pointer environment; it is not evidence of a portable interface for wider pointers.

### Required storage

For a defined execution:

- Every input byte actually inspected must be readable.
- Each selected membership-table entry must be readable.
- Every value-table entry used for an accepted byte must be readable.
- A nonnull stop-address destination must be writable for four bytes when the scanning path finishes.
- The scan must eventually encounter a rejected byte if normal termination is required.

The interface does not provide a byte count. A zero byte is not an unconditional scan terminator: after the initial-byte check, termination is controlled by the selected membership table.

## Behavior

### 1. Immediate-return cases

If the input address is null, the function returns zero.

In this case, it does not:

- Read input storage.
- Consult membership-table or value-table reference storage.
- Write the stop-address destination.

The auxiliary input does not turn a null input into a usable input.

If the input address is nonnull, the function first reads its initial byte as an unsigned eight-bit value. If that byte is zero, the function returns zero without writing the stop-address destination or consulting the tables.

This creates an important distinction: an initially zero byte causes an early return, whereas a zero byte encountered later is processed according to membership-table contents.

### 2. Mode and membership-table selection

For the remaining behavior, use these descriptive data labels:

- **T8:** membership table associated with multiplier 8.
- **T16:** membership table associated with multiplier 16.
- **T10:** membership table associated with multiplier 10.
- **V:** byte-to-value table.

These labels describe roles, not verified table contents.

The function first reads the stored reference for T8.

If the requested mode is not 8, it also reads the stored reference for T16. If the requested mode is neither 8 nor 16, it sets the working multiplier to 10 and reads the stored reference for T10.

Consequently:

| Requested mode | Initial working multiplier | Initial selected table |
|---|---:|---|
| 8 | 8 | T8 |
| 16 | 16 | T16 |
| Any other signed 32-bit value | 10 | T10 |

The preliminary reference reads occur even when a later selection supersedes them. For example, an unsupported mode leads to reference reads for T8, T16, and T10, in that order.

Unsupported mode values are not reported as errors. They select the decimal-associated path.

### 3. Initial hexadecimal-prefix handling

Prefix recognition is attempted only when the initially read byte equals `0x30`.

The function then examines the byte at input offset one:

- A value of `0x78` recognizes the lowercase prefix.
- Otherwise, an additional access to that position checks for `0x58`, recognizing the uppercase prefix.

The supplied expression contains two separate accesses to the second byte when the first comparison fails. This is a source-level access distinction, not a guarantee about the number of physical memory transactions after compilation.

When either prefix is recognized:

- The working multiplier becomes 16.
- The scan address advances by two bytes.
- The stored reference for T16 is read and becomes the selected membership-table reference.

Prefix recognition applies regardless of the requested mode. It does not require a following accepted byte.

If the initial byte is `0x30` but the next byte is neither recognized prefix marker, the scan address remains at the original input address. A lone leading zero does not itself change the selected mode.

No other prefix handling, whitespace skipping, or sign handling is explicit in the supplied evidence.

### 4. Membership testing

The accumulator starts at zero.

At each scan position, the function reads one unsigned byte. For a byte value \(b\):

- The membership-table byte index is \(\lfloor b/8 \rfloor\).
- The tested bit is \(b \bmod 8\).
- A set bit accepts the byte.
- A clear bit terminates scanning.

Because \(b\) is eight bits wide, the membership-table index ranges from 0 through 31, and the tested bit ranges from 0 through 7. A table capable of classifying every possible byte therefore requires 32 readable bytes.

The mask calculation uses bit positions no greater than seven. It does not introduce a shift-by-width edge case.

The selected table reference is retained for scanning. Its stored reference is not explicitly reloaded on every iteration, although the selected table’s contents are consulted for each byte.

### 5. Accumulation

For each accepted byte, the function:

1. Advances the scan address by one byte.
2. Obtains the byte’s unsigned eight-bit contribution from V, indexed by the accepted byte value.
3. Updates the accumulator using the working multiplier and that contribution.

The numerical relationship is:

\[
A_{\text{next}} = rA_{\text{current}} + V[b]
\]

Here, \(r\) is 8, 16, or 10, and \(V[b]\) is between 0 and 255.

The stored reference for V is consulted for each accepted byte. Unlike the selected membership-table reference, it is not explicitly captured once before the loop.

The membership table alone determines acceptance. There is no additional check that the contribution is less than the multiplier. An accepted byte can therefore contribute a value greater than or equal to the multiplier if the tables contain such a mapping.

The source establishes that the scan address advances before the accumulation statement. It does not establish a complete machine-level ordering among the arithmetic and the reference and entry reads within that statement.

### 6. Termination and output

When a byte fails membership testing, scanning stops without advancing past that byte.

If the stop-address destination is nonnull, the function writes the current scan address into its four-byte slot. It then returns the accumulator.

This output write occurs even when no bytes were accepted on the scanning path. Thus:

- An initially zero input leaves the destination unchanged.
- A nonzero initial byte rejected by the selected table causes the destination to receive the original input address.
- A recognized prefix followed immediately by a rejected byte causes the destination to receive the address two bytes after the original input.

There is no separate success flag and no explicit error code.

## Side effects

The only explicit write is the optional four-byte stop-address write after scanning terminates.

There are no explicit writes to:

- Input bytes.
- Membership-table contents.
- Value-table contents.
- Stored table references.

The function does not allocate or release storage, and no ownership transfer is expressed. Input and table storage are used as borrowed readable storage; the output slot is caller-provided writable storage.

No disjointness check is performed. If the output slot overlaps input or table storage, its final write can modify that storage. The write occurs after the last membership test, so it does not change earlier parsing decisions within the same ordinary execution.

No explicit external calls appear. There is consequently no basis for attributing hidden allocation, deallocation, callbacks, or global preservation obligations to a callee.

Concurrent modification, memory-mapped I/O, and asynchronous mutation of input or tables are not given a synchronization contract.

## Dependencies

F-b5b672be5c1a has no explicit function dependencies in the supplied body.

Its data dependencies are:

| Dependency | Use | Minimum accessible region for unrestricted byte indices |
|---|---|---:|
| Stored reference for T8 | Read during initial mode selection | One stored reference |
| Stored reference for T16 | Read during applicable mode selection and again upon recognized prefix | One stored reference |
| Stored reference for T10 | Read for requested modes other than 8 and 16 | One stored reference |
| Selected membership table | Determines byte acceptance | 32 bytes |
| Stored reference for V | Consulted for every accepted byte | One stored reference |
| Value table V | Supplies an unsigned contribution indexed by input byte | 256 bytes |

These are data-role labels, not additional function IDs.

Only entries reached by a particular execution must actually be accessible for that execution. The full table extents above describe support for all possible eight-bit indices.

## Acceptance examples

The examples below are conditional behavioral criteria derived from the supplied static evidence. No runtime tests have been performed.

### Null input

Given a null input address, any requested mode, any auxiliary input, and a stop-address slot initially containing a sentinel:

- Return value: zero.
- Stop-address slot: unchanged.
- Input and table contents: not read.

### Initially zero byte

Given readable input beginning with `0x00` and a stop-address slot containing a sentinel:

- Return value: zero.
- Stop-address slot: unchanged.
- No mode-table selection occurs.

This remains true even if a membership table would otherwise accept zero.

### First byte rejected

Given a nonzero first byte that the selected table rejects, with no recognized initial prefix:

- Return value: zero.
- Stop address: original input address.
- V is not consulted.

### Conditional decimal accumulation

Assume T10 accepts the bytes for `1` and `2`, rejects the following `!`, and V assigns contributions 1 and 2 to those accepted bytes.

For input `12!` and requested mode 10:

- Return value: 12.
- Stop address: the `!` byte.

The same result follows for any requested mode other than 8 and 16, under the same table assumptions.

### Prefix overrides octal mode

Assume T16 accepts `2` and `A`, V maps them to 2 and 10, and T16 rejects the following `!`.

For input `0x2A!` and requested mode 8:

- The prefix selects multiplier 16 and T16.
- Return value: 42.
- Stop address: the `!` byte.

The prefix bytes do not contribute to accumulation.

### Prefix without accepted content

Assume the byte immediately after `0X` is rejected by T16.

For input beginning with that prefix:

- Return value: zero.
- Stop address: input address plus two.
- No V entry is read.

### Leading zero without prefix

Assume T10 accepts `0` and `9`, maps them to 0 and 9, and rejects the next byte.

For input `09!` and requested mode 10:

- No octal-mode switch occurs.
- Return value: 9.
- Stop address: the `!` byte.

### Table-driven nonstandard contribution

Assume the selected table accepts a byte whose V contribution is 200, then rejects the next byte.

With an initially zero accumulator:

- Return value: 200.
- No range check rejects the contribution, even with multiplier 8.

### Zero encountered after the start

Assume the selected table accepts zero and V maps it to zero.

A zero byte encountered during scanning does not itself stop the function. Scanning proceeds to the next byte. The caller must therefore provide accessible storage beyond it until a rejected byte is reached.

## Uncertainties

### documented_only: Table contents

Actual membership bits and value mappings are absent. Conventional decimal, octal, and hexadecimal character sets are inferred intent, not established behavior. Treatment of signs, spaces, extended bytes, and zero during scanning remains table-dependent.

### documented_only: Arithmetic overflow

The accumulator and return representation are signed 32-bit integers. No explicit overflow detection, saturation, or error handling appears.

For values within the signed 32-bit range, the accumulation relationship is established. Beyond that range, the supplied source does not establish a portable signed-arithmetic result. Low-32-bit machine wrapping is a possible target behavior, but it is not promoted here to a verified contract.

### documented_only: Invalid storage and unterminated scans

There is no explicit handling for inaccessible input, invalid table references, unwritable output, address-space exhaustion, or a sequence with no rejected byte. No return value is promised for those cases.

### documented_only: Pointer portability and concurrency

The four-byte address output is established by the supplied representation. Behavior on a platform requiring wider address storage is not established.

No synchronization or stability guarantee is supplied for input, table references, or table contents.

## Evidence

This contract is based solely on the supplied function body and its displayed type widths.

Established static evidence covers the early-return conditions, table-reference selection, prefix override, membership-bit calculation, byte-indexed contributions, accumulator update, optional output write, and absence of explicit calls or allocation.

Numeric-parser intent and conventional character mappings are identified as inference. Table contents, overflow results, invalid-memory outcomes, and concurrent-mutation behavior remain documented uncertainties.

No implementation was executed, and no runtime tests have been performed.
