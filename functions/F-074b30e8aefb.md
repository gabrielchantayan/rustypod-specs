<!-- Trusted provenance: F-074b30e8aefb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-074b30e8aefb

## Purpose

F-074b30e8aefb performs condition-dependent memory and coprocessor operations, prepares two 32-bit working values, and executes a nested loop containing an ordered pair of dependency calls.

Its behavior depends on more than its four explicit arguments. Incoming processor flags, additional architectural registers, a stack-reference address, and pre-existing stack contents all affect execution. The explicit argument list is therefore insufficient to describe a standalone calling interface.

The paired calls and working-value increments suggest a two-dimensional traversal. That interpretation is **inferred intent**, not an established semantic purpose. The supplied evidence does not identify the data being traversed, a rendered result, or any other application-level output.

## Interface

### Explicit arguments and result

| Position | Width and interpretation | Use |
|---|---|---|
| First | Unsigned 32-bit word | May be masked; contributes to an initialization argument, a conditional byte write, and a coprocessor load address. |
| Second | Opaque 32-bit word | No direct use is visible. |
| Third | Signed 32-bit quantity | Passed to the initialization dependency unless replaced by a flag-dependent calculation. |
| Fourth | Address of a 32-bit integer | Receives an unconditional word write. Addresses before it are also used. |

No return value is exposed. Observable results consist of memory changes, dependency effects, coprocessor effects, and possible failure to complete.

The fourth argument must not be treated as merely a pointer to one writable integer: the function also constructs addresses 32 or 816 bytes before it.

### Implicit execution inputs

The supplied representation consumes the following incoming state:

| State | Role |
|---|---|
| Negative, zero, carry, and overflow flags | Select early operations and, on some paths, later address and argument calculations. |
| Architectural register 4 | Supplies a stored word, a conditional address calculation, the inner-loop count, and row corrections. |
| Architectural register 5 | Supplies three times its value as an initialization argument. |
| Architectural register 6 | Supplies the first unconditional stored word and, conditionally, an address to read. |
| Architectural register 7 | Supplies a masked address expression and a multiplier for one working value. |
| Architectural register 8 | Supplies a multiplier unless replaced by a conditional memory read. |
| Architectural register 9 | Supplies the outer-loop count. |
| Architectural register 10 | Supplies an address base, a multiplier, and a row increment. |
| Architectural register 12 | Supplies the base for an early conditional byte write. |
| Link register | Supplies part of the alternate third argument. |
| Coprocessor state | Supplies operands for opaque coprocessor operations. |
| Stack-reference address | Supplies both an address-derived value and several dependency-visible addresses. |

These are dependencies of the supplied representation. The evidence does not establish a conventional ABI that makes them valid inputs.

### Stack-relative layout

All offsets below are hexadecimal byte offsets from the supplied stack-reference address.

| Offset | Visible role |
|---|---|
| `+0x20` | Passed to both per-iteration dependencies. |
| `+0x24` | First working word; explicitly initialized and updated. |
| `+0x28` | Second working word; explicitly initialized and updated. |
| `+0x2c` | Read as an initialization multiplier. |
| `+0x30` | Passed to the second per-iteration dependency. |
| `+0x34` | Passed to the first per-iteration dependency. |
| `+0x40` | Read as an initialization addend. |
| `+0x4c` | Read as another initialization addend. |

The visible arithmetic reads and writes are 32-bit operations. The sizes of objects consumed through the passed addresses are unknown. In particular, adjacency does not establish separate, non-overlapping objects.

## Behavior

### 1. Early flag-dependent operations

The first operation is an opaque coprocessor operation, performed only when the **incoming** zero flag is clear and the incoming negative and overflow flags are equal.

Next, if the incoming negative flag is set:

- The first argument is restricted to bits selected by `0x61`.
- The working zero flag becomes true exactly when that masked value is zero.

Otherwise, the first argument and working zero flag remain unchanged.

A 32-bit difference between the resulting first argument and `0x3f0` is then formed for later use as an initialization argument. A subsequent replacement of the first argument does not recompute this difference.

If the incoming carry flag is clear, the low eight bits of the current first argument are written to the address 3,342 bytes before the incoming value of architectural register 12.

The function then unconditionally writes the incoming 32-bit value of architectural register 6 through the fourth explicit argument. This write precedes the later conditional replacement of the working address derived from register 6.

An additional opaque coprocessor operation follows unconditionally.

If the working zero flag is set at this point, the first argument is replaced by the bitwise intersection of:

- Incoming architectural register 7.
- The 32-bit stack-reference address shifted logically right by one bit.

The shift applies to the address before the intersection.

If the incoming overflow flag is clear, a coprocessor load is attempted using the current first argument as its address. Its access size, address constraints, transfer semantics, and failure behavior are not established.

### 2. Conditional address and predicate preparation

When the incoming overflow flag is clear, the function derives a new address and predicates from architectural register 4:

- The working address is its 32-bit value minus `0x07600000`.
- The working carry predicate is true when its unsigned value is at least `0x07600000`.
- The working overflow predicate is the signed-subtraction overflow indication for that subtraction.
- A separate negative-result predicate records whether the resulting 32-bit value is signed-negative.
- The working zero predicate records whether the result is zero.

The separate negative-result predicate starts false. When incoming overflow is set, this entire preparation is skipped: the address remains the incoming register-6 value, carry and overflow retain their preceding values, zero retains its potentially mask-updated value, and the separate negative-result predicate remains false.

The next address selection uses these predicates:

- If carry is clear **or** zero is set, the selected address is 3,497 bytes beyond the working address.
- On that same branch, a 32-bit word is read from the **unadjusted** working address and replaces the register-8-derived multiplier.
- Otherwise, the selected address is the working address, and the incoming register-8 multiplier remains in use.

The read is not from the selected, incremented address.

After that conditional read, the incoming register-4 word is stored 1,608 bytes before the incoming register-10 value if either:

- The working zero predicate is true; or
- The separate negative-result predicate differs from the working overflow predicate.

This store is independent of the earlier byte write and the unconditional write through the fourth argument.

### 3. Initialization dependency

The default fourth argument to F-082b668da333 is the address 32 bytes before the function’s fourth explicit argument.

If the working zero predicate is true, three changes occur:

1. The third argument becomes the link-register value minus the selected-address value multiplied by `0x8000`.
2. The dependency’s fourth argument becomes the address 816 bytes before the function’s fourth explicit argument.
3. The low byte of the selected-address value is written 32 bytes before the function’s fourth explicit argument.

F-082b668da333 is then called with:

| Position | Value |
|---|---|
| First | Three times incoming architectural register 5. |
| Second | The previously formed first-argument difference from `0x3f0`. |
| Third | The original third argument, or its zero-dependent replacement. |
| Fourth | The selected address relative to the fourth explicit argument. |

Its returned 32-bit integer becomes the horizontal step used below.

The call occurs after the preceding conditional writes and reads. It is not established to preserve globals or memory reachable through its arguments.

### 4. Working-value initialization

After F-082b668da333 returns, the function initializes the stack-relative working words in this order:

1. `+0x24` receives incoming register 7 multiplied by the returned horizontal step, plus the word at `+0x40`.
2. `+0x28` receives the sum of:
   - Incoming register 10 multiplied by the selected-address value.
   - The current register-8-derived multiplier multiplied by the word at `+0x2c`.
   - The word at `+0x4c`.

The source reads occur after the initialization dependency returns. The evidence does not justify replacing them with values captured before that call.

Only the order of the two destination assignments is established here. An ordering among independent reads within either arithmetic expression is not established.

### 5. Nested dependency calls and row updates

The inner loop begins with the unsigned 32-bit value from architectural register 4. Each iteration performs, in order:

1. F-a0d7716b0f77, receiving addresses `+0x20`, `+0x34`, and `+0x28`.
2. F-e2db7eabd42d, receiving addresses `+0x30`, `+0x24`, and `+0x20`.
3. An unsigned 32-bit decrement of the inner counter.

The inner loop repeats while the decremented counter is nonzero.

This is a post-tested loop. A zero starting value does **not** skip the body. Its unsigned decrement wraps, yielding \(2^{32}\) iterations if every call returns and execution remains valid. A nonzero starting value yields that many iterations.

After each completed inner loop, the function:

1. Decrements the outer counter.
2. Adds incoming register 10 minus twice incoming register 4 to the current word at `+0x28`.
3. Adds the returned horizontal step minus three times incoming register 4 to the current word at `+0x24`.
4. Resets the inner counter to incoming register 4.
5. Repeats while the decremented outer counter is nonzero.

Both working-word updates occur even on the final outer iteration. They operate on the words as they exist after the dependency calls, not necessarily their initialized values.

The outer loop is also post-tested. For a positive outer count, it completes that many rows, subject to dependency completion and valid execution. Behavior for zero or negative starting counts requires the signed-arithmetic qualification below.

### Arithmetic qualification

The supplied representation uses 32-bit words and addresses. Unsigned decrement and associated wraparound are explicit.

Several products, additions, and the outer decrement are represented using signed integer types. Under a 32-bit machine-word interpretation, their low 32 bits determine subsequent word values. However, the supplied text alone does not establish a language-level signed-overflow contract.

Accordingly:

- Unsigned inner-counter wraparound is established.
- Signed arithmetic outside its representable range is **documented_only**.
- If wrapping outer-counter arithmetic is assumed, a nonzero initial bit pattern gives that unsigned number of rows, and zero gives \(2^{32}\) rows.
- That outer-counter rule is conditional, not an unconditional language-level guarantee.

## Side effects

The visible side-effect sequence is:

1. Optional first coprocessor operation.
2. Optional register-12-relative byte write.
3. Unconditional word write through the fourth argument.
4. Unconditional second coprocessor operation.
5. Optional coprocessor load.
6. Optional word read through the working address.
7. Optional register-10-relative word write.
8. Optional byte write before the fourth argument.
9. F-082b668da333 and its unknown effects.
10. Working-word initialization.
11. Repeated ordered dependency pairs and working-word updates.

Aliasing can make this order consequential. For example, the unconditional fourth-argument write can affect a later read if their addresses coincide. No non-aliasing guarantee is supplied.

No explicit allocation, release, ownership transfer, validation, error return, or recovery path is visible. Passing an address does not establish whether a dependency retains it, releases associated storage, or accesses neighboring memory.

Successful completion requires all executed memory accesses and coprocessor operations to be valid, and all dependencies to return. Alignment requirements, exception delivery, and behavior after an invalid access are unknown. No atomicity, rollback, reentrancy, or thread-safety guarantee is established.

## Dependencies

| Opaque ID | Visible contract at this call site | Unknowns |
|---|---|---|
| F-082b668da333 | Called once with four arguments; its 32-bit result supplies a multiplier and later increment. | Argument semantics, memory-access extent, global effects, ownership behavior, and failure behavior. |
| F-a0d7716b0f77 | First call in every inner iteration; receives stack-relative addresses `+0x20`, `+0x34`, `+0x28`. | Read/write directions, object layouts, retained references, and other effects. |
| F-e2db7eabd42d | Second call in every inner iteration; receives `+0x30`, `+0x24`, `+0x20`. | Same categories of unknown behavior. |

The shared `+0x20` address and strict call order establish a possible communication path between the per-iteration dependencies. They do not establish what is communicated.

The coprocessor operations have no supplied opaque function IDs. Their conditional invocation is established; their functional meaning is **documented_only**.

## Acceptance examples

These are conditional contract examples, not executed tests.

### A. One inner iteration and one row

Given incoming register 4 equal to one and outer count equal to one, with valid accesses and returning dependencies:

- F-082b668da333 is called once.
- F-a0d7716b0f77 is called once, followed by F-e2db7eabd42d once.
- Both row-end working-word updates occur.
- The function then returns.

An implementation that omits the final row updates would differ observably.

### B. Zero inner count

Given incoming register 4 equal to zero and a positive outer count:

- The inner body executes before its first counter test.
- Each completed row requires \(2^{32}\) ordered dependency pairs, assuming continued valid execution.

Treating zero as an empty inner range would not match the supplied behavior.

### C. Zero result from address preparation

Given incoming overflow clear and register 4 equal to `0x07600000`:

- The derived working address is zero.
- The zero predicate is set.
- The selected address becomes `0x0da9`.
- A word read from address zero is attempted before initialization.

A fault at that read prevents the later initialization call. Acceptance of the complete path therefore requires address zero to be readable in the execution environment.

### D. Masking without recomputing the saved difference

Given incoming negative set and first argument `0x80`:

- Masking by `0x61` produces zero and sets the working zero predicate.
- The saved initialization difference is the 32-bit result of zero minus `0x3f0`.
- A later stack-derived replacement of the first argument does not alter that saved difference.

### E. Incoming overflow set

Given incoming overflow set:

- The coprocessor load and register-4-based address preparation are skipped.
- The separate negative-result predicate remains false.
- Its mismatch with the retained true overflow predicate makes the register-10-relative word store execute, assuming execution reaches it.

Using the incoming negative flag in place of that separately initialized predicate would change this behavior.

## Uncertainties

**Established static evidence:** explicit argument use, operation conditions, visible access widths, stack offsets, dependency ordering, unsigned inner-loop behavior, and absence of explicit error handling.

**Inferred intent:** a row-oriented or two-dimensional traversal with two cooperating per-element operations.

**Documented_only:**

- The validity of the recovered entry signature and implicit register inputs as a callable ABI.
- Coprocessor instruction meaning and execution requirements.
- Dependency semantics, preservation guarantees, memory ownership, and access extents.
- Concrete stack-object layouts beyond the visible offsets.
- Signed-overflow behavior and address validity after arithmetic.
- Platform alignment and fault behavior.
- Application-level output and intended useful input ranges.

These unknowns must not be replaced with assumptions that dependencies are pure, that globals remain unchanged, or that supplied addresses denote independently owned objects.

## Evidence

This contract is based solely on the supplied restricted source representation. Its unusual dependence on incoming flags, otherwise unbound architectural state, and opaque coprocessor operations limits conclusions about the intended interface and purpose.

No implementation access, dependency bodies, instruction-level verification, runtime observations, or tests were available. No runtime tests have been performed. The acceptance examples specify consequences of the stated conditions; they are not reports of observed executions.
