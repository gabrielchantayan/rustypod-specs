<!-- Trusted provenance: F-0b16aecfaf7f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b16aecfaf7f

## Purpose

F-0b16aecfaf7f dispatches a comparison-like operation over caller-supplied operands and writes a one-byte result. A packed selector chooses the operation, selects one of two modes passed to dependencies, and optionally complements the final result.

The function has two distinct outputs:

- A 32-bit status returned directly.
- A byte written through a caller-provided result address.

A zero status does not imply that the result byte is true. In particular, a first operand shorter than the requested prefix, suffix, or specialized-match length produces a zero status without invoking a comparison dependency.

The operation names used below describe the apparent role of each branch. Actual data interpretation, ordering, and matching semantics depend on opaque dependencies and are not established here.

## Interface

The interface has eight arguments, in order:

| Position | Contract role | Width and interpretation |
|---|---|---|
| 1 | Packed selector | Unsigned 32-bit value. Its low 24 bits select an operation; two higher bits control dependency mode and result complementation. |
| 2 | First operand address | A 32-bit address-like value. Zero is rejected. |
| 3 | First operand extent | Unsigned 32-bit value. Used as an unsigned length in local guards and passed to dependencies. |
| 4 | Second operand address | A 32-bit address-like value. Zero is rejected for every operation. |
| 5 | Second operand extent | Unsigned 32-bit value. Used in local guards, suffix-address calculation, and dependency calls. |
| 6 | Additional operand address or raw value | Uninterpreted 32-bit argument locally. Passed to a dependency only for the two interval operations. |
| 7 | Additional operand extent or raw value | Uninterpreted 32-bit argument locally. Passed to a dependency only for the two interval operations. |
| 8 | Result address | Address of a writable byte. A null address is rejected. |

The return value is a 32-bit status:

- `0x00000000`: normal local completion, including a false result or a guarded length mismatch.
- `0xFFFFFFCE`: rejected required argument or unsupported operation. If interpreted as signed 32-bit two’s complement, this is −50.

No other locally generated return value is shown. Dependency results are used to determine the result byte, not propagated as status values.

### Selector layout

| Bits | Meaning |
|---|---|
| 0–23 | Operation identifier |
| 24, mask `0x01000000` | Select dependency mode `0x19` instead of `0x18` |
| 25, mask `0x02000000` | Apply the final byte-complement transformation |
| 26–31 | No local effect |

The two mode values are passed as 32-bit arguments. Their semantic meaning is unknown.

### Caller obligations

After the explicit null checks, the function assumes that required memory accesses and dependency calls are valid. It performs no address-range, allocation-size, alignment, or lifetime validation.

The second operand address must be nonzero even when the selected operation will not use it because of a length guard. The additional operand is not locally checked, including for interval operations.

No ownership transfer, allocation, or release is established by this function. Dependency ownership behavior is unknown.

## Behavior

### Validation and initialization

The first observable decision checks whether the first operand address, second operand address, or result address is zero.

If any is zero:

- Return `0xFFFFFFCE`.
- Do not initialize or complement the result byte.
- Do not invoke either dependency.

Consequently, if the result address is valid but another required address is zero, its previous byte remains untouched by this function.

Otherwise, the function selects the dependency mode, extracts the low-24-bit operation identifier, and writes zero to the result byte before invoking any dependency.

This initial write is observable. If the result byte aliases operand storage, dependencies can therefore observe the modified storage rather than its original contents.

### Dependency result conventions

For the operations using F-a9e7346d7015, its return value is interpreted as a signed 32-bit integer:

- Negative: less-than result.
- Zero: equality result.
- Positive: greater-than result.

These labels describe how this function consumes the returned sign; they do not establish that the dependency implements any particular ordering.

For the operation using F-f87605217bc7, a byte-valued result is retained directly. This function does not normalize that result to zero or one before the optional complement.

### Operation table

The following results are the bytes selected before optional complementation.

| Operation | Behavior |
|---|---|
| `0x000001` | Compare the complete first and second operands using F-a9e7346d7015. Select 1 exactly when its result is zero. |
| `0x000002` | If the first extent is at least the second extent, call F-f87605217bc7 with both original operand addresses and extents, then retain its returned byte. Otherwise skip the call and the normal result write. |
| `0x000004` | If the first extent is at least the second extent, compare from the original first address, using the second extent for both operands. Select 1 exactly when the dependency result is zero. Otherwise skip the call and the normal result write. |
| `0x000008` | If the first extent is at least the second extent, adjust the first address to its apparent two-byte-unit suffix and compare using the second extent for both operands. Select 1 exactly when the dependency result is zero. Otherwise skip the call and the normal result write. |
| `0x000010` | Compare the complete operands. Select 1 exactly when the signed result is greater than zero. |
| `0x000020` | Compare the complete operands. Under the signed-byte and arithmetic-right-shift interpretation of the displayed expression, select 1 for a nonnegative result and 0 for a negative result. |
| `0x000040` | Compare the complete operands. Select 1 exactly when the 32-bit result’s sign bit is set. |
| `0x000080` | Compare the complete operands. Select 1 exactly when the signed result is less than or equal to zero. |
| `0x000100` | Perform both interval comparisons, in order. Select 1 when the first result is nonnegative and the second result is nonpositive. |
| `0x000200` | Perform both interval comparisons, in order. Select 1 when the first result is strictly positive and the second result is strictly negative. |
| Any other low-24-bit value | Perform one complete-operand comparison, discard its result for predicate purposes, and set the status to `0xFFFFFFCE`. Skip the normal result write. |

A guarded length mismatch for operations `0x000002`, `0x000004`, and `0x000008` leaves the initialized zero byte in place before complementation. It is not an error status.

### Complete-operand comparison

A complete-operand comparison invokes F-a9e7346d7015 with, in order:

1. Original first operand address.
2. Original first operand extent.
3. Original second operand address.
4. Original second operand extent.
5. Selected mode.

Unsupported operations also make this call. Operation recognition therefore does not prevent dependency side effects or dependency failures on that path.

### Prefix-like and suffix-like comparisons

For operation `0x000004`, the first operand’s declared extent is used only for the unsigned length guard. The dependency receives the second extent as the extent of both operands.

For operation `0x000008`, the first address is advanced by twice the difference between the first and second extents. The dependency again receives the second extent for both operands.

This arithmetic establishes a two-byte stride for suffix positioning. It does not independently establish text encoding, element validity, alignment, or the dependencies’ unit conventions.

The address calculation uses 32-bit quantities. There is no wider overflow check. A caller requiring an ordinary in-bounds suffix must ensure that the scaled offset and resulting address are representable and refer to accessible storage. Exact machine behavior when those arithmetic assumptions fail is not independently verified.

The length guards use unsigned comparison. Extents with the high bit set are therefore large positive extents for the guard, not negative lengths.

### Interval comparisons

Both interval operations invoke F-a9e7346d7015 twice:

1. Compare the first operand and its original extent against the second operand and its extent.
2. Compare the same first operand and extent against the additional operand arguments.

Both calls occur before the final predicate is selected. There is no short-circuit omission of the second call based on the first result.

The second call can observe memory or global-state changes caused by the first call. The contract does not assume a stable snapshot of operand contents.

The apparent interval interpretations are:

- `0x000100`: inclusive at both boundaries.
- `0x000200`: exclusive at both boundaries.

Those interpretations are conditional on the dependency’s sign convention representing the intended ordering. The function does not validate boundary order.

### Optional complement

Except for the initial argument-rejection path, selector bit 25 causes a final read and rewrite of the result byte.

For the byte actually stored at that time:

| Byte read | Byte written |
|---|---|
| 0 | 1 |
| 1 | 0 |
| 2–255 | 0 |

This is not a general bitwise complement. It maps zero to one and every nonzero byte to zero.

The complement applies even after an unsupported operation has selected error status. It also applies after a guarded length mismatch.

Recognized operations that select a result normally overwrite the result byte after dependency calls. Unsupported operations do not perform that overwrite. If a dependency can modify the result byte through aliasing or other accessible state, its modification can therefore survive an unsupported operation, or become the input to complementation.

## Side effects

Locally established side effects are:

- An initial one-byte zero write after successful argument validation.
- Dependency calls in the order described above.
- A selected-result byte write for recognized operations that do not take a length-mismatch exit.
- A final result-byte read and write when complementation is enabled.

No local allocation, deallocation, or direct global-state write is visible.

The function does not itself read operand contents. Such reads, and any operand writes, are delegated. Their sizes, ordering, alignment requirements, and effects are unknown.

Operand and output overlap is not rejected. Because output initialization precedes dependency calls, overlap can change comparison inputs. Neither dependencies’ preservation of globals nor their preservation or release of caller memory may be assumed.

Invalid nonnull addresses, insufficient storage, and dependency failures have no specified local recovery path. No exception translation or rollback is shown.

## Dependencies

### F-a9e7346d7015

Called with five 32-bit arguments: two operand/extent pairs and the selected mode.

Its signed 32-bit return is consumed through equality or sign tests. It is called:

- Once for ordinary comparisons, prefix-like comparisons, suffix-like comparisons, and unsupported operations that pass initial validation.
- Twice for interval operations.
- Not at all for the specialized-match operation or a guarded length mismatch.

Its data interpretation, mode semantics, error behavior, ownership rules, and state effects are documented_only unknowns. Any returned negative value is treated as a comparison result; this function has no separate dependency-error check.

### F-f87605217bc7

Called only for operation `0x000002` when the unsigned length guard passes.

It receives the original operand/extent pairs and selected mode. Its returned byte becomes the selected result without local Boolean normalization.

Its matching semantics, return-value range, ownership rules, and state effects are documented_only unknowns.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except where aliasing is explicitly discussed, assume valid disjoint storage and dependencies that do not modify the result byte.

1. **Rejected required address:** With a zero first operand address and an otherwise writable result byte initially equal to 7, return `0xFFFFFFCE` and leave the byte equal to 7. Enabling complementation does not change this outcome.

2. **Prefix length mismatch:** For operation `0x000004`, first extent 3 and second extent 4 produce no dependency call, status zero, and result zero. With complementation enabled, the result is one.

3. **Suffix argument positioning:** For operation `0x000008`, first extent 9 and second extent 4 advance the first address by 10 bytes. The comparison receives extent 4 for both operands. A zero dependency result selects one.

4. **Ordinary equality:** For operation `0x000001`, any nonzero dependency result selects zero; an exact zero selects one.

5. **Signed comparisons:** If the complete comparison returns −7, operation `0x000040` selects one, operation `0x000010` selects zero, and operation `0x000080` selects one.

6. **Inclusive interval boundary:** For operation `0x000100`, dependency results of zero and −3 select one. Both calls must occur. Results of zero and positive 3 select zero.

7. **Exclusive interval boundary:** For operation `0x000200`, results of positive 2 and negative 2 select one. Replacing either result with zero selects zero.

8. **Specialized non-Boolean result:** If F-f87605217bc7 returns byte 5, operation `0x000002` stores 5 without complementation and stores zero with complementation.

9. **Unsupported operation:** An unsupported low-24-bit identifier still causes one complete comparison, then returns `0xFFFFFFCE`. With no dependency mutation of the output, the result is zero without complementation and one with complementation.

10. **Unused additional arguments:** For an ordinary equality operation, changing arguments 6 and 7 does not change the arguments supplied to the sole dependency call.

## Uncertainties

- **documented_only — Operand representation:** The two-byte suffix stride suggests two-byte units, but encoding and element semantics are not established.
- **documented_only — Dependency behavior:** Neither dependency’s implementation, side effects, failure modes, or memory ownership contract is supplied.
- **documented_only — Arithmetic outside valid-address preconditions:** No overflow protection is visible; the exact execution model for overflowed address arithmetic is not independently established.
- **documented_only — Operation `0x000020`:** Its nonnegative predicate follows from interpreting the intermediate character value as signed eight-bit and right shift as arithmetic. An alternative language-level interpretation of character signedness would affect the expression; no independent machine-level confirmation is supplied.
- **Inferred intent:** Prefix, suffix, matching, and interval terminology describe structural use of the dependencies, not verified domain semantics.

## Evidence

The supplied static representation establishes the argument checks, selector masks, dependency mode values, initial result write, branch-specific call arguments, unsigned extent guards, dependency call order, signed return tests, optional byte transformation, and two local status values.

The behavioral interpretation above distinguishes these local effects from inferred operation names and undocumented dependency semantics.

No runtime tests, execution traces, or independent implementation inspection have been performed.
