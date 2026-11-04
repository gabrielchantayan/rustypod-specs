<!-- Trusted provenance: F-a3bf3b3dcd18; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3bf3b3dcd18

## Purpose

F-a3bf3b3dcd18 has no established successful return path or application-level purpose. The supplied static representation describes an initial byte read, a conditional choice of a 32-bit value and storage base, one 32-bit store, and then either a nonterminating loop or a boundary marked as invalid instruction data.

The recoverable contract is therefore a partial execution contract, not a specification of an ordinary returning operation. In particular, the invalid-instruction boundaries must not be interpreted as successful completion, a defined error return, or a known termination service.

**Established static evidence:** all represented paths either remain in a loop or reach an invalid-instruction boundary.

**Inferred intent:** none is sufficiently supported to assign a higher-level purpose to the arithmetic, byte flag, or destination field.

## Interface

### Explicit inputs

The representation exposes three arguments:

| Argument | Represented width and interpretation | Established use |
|---|---|---|
| First | 32-bit value with unspecified interpretation | No represented use |
| Second | Signed 32-bit integer | Tested for signed overflow when adding 198 |
| Third | 32-bit integer | Supplies the bit pattern used in an addition of 7 and subsequent logical right shift |

There is no supported basis for interpreting the first argument as a pointer, handle, length, or ownership token. Its lack of represented use does not establish what unrecovered instructions might do.

The second argument’s overflow predicate is true exactly for values from **2,147,483,450 through 2,147,483,647**, inclusive. Adding positive 198 cannot cause signed underflow.

For the third argument, the relevant machine-width arithmetic is:

- Take its 32-bit bit pattern.
- Add 7 modulo \(2^{32}\).
- Unless overridden by the second argument’s overflow condition, logically shift the resulting unsigned value right by 23 bits.

The reconstructed signed-looking addition does not establish an abstract-language promise concerning signed overflow. The behavioral interpretation here is the 32-bit bit-pattern computation represented by the supplied material.

### Implicit inputs

The function also depends on:

- A global-held base address used for a one-byte read at displacement **24 bytes**.
- An incoming register-derived base used for the destination store when the second argument’s addition does not overflow.
- A stack-associated reference used to replace that destination base when the addition does overflow.
- The readability and writability of the resulting addresses.

The provenance, lifetime, bounds, alignment, and ownership of these storage regions are not established.

### Outputs

No explicit result value is represented. There is no normal return path in the supplied control flow.

The principal represented output is a **32-bit memory store**. Its value is either zero or the upper nine bits of the third argument’s wrapped sum with 7. Consequently, the stored numeric value is always in the range **0 through 511**, inclusive.

No byte order is established for that 32-bit store.

## Behavior

### Initial arithmetic and byte read

The representation first forms the third argument’s 32-bit sum with 7. It then reads one byte at displacement 24 from the global-held base.

The byte read occurs before the branch on signed overflow of the second argument plus 198. This matters even when the overflow branch later discards the byte’s value: the initial read is still present on that path.

The supplied material contains no null check, bounds check, alignment repair, or explicit failure handling for this read. Successful access to the referenced byte is therefore a precondition for reaching later represented effects without an access failure. The concrete fault behavior, if any, is outside the established contract.

### Overflow-dependent state selection

The function tests whether adding 198 to the signed 32-bit second argument would overflow.

If the predicate is **false**:

- The third argument’s wrapped sum with 7 remains the selected value.
- The incoming register-derived destination base remains selected.
- The previously read byte remains available for a later zero-versus-nonzero test.

If the predicate is **true**:

- The selected value becomes zero.
- The destination base is replaced by a stack-associated address.
- The selected byte value becomes zero, irrespective of the byte read earlier.

The stack-associated replacement is represented as a reference displaced by 16 bytes. The subsequent store adds another 384 bytes, giving a combined displacement of **400 bytes from that represented stack reference**. This arithmetic does not establish a displacement from the actual entry stack pointer: the relationship between the reconstructed reference and the machine’s stack state is unknown.

### Destination store

After the overflow-dependent selection, the function stores the selected value logically shifted right by 23 bits.

The represented destination is:

- **No overflow:** incoming register-derived base plus **384 bytes**.
- **Overflow:** the stack-associated replacement base plus **384 bytes**, equivalently the represented stack reference plus **400 bytes**.

The store is 32 bits wide. It precedes the subsequent decision between the zero-result path and the nonzero-result path.

On an overflow path, the stored value is necessarily zero.

On a non-overflow path, the stored value is the wrapped sum’s upper nine bits. It is zero exactly when the wrapped sum lies between 0 and 8,388,607, inclusive. Expressed as signed third-argument values, that condition is exactly:

**−7 through 8,388,600, inclusive.**

This interval includes the wraparound cases −7 through −1 as well as zero and the positive values through 8,388,600.

No represented read-back of the destination follows the store. The next branch depends on the selected arithmetic value, not on a demonstrated reload of destination memory.

### Zero-result path

When the shifted value is zero, execution enters a loop whose continuation condition is the negation of the same signed-overflow predicate involving the second argument and 198.

The represented loop body is empty. It performs no additional explicit memory reads or writes.

There are two distinct cases:

- **The second-argument addition does not overflow:** the loop continuation condition remains true for the unchanged argument. In the represented execution model, this is a nonterminating loop after the store.
- **The second-argument addition overflows:** the loop continuation condition is false on its first test. Execution leaves the empty loop and reaches an invalid-instruction boundary.

No represented operation changes the second argument inside the loop. There is no established mechanism by which the loop polls memory, responds to the initially read byte, or observes a change in destination storage.

This nontermination statement is limited to the represented control flow. It does not assert immunity to external interruption, faults, or unrecovered machine behavior.

### Nonzero-result path

A nonzero shifted value is possible only when the second-argument addition does not overflow.

On that path, the function tests whether the initially read byte is nonzero:

- A nonzero byte leads to an invalid-instruction boundary.
- A zero byte also leads to an invalid-instruction boundary.

Thus, the byte test distinguishes represented control-flow paths but does not establish different successful outcomes. It would be unjustified to remove the byte dependency from this contract: the read is explicit, and both terminal boundaries have unknown concrete behavior.

The byte’s signedness does not affect this particular test; only zero versus nonzero matters.

## Side effects

The established ordering of represented memory effects is:

1. Read one byte from the global-held base at displacement 24.
2. Select the arithmetic value and destination base according to signed overflow.
3. Store one 32-bit value at the selected destination plus displacement 384.
4. Enter the loop or proceed toward an invalid-instruction boundary.

The first arithmetic calculation precedes the byte read in the representation, but is not itself a memory effect.

Conditional consequences include:

- If the byte access fails, the later store is not established to occur.
- If the destination access fails, later loop or terminal behavior is not established to occur.
- If the store succeeds and the zero-result, non-overflow case applies, the represented function remains in the loop without further explicit memory effects.
- If an invalid-instruction boundary is reached, any behavior beyond that boundary is unknown.

The overflow branch also replaces a register-derived base in the representation. Whether this corresponds to a meaningful architectural register update, and whether any observer could inspect it, is not established.

There is no represented allocation, release, reference-count adjustment, synchronization operation, or ownership transfer. This is not evidence that unknown behavior beyond the recovered boundary preserves memory or global state.

Aliasing between the byte-read region and destination region is not excluded. The represented read nevertheless occurs before the store. No volatility, atomicity, memory-ordering, or concurrent-access guarantees are established.

## Dependencies

No ordinary callable dependency with a supplied opaque function ID appears in the recoverable body.

The invalid-instruction markers are treated as limits of the supplied representation, not as identified callable services. They do not establish a known exception API, process-exit operation, error-reporting routine, or memory-management action.

The behavior additionally depends on storage and execution context:

| Dependency | Required property for the represented path | Status |
|---|---|---|
| Global-held base | Supports the byte access at displacement 24 | Address and object layout unknown |
| Incoming register-derived base | Supports the 32-bit store at displacement 384 on non-overflow paths | Origin and bounds unknown |
| Stack-associated reference | Supports the overflow-path store at combined displacement 400 | Relationship to actual stack layout unknown |
| Execution at invalid-instruction boundaries | Determines what follows the recovered path | `documented_only` |

No assumption is made that unknown execution preserves globals, retains ownership, or releases memory.

## Acceptance examples

These are conditional static acceptance examples, not runtime results. Unless stated otherwise, they assume the required memory accesses succeed and address formation behaves as represented.

1. **Zero-result loop without signed overflow**

   With second argument 0 and third argument 0, the wrapped sum is 7 and its logical shift by 23 is zero. The function reads the byte, stores zero at the incoming base plus 384, and remains in the represented empty loop. Either initial byte value leads to the same loop because the later byte test is not reached.

2. **Upper zero-result boundary**

   With second argument 0 and third argument 8,388,600, the wrapped sum is 8,388,607. The stored value is zero, followed by the represented nonterminating loop.

3. **First positive nonzero-result case**

   With second argument 0 and third argument 8,388,601, the wrapped sum is 8,388,608. The function stores 1 at the incoming base plus 384, then tests the initial byte. Both byte outcomes reach an invalid-instruction boundary; neither establishes normal return.

4. **Wraparound yielding zero**

   With second argument 0 and third argument −7, the 32-bit sum wraps to zero. The function stores zero at the incoming destination and enters the nonterminating loop.

5. **Value immediately below the wraparound zero interval**

   With second argument 0 and third argument −8, the wrapped sum is 4,294,967,295. The stored value is 511, followed by the byte-dependent branch toward an invalid-instruction boundary.

6. **Largest non-overflowing second argument**

   With second argument 2,147,483,449 and third argument 0, adding 198 does not overflow. The incoming destination base is retained, zero is stored, and the represented loop does not terminate.

7. **First overflowing second argument**

   With second argument 2,147,483,450, any third-argument value, and either byte value, the initial byte read still occurs. The function then selects the stack-associated destination, stores zero, leaves the empty loop on its first condition test, and reaches an invalid-instruction boundary.

8. **Inaccessible byte on an overflow path**

   Even when the second argument guarantees the overflow branch, the supplied ordering does not permit bypassing the initial byte read. A successful overflow-path store cannot be promised if that access fails.

## Uncertainties

The following remain `documented_only`:

- The higher-level purpose of the function and the meaning of its memory fields.
- The actual machine instructions at and beyond the invalid-instruction boundaries.
- Whether those boundaries trap, terminate, continue elsewhere, or reflect incomplete recovery.
- The concrete origin of the register-derived destination base.
- The relationship between the stack-associated reference and the actual stack pointer or frame.
- Address-space width and address-wrap behavior; the reconstruction suggests 32-bit address manipulation but does not independently establish the execution environment.
- Destination alignment requirements and byte order.
- Fault delivery and partial-store behavior.
- Whether either storage region is shared, volatile, device-backed, or concurrently modified.
- Any externally visible distinction between the terminal byte-test branches.
- Effects absent from the recovered representation.

These unknowns do not justify inventing validation, cleanup, return values, or recovery behavior.

## Evidence

This contract is based solely on the supplied static representation: three arguments, a 32-bit addition by 7, a byte access at displacement 24, a signed 32-bit overflow test for addition by 198, conditional base replacement, a 32-bit store at displacement 384, a logical right shift by 23, an empty overflow-conditioned loop, and invalid-instruction boundaries.

The numeric thresholds and acceptance examples are arithmetic deductions from those operations. They are not observed executions.

The supplied warning about bad instruction data materially limits completeness. No runtime tests have been performed, and no behavior beyond the represented boundaries is claimed.
