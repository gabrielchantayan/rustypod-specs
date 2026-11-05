<!-- Trusted provenance: F-ae7115ea5ec1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae7115ea5ec1

## Purpose

F-ae7115ea5ec1 writes destination metadata from a four-word descriptor, calculates an aligned storage quantity, attempts to obtain a nonzero storage-related result, and conditionally delegates final setup.

The established contract concerns memory accesses, arithmetic, dependency calls, and control flow. Interpreting the descriptor as rectangular bounds, the aligned quantity as a row stride, and the storage calls as reuse and allocation is **inferred intent**, not an established dependency contract.

There is no explicit return value. If the fallback dependency returns zero, this function returns without final setup and without performing rollback. Because dependency memory effects are unknown, the destination’s contents at return cannot generally be inferred solely from this function’s earlier writes.

## Interface

The function takes five arguments:

| Argument | Established representation | Use |
|---|---|---|
| Destination | 32-bit address-bearing value | Base for direct field accesses; also passed to final setup. |
| Descriptor | Address of four consecutive 32-bit words | Supplies metadata and sizing operands; passed by reference to final setup. |
| Format-like value | Opaque 32-bit value | Passed to the sizing dependency and final setup. |
| Auxiliary value | Opaque 32-bit value | Written to destination offset +0xA8. |
| Final context | Opaque 32-bit value | Passed to final setup. |

The descriptor words are designated D0, D1, D2, and D3:

| Word | Byte offset |
|---|---:|
| D0 | +0 |
| D1 | +4 |
| D2 | +8 |
| D3 | +12 |

Descriptor words participate in signed 32-bit arithmetic before the unsigned sizing operations described below.

The directly accessed destination fields are:

| Byte offset | Access width | Direct operation |
|---|---:|---|
| +0x98 | 32 bits | Write a sampled D0 value. |
| +0x9C | 32 bits | Write a sampled D1 value. |
| +0xA0 | 32 bits | Write a sampled D2 value. |
| +0xA4 | 32 bits | Write a sampled D3 value. |
| +0xA8 | 32 bits | Write the auxiliary argument. |
| +0xB0 | 32 bits | Test and read an existing storage-related value; write zero on fallback. |
| +0xB4 | 8 bits | Write 1. |

Final setup additionally receives destination +4. This function does not directly dereference that address.

Normal completion requires the memory accesses actually reached to be valid. The body contains no explicit null, alignment, object-size, address-overflow, or descriptor-range checks. Dependency-specific preconditions are unknown.

Descriptor and destination overlap is not rejected. Repeated descriptor reads mean that overlapping writes can affect later calculations. No synchronization or atomic snapshot mechanism is visible. Byte order and architectural alignment requirements are not established by the supplied representation.

## Behavior

### Initial dependency

The first visible action is a call to F-854a6c2c6fa3 with no explicit arguments.

All direct descriptor reads and destination accesses occur afterward. This dependency’s memory and global-state effects are unknown. In particular, descriptor contents and destination fields must not be assumed unchanged across this call.

### Metadata writes

The function first reads D1, then D2, then D3, retaining those three values. It subsequently reads D0 and writes the sampled D0 value to destination +0x98.

It then performs these writes in order:

1. Retained D1 to +0x9C.
2. Retained D2 to +0xA0.
3. Retained D3 to +0xA4.
4. Auxiliary argument to +0xA8.
5. The single byte 1 to +0xB4.

These actions are neither an atomic descriptor snapshot nor an atomic destination update. The D0 sample occurs after the initial samples of D1 through D3. The writes do not cause the retained samples to be refreshed.

These writes precede storage acquisition. Their occurrence is established; their persistence through later dependency calls is not. This function has no rollback sequence for them.

### Aligned quantity

After writing metadata, the function reads D3 and then D1 again. It retains those two samples across a call to F-8939cedb2807, which receives the format-like argument and returns a signed 32-bit integer.

The arithmetic uses this retained D3/D1 pair, not later descriptor contents. Any descriptor mutation by F-8939cedb2807 can affect subsequent reads but does not replace these retained operands.

For inputs whose signed intermediate operations do not overflow:

- Form the signed difference D3 − D1.
- Multiply it by the signed result from F-8939cedb2807.
- Add 7, obtaining a signed 32-bit intermediate T.
- Divide T by 8 with rounding toward zero, obtaining Q.
- Convert Q to unsigned 32-bit form, add 3 modulo 2³², and clear the lowest two bits.

Call the resulting unsigned 32-bit quantity **S**. Its low two bits are zero.

The division is implemented with sign-dependent adjustment before an arithmetic right shift. Consequently, it is not equivalent to simply shifting every negative T right by three bits.

For an ordinary nonnegative product without signed overflow, S corresponds to rounding the product upward into groups of eight, then rounding that result upward to a multiple of four. This interpretation is not a validation rule: negative operands and zero are not rejected.

The supplied representation does not establish a portable signed-overflow contract. A machine-width interpretation takes the low 32 bits of the subtraction, multiplication, and addition when constructing T. Conclusions requiring such overflow behavior are conditional on that interpretation. The subsequent unsigned addition and masking are 32-bit operations.

S is calculated once. It is not recomputed after either storage dependency, even if descriptor memory or other state changes.

### Existing-storage branch

The function tests the 32-bit field at destination +0xB0.

If the tested value is zero, F-bc7ecdbf25ab is skipped and execution enters fallback.

If the tested value is nonzero, the function calls F-bc7ecdbf25ab with three arguments:

1. A separately read value from destination +0xB0.
2. A requested-size quantity derived from S and fresh descriptor reads.
3. The constant 0x1C.

The requested-size quantity is the unsigned 32-bit product of S and the difference D2 − D0. Subject to the signed-subtraction qualification above, the product is reduced modulo 2³².

The D2 and D0 samples are not taken from the earlier metadata copies. Their relative read order is not established by the supplied expression. Nor is the relative evaluation order between those descriptor reads and the call-argument read of +0xB0.

The +0xB0 test and call-argument read are distinct accesses. There is no intervening dependency call, but no synchronization guarantee is visible.

A nonzero result from F-bc7ecdbf25ab selects final setup and supplies its storage-related argument. Any nonzero 32-bit result qualifies, including one whose signed interpretation is negative. A zero result selects fallback.

On the successful existing-storage branch, this function does not directly clear +0xB0 or store the dependency result into it. The dependency itself may alter that field or other memory.

### Fallback branch

Fallback occurs if the initial +0xB0 test is zero or F-bc7ecdbf25ab returns zero.

The function first writes a 32-bit zero to destination +0xB0. It then calls F-638beded7699 with:

1. S multiplied by a freshly sampled D2 − D0 difference, using the same width rules as the existing-storage request.
2. The constant 0x1C.

The descriptor difference is recomputed; S is reused. Therefore, if both storage calls occur, their requested sizes need not match. F-bc7ecdbf25ab may mutate the descriptor before returning zero. The fallback write to +0xB0 may itself change a descriptor word when the memory regions overlap.

If F-638beded7699 returns zero, this function returns immediately. It performs no final-setup call, restoration, rollback, or further direct write.

This establishes the function’s actions, not the destination’s final contents. F-638beded7699 and earlier dependencies have unknown memory effects and may overwrite the metadata, auxiliary field, flag, or +0xB0. A zero fallback result therefore does not guarantee that any earlier written value remains present at return.

If F-638beded7699 returns nonzero, that result is passed to final setup. This function does not directly store it into +0xB0.

### Final setup

After either storage dependency returns nonzero, the function calls F-940bdd7a50cc with seven arguments, in this positional order:

1. Destination +4.
2. Original descriptor address.
3. Successful nonzero storage-related result.
4. S.
5. Original format-like argument.
6. Original destination address.
7. Original final-context argument.

The descriptor is passed by reference, not as the values sampled earlier. Final setup may therefore encounter descriptor contents different from the metadata samples or sizing samples.

This function does not inspect a return value from final setup and performs no subsequent direct writes. It then returns.

## Side effects

The direct writes consist of the metadata transfer, auxiliary assignment, one-byte flag assignment, and conditional zeroing of +0xB0. Dependencies may add arbitrary effects not constrained by the supplied body.

The following distinctions are required:

- Metadata and flag writes occur before a successful storage result is known.
- On fallback, the direct zero write to +0xB0 occurs before F-638beded7699 is called.
- After F-638beded7699 returns zero, this function performs no rollback.
- Earlier writes are not guaranteed to persist through dependency calls. In particular, a zero fallback result does not establish final metadata, flag, auxiliary, or +0xB0 contents.
- No successful storage result is directly assigned to +0xB0 by this function.
- No direct release operation, diagnostic, explicit error return, or exception handler is visible.

Ownership is unresolved. A zero result from F-bc7ecdbf25ab does not establish whether the old storage remains valid, has been released, or is still owned by the destination. Clearing +0xB0 is a field write, not evidence of deallocation.

Likewise, passing a nonzero result to final setup does not establish borrowing, adoption, retention, or release. No dependency may be assumed to preserve globals or memory merely because it receives no explicit pointer to that memory.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-854a6c2c6fa3 | Called first, once, without explicit arguments. | Purpose, memory and global effects, failure behavior, and whether it returns. |
| F-8939cedb2807 | Receives the format-like value; its signed 32-bit result participates in sizing. | Valid inputs, units, result range, side effects, and error convention. |
| F-bc7ecdbf25ab | Called only after a nonzero +0xB0 test; receives an existing value, requested size, and 0x1C. Zero selects fallback. | Storage operation, ownership, mutations, and behavior on failure. |
| F-638beded7699 | Called on fallback after a direct zero write to +0xB0; receives requested size and 0x1C. Zero selects immediate return. | Allocation semantics, zero-size handling, initialization, ownership, and memory effects. |
| F-940bdd7a50cc | Called once after a nonzero storage result, with the seven arguments specified above. | Final state, ownership, validation, mutation, and failure reporting. |

The specified sequence describes calls that are reached and return normally. No handling is visible for a dependency that does not return normally.

## Acceptance examples

These are conditional static deductions, not executed tests. Except where an example explicitly permits mutation, assume valid nonoverlapping memory, stable descriptor contents, normally returning dependencies, and no dependency memory effects.

### Ordinary fallback success

Given descriptor (10, 20, 15, 29), a sizing result of 8, and initial +0xB0 equal to zero:

- The metadata writes supply 10, 20, 15, and 29.
- The auxiliary argument is written, followed by flag byte 1.
- The first difference is 29 − 20 = 9.
- The product plus 7 is 79.
- Division by 8 toward zero gives 9.
- Four-unit alignment produces S = 12.
- The requested size is 12 × (15 − 10) = 60.
- F-bc7ecdbf25ab is skipped.
- The function writes zero to +0xB0, then calls F-638beded7699 with (60, 0x1C).
- A nonzero fallback result is passed to final setup with S = 12.

Under the no-mutation assumption, the directly written metadata and flag remain present. That persistence is not unconditional.

### Existing-storage success

With the same arithmetic inputs, a nonzero initial +0xB0 value, and a nonzero result from F-bc7ecdbf25ab:

- The existing-storage call receives requested size 60.
- F-638beded7699 is not called.
- Final setup receives the returned storage-related value.
- This function performs no direct clear or replacement of +0xB0 on this branch.

### Fallback failure without dependency mutation

If fallback returns zero under the baseline no-mutation assumptions, final setup is skipped. The earlier metadata, auxiliary, and flag writes remain observable, and +0xB0 contains zero.

This retained state depends on the stated assumption. The unconditional requirement is only that this function performs no rollback or further direct write after the zero result.

### Fallback failure with dependency mutation

Allow F-638beded7699 to overwrite the metadata and +0xB0 before returning zero.

The required behavior is immediate return without final setup or restoration. The contract does not require the metadata to retain the copied descriptor or +0xB0 to remain zero.

### Changed descriptor between requests

Suppose S = 12 and the existing-storage request samples D2 − D0 = 5. If F-bc7ecdbf25ab changes that difference to 7 and returns zero, the two requests are 60 and 84, provided the intervening clear causes no further descriptor change.

S remains 12.

### Zero and negative arithmetic

If D2 − D0 is zero, the requested size is zero. The function still follows the applicable storage path; dependency behavior determines the outcome.

With S = 12 and a non-overflowing signed difference of −1, unsigned multiplication produces 0xFFFFFFF4. There is no clamp or negative-size rejection.

If the first product is −8, adding 7 gives −1. Division toward zero gives 0, and alignment yields S = 0. This distinguishes the observed division rule from an unadjusted arithmetic right shift.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the descriptor, format-like value, auxiliary field, flag, and constant 0x1C.
- Supported inputs and intended signed-overflow behavior.
- Dependency implementations, memory effects, global effects, ownership, and release behavior.
- Final destination contents after dependencies execute.
- Whether final setup stores a storage result into +0xB0.
- Architectural byte order, alignment requirements, and address-wrap behavior.
- Supported aliasing arrangements and concurrency guarantees.
- Failure mechanisms other than the visible zero/nonzero storage results.

Rectangle handling, format-dependent stride calculation, and buffer reuse remain inferred intent only.

## Evidence

The supplied static function representation supports the access widths and offsets, ordered direct writes, descriptor resampling, arithmetic structure, zero/nonzero branches, dependency arguments, and lack of direct rollback described above.

It does not supply dependency implementations or establish their preservation, ownership, or allocation contracts. In particular, the absence of rollback cannot establish that earlier writes survive dependency execution.

No implementation access, runtime execution, or tests were performed. Acceptance examples are conditional arithmetic and control-flow deductions, not runtime observations.
