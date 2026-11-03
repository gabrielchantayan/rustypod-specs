<!-- Trusted provenance: F-b9a9351bdaff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9a9351bdaff

## Purpose

F-b9a9351bdaff conditionally performs an operation on an indexed object reached through global dispatch state.

Its established behavior is:

- Retain a globally stored cell address and initialize two temporary words.
- Independently read the global cell address again to resolve a bound-producing operation.
- Compare the original index with that operation’s result using a strict unsigned comparison.
- On success, resolve an indexed lookup through the retained cell, obtain a selected object, and invoke an operation on it.
- Pass the temporary storage through two direct dependencies, then pass the selected object to a final dependency marked non-returning in the supplied representation.

The distinct reads of global and cell state are material. The function does not establish a single consistent snapshot for the bound operation, lookup target, and lookup receiver.

A possible intent is to obtain a value from an indexed object, apply it to a destination, and then clean up temporary and object state. That interpretation is **inferred intent**, not an established ownership or lifecycle contract.

This document describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments

The function has four arguments, in this order:

| Position | Width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the displayed body. |
| Second | 32 bits | No explicit use in the displayed body. |
| Third | 32 bits | An unsigned index, used for comparison and lookup and copied into the first temporary word. |
| Fourth | 32 bits | An opaque value, copied into the second temporary word and later passed independently to F-46f0b3cb316e. |

The fourth argument’s semantic type is unknown. It must not be specified as a destination pointer, handle, or scalar with additional meaning without further evidence.

The first two arguments are not explicitly forwarded. This does not establish a complete machine-level calling convention or rule out implicit argument mechanisms outside the displayed expressions.

### Result and termination

The function declares no return value.

If the index fails the unsigned comparison, the function reaches its ordinary return without performing the indexed lookup or subsequent calls.

On the successful branch, the last dependency is F-feee9156a257. The supplied representation annotates that dependency as non-returning. This establishes the annotation, not its runtime accuracy or mechanism.

If that annotation accurately describes the dependency, the successful branch does not return normally. If the dependency instead returns normally, the displayed body contains no subsequent operation other than the trailing void return. Process termination, exception propagation, nonlocal transfer, and indefinite waiting are not individually established.

### Temporary storage and representation

The function initializes two 32-bit temporary words:

1. The first receives the original index.
2. The second receives the original fourth argument.

The supplied stack representation places the second word four bytes after the first. The observed initial footprint is therefore two adjacent four-byte words. This does not establish a declared eight-byte structure, a semantic field layout, or permission for a dependency to access any particular extent beyond what its unknown contract allows.

Every explicit temporary-address argument designates the first word. The second word is never passed by its own address.

The representation uses 32-bit words for scalar values and address-bearing object fields. Dispatch offsets are byte offsets. A complete ABI, pointer model, alignment contract, and byte order are not independently supplied.

### Preconditions

The body provides no explicit validation of:

- The global cell addresses it reads.
- The object references obtained through those cells.
- Dispatch-table addresses or callable entries.
- The object address returned by the indexed lookup.

Every dereference and indirect invocation that is reached requires suitable readable state and a callable target. Requirements for the lookup’s explicit receiver and for the direct dependencies remain dependent on their unknown contracts.

Passing the range comparison does not itself establish any of these validity conditions.

## Behavior

### Initial capture and temporary initialization

The function first reads the global location containing a cell address and retains the resulting address locally.

It then writes the index into the first temporary word and the fourth argument into the second temporary word, in that order.

At this stage, the retained value is the **cell address**, not a captured reference to the object stored in that cell.

### Bound-target resolution uses a fresh global read

After initializing the temporary words, the function evaluates the global cell-address location again.

Using this freshly obtained cell address, it:

- Reads an object reference from the cell.
- Reads that object’s dispatch-table address.
- Resolves the entry at byte offset **0x30**.
- Invokes that entry, identified here as **F-indirect-0030**.

The retained cell address from the initial read is not the address expression used for this bound-target resolution.

The two global reads can yield equal cell addresses under a stability assumption, but their equality is not an independently established invariant. The function contains no explicit global write between them; that fact does not establish synchronization or exclude external modification.

F-indirect-0030 has no explicit arguments in the displayed invocation. In particular, the object used to locate the target is not explicitly passed as a receiver. Any implicit receiver convention remains unknown.

Its result is used as an unsigned 32-bit comparison bound. Calling it a collection count would be inferred intent rather than a proven semantic type.

### Unsigned range decision

The function compares the original third argument with the returned bound:

- An index strictly less than the bound enters the successful branch.
- An index equal to or greater than the bound takes the ordinary return path.

The comparison uses the original argument, not a read from temporary storage.

All 32 bits participate in the unsigned comparison. A high-bit-set index is a large unsigned value, not a negative value. A zero bound rejects every index. Even a bound of 0xFFFFFFFF rejects index 0xFFFFFFFF.

There is no explicit index adjustment, scaling, or arithmetic in this body. Any indexed addressing performed inside a dependency is outside the available evidence.

### Indexed lookup uses the retained cell

On the successful branch, the function uses the cell address retained by the **first** global read. It does not reread the global cell-address location for this lookup.

The invocation at dispatch offset **0x34**, identified as **F-indirect-0034**, contains two separate accesses through that retained cell:

- One reads an object reference for target resolution, then reads its dispatch-table address and the entry at offset 0x34.
- Another reads an object reference to supply the first explicit argument.

The second explicit argument is the original unsigned index.

The supplied expression does not establish an evaluation order between the target expression and the explicit receiver argument. Neither cell read should be specified as necessarily occurring first.

The references obtained by those reads are not unconditionally identical. With stable cell contents, they refer to the same object. Without that assumption, the target can be selected through one object while the explicit receiver argument comes from another.

Consequently, the established explicit argument order is:

1. The object reference read through the retained cell for the receiver argument.
2. The original index.

It is not established that this receiver is the object whose dispatch table supplied the target.

F-indirect-0034 returns a value subsequently used as the selected object’s address. The function performs no second bound check, no validation of this result, and no reconciliation between the state used for the bound and the state used for lookup.

### Selected-object operation

Using the selected object address, the function reads its dispatch-table address and resolves the entry at byte offset **0x14**, identified as **F-indirect-0014**.

The explicit arguments, in order, are:

1. The address of the first temporary word.
2. The selected object address.

No return value from this call is explicitly consumed.

The function itself has not rewritten either temporary word since initialization. Earlier dependencies have not received the temporary address explicitly, but their unknown behavior does not justify a universal claim that ambient state or accessible memory is preserved.

F-indirect-0014 receives writable local storage. The meaning of any contents it produces, and the extent it may validly access, are not defined by this caller.

### Direct dependency sequence

After F-indirect-0014 returns normally, the function invokes these dependencies in order:

| Dependency | Explicit arguments |
|---|---|
| F-46f0b3cb316e | Original fourth argument; address of the first temporary word |
| F-289ec4311c64 | Address of the first temporary word |
| F-feee9156a257 | Previously selected object address |

The first argument to F-46f0b3cb316e comes from the original fourth argument, not from the second temporary word. Modifying that word does not change this by-value argument.

The temporary storage is not reinitialized between calls. F-289ec4311c64 receives the same address after F-46f0b3cb316e has had an opportunity to modify its contents.

The final call uses the previously selected object address. There is no new lookup, dispatch-state refresh, or validity check before that call. If an earlier dependency invalidates the object, the caller does not visibly compensate.

Each later call is conditional on preceding dependencies returning normally. No explicit recovery or alternate sequencing appears.

## Side effects

### Explicit writes

The only explicit data writes in the body initialize the two temporary words.

There is no explicit assignment to global storage or object fields. This does not imply purity: every dependency may have effects not specified by the available body.

### Observable access relationships

The established source-level access sequence is:

1. Read and retain the global cell address.
2. Initialize the first temporary word.
3. Initialize the second temporary word.
4. Read the global cell address again and resolve F-indirect-0030 through that freshly obtained cell.
5. Invoke F-indirect-0030 and compare its result with the original index.
6. If accepted, evaluate the target and arguments for F-indirect-0034 through the retained cell, including separate cell reads for target resolution and receiver formation.
7. Invoke the lookup, then resolve and invoke the selected object’s operation.
8. Invoke the three direct dependencies in their stated order.

Within step 6, no relative evaluation order is established between the target expression and receiver argument.

These are properties of the displayed access expressions, not a verified count of emitted machine loads or a guarantee of volatile access semantics.

The function provides no visible snapshot, locking, or consistency mechanism across these accesses.

### Ownership and lifetime

No allocation or release primitive is explicitly established by the caller.

In particular:

- The selected object’s ownership is unknown.
- The temporary words’ contents may or may not represent resources.
- F-289ec4311c64 is not proven to be a destructor.
- F-feee9156a257 is not proven to release the selected object.
- No dependency is proven to preserve globals, cell contents, or object validity.

The temporary address refers to storage belonging to this invocation. The evidence establishes no permission for retaining and dereferencing that address after the storage’s lifetime ends.

### Errors

The function defines no explicit error result, null-result branch, retry, or recovery path.

Dependency failures and invalid memory or dispatch state have no locally specified handling. Their consequences must remain unknown rather than being assigned a particular fault, exception, or termination behavior.

## Dependencies

| Opaque identifier | Established role |
|---|---|
| F-indirect-0030 | Target at offset 0x30, resolved using a fresh global cell-address read; invoked without explicit arguments; supplies the unsigned bound. |
| F-indirect-0034 | Target at offset 0x34, resolved through the retained cell; receives a separately read receiver reference and the original index; returns the selected object address. |
| F-indirect-0014 | Target at offset 0x14 on the selected object; receives the temporary address followed by the selected object address. |
| F-46f0b3cb316e | Receives the original fourth argument and temporary address. |
| F-289ec4311c64 | Receives the temporary address after F-46f0b3cb316e returns. |
| F-feee9156a257 | Receives the selected object address last; annotated as non-returning. |

The indirect identifiers designate unresolved call sites, not proven globally unique functions. Different dispatch state can select different implementations.

No dependency body or independent dependency contract is available here.

## Acceptance examples

These are conditional obligations derived from static evidence, not executed tests.

### Rejected boundary

If F-indirect-0030 returns 5 normally and the original index is 5, the function returns without calling F-indirect-0034, F-indirect-0014, or any of the three direct dependencies.

An index greater than 5 has the same branch outcome.

### Zero and maximum bounds

A bound of zero rejects every unsigned 32-bit index.

A bound of 0xFFFFFFFF accepts index 0xFFFFFFFE and rejects index 0xFFFFFFFF. This distinguishes the comparison from signed ordering.

### Ordinary successful branch

Given index 2, bound 3, valid required state, and normal return from every prerequisite dependency:

- The lookup’s second explicit argument is 2.
- The selected-object operation receives the temporary address first and selected object address second.
- The direct calls occur in this order: F-46f0b3cb316e, F-289ec4311c64, F-feee9156a257.
- The final dependency’s actual termination behavior remains subject to its unresolved contract.

### Different results from the two global reads

Suppose the initial global read yields one cell address, while the later global read yields another, and both lead to valid state.

The bound target is resolved through the later cell address. If the comparison succeeds, the lookup uses the initially retained cell address.

This example is conditional on such differing observations being possible in the execution environment; the function supplies no mechanism that enforces their equality.

### Cell mutation during the bound operation

Suppose F-indirect-0030 changes the object reference in the retained cell and returns a bound greater than the index.

The subsequent lookup’s two cell-read expressions access that retained cell’s later contents. If those contents remain stable throughout lookup evaluation, both reads obtain the replacement reference. The bound is not recomputed.

Changing only the global cell-address location during the bound operation does not replace the already retained cell address.

### Different lookup target and receiver references

Suppose the lookup’s target-resolution read obtains one valid object reference and its receiver-argument read obtains another, with the chosen target accepting that receiver.

The function invokes the target selected through the first reference with the separately obtained receiver as its first argument. It performs no identity check.

Acceptance must not require a particular order for those two reads or silently require them to agree.

### Temporary mutation

If F-indirect-0014 changes the second temporary word and returns, F-46f0b3cb316e still receives the original fourth argument by value.

If F-46f0b3cb316e then changes the temporary storage and returns, F-289ec4311c64 receives the same storage without intervening reinitialization.

## Uncertainties

The following remain `documented_only`:

- Calling convention, implicit arguments, and complete pointer representation.
- Actual identities and contracts of indirect targets.
- Whether the bound represents a count or stable collection size.
- Equality of the two global cell-address observations.
- Equality of the lookup’s separately read target object and receiver.
- Evaluation order between lookup target resolution and receiver-argument evaluation.
- Temporary contents’ semantic types and permitted access extent.
- Meaning of the fourth argument.
- Ownership, retention, release, aliasing, and object-validity rules.
- Dependency effects on globals and other accessible memory.
- Concurrency, reentrancy, synchronization, and memory-order guarantees.
- Accuracy and mechanism of the final non-returning annotation.
- Failure behavior outside the displayed normal control flow.

These unknowns must not be replaced with presumed container, receiver, destructor, or exception conventions.

## Evidence

Established evidence is limited to the supplied static function representation: argument and temporary widths; initial cell-address retention; temporary initialization; a separate global read for bound-target resolution; the strict unsigned comparison; separate retained-cell accesses for lookup target and receiver; dispatch offsets; explicit argument order; subsequent call ordering; and the final non-returning annotation.

The temporary adjacency is supplied by the displayed stack representation, not by an independently defined aggregate type.

Interpretations involving extraction, transfer, cleanup, or finalization are inferred intent only. No runtime behavior, dependency implementation, ownership contract, or synchronization guarantee has been independently verified. No runtime tests were performed.
