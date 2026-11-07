<!-- Trusted provenance: F-be9b941ea6a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be9b941ea6a0

## Purpose

F-be9b941ea6a0 conditionally updates accounting associated with an encoded object reference and a context, then passes the object’s metadata to a dependency.

The established behavior consists of reference decoding, a metadata validity check, ordered 32-bit counter updates, and calls to four dependencies on the active path. The counter decrements and metadata handoff suggest an object-release or deallocation-accounting operation. That interpretation is inferred intent, not an established ownership contract: the available evidence does not establish that this function frees storage, transfers ownership, or makes the reference unusable.

The function has no explicit return value. Normal completion includes several no-op cases. One metadata condition invokes a dependency identified as non-returning.

## Interface

### Inputs and result

The function accepts two machine-width arguments:

- **Encoded reference:** a 32-bit unsigned value used both as an address and as input to reference decoding.
- **Context address:** a 32-bit value represented as an integer but used as the base address of a mutable structure.

There is no explicit result. Observable effects are memory accesses, counter writes, dependency calls, and possible non-returning termination.

The supplied representation uses 32-bit integers and addresses, 16-bit unsigned metadata fields, and an 8-bit context flag. Address calculations and counter arithmetic are described below at those widths. The evidence does not establish host-language portability, alignment guarantees, or the behavior of invalid addresses.

### Context layout

Offsets are byte offsets from the context address.

| Offset or address family | Access | Established role |
|---|---|---|
| `0x414` | Read one byte | Enables the active path when nonzero |
| `0x7c` | Address passed to dependencies | Base of an embedded accounting region |
| `0x2a0 + 4 × first metadata index` | Read/write 32 bits | First indexed amount counter |
| `0x388` | Read/write 32 bits | First fixed amount counter |
| `0x15c + 4 × second metadata index` | Read/write 32 bits | Second indexed amount counter |
| `0x29c` | Read/write 32 bits | Second fixed amount counter |
| `0x38c + 4 × dependency-selected index` | Read/write 32 bits | Indexed count |
| `0x40c` | Read/write 32 bits | Fixed count |
| `0x84` | Read/write 32 bits | Final amount counter |

The address families describe arithmetic, not proven array boundaries. In particular, the function does not establish that these locations are disjoint.

### Reference decoding and metadata layout

For a nonzero encoded reference on the active path:

1. A 32-bit displacement is read from the address four bytes before the **original encoded reference**.
2. Bit 31 of the encoded reference is cleared.
3. The displacement is subtracted from that masked reference to obtain the **decoded address**.

The displacement read does not use the masked reference. This distinction matters when bit 31 is set.

A zero decoded address ends processing without reading a metadata header or calling dependencies.

For a nonzero decoded address, the function uses the following layout:

| Location relative to decoded address | Width | Use |
|---|---:|---|
| Eight bytes before | 32 bits | Header word; validity checks and masked accounting amount |
| Four bytes before | 16 bits, unsigned | First metadata index |
| Two bytes before | 16 bits, unsigned | Second metadata index |

The accounting amount is the header word masked with `0x3ffffffc`. Bits 0–1 and 30–31 do not contribute to the amount. The resulting amount is between zero and `0x3ffffffc`, inclusive, and is divisible by four.

### Preconditions for a defined normal path

The function performs no general address or bounds validation. The caller must provide accessible storage for every location reached by the selected branch.

In particular:

- Unless both arguments are zero, the context flag address must be readable—even when the encoded reference is zero.
- A nonzero encoded reference with a nonzero context flag requires the displacement location to be readable.
- A nonzero decoded address requires readable metadata.
- Reached counter locations must support 32-bit reads and writes.
- Metadata-derived and dependency-derived indices must lead to valid storage.
- Dependencies must complete in a manner compatible with continued access to the context and metadata if normal completion is expected.

No ownership, exclusivity, or synchronization guarantee is established merely by satisfying these address requirements.

## Behavior

### 1. Initial gating

When both arguments are zero, the function returns immediately. It does not read the context flag.

For every other argument combination, it first reads the byte at context offset `0x414`.

- If that byte is zero, the function returns.
- If that byte is nonzero but the encoded reference is zero, the function returns.
- Only a nonzero flag together with a nonzero encoded reference enters reference decoding.

Consequently, a zero encoded reference is not an unconditional no-access case. With a nonzero context argument, the flag is still read. Likewise, a zero context with a nonzero reference is not rejected before the flag access.

### 2. Decode the reference

The function reads the displacement and calculates the decoded address as described in the interface.

If the decoded address is zero, it returns. There are no explicit counter writes or dependency calls on this branch, although the flag and displacement have already been read.

If the decoded address is nonzero, the function identifies the header eight bytes before it.

### 3. Check the header before calling accounting dependencies

The header is read and compared with zero.

- A zero header invokes F-a817d8d58c10.
- For a nonzero header, the header is read again and bit 0 is tested. If bit 0 is set, F-a817d8d58c10 is invoked.

F-a817d8d58c10 is identified as non-returning. No normal continuation after that call is represented.

The two checks are separate reads in the supplied evidence. If storage can change between them, the values need not agree. No lock, snapshot, or retry is established before these reads.

Passing this check establishes only that the first read was nonzero and the second read had bit 0 clear. It does not establish that the masked accounting amount is nonzero, that either index is in range, or that the header remains valid later.

### 4. Enter the dependency-mediated accounting phase

The function calls F-c0fc6d033caa with the address at context offset `0x7c`.

Its effect is unknown. It may plausibly be an acquisition or serialization operation, but that interpretation is not required by the observed behavior. The function does not explicitly capture its return value.

The first metadata index and accounting amount used next are read **after** this call. They are not frozen by the earlier header check.

### 5. Apply the first four amount updates

The function reads the unsigned 16-bit index four bytes before the decoded address and uses it to select the first indexed counter.

It then reads the header and retains the amount selected by `0x3ffffffc`. This retained amount is used for all four updates below, in order:

1. Subtract it from the first indexed amount counter.
2. Subtract it from the fixed counter at context offset `0x388`.
3. Read the unsigned 16-bit index two bytes before the decoded address, then subtract the retained amount from the second indexed amount counter.
4. Subtract it from the fixed counter at context offset `0x29c`.

Each update reads the counter’s current 32-bit value and writes the subtraction result. There is no check for an insufficient balance.

The second metadata index is read only after the first two counter writes. Therefore, the evidence does not support treating both indices and all counters as an immutable entry-time snapshot.

### 6. Apply count updates

The function calls F-09dcb55e8e0a without explicit arguments and uses its returned integer as an index.

It then performs these updates in order:

1. Decrement the 32-bit indexed count at context offset `0x38c + 4 × returned index` by one.
2. Decrement the 32-bit fixed count at context offset `0x40c` by one.

The return value is not range-checked. The function does not establish that it is nonnegative or bounded by an array size.

### 7. Apply the final amount update

The function reads the header again and masks it with `0x3ffffffc`. It subtracts this newly read amount from the 32-bit counter at context offset `0x84`.

This is a fresh header read, not reuse of the amount retained for the preceding four amount updates. F-09dcb55e8e0a and intervening memory writes occur between those amount reads. Accordingly, different amounts can be used within one invocation.

The available expression establishes that the counter value and masked header value are read before the resulting counter write. It does not establish a finer ordering between those two operand reads.

### 8. Hand off metadata and finish

After the explicit counter updates, the function calls:

1. F-cc6c9b4a0f68 with the embedded accounting-region address and the header address.
2. F-e586bff64fb7 with the embedded accounting-region address.

It then returns normally if both calls return.

There is no explicit error inspection, rollback, or compensating update around these calls.

### Arithmetic and aliasing

Counter operations are 32-bit updates. No saturation, underflow rejection, or overflow rejection is present. Under a wrapping 32-bit execution interpretation, decrementing zero by one stores `0xffffffff`; subtraction similarly retains the low 32 bits. The supplied representation alone does not establish portable language-level signed-overflow semantics.

The 16-bit indices range from 0 through 65,535. Multiplication by four produces offsets from 0 through 262,140 before addition to the context-derived base. The function does not establish that the context allocation covers that span.

If calculated addresses overlap, later operations read the state left by earlier writes. An implementation must not consolidate or reorder updates on an assumption that all counter locations are independent.

## Side effects

On the complete active path, the function explicitly performs:

- Five amount-counter writes.
- Two count-counter writes.
- Four returning dependency calls, assuming each dependency returns.

These are seven ordered counter updates, not necessarily seven distinct storage locations.

Before the fatal dependency is called, the function has performed metadata reads but no explicit counter writes. Dependencies can have additional effects not visible in this function.

The function contains no explicit allocation, direct memory-release operation, output generation, or assignment to a named global. This does not establish that dependencies lack those effects.

No ownership transfer is established for either argument or the metadata. In particular:

- Passing the header address to F-cc6c9b4a0f68 does not by itself prove release or transfer.
- F-c0fc6d033caa and F-e586bff64fb7 are not proven to form a lock/unlock pair.
- No dependency is known here to preserve globals, context fields, or metadata.
- No recovery is shown for inaccessible memory, dependency failure, or interruption after partial accounting updates.

## Dependencies

| Opaque ID | Invocation and established use | Unknowns |
|---|---|---|
| F-a817d8d58c10 | Called without explicit arguments when the header check fails; marked non-returning | Termination mechanism, diagnostics, cleanup, global effects |
| F-c0fc6d033caa | Called with the embedded accounting-region address before counter updates | Synchronization, mutation, failure behavior, ownership effects |
| F-09dcb55e8e0a | Called without explicit arguments; returned integer selects a count slot | Meaning, range, sign constraints, side effects |
| F-cc6c9b4a0f68 | Called with the embedded accounting-region address and header address after updates | Whether it releases, links, records, or otherwise changes metadata |
| F-e586bff64fb7 | Called with the embedded accounting-region address after F-cc6c9b4a0f68 | Synchronization, finalization, mutation, failure behavior |

The dependency ordering is established. Their semantic roles beyond argument use and the indicated non-returning behavior are documented_only.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Both arguments zero

Given both arguments equal zero, the function returns without reading the context flag or invoking a dependency.

### Disabled context

Given an accessible context whose flag byte is zero, the function returns after reading the flag. It does not read the reference displacement, even if the reference is nonzero.

### Zero reference with an enabled context

Given a nonzero, accessible context whose flag is nonzero and an encoded reference of zero, the flag is read and the function returns without decoding metadata.

### Decoding to zero

Given an enabled context and a nonzero reference whose masked value equals the displacement read from the original reference minus four, the decoded address is zero. The function returns without reading the header or calling accounting dependencies.

### Rejected header

Given a nonzero decoded address:

- If the initial header read is zero, F-a817d8d58c10 is invoked.
- If the initial read is nonzero and the subsequent read has bit 0 set, F-a817d8d58c10 is invoked.

Neither case reaches an explicit counter write.

### Stable accounting amount

Assume all relevant locations are valid and nonoverlapping, dependencies return without mutating them, both selected indexed counters are initially 1,000, the two fixed intermediate amount counters are initially 2,000, and the final amount counter is initially 3,000.

With a stable header of `0x42`, the accepted header produces an amount of 64. The indexed amount counters become 936, the fixed intermediate amount counters become 1,936, and the final amount counter becomes 2,936. Both selected count locations are decremented by one.

### Zero amount with valid flags

A stable header of `0x2` is nonzero and has bit 0 clear, but produces an accounting amount of zero. The function still performs the amount read/write operations, decrements both counts, and invokes the normal-path dependencies.

### Amount changes before the final update

Assume the header produces amount 64 after F-c0fc6d033caa, but produces amount 128 when reread after F-09dcb55e8e0a. The first four amount updates subtract 64; the final amount update subtracts 128. No consistency check or restart is performed.

## Uncertainties

The following remain documented_only:

- The semantic type of the encoded reference and the meaning of its high bit.
- Whether reference subtraction is intended to wrap and which addresses are valid.
- Metadata endianness, alignment requirements, and allocation boundaries.
- The meanings and valid ranges of the two metadata indices.
- The meaning and valid range of the dependency-selected index.
- Whether overlapping counter or metadata locations are valid inputs.
- Whether any dependency provides synchronization or modifies storage used later.
- Whether the metadata handoff releases memory or transfers ownership.
- The concrete behavior of the non-returning dependency.
- Failure behavior after a dependency changes state or after some counters have been written.
- Machine-level behavior beyond the supplied integer and address representation.

No unknown is resolved by assuming conventional allocator, thread-local accounting, or lock semantics.

## Evidence

This contract is based solely on the supplied static representation of F-be9b941ea6a0 and the supplied opaque dependency mapping.

Established static evidence includes branch conditions, access widths and offsets, masks, arithmetic operations, explicit write order, dependency arguments, and call order. Release-related purpose and synchronization-related interpretations are explicitly separated as inferred intent or unknowns.

No runtime tests, memory inspection, dependency execution, or behavioral experiments have been performed.
