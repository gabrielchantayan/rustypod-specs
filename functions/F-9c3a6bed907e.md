<!-- Trusted provenance: F-9c3a6bed907e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c3a6bed907e

## Purpose

F-9c3a6bed907e dispatches one indirect call using a caller-supplied object reference. It prepares two temporary 32-bit values, selects a callable target through the object’s first word and a fixed byte offset, and invokes that target with the original object reference and the address of the first temporary.

The established behavior is a small dispatch wrapper. Interpreting the first object word as a virtual-method-table reference and the temporary values as a two-field argument package is plausible intent, not an established interface definition.

The function has no explicit result. Its externally meaningful effects can include anything performed by the selected target, as well as faults caused by invalid object, table, or target references.

## Interface

### Inputs

The supplied representation identifies three inputs:

1. **Object reference.** A pointer through which the function reads an integer-sized first word. That word participates in selecting the indirect-call target. The same object reference is passed as the first argument to that target.
2. **First value.** An uninterpreted 32-bit value copied into temporary local storage. The address of this storage is passed as the target’s second argument.
3. **Second value.** A separate uninterpreted 32-bit value copied into another temporary local storage location before dispatch.

Neither value has an established signed interpretation, numeric unit, encoding, or application-level meaning. Their supplied 32-bit representations are retained by the local assignments; the wrapper shows no arithmetic, conversion, filtering, or validation of either value.

The recovered interface uses an integer-sized word for the object’s first field. A 32-bit interpretation of that field is consistent with the supplied representation, but the complete platform ABI, pointer width, and callable-pointer representation are not independently supplied.

### Output

The declared interface returns no value. After the indirect call returns normally, the wrapper returns normally without explicitly forwarding any result from that call.

A physical return register might contain residual data under a particular ABI. Such data is not an output guaranteed by this contract.

### Preconditions for normal completion

Normal completion requires all of the following:

- The object reference supports the initial word read.
- The resulting word can be used in the represented address calculation.
- The location reached by adding 28 bytes supports reading a callable reference.
- The selected reference identifies executable behavior compatible with the invocation made here.
- The indirect target returns normally.
- Any further requirements imposed by that target are satisfied.

The function does not establish these conditions itself. In particular, it contains no null check, range check, callable-target check, or application-level validation of either value.

## Behavior

### Established operation and ordering

The supplied representation establishes this sequence:

1. The first value is copied into one local 32-bit storage location.
2. The second value is copied into a separate local 32-bit storage location.
3. The first integer-sized word is read through the object reference.
4. A byte offset of 28, hexadecimal `0x1c`, is added to that word.
5. A callable reference is read from the resulting location.
6. That target is invoked with two arguments, in this order: the original object reference, then the address of the first local value.
7. If the invocation returns normally, the wrapper returns without further explicit work.

The local initializations precede the indirect invocation in the supplied representation. They are not writes into the caller’s original value arguments or into the object. No externally visible writes by the wrapper are shown before dispatch.

This ordering describes the represented operations, not an independently verified machine-instruction trace. It does not establish processor-level memory-ordering, atomicity, or concurrency guarantees.

### Dispatch layout

Two levels of access are represented:

- The object reference identifies storage whose first word supplies the base used for dispatch.
- A callable reference is loaded from that base plus 28 bytes.

The offset is a byte displacement, not a count of callable entries. Calling it a particular numbered table slot would require an established entry width and table layout, neither of which is independently provided.

No other object fields or table entries are explicitly inspected by the wrapper. It selects the target from the current first word at invocation time rather than from a fixed, directly named dependency.

The representation does not settle the exact machine-width behavior of the address addition, including overflow, wrapping, sign extension, or address canonicalization. Normal-use acceptance therefore requires an address calculation that reaches valid target-reference storage. Behavior at arithmetic or address-space boundaries is documented_only.

### Temporary-value layout and access

The first temporary has an established 32-bit extent, and its address is explicitly passed to the target. The second temporary is initialized, but its address is not separately passed.

The supplied local declarations do not by themselves establish a portable aggregate containing both values. In particular, they do not prove that the second value is accessible exactly four bytes after the first, that there is no intervening padding, or that reading beyond the first temporary is valid.

Consequently:

- The target receives an address associated with the first value.
- The wrapper does not explicitly pass the second value as a separate call argument.
- Treating the addressed storage as an eight-byte, two-field record requires additional layout evidence.

A likely explanation for the second initialization is that the intended target consumes a packed pair of values. That remains inferred intent, not a guaranteed layout in this contract.

### Mutation and lifetime

The first temporary is mutable local storage. The representation does not impose a read-only restriction on the pointer supplied to the target. If the target changes that temporary, the wrapper performs no explicit subsequent read of it and does not return its final contents.

The temporary storage is associated with this invocation. No allocation, ownership transfer, or lifetime extension is shown. A retained pointer must not be treated as valid after the wrapper returns merely because the target received it.

The object reference is forwarded directly. There is no explicit object copy, retain operation, release operation, or ownership transfer by the wrapper.

These observations do not constrain the target’s own actions. It may modify reachable objects, affect global state, retain references, or perform resource-management operations according to its separate contract. None of those behaviors is established here.

### Failure and abnormal control flow

There is no local error result or recovery branch.

An invalid object reference can prevent the initial read. An invalid dispatch base can prevent the target-reference read. An unusable callable reference can prevent successful invocation. The exact failure mechanism depends on the execution environment and is not specified by the supplied representation.

The wrapper provides no explicit handling for a target that faults, throws, terminates execution, or never returns. Normal wrapper return is conditional on the target returning normally.

## Side effects

The wrapper itself explicitly performs:

- Two local 32-bit initializations.
- One read of the object’s first word.
- One read of a callable reference at the computed dispatch location.
- One indirect invocation.

It shows no direct writes to object fields, caller-owned value storage, global variables, files, or external devices. It shows no explicit allocation or deallocation.

The indirect invocation prevents a broader claim of purity. The target’s side effects are unknown, including whether it changes global state or releases memory. The wrapper does not restore state after the call and provides no rollback behavior.

No synchronization, locking, memory barrier, or thread-safety mechanism is shown.

## Dependencies

The only explicit call is to a dynamically selected dependency. No opaque function ID for that target is supplied, so its identity and implementation are documented_only.

Its observable invocation contract is limited to:

- Selection through the object’s first word and the 28-byte displacement.
- Receipt of the original object reference as its first argument.
- Receipt of the first temporary’s address as its second argument.
- Invocation after both local value initializations.

The target’s return type, calling-convention details, additional memory accesses, ownership behavior, and global-state effects are not established. The wrapper does not explicitly consume a target result.

F-9c3a6bed907e therefore cannot be specified as preserving all state other than its temporaries. That conclusion would require a separate contract for the selected dependency.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Normal dispatch

Given a valid object reference, a valid dispatch base in its first word, and a compatible callable reference at base plus 28 bytes:

- Exactly one indirect invocation is represented.
- The first argument identifies the original object.
- The second argument addresses temporary storage initialized with the first supplied 32-bit value.
- Both local initializations precede the invocation.
- Normal target return is followed by normal wrapper return without an explicit result.

### Boundary bit patterns

For first-value inputs such as `0x00000000`, `0x80000000`, or `0xffffffff`, the wrapper applies the same copy-and-dispatch behavior. It does not explicitly reject these patterns or interpret them as positive, negative, absent, or erroneous.

The same absence of interpretation applies to the second value. This does not guarantee that the target accepts any particular pattern.

### Target mutation

If the target writes a different 32-bit value through the second argument and then returns normally, the wrapper does not explicitly read back or publish that changed value. Any further observable effect depends on the target.

### Target result

If the target produces a result under its own calling convention, the wrapper still has no declared output and performs no explicit result forwarding.

### Invalid dispatch data

For a null object reference or an invalid dispatch entry, acceptance must not require a clean error code, a skipped call, or successful return. No such handling is represented.

### Proposed two-field consumer

A requirement that the target read the second value four bytes after the first cannot be accepted solely from the supplied declarations. That requirement needs independent temporary-layout evidence. Its status here is documented_only.

## Uncertainties

The following remain documented_only:

- Exact machine ABI, pointer width, alignment requirements, and callable-reference width.
- Address-arithmetic width and boundary behavior.
- The concrete meaning of the object’s first word.
- Whether the dispatch structure is a virtual table or another table-like structure.
- Adjacency and ordering of the two temporary storage locations in actual memory.
- Whether the selected target accesses the second value.
- Target identity, return type, side effects, ownership rules, and error behavior.
- Exception propagation, unwinding, and platform-specific fault behavior.

The principal inferred intent is dispatch to an object-associated operation using two prepared values. The established behavior is narrower: only the first temporary’s address is explicitly supplied, and the second temporary’s relationship to it is not independently established.

## Evidence

This contract is based solely on the supplied static function representation. It shows two 32-bit local assignments, an indirect target lookup using the object’s first word plus 28 bytes, a two-argument invocation, and an explicit return without a result.

No runtime tests, memory-layout observations, instruction traces, dependency implementations, or ABI documentation were provided or examined. Acceptance examples are specification conditions, not reported execution outcomes.
