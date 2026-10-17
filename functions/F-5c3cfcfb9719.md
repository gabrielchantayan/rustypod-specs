<!-- Trusted provenance: F-5c3cfcfb9719; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5c3cfcfb9719

## Purpose

F-5c3cfcfb9719 dispatches one operation through a function pointer obtained from a caller-supplied object-like structure. It passes the original input pointer and the fixed numeric argument **0x60BC**, then returns the callee’s two-byte result.

The established behavior is a single indirect call and result forwarding. The supplied representation supports an object-method interpretation, but it does not establish the object’s type, the operation’s meaning, or the significance of 0x60BC.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Input

The function takes one pointer argument.

The supplied representation types this argument as a pointer to an integer-sized field. Operationally, the pointer must provide access to a leading field whose value is used as an address base. At byte offset **0xDC**, or **220 decimal**, from that base, the function obtains a callable function pointer.

The caller must therefore supply a structure satisfying these access requirements:

- The input pointer permits reading its leading integer-sized field.
- The value read from that field identifies an address base suitable for the subsequent lookup.
- The location at base plus 0xDC permits reading a function pointer.
- That function pointer identifies a callable target compatible with the invocation performed here.
- The original input pointer is suitable for use as the target’s first argument.

The wrapper does not check these conditions before use.

The supplied representation uses an integer value as the lookup base. Its exact storage width and its relationship to the target’s pointer width require confirmation from the execution environment. The record offset is a **byte offset**, not an element count.

### Output

The function returns a **two-byte value**, equivalent in storage width to 16 bits on an eight-bit-byte target.

No signedness, Boolean interpretation, enumeration, status-code convention, or ownership meaning is established for this result. The wrapper returns the value produced by the indirect call without an explicit comparison, mask, arithmetic adjustment, or mapping.

The supplied representation assigns the call result to a two-byte temporary before returning it. This establishes the wrapper’s two-byte result boundary, but not the indirect target’s independently declared return type or the processor-level handling of other return-register bits.

### Additional argument supplied internally

The indirect target receives **0x60BC**, or **24,764 decimal**, as its second explicit argument. This is a numeric value, not an address dereferenced by this wrapper.

The constant fits within both signed and unsigned 16-bit ranges. That fact does not establish a 16-bit argument-passing convention. Its actual argument width, extension rules, register placement, and stack layout remain ABI-dependent.

## Behavior

### Lookup and call sequence

On the ordinary path, the wrapper performs the following observable operations in this order:

1. It reads the leading integer-sized field through the input pointer.
2. It uses that field’s value as an address base and locates the function-pointer entry at byte displacement 0xDC.
3. It reads the function pointer from that entry.
4. It invokes the selected target with two explicit arguments: the original input pointer, followed by 0x60BC.
5. If the target returns normally, it returns the target’s two-byte result.

There is one indirect invocation. The supplied body contains no conditional branch, loop, retry, fallback target, or alternate result path.

The first argument is the original pointer, not the address of the function-pointer entry and not the base value read from the leading field. The callee can consequently access the original object-like storage according to its own contract.

The dispatch entry is selected before the callee executes. A later change to that entry cannot change the target already selected for this invocation. Such a change may affect subsequent invocations, subject to the environment’s memory and concurrency rules.

### Layout and arithmetic

The required layout has two distinct locations:

- An integer-sized leading field at the input address.
- A function-pointer entry at displacement 0xDC from the base represented by that field.

No other object fields or table entries are explicitly read by the wrapper.

The evidence does not establish the total table size, the width of each table entry, or the number of entries preceding the selected entry. In particular, interpreting 0xDC as a particular method index would require an independently established entry width.

The address calculation must produce a valid location under the target environment’s address model. No overflow check, range check, alignment check, or address normalization is visible. The supplied representation is insufficient to specify portable behavior for an overflowing or unrepresentable base-plus-offset calculation.

### Completion and failure

Normal completion depends on the selected target returning normally.

There is no explicit wrapper-generated error result. The wrapper does not distinguish a successful result from an error-like result and does not interpret any particular two-byte pattern.

A null or inaccessible input, unusable base value, unreadable entry, invalid function pointer, or incompatible target is outside the established valid-input contract. No defined fallback behavior is shown for these conditions. Depending on the execution environment, they may cause a memory-access fault, an invalid control transfer, or other invalid execution.

If the target does not return, the wrapper has no established normal return. No local recovery or cleanup mechanism is visible.

## Side effects

The wrapper explicitly reads dispatch-related memory and transfers control to the selected target. It contains no explicit write to the caller’s object, the dispatch table, or global state.

This is not a guarantee that the overall call is read-only. The indirect target receives the original input pointer and may have additional access to shared state. Its behavior is not provided.

In particular, the evidence does not establish whether the target:

- Mutates the input object or its dispatch field.
- Changes the dispatch table or global state.
- Allocates, retains, transfers, or releases memory.
- Performs I/O.
- Invokes callbacks or reenters this wrapper.
- Returns normally, raises an exception, or terminates execution.

The wrapper performs no visible allocation, deallocation, reference-count adjustment, or ownership transfer of its own. Passing the pointer to the target does not, by itself, establish borrowing or ownership transfer. The target’s ownership contract is **documented_only**.

After normal target completion, the wrapper does not explicitly read the object again. No post-call access to the object is therefore shown. This observation does not establish that releasing the object during the target call is permitted; that depends on the target and calling environment.

No synchronization operation is visible. Safe concurrent modification of the object or dispatch entry is not established.

## Dependencies

### F-unresolved

**F-unresolved** is a documentation-only opaque identifier for the dynamically selected callee. It does not assert a known implementation identity or a single fixed target across calls.

The established invocation boundary is:

- Selection: function pointer read at byte offset 0xDC from the base obtained through the input’s leading field.
- First explicit argument: the original input pointer.
- Second explicit argument: numeric value 0x60BC.
- Result consumed by the wrapper: a two-byte value.
- Invocation count: once on the ordinary path.

The target must be callable under the applicable ABI with this invocation shape. Beyond that boundary, its accepted object types, interpretation of the numeric argument, side effects, ownership rules, and error conventions are unknown.

No assumption is made that F-unresolved preserves global variables, retains the dispatch table unchanged, or leaves the input object alive.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not executed tests.

### Ordinary result forwarding

Given a valid input and a compatible selected target that returns the two-byte pattern **0x1234**, the wrapper must call that target with the original pointer and 0x60BC, then return **0x1234**.

It must not replace that result with a success flag or reinterpret it as a different status value.

### Boundary result patterns

Given otherwise valid execution, target results **0x0000** and **0xFFFF** must pass through as the corresponding two-byte patterns.

Neither pattern is established as success, failure, true, false, or a negative number.

### Dispatch selection

Given two valid inputs whose leading fields resolve to different dispatch tables, the entry at offset 0xDC in each respective table determines the invoked target.

No other entry is an established fallback. Targets stored elsewhere in either table are irrelevant to this wrapper’s explicit lookup.

### Mutation during the target call

If a selected target changes the input’s leading field or replaces the selected table entry and then returns normally, the wrapper must return that invocation’s result without a second explicit dispatch lookup.

This criterion does not imply that such mutation is safe under concurrent access.

### Invalid dispatch data

If the selected entry is null or otherwise non-callable, no particular returned value is required by the established contract. Accepting such input as a recoverable error case would require additional evidence or an explicitly expanded contract.

## Uncertainties

The following remain **documented_only**:

- The semantic type and complete layout of the input object.
- The integer-field width, pointer width, alignment requirements, and ABI.
- Whether the lookup structure is a language-level virtual table or another dispatch mechanism.
- The operation represented by 0x60BC.
- The independently declared argument and return types of the selected target.
- The meaning of returned values.
- Target-side mutation, ownership, allocation, release, I/O, and global-state behavior.
- Exception propagation, invalid-input consequences, and concurrency guarantees.

The object-method interpretation is inferred intent. The fixed-offset indirect invocation and two-byte result forwarding are established static behavior.

## Evidence

The supplied static representation shows one pointer input, a leading-field read used as a lookup base, an indirect function-pointer lookup at byte displacement 0xDC, and one call with the original pointer and constant 0x60BC.

It also shows a two-byte result captured from that call and returned without explicit transformation. No validation, branching, repeated lookup, explicit object write, or local resource management appears in the supplied body.

No callee implementation, runtime trace, ABI declaration, or execution result was provided. All acceptance examples are therefore conditional specifications, not claims of observed execution.
