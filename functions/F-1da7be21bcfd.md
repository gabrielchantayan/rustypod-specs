<!-- Trusted provenance: F-1da7be21bcfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1da7be21bcfd

## Purpose

F-1da7be21bcfd obtains six 32-bit values through an indirect callback, derives a common scale operand from the fourth value, conditionally transforms five values through F-73d099a86adb, and publishes a reordered result into a destination object.

It also writes a separate 16-bit result associated with an initial call to F-73d099a86adb. The supplied representation does not establish how that 16-bit result is produced.

The constants and shifts suggest fixed-point normalization, with 65,536 representing unity. That interpretation is inferred intent, not an established arithmetic contract: the implementation of F-73d099a86adb is unavailable.

## Interface

### Inputs

The function accepts two arguments represented as 32-bit integers and used as base addresses:

- **Destination base:** identifies writable storage receiving the results.
- **Provider base:** identifies an object containing an indirect callback address at byte offset 0x34.

The wrapper does not check either address for nullness, accessibility, alignment, object type, or sufficient storage.

The destination must support the specific writes listed below. The highest write covers offsets 0x1C4 through 0x1C7. This does not establish that every intervening byte is accessed or must belong to a particular declared structure.

The provider must support reading its callback slot at offset 0x34. The callback must be callable with the argument arrangement described below.

### Callback interface and temporary layout

The callback is invoked with four arguments, in this order:

1. The provider base.
2. The integer constant 6.
3. An address designating temporary result storage.
4. The integer constant 3.

The represented temporary result contains six consecutive 32-bit words. This contract names them **A**, **B**, **C**, **D**, **E**, and **F**, in increasing address order:

| Relative byte offset | Word |
|---|---|
| 0x00 | A |
| 0x04 | B |
| 0x08 | C |
| 0x0C | D |
| 0x10 | E |
| 0x14 | F |

These labels are descriptive positions, not source identifiers.

The wrapper does not initialize the six words before invoking the callback. For determinate behavior, the callback must supply every word that the subsequent execution reads. In particular, D is read to select the scale operand, and all six positions eventually contribute to the publication process unless a call fails to return normally.

The callback’s return value is not inspected.

### Outputs

The function has no explicit return value. Its directly visible outputs are writes relative to the destination base:

| Destination offset | Width | Published value |
|---|---:|---|
| 0x44 | 16 bits | Auxiliary result associated with the initial F-73d099a86adb call |
| 0x1B0 | 32 bits | Final A |
| 0x1B8 | 32 bits | Final B |
| 0x1B4 | 32 bits | Final C |
| 0x1BC | 32 bits | Final D |
| 0x1C0 | 32 bits | Final E arithmetically shifted right by 16 |
| 0x1C4 | 32 bits | Final F arithmetically shifted right by 16 |

The B write precedes the C write even though B’s destination address is higher.

No byte order is established by the supplied fragment. The table specifies access widths and relative addresses, not an external serialization format.

## Behavior

### Acquisition and scale selection

The first external operation is the indirect callback invocation through the provider’s slot at offset 0x34. No destination-field write is shown before that invocation.

After the callback returns, the wrapper reads D as a signed 32-bit integer and derives a scale operand:

- If D is nonnegative, the scale operand is D.
- If D is negative, the wrapper applies signed negation to obtain the intended magnitude.

For D values from −2,147,483,647 through 2,147,483,647, this describes the mathematical absolute value without a range ambiguity. D equal to −2,147,483,648 is a special uncertainty: its positive magnitude cannot be represented in a signed 32-bit integer. The supplied representation does not establish whether execution wraps, traps, or has another target-specific consequence.

The derived scale operand is selected before the initial helper call. Subsequent helper calls use that selected operand; the shown wrapper does not recompute it from D.

### Initial helper call and 16-bit publication

The wrapper always calls F-73d099a86adb once with these arguments:

1. 0x03E80000, equal to 65,536,000.
2. The selected scale operand.

This call occurs even when the scale operand is zero or exactly 65,536.

The ordinary return value from this call is not assigned to one of the six temporary words. Instead, the supplied representation associates a separate 16-bit value with the call and writes that value to destination offset 0x44 immediately afterward.

The provenance of this 16-bit value is not resolved. It must not be specified as the low half, high half, quotient, remainder, or truncation of the ordinary helper return without additional evidence.

### Conditional transformation

After the 16-bit write, the wrapper compares the selected scale operand with 0x00010000, equal to 65,536.

If the scale operand equals 65,536:

- There are no additional calls to F-73d099a86adb.
- The wrapper does not replace any of A through F during this phase.
- In particular, it does not force D to positive 65,536.

Consequently, in an execution without intervening mutation, both D = 65,536 and D = −65,536 skip transformation. The negative case retains the negative D value for publication.

If the scale operand differs from 65,536, the wrapper makes five additional calls to F-73d099a86adb. Their first arguments are the current values of A, B, C, E, and F, in that order. Every call receives the same selected scale operand as its second argument.

Each ordinary 32-bit helper result replaces its corresponding temporary word before the next helper call begins. After the call transforming F returns and its result is stored, the wrapper replaces D with positive 65,536.

No helper call transforms D itself.

These facts constrain call count and sequencing, but not the helper’s arithmetic. For example, the supplied evidence does not establish that F-73d099a86adb performs division, multiplication, saturation, rounding, or any particular fixed-point conversion.

### Final publication

After the conditional phase, the wrapper performs the six 32-bit destination writes in this order:

1. A to offset 0x1B0.
2. B to offset 0x1B8.
3. C to offset 0x1B4.
4. D to offset 0x1BC.
5. E shifted right by 16 to offset 0x1C0.
6. F shifted right by 16 to offset 0x1C4.

E and F are represented as signed 32-bit values. Their shifts therefore express arithmetic right shifts in the supplied representation: the sign is retained rather than replaced with zero bits. The shifted results are written as complete 32-bit values, not as 16-bit stores.

For a target implementing that arithmetic shift, shifting a negative value by 16 corresponds to rounding its division by 65,536 toward negative infinity. It is not generally equivalent to signed division that truncates toward zero.

The function then returns without an explicit result.

## Side effects

### Direct writes and visibility

The shown wrapper directly writes one 16-bit destination field, followed later by six 32-bit destination fields. These writes are not presented as an atomic transaction.

The 16-bit field is updated before any of the five conditional transformation calls. Thus, if a later helper call does not complete normally, that field may already have changed while the six final fields have not yet been written.

During final publication, a fault or interruption can leave a partially updated destination. There is no shown rollback, completion flag, or synchronization operation.

No clearing of adjacent fields or padding is shown.

### External effects

The callback and F-73d099a86adb can have effects not visible in this fragment. Their implementations are not supplied. This contract does not assume that they:

- Preserve globals or other reachable memory.
- Avoid reentrancy.
- Leave the provider or destination unchanged.
- Avoid retaining pointers.
- Allocate or release no memory.
- Return normally for every input.

The stated ordering describes the wrapper’s own operations. It does not assert that these are the only memory accesses performed during the invocation.

If external code mutates the temporary result storage through a retained or otherwise reachable reference, later reads can observe those mutations. In particular, selecting the scale operand does not freeze the remaining temporary words. Acceptance examples that rely on stable callback output explicitly exclude such interference.

### Ownership and lifetime

The wrapper shows no allocation or explicit release operation. It receives the destination and provider addresses from its caller and passes temporary result storage to the callback.

The temporary storage is valid for use during this invocation; no supported post-return lifetime is established. The callback is not shown receiving ownership of it. Whether either dependency retains references, acquires ownership of other objects, or releases memory remains unknown.

### Errors

There is no explicit error return, validation branch, or recovery path.

A callback failure is not recognized through a return status. Likewise, helper results are accepted without an error check. Inaccessible addresses, invalid callback targets, incompletely populated temporary storage, and dependency failures are outside any demonstrated handled-error contract.

A zero scale operand is forwarded to the helper. The wrapper does not substitute a default or reject the input.

## Dependencies

### F-73d099a86adb

This is the only statically identified direct dependency.

Its observed use has two 32-bit arguments and an ordinary 32-bit result used by the conditional transformation calls. The initial call also has an unresolved associated 16-bit result used for destination offset 0x44.

Observed invocation pattern:

- One unconditional call with first argument 0x03E80000.
- Five additional calls if, and only if, the selected scale operand is not 0x00010000.
- The five additional first arguments come from A, B, C, E, and F, in that order.
- All calls receive the same selected scale operand as their second argument.

Its arithmetic, signedness rules, exceptional cases, global effects, and auxiliary-result convention are documented_only unknowns.

### Indirect callback

No opaque function ID is supplied for the runtime target.

Its address comes from the provider at offset 0x34. Its observed argument sequence is provider base, 6, temporary result address, 3. The meanings of the two constants are unknown.

The wrapper relies on callback-populated data but performs no explicit success check. No stronger callback ownership or side-effect contract is established.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied static evidence. They are not reports of executed tests.

### Positive unity scale

Assume the callback supplies D = 65,536, all six words remain stable except for the wrapper’s own operations, and every dependency returns normally.

Expected behavior:

- Exactly one call to F-73d099a86adb, with arguments 65,536,000 and 65,536.
- The associated 16-bit result is written to offset 0x44.
- No transformation calls for A, B, C, E, or F.
- A, B, C, and positive 65,536 are published to their specified fields.
- E and F are published after signed right shifts by 16.

### Negative unity scale

Under the same stability assumptions, let D = −65,536.

Expected behavior:

- The selected scale operand is positive 65,536.
- Only the initial helper call occurs.
- The published D field contains −65,536, not positive 65,536.

This distinguishes the branch predicate from the value eventually stored in the D field.

### Non-unity magnitude

Assume D = −131,072 and no external interference.

Expected behavior:

- The selected scale operand is 131,072.
- Six helper calls occur in total.
- After the initial call, the remaining calls transform A, B, C, E, and F in that order.
- Each transformation receives 131,072 as its second argument.
- Final D is positive 65,536.
- Final A, B, and C are the corresponding ordinary helper results.
- The two remaining published fields are the arithmetic right shifts of the helper results for E and F.

Exact numeric transformation results cannot be required without the dependency contract.

### Zero scale operand

Assume D = 0.

The initial helper call receives zero as its second argument. If it returns normally, the wrapper writes the associated 16-bit field and enters the five-call transformation branch.

The wrapper provides no zero-denominator guard. Whether any call fails, returns a sentinel, or computes a value is unknown; acceptance must not invent one of those outcomes.

### Signed shift boundary

Assume D = 65,536, E = −1, F = 65,535, stable temporary storage, and arithmetic signed right shift.

Expected final writes:

- Offset 0x1C0 receives the 32-bit representation of −1.
- Offset 0x1C4 receives zero.

This checks signed extraction rather than unsigned shifting or truncation to a 16-bit field.

### Failure during transformation

Assume a non-unity scale operand and that a transformation call does not return normally.

The write at offset 0x44 has already occurred. The wrapper has not yet begun its final six-field publication sequence. No rollback by the wrapper is shown.

## Uncertainties

The following are documented_only:

- The arithmetic implemented by F-73d099a86adb.
- The origin and meaning of the auxiliary 16-bit result.
- The meaning of callback arguments 6 and 3.
- The semantic types and units of the six callback words.
- The exact outcome of negating the most negative signed 32-bit value.
- Dependency behavior for a zero scale operand.
- Callback and helper memory effects, reentrancy, pointer retention, and ownership effects.
- Calling-convention details beyond the argument and result uses shown.
- Byte order, alignment requirements, and complete object layouts.
- Thread-safety and concurrent-observer guarantees.

Fixed-point normalization is inferred intent only. It does not resolve any of these unknowns.

## Evidence

The supplied static representation establishes the callback slot offset and argument order, six-word temporary layout, signed scale-selection branch, helper-call constants and order, unity comparison, conditional replacement of D, destination access widths and offsets, final write order, and signed shift expressions.

It does not provide either dependency’s implementation or an authoritative explanation of the auxiliary 16-bit result.

No runtime tests, execution traces, or independent implementation checks have been performed. All acceptance examples above are conditional expectations from static evidence.
