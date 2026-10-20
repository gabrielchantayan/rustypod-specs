<!-- Trusted provenance: F-2cfb687bb8ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cfb687bb8ab

## Purpose

F-2cfb687bb8ab is a minimal indirect-dispatch entry point. Its established behavior is to obtain a dispatch base from memory addressed by its sole input, resolve a callable target at byte offset 20 from that base, and transfer control to that target.

The supplied representation shows no argument expressions at the indirect invocation and no returned value from this entry point. It does not establish the target’s actual calling convention, its effective inputs, or whether the transfer is implemented as a conventional call or a tail transfer.

A table-based object dispatch is a plausible interpretation, but object identity, class structure, method identity, and receiver passing are not established. The contract therefore describes the observable memory accesses and control transfer without requiring an object-oriented interpretation.

## Interface

**Identifier:** F-2cfb687bb8ab.

**Explicit input:** One pointer to an integer-valued memory location. That location supplies the base used to resolve the indirect target; it is not itself shown to contain the target address.

**Explicit output:** None. The represented interface has no return value. A target’s possible return value is not explicitly inspected, transformed, or returned as a defined result by this entry point.

**Input layout:**

| Location | Represented content | Role |
|---|---|---|
| Address supplied as the input | One integer value | Supplies the dispatch base |
| Dispatch base plus 20 bytes | A callable-address slot | Supplies the indirect target |

The offset is hexadecimal 14, equivalent to decimal 20. It is an address displacement, not an index multiplied by the size of the input’s integer type.

**Widths and representation:** The recovered interface represents the first loaded value as a signed integer. A 32-bit signed value is the conventional interpretation of this recovered form, but the supplied material does not independently establish the compilation data model. It also does not explicitly establish the width of a callable-address slot.

Consequently, a byte-exact implementation contract requires additional ABI evidence for:

- The integer width and signed representation used for the initial load.
- The width of the indirect-target address.
- How the integer-derived address is interpreted.
- The indirect target’s calling convention.

Under a separately established 32-bit model, the initial value and a conventional callable-address slot would each occupy four bytes. That is a conditional layout, not an independently verified property of this entry point.

**Successful-dispatch preconditions:**

- The input identifies readable storage for the initial integer load.
- Adding 20 to the loaded value produces a usable address under the actual machine’s arithmetic and address rules.
- The resulting location contains a readable target-address representation.
- That representation identifies a destination compatible with the actual control-transfer convention.

These are conditions for a successful dispatch, not conditions checked by this function. No input validation is shown.

## Behavior

### Target resolution

The first represented data access reads the integer at the input address. That value becomes the dispatch base.

The next represented operation forms an address by adding 20 to that value. There is no shown mask, normalization, bounds check, table-length comparison, or alternative offset.

The function then obtains a callable target through the slot at the computed address. The established data dependency is therefore:

1. Read the base from the input-addressed storage.
2. Resolve the target slot at base plus 20.
3. Read the target address from that slot.
4. Transfer control to that target.

This describes the logical access order in the supplied representation. It does not claim a particular instruction sequence, atomicity, or hardware memory-ordering behavior.

The additional dereference notation used to invoke a function does not establish a second data-pointer link beyond the target-address slot. The evidence supports one initial base read and one target-slot read, followed by control transfer.

### Invocation

The indirect invocation contains no explicit argument expressions. In particular, the input pointer is not explicitly supplied as an argument to the resolved target.

This does not establish that the target receives no effective inputs. Register contents, stack contents, an implicit receiver, or other calling-convention state may matter. None of those details can be recovered reliably from the supplied representation alone.

There is one represented indirect invocation. No retry, alternate-target selection, loop, or local fallback is shown.

### Completion

After an ordinary return from the indirect invocation, the represented entry point returns without producing a defined value.

That completion is conditional on the target returning normally. If the target terminates execution, faults, performs a nonlocal transfer, or never returns, this entry point does not independently guarantee a normal return.

The supplied recovery warning identifies uncertainty in the control-transfer reconstruction. The underlying operation may be a transfer whose continuation differs from a conventional nested call. Accordingly, the strongest established requirement is target resolution followed by an indirect transfer; precise stack and return-address effects remain unknown.

### Arithmetic and boundary cases

The displacement is fixed at positive 20. No special treatment is shown for zero, negative, or unusually large base values.

The representation performs integer addition before interpreting the result as a location containing a callable address. It does not establish whether an overflowing addition wraps in the underlying machine, is constrained by an address width, or is merely an artifact of the recovered expression. A specification must not silently replace that uncertainty with guaranteed unsigned modular arithmetic.

Likewise, neither alignment requirements nor support for unaligned reads are established. A candidate input that violates the actual platform’s requirements has no documented successful outcome here.

## Side effects

**Direct memory effects:** The supplied body shows reads but no explicit writes to input storage, the target slot, global storage, or any other data location.

**Indirect effects:** The selected target may read or modify memory, alter global state, allocate or release resources, perform external I/O, or transfer control elsewhere. No target body or target contract is supplied, so none of these effects can be ruled out or required.

**Ownership:** This entry point shows no explicit allocation, release, ownership transfer, or reference-count operation. Its initial use of the input is a read. That observation does not imply that the resolved target preserves the input, preserves the dispatch storage, or refrains from releasing related resources.

**Errors:** No explicit error result, diagnostic, exception handler, or recovery path is shown. Invalid input storage, an unreadable slot, an invalid target, or an incompatible calling convention may prevent completion. The exact failure mechanism is not established.

**Concurrency:** No synchronization, atomic operation, or snapshot mechanism is shown. The two dependent reads are not documented as a transaction. Behavior under concurrent mutation of the input-addressed value or target slot is unknown.

## Dependencies

The private dependency map identifies only F-2cfb687bb8ab itself. It supplies no opaque identifier for the dynamically selected target.

| Dependency | Established use | Undocumented properties |
|---|---|---|
| Unidentified runtime-selected target | Receives control through the slot at dispatch base plus 20 | Identity, signature, effective arguments, return convention, memory effects, ownership effects, global-state effects, and failure behavior |

No named helper, allocator, validator, logging routine, or error handler appears in the supplied body.

The dynamic target is a real dependency despite lacking an identifier. Assigning it a fabricated opaque function ID would imply provenance not present in the evidence.

## Acceptance examples

These examples are conditional contract illustrations, not executed tests.

### Ordinary dispatch

Given readable input storage containing a valid dispatch base, and a valid callable address stored at base plus 20, the entry point resolves that slot and transfers control to its target.

If the target returns normally and the recovered call-and-return model is accurate, the entry point then returns without a defined result value.

### Distinguishing the slot offset

Given callable targets at both base plus 16 and base plus 20, with all relevant accesses valid, the represented behavior selects the target at base plus 20. The presence of another callable address nearby does not change the fixed displacement.

This example establishes slot selection, not a complete table layout or the number of preceding entries.

### Changed dispatch base

Given two separate invocations whose input-addressed storage contains different valid bases when read, each invocation resolves its target using the base read for that invocation. The supplied body shows no retained target or cross-invocation cache.

### Target modifies state

Given a resolved target that changes global state or releases a resource, those effects are compatible with the available contract. An acceptance criterion requiring preservation would need additional evidence about that target.

The absence of direct writes in F-2cfb687bb8ab is insufficient to require overall purity.

### Target does not return

Given a target that does not return normally, normal completion of this entry point is not required. No local timeout, cancellation, or recovery behavior is shown.

### Invalid input or target

Given an unreadable input address, an unusable base-plus-20 address, or an invalid callable address, no specific error value or graceful rejection is required by the evidence. These cases are documented_only until platform and runtime behavior are established.

## Uncertainties

The following remain **documented_only**:

- Exact integer, pointer, and callable-address widths.
- Integer-to-address interpretation and overflow behavior.
- Alignment and readable-region requirements.
- Whether the initial value represents an object dispatch table, another table, or an unrelated address structure.
- The dynamic target’s identity and full interface.
- Whether the original input is available to the target through implicit machine state.
- Whether the indirect transfer is a conventional call or a tail transfer.
- Stack adjustment, register preservation, and return-address behavior.
- Target-induced writes, resource ownership changes, and global-state changes.
- Fault, exception, and nonlocal-control-transfer behavior.
- Behavior during concurrent mutation.

These unknowns do not justify adding validation, fallback dispatch, ownership guarantees, or a particular ABI to the contract.

## Evidence

**Established static evidence:** The supplied recovered body represents one pointer input, one initial integer dereference, a fixed displacement of 20, a target-address lookup, an indirect invocation with no explicit argument expressions, and a return without a value. It shows no explicit data writes or local error handling.

**Recovery limitation:** The supplied warning reports unresolved indirect-control-flow reconstruction and states that an indirect jump was treated as a call. This limits confidence in invocation and return mechanics; it does not justify inventing additional branch destinations.

**Inferred intent:** Fixed-offset dispatch through a base read from input storage is consistent with a dispatch-table entry point. That interpretation is useful context, not an established object model.

**Validation status:** No runtime tests have been performed. No target implementation, ABI documentation, independent disassembly, or execution trace was supplied. All acceptance examples are conditional consequences of the static representation.
