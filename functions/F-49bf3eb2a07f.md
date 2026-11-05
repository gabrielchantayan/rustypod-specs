<!-- Trusted provenance: F-49bf3eb2a07f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-49bf3eb2a07f

## Purpose

F-49bf3eb2a07f performs a conditional ownership-sensitive state transition on an embedded object. It first invokes two preparation dependencies and records the low byte of its second argument. It may then invoke a guarded dependency and propagate its nonzero result. Otherwise, it compares an object field with a value supplied by F-196812b4b122 and either reports a mismatch, decreases a 16-bit count, or clears ownership-related state and invokes F-259d2fe281b0.

**Established behavior:** Preparation and the byte write occur before every explicit return. An ownership mismatch produces the value 5. A particular count-controlled path can return zero without invoking the final dependency. When that dependency is invoked, its result becomes the function result.

**Inferred intent:** The ownership comparison, mode-dependent count, and final operation are consistent with releasing an owned, possibly nested resource. Their actual domain meaning is not established. This contract therefore describes fields and transitions without assigning a synchronization primitive or a particular owner identity scheme.

## Interface

### Arguments and result

The supplied representation exposes four explicit arguments:

| Argument | Represented width | Contract |
|---|---:|---|
| Object base | 32 bits | Address used to derive the embedded-object address and access the fields listed below. |
| Preparation value | 32 bits | Passed in full to F-7daf8eda3944. Its low eight bits are subsequently stored in the object. |
| Forwarded value A | 32 bits | Passed to F-621dae46a146 if that dependency is reached. |
| Forwarded value B | 32 bits | Passed to F-621dae46a146 if that dependency is reached. |

The result is represented as a signed 32-bit integer. Explicit local results are 0, 5, and hexadecimal 0x1A, or decimal 26. Results obtained from F-621dae46a146 and F-259d2fe281b0 are propagated as described below; their meanings and allowed ranges are unknown.

The represented call to F-621dae46a146 also includes four incoming machine-context words not established as explicit arguments of this function. Their provenance, semantic meaning, and relationship to the true calling convention are **documented_only**. They must not be assigned invented constant values or treated as known object fields.

### Object layout

Offsets below are relative to the object base. They describe the accesses visible in this function, not a complete object type.

| Offset | Width | Observed use |
|---|---:|---|
| 0x20 | 32 bits | First word of an embedded region; compared with a global comparison token. The region’s address is passed to preparation and guarded dependencies. |
| 0x24 | 32 bits | Ownership-like field; compared with the result of F-196812b4b122 and possibly cleared. |
| 0x2E | 16 bits | Unsigned flags snapshot. Bits 4 and 5 select the count-controlled behavior. |
| 0x32 | 16 bits | Count interpreted through a signed 16-bit representation, decremented conditionally, or explicitly cleared. |
| 0x34 | Not established | Address passed to F-259d2fe281b0. This function does not directly access the contents at that address. |
| 0x3E | 8 bits | Receives the low byte of the preparation value. |

These accesses require an addressable object representation through at least offset 0x3E. Dependencies may require additional storage, alignment, or valid internal structure; those requirements are unknown.

The early null comparison does not make this a safely null-checking interface. The function calls dependencies and writes at offset 0x3E before that comparison.

## Behavior

### 1. Preparation and byte recording

The function derives the embedded-region address by adding 0x20 to the object base.

It then performs the following operations in order:

1. Invokes an initialization-like dependency whose opaque ID is not supplied, passing the embedded-region address.
2. Invokes F-7daf8eda3944 with the full 32-bit preparation value.
3. Writes the low eight bits of the preparation value to object offset 0x3E.

Neither preparation call supplies a result used by the visible control flow. That does not establish that either call succeeds, has no side effects, or returns normally.

The byte write occurs after both calls. Consequently, any earlier modification of this byte by either dependency is overwritten if execution reaches the write. Only this byte is directly written at this stage; preservation of neighboring storage by dependencies is unknown.

### 2. Embedded-address null branch

After preparation and the byte write, the function tests whether the derived embedded-region address is zero.

If it is zero, the function returns 26. It does not proceed to the comparison-token check, flag read, ownership comparison, or final dependency.

This branch tests the derived address, not the original object base. In a flat 32-bit wrapping-address interpretation, a zero base would instead produce an embedded address of 0x20. Conversely, a base bit pattern of 0xFFFFFFE0 would produce a zero embedded address. These are arithmetic observations, not promises that such addresses are valid or that execution can reach the branch without a fault.

### 3. Comparison-token gate and optional early result

If the embedded-region address is nonzero, the function compares its first 32-bit word with a global comparison token.

If the values differ, it skips F-621dae46a146 and continues to the flags and ownership logic.

If the values match, it invokes F-621dae46a146 with:

- The embedded-region address.
- A zero-valued argument.
- Forwarded value A.
- Forwarded value B.
- The four additional incoming context words represented in the supplied call.

If this dependency returns a nonzero 32-bit result, F-49bf3eb2a07f immediately returns that result. It performs none of the subsequent direct flag, ownership, or count operations.

If the dependency returns zero, execution continues.

The comparison uses state available after both preparation calls. It is not a comparison against the object’s entry-state word or an assumed immutable token. The relative evaluation order of the two comparison operands is not established.

### 4. Flags snapshot and ownership comparison

The function reads the unsigned 16-bit flags field at offset 0x2E and retains that value for the later mode decision.

It then invokes F-196812b4b122 with no explicit arguments. After that call returns, it reads the 32-bit field at offset 0x24 and compares it with the dependency result.

This timing is significant:

- Mode selection uses the flags value read **before** F-196812b4b122.
- Ownership comparison uses the field read **after** F-196812b4b122.
- If F-196812b4b122 changes the flags, those changes do not replace the retained mode value.
- If it changes the ownership-like field, the comparison observes the resulting field value.

If the field and dependency result differ, the function returns 5. It does not directly read or modify the count, clear the ownership-like field, or invoke F-259d2fe281b0 on this path.

### 5. Mode-dependent count handling

When the ownership comparison matches, the retained flags select one of two paths.

The count-controlled mode is selected exactly when bits 4 and 5 of the flags encode binary 10. Equivalently, masking with 0x0030 yields 0x0020. Bits outside that mask do not affect this decision.

#### Any other mode

The function does not read the old count. It proceeds directly to the clearing and final-call sequence below.

#### Count-controlled mode

The function reads the 16-bit count at offset 0x32, subtracts one, stores the resulting 16-bit value back to the same field, and checks that narrowed value for zero.

The represented subtraction is performed after promotion of the signed 16-bit operand. Its mathematical range is −32769 through 32766, so the subtraction itself fits in a signed 32-bit value. The subsequent narrowing back to 16 bits is the relevant boundary.

Under the conventional two’s-complement target interpretation:

- 0x0001 becomes 0x0000.
- 0x0000 becomes 0xFFFF.
- 0x8000 becomes 0x7FFF.
- Every other input also decreases by one modulo 65536.

If the stored result is nonzero, the function returns 0 without directly clearing the ownership-like field or invoking F-259d2fe281b0.

If the stored result is zero, it continues to the clearing sequence. In the stated 16-bit interpretation, an initial count of one is the only count that takes this branch. There is no visible rejection of an initial zero or negative signed count.

### 6. Clearing and final call

When reached through either the non-count-controlled mode or a count decrement to zero, the function performs these operations in order:

1. Writes 16-bit zero to offset 0x32.
2. Writes 32-bit zero to offset 0x24.
3. Invokes F-259d2fe281b0 with the address at object base plus 0x34.
4. Returns that dependency’s result.

On the decrement-to-zero path, the explicit clear is a second write of zero to the count field.

Both clears occur before the final call. If that call returns a failure-like value, the function still returns that value without any visible rollback. The dependency may itself change the object or other state, so the final object contents cannot be inferred solely from the two preceding writes.

## Side effects

Direct writes by this function are limited to:

- The preparation-value byte at offset 0x3E, before the first explicit return opportunity.
- A conditional 16-bit count decrement.
- A conditional 16-bit count clear.
- A conditional 32-bit ownership-like field clear.

The function directly reads the embedded region’s first word, the comparison token, flags, ownership-like field, and conditionally the count. Each read occurs only after its preceding control-flow gates have been passed.

All dependency side effects remain additional to these direct effects. No dependency is established to preserve globals, preserve the object, allocate memory, free memory, acquire ownership, or release ownership. No memory-management guarantee follows merely from passing an address to a dependency.

No direct allocation or deallocation is visible. No atomicity, locking, memory-ordering, or concurrent-access safety guarantee is established. The documented order is the represented execution order; hardware-level observability is not established.

## Dependencies

| Opaque ID | Invocation condition and role |
|---|---|
| F-7daf8eda3944 | Invoked after the first preparation dependency, before the byte store. Receives the full preparation value. Its result is not used by the visible flow. |
| F-621dae46a146 | Invoked only when the embedded word matches the comparison token. A nonzero result is returned immediately; zero allows continuation. |
| F-196812b4b122 | Invoked after the flags snapshot. Its result is compared with the subsequently read ownership-like field. |
| F-259d2fe281b0 | Invoked after the count and ownership-like fields have been cleared. Receives the address at offset 0x34; its result is returned unchanged. |

One additional dependency is invoked first with the embedded-region address. No opaque ID was supplied for it. Its identity and behavioral contract are **documented_only**; no private symbol is substituted here.

The comparison token is a separate global input to the guarded-call decision. Its value, lifecycle, and mutation rules are unknown.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume valid accessible storage, normal dependency returns, a nonzero embedded-region address, and no dependency mutations that alter the stated conditions.

1. **Low-byte recording and ownership mismatch.**  
   With preparation value 0x123456AB, the function writes 0xAB at offset 0x3E after both preparation calls. If the guarded path is skipped or returns zero, and the ownership comparison then fails, the result is 5. No direct count access occurs.

2. **Guarded nonzero result.**  
   If the embedded word matches the comparison token and F-621dae46a146 returns −7, the result is −7. The preparation byte has already been written. The function does not subsequently read flags or invoke F-196812b4b122.

3. **Count remains nonzero.**  
   With matching ownership, count-controlled mode, and count 3, the count becomes 2 and the result is 0. The function does not directly clear the ownership-like field or invoke F-259d2fe281b0.

4. **Count reaches zero.**  
   With matching ownership, count-controlled mode, and count 1, the function stores zero to the count, explicitly clears it again, clears the ownership-like field, and invokes F-259d2fe281b0. If that dependency returns 9, the result is 9.

5. **Zero count is not rejected.**  
   Under the stated two’s-complement interpretation, matching ownership and a zero count in count-controlled mode produce count 0xFFFF and result 0. This path does not invoke F-259d2fe281b0.

6. **Other mode ignores the old count.**  
   With matching ownership and mode bits other than binary 10, the function clears both fields and invokes F-259d2fe281b0 without first reading or decrementing the old count.

7. **Flags change during identity retrieval.**  
   If the saved flags select count-controlled mode but F-196812b4b122 changes the stored flags to another mode, the current invocation still uses count-controlled behavior. The ownership comparison nevertheless uses the field value present after that call.

8. **Final-call failure has no visible rollback.**  
   If F-259d2fe281b0 returns a nonzero failure-like value, that value is returned. The two clears have already occurred; restoration by this function is not present.

## Uncertainties

- Exact ABI details, pointer arithmetic semantics, byte order, alignment requirements, and fault behavior are not fully established.
- The widths above follow the supplied representation. The count-boundary examples additionally assume conventional two’s-complement narrowing to 16 bits.
- The extra context words supplied to F-621dae46a146 may reflect calling-convention recovery rather than a complete semantic interface.
- The first dependency’s identity and all dependencies’ internal contracts are unavailable.
- Error meanings for 5 and 26 are unknown. They must not be assigned platform error names without further evidence.
- The fields’ domain meanings are inferred, not established.
- Behavior under concurrent mutation, dependency-triggered nonlocal control flow, inaccessible storage, or dependency release of referenced memory is **documented_only**. This function supplies no visible recovery mechanism for those conditions.

## Evidence

This contract is based solely on the supplied static representation and dependency-ID mapping. The field offsets, access widths, call ordering, branch conditions, direct writes, and return propagation are established by that representation.

Ownership and release terminology expresses inferred intent only. ABI details, dependency internals, and unspecified environmental behavior remain unknown.

No runtime tests, execution traces, or independent implementation inspection have been performed.
