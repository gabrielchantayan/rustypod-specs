<!-- Trusted provenance: F-a97edcae14bb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a97edcae14bb

## Purpose

F-a97edcae14bb coordinates an object-associated update using an opaque 32-bit value, two 32-bit controls, and an 8-bit value.

Its established behavior has two paths:

- When an object status byte is nonzero, it stores the 32-bit input in the object and returns without calling any dependency.
- Otherwise, it stores the byte input, resolves an associated value, performs ordered dependency calls, prepares local temporary storage, conditionally invokes another operation, stores the original 32-bit input, and passes the temporary storage to a final dependency.

The conditional operation can require a 16-bit object-relative read. The displacement for that read is itself obtained by reading global backing storage at that point in execution. It is not established as an immutable constant.

“Resolution,” “context,” “preparation,” and “finalization” below describe call-site roles. Domain interpretations such as notification, selection change, or resource release are not established.

## Interface

### Inputs

The function has five positional inputs:

| Position | Role | Width and interpretation |
|---|---|---|
| 1 | Object address | A 32-bit address-bearing value in the supplied representation. Used for fixed-offset accesses and a conditionally computed address. |
| 2 | Associated value | An opaque 32-bit value supplied to a dependency and later stored unchanged. |
| 3 | Forwarded control | A 32-bit value tested for zero and forwarded unchanged to the conditional operation. |
| 4 | Suppression control | A 32-bit value tested for zero versus nonzero. |
| 5 | Byte value | An opaque 8-bit value stored on the longer path. |

The controls need not be normalized booleans. Every nonzero bit pattern satisfies their nonzero tests, including values that would be negative under a signed interpretation.

### Result

There is no explicit return value. Observable results consist of memory accesses, dependency calls, and their ordering.

A zero resolution result leads to a dependency annotated as nonreturning. No recoverable failure result is provided by this function.

### Directly accessed storage

| Storage | Width | Use |
|---|---:|---|
| Object offset `0xFC` | Address only | Its address is passed to F-1f46730b1ced; this function does not directly read its contents. |
| Object offset `0x12C` | 32 bits | Receives the second input on either normal-return path. |
| Object offset `0x130` | 8 bits | Receives the fifth input when the entry status is zero. |
| Object offset `0x134` | 8 bits | Read at entry to select the path. |
| First context result plus `0x2C` | 8 bits | Supplies one input to temporary preparation. |
| Second context result plus `0x28` | 32 bits | Supplies another input to temporary preparation. |
| Global displacement backing storage | Not independently established | Conditionally read to obtain the displacement for an object-relative load. |
| Object address plus the just-read displacement | 16 bits | Conditionally supplies the final argument of F-818c773e8c8a. |
| Invocation-local temporary storage | 92 bytes reserved | Passed to preparation, the conditional operation, and finalization. |

The global displacement’s storage location, declared type, access width, and signedness are not supplied. Its role in address formation is established; those additional properties are not.

### Preconditions and representation limits

Each executed memory access requires valid storage for its access width. Context results used for field reads must identify suitable storage. When the conditional object read executes, both the global displacement storage and the resulting object-relative address must be readable.

The fixed object accesses extend through byte offset `0x134`. This does not establish a complete allocation size: the conditional displacement and dependency behavior can require other storage.

The supplied representation uses 32-bit address-bearing values. The fragment does not establish portable semantics for address overflow, misalignment, invalid addresses, or arithmetic outside the target’s usable address range. The displacement’s missing type also prevents a complete account of its extension or conversion during address formation.

No local null checks, bounds checks, synchronization, or alignment recovery are visible. Endianness and precise calling-convention details are not established.

## Behavior

### Entry decision and short path

The first explicit object access is an 8-bit read at offset `0x134`.

If this byte is nonzero, the function writes the second input as 32 bits at offset `0x12C` and returns. It makes no dependency calls and does not directly write offset `0x130`.

Neither control affects this path. There are no context acquisitions, global displacement reads, conditional 16-bit loads, or temporary-storage dependency calls.

The status byte is not read again on either path. A later dependency changing that byte does not cause this invocation to revisit its entry decision.

### Initial longer-path write and resolution

If the entry status is zero, the function first writes the fifth input as one byte at object offset `0x130`.

It then calls F-1f46730b1ced with:

1. The object address plus `0xFC`.
2. The original second input.

The returned value is a 32-bit result tested against zero. Its use does not establish whether it represents a pointer, handle, index, or another kind of value.

When the result is zero, F-a817d8d58c10 is invoked without explicit arguments. That dependency is annotated as nonreturning. Under that contract, execution does not reach the subsequent operations, final 32-bit object write, or temporary finalization.

The earlier byte write has already occurred. No rollback or recoverable error status is visible. The evidence does not establish whether the nonreturning dependency terminates execution, throws, or transfers control by another mechanism.

### Operations following successful resolution

For a nonzero resolution result, the function performs these calls in order:

1. F-d2f99870cc43 receives the resolution result and produces an opaque 32-bit result.
2. F-bf557f07179f receives the original object address and that derived result.
3. F-fa8898fa75ae receives the original resolution result.

No result from the third call is used. The original resolution result is retained for the later conditional operation.

These calls do not establish ownership transfer, destruction, or release. In particular, F-fa8898fa75ae cannot be specified as freeing the resolution result from this evidence. If later use requires an underlying resource to remain valid, that is a dependency-level requirement not resolved here.

### Temporary preparation and ordered context reads

The invocation provides 92 bytes of local temporary storage. No direct initialization of its contents is shown before it is passed to F-2c5a9f9c1eb3.

The preparation inputs are obtained in this order:

1. Call F-fdd8589eca2d.
2. Read one byte at offset `0x2C` from its result and retain that byte.
3. Call F-fdd8589eca2d again.
4. Read 32 bits at offset `0x28` from this second result.
5. Call F-2c5a9f9c1eb3 with the temporary-storage address, the 32-bit value from the second context, and the byte retained from the first context.

The first byte read therefore occurs before the second context acquisition. Side effects of the second acquisition do not cause that byte to be reread.

The two context results need not be equal. Replacing the two calls with a single acquisition is not justified by the supplied evidence.

No preparation result is checked. The storage’s internal layout, initialized extent, and resource obligations remain unspecified.

### Mode-dependent operation

After temporary preparation, the function calls F-5c5958cfd5fe and compares its 32-bit result with exactly `8`.

If the result equals `8`, the function skips all of the following:

- A third call to F-fdd8589eca2d.
- The global displacement read.
- The displacement-based 16-bit object read.
- F-818c773e8c8a.

It then proceeds to the final object write and finalization.

For every mode result other than `8`, the function calls F-fdd8589eca2d a third time. This occurs regardless of either control’s value. Its result becomes the first argument of the conditional operation.

The operation’s final argument is selected as follows:

| Fourth input | Third input | Final argument | Global displacement and object reads |
|---|---|---|---|
| Nonzero | Any | 16-bit zero | Neither occurs |
| Zero | Zero | 16-bit zero | Neither occurs |
| Zero | Nonzero | 16-bit value from the computed object address | Both occur, in that order |

In the last row, the ordered accesses are specifically:

1. Complete the third context-acquisition call.
2. Read the displacement value from its global backing storage.
3. Form the address from the original object address and that displacement.
4. Read 16 bits at the resulting address.

The global read is conditional, not an entry-time acquisition or a fixed constant substitution. It occurs only when the mode result is not `8`, the fourth input is zero, and the third input is nonzero.

Dependencies executed before this read may change the displacement’s backing storage. This includes the third context-acquisition call itself. The displacement used must therefore be the value observed at the specified read point; caching a value before those calls is not behaviorally justified without an additional immutability or equivalence contract.

Likewise, a dependency may change the object storage before the eventual 16-bit read. The source does not establish a snapshot spanning these calls and reads.

F-818c773e8c8a then receives:

1. The third context-acquisition result.
2. The local temporary-storage address.
3. The original resolution result.
4. The third input, unchanged.
5. The selected 16-bit value.

The full third input is forwarded even though only its zero/nonzero status controls the optional read. No result from the operation is checked.

### Final write and finalization

After the conditional operation, or immediately after its mode-based omission, the function:

1. Writes the original second input as 32 bits at object offset `0x12C`.
2. Calls F-920630e53283 with the local temporary-storage address.
3. Returns if that dependency returns normally.

The object write precedes finalization. A finalization dependency with access to the object by another route can observe it.

This establishes a write event, not a guaranteed value at return. Finalization may have unknown side effects, and there is no subsequent local restoration or verification.

## Side effects

The function’s explicit writes to object storage are limited to:

- One byte at offset `0x130` on the longer path.
- The original 32-bit second input at offset `0x12C` on either normal-return path.

Its explicit reads include the entry status byte, two context fields, and—only under the three required conditions—the global displacement followed by a 16-bit object-relative load.

The function does not explicitly write the global displacement storage. That does not imply the storage remains unchanged: dependency side effects are unspecified.

Dependencies receive object-related addresses, opaque values, context results, and local temporary storage. Their writes, allocations, releases, retained references, global effects, and reentrant behavior are not established.

The temporary storage belongs to this invocation. No direct heap allocation or deallocation is visible. The local call sequence does not establish that dependencies avoid either operation or that they may safely retain the temporary-storage address after return.

There is one visible finalization call after preparation and continued normal execution. No local exception handler or guaranteed cleanup path is shown for faults, nonlocal exits, or dependencies that fail to return.

The routine is not established as atomic or thread-safe. In particular, the conditional displacement read and subsequent object read are separate accesses without visible synchronization.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-1f46730b1ced | Receives an object-relative address and the second input; returns the value tested for resolution failure. |
| F-a817d8d58c10 | Called on a zero resolution result; annotated as nonreturning. |
| F-d2f99870cc43 | Derives an opaque 32-bit value from the nonzero resolution result. |
| F-bf557f07179f | Receives the object address and derived value. |
| F-fa8898fa75ae | Receives the original resolution result before temporary preparation. |
| F-fdd8589eca2d | Supplies two context results for field reads and, when the mode is not `8`, a third result for the conditional operation. |
| F-2c5a9f9c1eb3 | Receives local temporary storage, a 32-bit context field, and an 8-bit context field. |
| F-5c5958cfd5fe | Supplies the mode value compared with exactly `8`. |
| F-818c773e8c8a | Receives the five arguments of the conditional operation. |
| F-920630e53283 | Receives temporary storage after the final object write. |

These are call-site observations, not complete dependency contracts. None establishes preservation of globals, object contents, or resource lifetimes.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. All referenced accesses are assumed valid, and dependencies are assumed to return normally except where stated otherwise.

### Nonzero entry status

With an entry status of `0x80`, the function writes the second input at `0x12C` and returns. It calls no dependencies, does not directly write `0x130`, and does not read the global displacement.

### Resolution failure

With zero entry status and a zero result from F-1f46730b1ced, the byte input is written at `0x130` before F-a817d8d58c10 is called. If the latter obeys its nonreturning annotation, no temporary preparation or final 32-bit object write follows.

### Mode equals eight

With successful resolution and mode `8`, both context field reads and temporary preparation still occur. There is no third context acquisition, global displacement read, conditional object read, or F-818c773e8c8a call. The final object write precedes F-920630e53283.

### Suppressed field selection

With mode other than `8` and a nonzero fourth input, the third context acquisition occurs and F-818c773e8c8a receives a zero final argument. Neither displacement-related read occurs. A nonzero third input is still forwarded unchanged.

With the fourth and third inputs both zero, the same read omissions and zero final argument apply.

### Displacement changed by a dependency

Suppose the mode is not `8`, the fourth input is zero, and the third input is nonzero. If the third context acquisition changes the global displacement from one valid displacement to another before returning, the subsequent global read supplies the updated value for address formation. An implementation using a displacement cached before that call would not satisfy this ordering.

This example assumes no further change intervenes between the dependency’s update and the global read; concurrent modification semantics are not established.

### Selected 16-bit pattern

Under the same enabling conditions, if the computed object-relative read yields `0xFEDC`, the operation receives that 16-bit pattern as its final logical argument. The fragment does not establish its wider ABI representation or a signed numerical interpretation.

### Distinct contexts and finalization effects

If the context acquisitions return different valid addresses, preparation uses the byte from the first and the 32-bit field from the second. The conditional operation receives the third result when applicable.

If finalization changes the recorded object field after the function’s final write, no local write restores the original second input afterward.

## Uncertainties

The following remain **documented_only**:

- The global displacement storage’s location, declared width, signedness, value range, and mutability rules. The conditional read and its ordering are established; immutability is not.
- Exact address-arithmetic conversion, overflow, alignment, and endianness behavior.
- The conditional 16-bit field’s location and possible overlap with other storage.
- Complete object layout, allocation requirements, and ownership.
- Dependency behavior beyond visible arguments, result use, and sequencing.
- Context stability and dependency modifications to globals or object storage.
- Lifetime effects of F-fa8898fa75ae on the resolution result.
- Temporary-storage layout, initialized extent, ownership, and retained-reference rules.
- Finalization’s resource effects and possible writes to the object or globals.
- The ABI representation of the logical 16-bit final argument.
- The nonreturning dependency’s mechanism and behavior if its annotation is not honored.
- Fault handling, reentrancy, concurrency safety, and cleanup after abnormal exits.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. Established evidence includes explicit access widths where shown, fixed offsets, input forwarding, branch conditions, dependency ordering, and the 92-byte local storage extent.

The displacement expression establishes a conditional global-value read feeding an object-relative 16-bit load. Its position is after the third context acquisition, not before the dependency sequence. The supplied material does not establish the global value’s type or immutability.

Nonreturning behavior is supported by an annotation rather than a dependency body. Domain intent, ownership, global preservation, and dependency internals are not established.

No runtime tests have been performed. The acceptance examples describe conditional observations derived from static evidence, not observed executions.
