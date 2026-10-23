<!-- Trusted provenance: F-ea9442e8ddbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea9442e8ddbb

## Purpose

F-ea9442e8ddbb traverses a linked sequence of objects and invokes an indirectly selected operation on each visited object. Traversal continues only when that operation returns a status equal to zero. The function returns a separate, four-byte result value that the operation can modify through a supplied address.

The operation’s status and the function’s returned result are distinct channels. A nonzero status stops traversal but is not itself returned. A zero status permits traversal but does not imply that the result value remains zero.

The established behavior is structural. The objects’ application-level meaning, the meaning of the captured global value, and the interpretation of the result are unknown.

## Interface

### Inputs

The function receives two arguments:

- **Initial object reference.** Either null or a reference to the first object to visit. Each visited object supplies an indirect-operation table reference and, when traversal continues, a successor reference.
- **Forwarded value.** An uninterpreted four-byte value passed unchanged to every operation invoked during this execution. Its signedness and application-level type are not established.

The function also reads one global four-byte value. It captures that value once, before traversal, and supplies the captured value to each operation. The global storage is identified here as **G-01**; this label carries no semantic meaning.

### Output

The function returns a four-byte value taken from a writable result slot belonging to the current invocation.

That slot starts at zero. Every operation receives the address of the same slot. There is no reset between operations. Consequently, the returned value is:

- Zero when no operation runs.
- Zero when operations run but leave the initially zero slot unchanged.
- Otherwise, the value present in the slot when traversal ends.

The wrapper does not interpret the returned bits as a pointer, integer, Boolean, or error code.

### Object and operation layout

The recovered representation establishes these accesses relative to a visited object:

| Location | Use |
|---|---|
| Object offset 0 | Read to obtain an operation-table reference. |
| Operation-table byte offset 100 | Read to obtain the operation entry to invoke. |
| Object byte offset 20 | Read as the successor reference, but only after an operation returns zero. |

The object offset of 20 follows from the sixth element of the recovered four-byte integer layout. The operation-table displacement of 100 is a byte displacement, not an index of 100 entries. Under a four-byte entry layout, it corresponds to entry index 25.

Four-byte scalar widths are established by the supplied representation. The corresponding 32-bit pointer interpretation is supported by its pointer-bearing four-byte fields, but the target ABI has not been independently verified. These offsets must not be reinterpreted using a host platform’s wider pointer size.

### Preconditions

For normal execution:

- Every nonnull visited object must support the required reads at offsets 0 and, when reached, 20.
- Its operation table must support the operation-entry read at byte offset 100.
- The selected entry must be callable with the four arguments described below and return a status usable as a 32-bit integer.
- The operation must return control normally for the wrapper to continue.
- Any object mutation performed by the operation must leave subsequent required accesses valid if traversal is to complete normally.

Null is explicitly accepted for the initial reference and for a successor reference. No other reference validation is present.

## Behavior

### Initialization

The function first reads G-01 and retains its four-byte value for this invocation. It then initializes its result slot to zero.

This ordering applies even when the initial object reference is null. An empty traversal therefore still performs the global read.

The function does not reread G-01 during traversal. If an operation changes G-01, later operations in this invocation still receive the originally captured value. This is a property of the wrapper’s retained argument, not an assertion that operations preserve global state.

### Visiting an object

When the current object reference is nonnull, the function resolves the operation through that object:

1. It obtains the object’s operation-table reference.
2. It obtains the operation entry at byte offset 100 in that table.
3. It invokes that entry with four arguments, in this order:
   - The current object reference.
   - The captured value of G-01.
   - The original forwarded four-byte input.
   - The address of the invocation’s result slot.

These are the logical argument positions. The physical calling convention and argument-evaluation details are not independently established.

Operation selection occurs for each visited object. The function does not resolve an entry once and reuse it across the entire chain. Different objects may therefore select different operations.

The result slot remains the same across all visits. An operation can inspect a value written by an earlier operation, replace it, modify only part of it, or leave it unchanged. Subject to the operation’s actual memory accesses, the wrapper returns the resulting four-byte contents without normalization.

### Continuation and termination

After an operation returns, the function compares its status with zero.

- **Status equal to zero:** The function reads the current object’s successor reference at byte offset 20. That reference becomes the next object to consider.
- **Status not equal to zero:** Traversal stops immediately. The wrapper does not read that object’s successor field.

Only exact equality with zero controls continuation. Positive and negative nonzero statuses have the same stopping effect. No particular status is recognized as success or failure by the wrapper.

The successor is read after the operation returns. It is not prefetched before the call. An operation that changes its current object’s successor field can therefore redirect the remainder of traversal.

When the newly selected successor is null, traversal ends without another operation call. The function then returns the current result-slot value.

### Cycles and repeated objects

The function has no visited-object set, iteration limit, or cycle detection.

A cycle may cause an object to be visited repeatedly. Each visit resolves and invokes its operation again. Traversal terminates only if a later operation returns nonzero, a successor read yields null, or execution is otherwise interrupted.

A cycle whose operations always return zero and whose links never change need not terminate. This behavior is not converted into an error.

### Errors and abnormal execution

The wrapper provides no explicit error-reporting mechanism for invalid objects, invalid tables, invalid operation entries, or invalid successor references.

It also does not:

- Translate an operation’s status into its own return value.
- Clear the result when an operation returns nonzero.
- Roll back modifications made during earlier visits.
- Supply a fallback operation when indirect dispatch is invalid.

If an operation fails to return normally, the supplied material establishes no subsequent wrapper behavior. The same limitation applies to invalid memory accesses. Any platform-specific fault, exception, or recovery behavior remains outside this contract’s established evidence.

## Side effects

### Direct wrapper effects

The wrapper’s direct memory effects are limited to initializing its invocation-local result slot and maintaining traversal state. It performs the following externally relevant reads:

- One read of G-01 before traversal.
- An object-table reference read and an operation-entry read for each visited object.
- A successor-reference read after each operation whose status equals zero.

There is no direct write to G-01 or to an object field in the supplied function.

The ordering matters: operation effects occur before a continuation link is read, and a stopping status prevents that link read entirely.

### Operation effects

The indirectly invoked operation can modify the result through the supplied address. Its other side effects are not established.

The contract does not assume that an operation:

- Preserves G-01 or other globals.
- Preserves object contents or operation tables.
- Preserves the successor field.
- Allocates, releases, or retains any particular storage.
- Leaves the result slot unchanged when returning a particular status.

For example, an operation may change G-01 and its object’s successor. The wrapper will continue using the old captured global value but will read the successor after that change.

### Ownership and lifetime

The wrapper itself performs no visible allocation, release, reference-count adjustment, or ownership transfer.

The result address refers to storage belonging to this invocation. The wrapper uses that storage through its final return; nothing establishes a valid post-return lifetime for the address. Retaining it does not establish ownership or permission to access it after the invocation ends.

If an operation releases the current object and returns zero, the subsequent successor read may access invalid storage. No wrapper safeguard is present. A nonzero status avoids that successor read, but does not by itself establish the safety of the operation’s other lifetime effects.

## Dependencies

### F-opaque-01

**F-opaque-01** is a documentation-local opaque function ID for the indirectly selected operation role. It does not identify a recovered concrete function. Multiple distinct concrete operations may occupy this role across visited objects.

Its observable interface consists of:

- Current object reference.
- Captured four-byte global value.
- Forwarded four-byte caller value.
- Writable address of the shared four-byte result slot.
- A returned 32-bit status, tested only for equality with zero.

The wrapper requires no particular relationship between the status and result-slot contents.

The implementation, semantics, ownership rules, and broader side effects of F-opaque-01 are unavailable. No additional concrete function dependency is established by the supplied material.

## Acceptance examples

The following are conditional expectations derived from static evidence, not executed tests.

| Conditions | Required behavior |
|---|---|
| Initial reference is null. | Read G-01, initialize the result to zero, invoke no operation, and return zero. |
| One object’s operation leaves the result untouched, returns zero, and its successor is null. | Read the successor after the call and return zero. |
| One object’s operation writes 37 to the result and returns nonzero. | Return 37 without reading that object’s successor field. |
| First operation writes 37 and returns zero; the next operation leaves the result untouched and returns nonzero. | Return 37. The result is not reset between calls. |
| First operation writes 37; a later operation writes zero and stops traversal. | Return zero, not the earlier value. |
| An operation returns a negative status and writes 12 to the result. | Stop traversal and return 12. The negative status is not returned. |
| An operation changes G-01 from one four-byte value to another and traversal continues. | Later operations receive the value captured before the first visit. |
| An operation changes its successor from object A to object B, then returns zero. | Read and follow the updated successor to B, assuming the access remains valid. |
| An operation returns nonzero while its successor field would be invalid to read. | Do not read that successor field. Earlier dispatch and operation execution must still be valid. |
| Two objects select different operation entries. | Resolve the entry separately for each visited object. |
| A self-linked object always returns zero and leaves its link unchanged. | Repeated visits are possible; finite completion is not guaranteed. |

A nonzero status with an untouched result slot is also valid: the function returns the slot’s previous contents, which may still be zero. Stopping does not imply a nonzero returned result.

## Uncertainties

### Inferred intent

The structure is consistent with a chain search, dispatch sequence, or accumulation process in which a status controls continuation and a separate output carries a result. None of these application-level interpretations is established.

The object layout resembles an operation-table-based object with an embedded link. That resemblance does not establish a language-level class, inheritance model, or ownership policy.

### documented_only

The following remain unknown:

- The meaning and initialization requirements of G-01.
- The forwarded value’s semantic type.
- The result’s semantic type and valid-value range.
- Whether zero status conventionally means “continue,” “not handled,” or something else beyond its observed control effect.
- The identities and contracts of concrete indirect operations.
- The independently verified pointer width, calling convention, alignment rules, and byte order.
- Synchronization requirements for shared globals, object fields, or tables.
- Fault handling and nonlocal control-transfer behavior.
- Whether callers guarantee acyclic chains or stable object lifetimes.

These unknowns do not change the established read order, exact-zero continuation rule, shared result-slot behavior, or single capture of G-01.

## Evidence

This contract is based solely on static inspection of the supplied representation for F-ea9442e8ddbb.

That evidence establishes the four-byte inputs and result, the initial global capture, zero initialization of the result slot, per-object indirect dispatch, the operation-table displacement, the shared writable result address, the zero-status continuation condition, the post-call successor read, and return of the slot rather than the status.

No runtime tests have been performed. No external operation implementation, caller behavior, ABI documentation, or ownership documentation was available. The acceptance examples are conditional consequences of the visible structure, not observations of execution.
