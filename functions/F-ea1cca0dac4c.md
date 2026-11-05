<!-- Trusted provenance: F-ea1cca0dac4c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea1cca0dac4c

## Purpose

F-ea1cca0dac4c performs eight explicit memory stores, obtains values from external data slots, computes a dispatch argument, and makes an unresolved indirect control transfer.

**Established static evidence:** the represented behavior includes byte, 16-bit, and 32-bit writes; reads of four external slots, including repeated evaluations of two slots; an unsigned comparison; and a final bitwise-OR argument. Several required inputs come from ambient machine state rather than the four declared arguments.

**Inferred intent:** the operations may implement a low-level state transformation followed by dispatch. No application-level purpose is established.

The contract below describes the supplied static representation. It does not establish that the represented function boundary, declared interface, or treatment of the final transfer as an ordinary call is correct.

## Interface

### Explicit inputs

The represented interface has four signed integer arguments and no explicit return value.

| Input | Uses |
|---|---|
| First argument | Added to an ambient signed base to form the first destination; also supplies the 32-bit value stored there. |
| Second argument | Contributes to two 16-bit-store destinations; supplies the value for one of those stores. |
| Third argument | Added to the second argument to form a 16-bit-store destination. |
| Fourth argument | Added to the captured shared index to form the final 16-bit-store destination. |

No buffer lengths, address ranges, ownership annotations, or lifetime information accompany these inputs.

### Ambient inputs

The represented body also consumes values without showing their initialization:

- A signed base used in the first destination address.
- A signed integer used both in address formation and as the source of a 16-bit store.
- A first unsigned integer whose low eight bits are stored and whose full value participates in a comparison.
- A second unsigned integer compared against the first.
- An unresolved indirect-transfer target.

Availability of these values is a precondition for interpreting the body as an executable operation. Their relationship to a real calling convention remains **documented_only**.

### External data slots

The following names describe data roles, not implementation identifiers.

| Slot | Represented contents and use |
|---|---|
| Payload slot | A 32-bit payload captured for a later word store. |
| Halfword-index slot | A signed integer captured and doubled to form a 16-bit-store destination. |
| Shared-index slot | A signed integer captured for later computations, and evaluated again for the first byte-store destination. |
| Base slot | An unsigned integer captured for dispatch, then evaluated again for the second byte-store destination. |

The shared-index capture and its later evaluation are distinct in the representation. Likewise, the two base-slot evaluations are distinct. This does not establish how many physical load instructions execute.

### Widths, representation, and arithmetic

The supplied representation supports the following working widths:

- Signed integers: 32 bits.
- Unsigned integers: 32 bits.
- Byte stores: 8 bits.
- Halfword stores: 16 bits.
- Word stores: 32 bits.

The exact ABI and address width are not independently established. The payload is a 32-bit quantity; its signed interpretation is unnecessary to describe its later store.

Narrow stores write the low eight or sixteen bits of their sources under the represented fixed-width interpretation. Narrowing a stored value does not narrow the value used in subsequent address calculations or dispatch.

The derived dispatch value requires two distinct arithmetic stages:

1. Add decimal 78 to the captured shared index using signed integer arithmetic.
2. Convert that intermediate result to unsigned arithmetic and add an unsigned contribution of either zero or one.

The contribution is one when the second ambient unsigned integer is less than or equal to the first, and zero otherwise.

The first stage is **not** established as modulo-\(2^{32}\) arithmetic. If the signed addition overflows, its behavior remains **documented_only** pending machine-level evidence. For a signed 32-bit interpretation, that addition is representable when the captured shared index is at most 2,147,483,569. Adding 78 cannot underflow a signed 32-bit input.

When the signed intermediate is representable, its conversion to unsigned and the subsequent addition follow modulo-\(2^{32}\) arithmetic. The second byte-store address also uses unsigned addition of the later base value and the derived value.

Other expressions use signed arithmetic, including the initial address sums, doubled indices, and final address sum. Their overflow behavior is also unresolved.

### Outputs and termination

There is no explicit scalar result. Observable outputs consist of memory writes and the indirect transfer.

The representation places a result-free return after the indirect invocation. A normal return is therefore conditional on that transfer returning normally and control continuing as represented. The underlying transfer may not actually be an ordinary returning call.

## Behavior

### Preconditions for normal progression

A normally progressing execution requires:

- Available explicit and ambient inputs.
- Valid readable external slots at each reached evaluation.
- Valid writable destinations covering the full width of each reached store.
- Any required alignment.
- Address computations with supported arithmetic and address-conversion behavior.
- A valid indirect target and compatible invocation state.

No visible checks enforce these conditions. The function has no represented status return, recovery path, or rollback.

### Ordered operations

The following order is established at the level of the supplied representation. It is not a guarantee of atomicity, hardware ordering, or concurrent observation.

1. **First word store.**  
   Add the ambient signed base to the first argument. At the resulting address, store the first argument as a 32-bit value.

2. **First halfword store.**  
   Add the third argument to the second argument. At that address, store the low sixteen bits of the ambient signed integer.

3. **Second halfword store.**  
   Add the second argument to the ambient signed integer. At that address, store the low sixteen bits of the second argument.

4. **Capture three external values.**  
   Read, in order:
   - The payload slot.
   - The halfword-index slot.
   - The shared-index slot.

   Retain these captured values for subsequent operations. These reads follow the first three stores, so overlap with those stores can affect the captured values.

5. **First byte store.**  
   Evaluate the shared-index slot again. Double this newly evaluated signed value to form the destination. Store the low eight bits of the first ambient unsigned integer there.

   The address expression uses this later slot evaluation, not explicitly the previously captured shared index. No explicit store occurs between the shared-index capture and this reevaluation. In ordinary stable memory, the two values agree; their equality is not an unconditional environmental guarantee.

6. **Fixed-value halfword store.**  
   Double the captured halfword index. At that address, store the 16-bit value `0x004E`, decimal 78.

7. **Capture the dispatch base.**  
   Read the base slot as an unsigned 32-bit value and retain it for the final indirect argument.

   This read follows both the first byte store and the fixed-value halfword store. If either overlaps the base slot, the captured base can reflect that modification.

8. **Compute the derived value.**  
   First add signed integer 78 to the captured shared index. Only afterward does the unsigned comparison contribution introduce unsigned arithmetic.

   Compare the two ambient unsigned integers at their full represented width. Add one when the second is less than or equal to the first; otherwise add zero.

   Subject to the signed intermediate being representable, the resulting derived value is a 32-bit unsigned quantity.

9. **Second byte store.**  
   Evaluate the base slot again. Add that later unsigned base value to the full derived value using unsigned 32-bit arithmetic. At the resulting address, store the low eight bits of the derived value.

   The earlier base capture remains the value used for dispatch. No explicit memory store occurs between the two base-slot evaluations. Stable ordinary memory therefore gives equal values, but the representation does not establish immutability or external synchronization.

10. **Payload word store.**  
    Double the captured shared index. At that address, store the captured 32-bit payload.

    If the shared-index reevaluation used in step 5 matched the captured shared index, this store starts at the same address as the first byte store. Otherwise, the two starting addresses need not match.

11. **Final halfword store.**  
    Add the fourth argument to the captured shared index. At that address, store the low sixteen bits of the captured shared index.

12. **Indirect transfer.**  
    Invoke F-unknown-indirect with one represented 32-bit argument: the bitwise OR of the full derived value and the earlier captured base.

    This argument is not the sum used for the second byte-store destination. The second byte store’s truncation does not truncate this argument.

13. **Conditional return.**  
    If the target returns and execution resumes as represented, return without an explicit result.

### Aliasing and retained values

No non-aliasing restriction is shown. Destinations may overlap each other or external slots.

Earlier stores can affect later slot reads. Conversely, modifying a slot after a value has been captured does not retroactively change that captured value. In particular:

- The payload store uses the captured payload.
- The fixed-value store uses the captured halfword index.
- Derived arithmetic, the payload destination, and the final halfword store use the captured shared index.
- Dispatch uses the earlier base capture, not a fresh read after the remaining stores.

Later overlapping stores can overwrite earlier stored bytes. No endianness is supplied, so the contract specifies multi-byte values and widths without assigning their component bytes to increasing addresses.

## Side effects

There are **eight explicit stores**.

| Order | Width | Destination basis | Stored value |
|---:|---:|---|---|
| 1 | 32 bits | Ambient signed base plus first argument | First argument |
| 2 | 16 bits | Third argument plus second argument | Low sixteen bits of ambient signed integer |
| 3 | 16 bits | Second argument plus ambient signed integer | Low sixteen bits of second argument |
| 4 | 8 bits | Twice the reevaluated shared-index slot | Low eight bits of first ambient unsigned integer |
| 5 | 16 bits | Twice the captured halfword index | Decimal 78 |
| 6 | 8 bits | Later base evaluation plus derived value | Low eight bits of derived value |
| 7 | 32 bits | Twice the captured shared index | Captured payload |
| 8 | 16 bits | Fourth argument plus captured shared index | Low sixteen bits of captured shared index |

The body shows no direct allocation, deallocation, resource acquisition, or ownership transfer. Address formation and writes do not establish ownership of the destinations.

The indirect target may perform additional effects, including effects on globals, memory, resources, or control flow. None are constrained by this body.

Intermediate writes may matter even when overwritten later, particularly for device-backed or externally observed memory. Atomicity, volatile semantics, synchronization, and hardware memory ordering remain unspecified.

A failure can leave earlier effects in place. There is no represented transactional boundary or compensating action.

## Dependencies

### F-unknown-indirect

This is an opaque documentary ID for the unresolved indirect target, not an established dependency identity.

The visible invocation supplies the bitwise OR of the derived unsigned value and the earlier captured unsigned base. The evidence does not establish:

- How the target is initialized or selected.
- Its complete calling convention or additional ambient inputs.
- Whether it returns.
- Whether it changes globals, external slots, or unrelated memory.
- Whether it acquires, transfers, or releases ownership.
- How it reports failure.

The supplied dependency information does not resolve this target to another identified function.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, they assume valid memory, suitable alignment, compatible invocation state, supported address conversion, representable signed address arithmetic, and a representable signed intermediate when adding 78.

### Equality contributes one

With captured shared index 100 and equal ambient unsigned comparison operands, the signed intermediate is 178 and the derived unsigned value is 179.

If the second ambient unsigned operand is greater than the first, the contribution is zero and the derived value is 178.

Equality belongs to the incrementing case.

### Byte truncation does not narrow later computations

With captured shared index 200 and a comparison contribution of one, the signed intermediate is 278 and the derived value is 279.

The second byte store writes decimal 23. Its address calculation and the dispatch OR still use the full value 279.

### OR differs from address addition

With earlier base capture `0x100` and derived value `0x112`, the indirect argument is `0x112`.

If the later base evaluation is also `0x100`, the second byte-store destination is `0x212`. That destination is not the dispatch argument.

### Matching shared-index evaluations produce an overwrite

If both shared-index evaluations yield `0x1000`, the first byte store and later payload word store begin at `0x2000`.

After the payload store, and absent subsequent overlap, that four-byte region contains the captured payload in the target’s byte order. The earlier byte write still occurred.

If an external change instead makes the reevaluated shared index `0x1001`, the first byte-store address is `0x2002`, while the payload store still begins at `0x2000`. Their starting addresses differ, although their written regions overlap.

### Earlier stores affect later captures

If one of the first three stores modifies the shared-index slot, its captured value reflects the slot when read after those stores, not necessarily its entry value.

If the fixed-value halfword store modifies bytes of the base slot, both later base evaluations can reflect that modification. Their exact numeric values depend on overlap and byte order.

### Signed overflow is not silently converted into wrapping behavior

If adding 78 to the captured shared index exceeds the signed 32-bit range, this contract does not assign a derived value by wrapping that signed addition.

The later unsigned contribution does not establish the behavior of the earlier overflowing signed operation. Other preconditions may already fail for the same input, including doubled-index address arithmetic.

### Failure preserves no rollback guarantee

If the fixed-value halfword store fails, preceding stores may already have occurred. The later base capture, derived byte store, payload store, final halfword store, and indirect transfer have not yet been reached in the represented sequence.

No restoration of earlier contents is shown.

## Uncertainties

The following remain **documented_only**:

- Correctness of the represented function boundary and parameter list.
- Initialization, provenance, and valid ranges of ambient inputs.
- Exact ABI, address width, integer-to-address behavior, alignment, and endianness.
- Signed-overflow behavior in address expressions and in the initial shared-index-plus-78 computation.
- Whether repeated slot evaluations correspond to distinct machine loads.
- Whether external slots are stable, externally modified, device-backed, or synchronized.
- Exact instructions, access atomicity, and hardware ordering.
- Identity and full behavior of F-unknown-indirect.
- Whether the final transfer is a call, jump, or another control-transfer mechanism.
- Fault, exception, signal, and nonlocal-transfer behavior.
- Memory ownership and application-level meaning.

The representation contains no visible validation, retry, cleanup, or error translation. This does not establish whether surrounding execution supplies such mechanisms.

## Evidence

This contract is based solely on the supplied static representation and dependency information. No implementation access, runtime execution, or tests have been performed.

The strongest established facts are the eight stores and their represented order and widths; the ordered external captures; the separate shared-index and base-slot reevaluations; the full-width unsigned comparison; the signed addition preceding unsigned arithmetic; and the final bitwise-OR argument.

The source material explicitly marks the indirect transfer as incompletely recovered and represented as a call despite that uncertainty. Ordinary call-and-return behavior is therefore conditional, not verified.

Domain purpose, ownership, environmental stability, overflow behavior beyond established arithmetic rules, and target behavior remain unresolved rather than inferred into requirements.
