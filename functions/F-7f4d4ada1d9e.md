<!-- Trusted provenance: F-7f4d4ada1d9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7f4d4ada1d9e

## Purpose

F-7f4d4ada1d9e conditionally advances a 32-bit state field by an unsigned quantity. Acceptance requires a non-null associated reference, a nonzero word obtained through that reference, a quantity no greater than 524,287, and a resulting candidate strictly below a derived unsigned threshold.

The operation invokes F-ccc494c80b38 before inspecting the supplied object and F-dd2544d72d08 after making its decision. The latter invocation occurs on both acceptance and rejection, provided execution reaches it normally.

**Established behavior:** the arithmetic used for the candidate and threshold is 32-bit unsigned arithmetic, including wraparound. Equality with the threshold is rejected.

**Inferred intent:** the state field may represent a position, and the associated word may supply a reference point for a bounded advance. Neither interpretation is necessary to describe the behavior. In particular, the operation does not establish ordinary non-wrapping range safety.

## Interface

### Inputs

The operation has two inputs:

| Input | Representation | Role |
|---|---|---|
| Object address | Address represented by a 32-bit integer in the supplied representation | Identifies the object whose fields are inspected and conditionally updated. |
| Advance quantity | Unsigned 32-bit integer | Added to the state field only when the structural and quantity checks permit candidate evaluation. |

There is no explicit null check on the object address.

The accessible object layout is:

| Byte offset | Width | Access | Meaning within this contract |
|---|---:|---|---|
| +4 | 32 bits | Read | Associated reference; zero makes the operation reject. |
| +8 | 32 bits | Read and conditionally written | State value to which the advance quantity is added. |

When the associated reference is nonzero, another 32-bit word is read at byte offset +4 from the referenced location. This is the **associated word**. Its zero/nonzero status is checked, and its bit pattern is subsequently used in unsigned threshold arithmetic.

No behavior is established for other object fields. The supplied representation does not establish byte order, packing rules beyond these offsets, or an allocation size beyond the accesses described.

### Output

The result is a logical Boolean:

- **True:** the operation accepts the candidate and performs a 32-bit write to the object's +8 field before calling F-dd2544d72d08.
- **False:** the operation rejects the candidate and performs no direct write to that field.

A true result does not guarantee that the written value remains in memory when the operation returns. F-dd2544d72d08 executes after the write and may have effects not established here.

A false result does not guarantee that the object is unchanged. Either dependency may modify it or other state.

### Preconditions and validity

For normal completion, the object and any followed reference must support the described accesses when those accesses occur. If a write is selected, the +8 field must also be writable.

The first dependency executes before any of these accesses. Consequently, validity must hold after that dependency returns, not merely at entry.

An excessive advance quantity does not make an invalid object safe to supply: the object fields and, when selected, the associated word are read before the quantity can cause rejection. The operation provides no established error result for invalid addresses, inaccessible memory, alignment faults, or a dependency that fails to return normally.

## Behavior

### Dependency entry and observation order

The first observable action established by the supplied representation is a call to F-ccc494c80b38 with no explicit arguments.

After that call returns, the operation observes memory in this order:

1. It reads the 32-bit state value at object offset +8.
2. It reads the 32-bit associated reference at object offset +4 and records whether that observation is zero.
3. If that observation was nonzero, it reads the associated reference at object offset +4 again, then reads the 32-bit associated word at offset +4 from the reference obtained by this second read.

The repeated reference read is part of the supplied representation. The first observation controls whether the nested read is attempted; the second supplies its address.

Under stable-memory conditions, both reference observations identify the same location. If the field changes between observations, the supplied behavior contains no validation that the second reference is still nonzero or still denotes the same object. No synchronization or atomicity guarantee is established.

The state value used in the decision is the earlier value read from +8. It is not refreshed before the conditional write.

### Structural and quantity rejection

The operation rejects if any of these conditions holds:

- The first associated-reference observation is zero.
- The associated word is zero.
- The advance quantity is greater than 524,287, equivalently greater than hexadecimal 0x0007FFFF.

If the first associated reference is zero, no nested associated-word read occurs.

If the reference is nonzero, the nested word is read even when the advance quantity is already outside the permitted range. The quantity limit therefore does not short-circuit the preceding memory accesses.

An associated word with its high bit set is not rejected merely because its signed interpretation would be negative. The structural check distinguishes zero from nonzero; the threshold calculation uses its 32-bit bit pattern as unsigned.

### Candidate and threshold comparison

When the structural conditions pass and the quantity is within range, acceptance depends on two values:

- **Candidate:** the originally observed state value plus the advance quantity, reduced modulo 2³².
- **Threshold:** the associated word's unsigned 32-bit value plus 524,288, reduced modulo 2³².

The operation accepts exactly when the candidate is **strictly less than** the threshold in an unsigned 32-bit comparison.

The threshold offset is hexadecimal 0x00080000. It is distinct from the maximum permitted advance quantity, which is one less.

Consequences include:

- A candidate equal to the threshold is rejected.
- A zero advance is allowed by the quantity check, but can still be rejected by the candidate comparison.
- Candidate addition can wrap to a small value and then be accepted.
- Threshold addition can wrap to a small value, sharply reducing the accepted candidate range.
- A threshold of zero rejects every candidate.
- No comparison establishes that the candidate is numerically greater than or equal to the original state.
- No comparison establishes that the candidate is greater than or equal to the associated word.

The function therefore must not be specified as a conventional saturating increment, checked addition, or non-wrapping interval check.

### State transition and exit

On acceptance, the operation writes the candidate as a 32-bit value to object offset +8. On rejection, it makes no direct write there.

After either decision, it calls F-dd2544d72d08 with the original object address as its sole explicit argument.

The returned Boolean reflects the previously computed acceptance decision. The supplied representation shows no post-call reread or revalidation of the object's state. Any normal effects of F-dd2544d72d08 therefore do not cause the decision to be recomputed.

These return guarantees are conditional on both dependencies and the intervening memory accesses completing normally.

## Side effects

The only direct memory write established for F-7f4d4ada1d9e itself is the conditional 32-bit write to object offset +8.

The direct reads are:

- Object offset +8, once.
- Object offset +4, once when the first reference observation is zero and twice when it is nonzero.
- Associated location offset +4, once when the first reference observation is nonzero.

These counts describe the supplied representation, not a guarantee about underlying machine transactions or volatile-access semantics.

There are two dependency calls on every normally completing path:

1. F-ccc494c80b38, before the direct reads.
2. F-dd2544d72d08, after the optional direct write.

No direct allocation, release, ownership transfer, reference-count update, or global-state write is visible in the supplied body. That absence does not constrain either dependency.

In particular, the dependency names and their placement are insufficient to establish that they acquire and release a lock, preserve globals, retain the object, or release memory. There is no established rollback if a dependency fails after a write.

The operation contains no direct reread of the object after F-dd2544d72d08 returns. Whether that dependency can invalidate or release the object remains unknown; this contract grants no ownership permission based on that possibility.

## Dependencies

| Opaque ID | Invocation | Established relationship |
|---|---|---|
| F-ccc494c80b38 | Once, with no explicit arguments, before direct object access | Its normal return precedes the state and reference observations. No result is inspected. |
| F-dd2544d72d08 | Once, with the original object address, after the decision and optional write | Called for both acceptance and rejection. No result is inspected. |

For both dependencies, effects on memory, globals, synchronization, allocation, ownership, and failure behavior are **documented_only** unknowns.

The no-argument call is not evidence of having no relevant inputs: it may depend on global or other implicit state. Likewise, ignoring a dependency's result does not imply that the dependency cannot fail or terminate control flow.

## Acceptance examples

The following are conditional examples, not executed tests. Unless stated otherwise, assume valid accessible memory, a stable nonzero associated reference, normally returning dependencies, and no dependency mutation that changes the values under discussion. The “write” column describes the operation's direct write before the final dependency call.

| Associated word | Initial state | Advance | Expected result | Direct write |
|---|---:|---:|---|---|
| 0x00001000 | 0x00080000 | 0x00000010 | True: candidate 0x00080010 is below threshold 0x00081000. | 0x00080010 |
| 0x00001000 | 0x00080FFF | 0 | True: unchanged candidate is below threshold 0x00081000. | 0x00080FFF, even though its value is unchanged |
| 0x00001000 | 0x00080FFF | 1 | False: candidate equals threshold 0x00081000. | None |
| 0x00001000 | 0x00081000 | 0 | False: zero advance does not bypass the threshold comparison. | None |
| 0x00001000 | 0 | 0x0007FFFF | True: maximum permitted advance produces candidate 0x0007FFFF. | 0x0007FFFF |
| 0x00001000 | 0 | 0x00080000 | False: advance exceeds the permitted maximum. | None |
| 0 | Any readable value | 0 | False: associated word is zero. | None |
| 0x00001000 | 0xFFFFFFF0 | 0x20 | True: candidate wraps to 0x00000010, below 0x00081000. | 0x00000010 |
| 0xFFF90000 | 0x0000FFFF | 0 | True: threshold wraps to 0x00010000. | 0x0000FFFF |
| 0xFFF90000 | 0x00010000 | 0 | False: candidate equals the wrapped threshold. | None |
| 0xFFF80000 | Any readable value | Any permitted quantity | False: threshold wraps to zero. | None |

Additional ordering examples:

- **Zero reference:** after F-ccc494c80b38 returns, the state is read and the reference is observed as zero. No nested word is read. The result is false, but F-dd2544d72d08 is still called.
- **Excessive quantity with nonzero reference:** the nested word is still read before rejection. An inaccessible nested address is not converted into a clean false result by the quantity limit.
- **Entry dependency mutation:** if F-ccc494c80b38 changes the state before returning, the subsequent state read uses that changed value.
- **Exit dependency mutation:** if the candidate is accepted and F-dd2544d72d08 then changes the +8 field, the Boolean remains true on normal return. The final field value need not equal the candidate.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the state field and associated word.
- The complete object layout and allocation requirements.
- The exact calling convention and physical Boolean return encoding.
- Endianness, alignment requirements, and target behavior for invalid accesses.
- The intended address-space model behind the integer-valued object address.
- Whether concurrent mutation is permitted or excluded by an external protocol.
- Whether either dependency supplies synchronization or preserves any global state.
- Whether either dependency allocates, releases, retains, or invalidates memory.
- Dependency behavior on exceptional or non-returning paths.
- Whether integer wraparound is intentional or an incidental property of the implementation.

The observed representation supports 32-bit unsigned wraparound for the decision arithmetic. It does not establish safe behavior for arbitrary address arithmetic or invalid pointers.

## Evidence

This contract is based solely on the supplied static function representation and the supplied mapping to opaque function IDs.

Established static evidence covers the field offsets and widths, dependency order and explicit arguments, repeated reference read, zero checks, quantity bound, unsigned candidate and threshold arithmetic, strict acceptance comparison, conditional write, and Boolean return decision.

The position-and-bound interpretation is inferred intent only. Dependency semantics and ownership rules are not established.

No runtime tests have been performed. The acceptance examples are deductions from the static behavior, not observed executions.
