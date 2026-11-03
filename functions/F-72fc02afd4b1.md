<!-- Trusted provenance: F-72fc02afd4b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72fc02afd4b1

## Purpose

F-72fc02afd4b1 conditionally performs a two-stage dispatch followed by an iterator-driven sequence of dependency calls. A 32-bit selector controls whether this work occurs.

One dependency is invoked on every entry, including entries whose selector does not match. For a matching selector, the function:

1. Makes two indirect calls through the returned receiver’s dispatch structure.
2. Initializes a local cursor to one byte beyond a statically referenced character-data address.
3. Repeatedly obtains a 32-bit value through a cursor-processing dependency.
4. For each nonzero value, invokes another dependency with that value and a fixed argument of one.
5. Returns one after the cursor-processing dependency returns zero.

For a nonmatching selector, it returns zero after the unconditional dependency call.

These are established control-flow properties. The meanings of the selector, receiver, global operands, character data, and produced values are not established. In particular, a return value of one is not evidence that the called operations succeeded.

## Interface

### Inputs

The recovered interface has four positional inputs:

| Position | Recovered representation | Established use |
|---|---|---|
| First | Opaque 32-bit value | Not explicitly read or forwarded by this function. |
| Second | 32-bit integer selector | Compared for equality with hexadecimal 0x7691, decimal 30353. |
| Third | Opaque 32-bit value | Passed to both indirect calls and to every per-item dependency call. |
| Fourth | Character pointer | Initially copied into a local cursor, but replaced before that cursor is supplied to any dependency. |

The first input has no established behavioral role.

The third input is forwarded without explicit arithmetic, validation, or dereferencing by this function. It could represent an address, handle, identifier, or other word-sized value; its interpretation belongs to the dependencies.

The fourth input is not directly dereferenced. On the matching path, the local copy is overwritten with a different address before iteration starts. On the nonmatching path, the copy is not used. Consequently, the supplied pointer does not select the sequence processed by the visible loop.

### Output

The function returns a 32-bit value:

- Zero when the selector does not equal 0x7691 and the unconditional dependency returns normally.
- One when the selector equals 0x7691 and all calls, including the final zero-producing cursor call, return normally.

There is no other explicit normal return value. Called operations that fail to return can prevent either result from being produced.

### Width and layout constraints

The recovered representation identifies the opaque values, selector, result, and cursor-produced values as 32-bit quantities. Pointer width and the complete calling convention are not independently established.

The receiver is interpreted as beginning with a dispatch-base word. An indirect-call target is obtained at byte offset 0x54, or 84 bytes, from that base. This is a byte displacement, not an established array index.

The supplied evidence does not establish the complete receiver layout, dispatch-table bounds, global operand storage widths, or indirect target’s declared parameter types.

## Behavior

### Unconditional dependency call

The function first initializes its local result to zero and copies the fourth input into a local cursor. It then invokes F-9d0ecc5f6adf without explicit arguments.

The returned value is treated as a receiver address if the selector matches. The function does not explicitly test that value for nullness or validity.

The selector comparison occurs after this dependency call. A nonmatching selector therefore does not bypass F-9d0ecc5f6adf or any effects it may have.

If the selector does not equal 0x7691, the function performs no explicit receiver dereference, no indirect call, and no cursor iteration. It returns zero, provided the dependency returned normally.

### Matching selector: two indirect calls

For a matching selector, two indirect calls occur in strict sequence.

For the first call, the function obtains the dispatch base from the receiver and resolves the callable entry at byte displacement 0x54. The call receives three explicit arguments:

1. The receiver returned by F-9d0ecc5f6adf.
2. A value obtained from the first referenced global operand.
3. The third input.

After that call returns, the function performs a second dispatch through the same receiver address and the same dispatch displacement. The second call receives:

1. That receiver address.
2. A value obtained from a second referenced global operand.
3. The third input.

The second indirect target is resolved again. It is not established to be the same target as the first: the first call could alter the receiver’s dispatch-base word or the dispatch entry.

Likewise, the second global operand is evaluated for the second call, after the first call returns. It must not be modeled as an entry-time snapshot immune to changes made by F-9d0ecc5f6adf or the first indirect call.

The relative evaluation order of argument expressions and dispatch-related reads within a single call is not established. The inter-call ordering is established: the first call completes before the second begins.

Neither indirect call’s return value is used to select subsequent behavior.

### Matching selector: cursor initialization

After both indirect calls return, the function replaces the local cursor with an address one character element beyond a statically referenced character-data address. Because the element type is character-sized, the displacement is one byte.

The original fourth input is not used as the iteration source.

The evidence establishes the address adjustment, but not the underlying byte contents, sequence length, encoding, or terminator convention. A source label suggesting textual contents is not sufficient to establish those contents as part of this contract.

The cursor is initialized only after both indirect calls. If those calls can modify the referenced storage, the iterator may observe modified contents. This function does not make a private copy of the sequence.

### Matching selector: iteration

The function invokes F-d3922c7fe0ec with the address of its local cursor.

This supplies access to the cursor variable itself, not merely to the character data it currently references. F-d3922c7fe0ec may therefore update the cursor value. Its advancement rules are unknown.

Each returned 32-bit integer determines the next action:

- Zero terminates iteration.
- Any nonzero value causes one call to F-008d77a63c85.

The per-item call receives, in order:

1. The third input.
2. The nonzero value returned by F-d3922c7fe0ec.
3. The fixed integer value one.

After that call returns, the function invokes F-d3922c7fe0ec again using the same local cursor variable.

There is no explicit filtering, range checking, accumulation, or transformation of the produced value. Negative values, under the recovered signed-integer interpretation, are nonzero and therefore take the per-item path. They are not treated as errors by this function.

The return value of F-008d77a63c85 is ignored. No retry, rollback, or early-exit decision is based on it.

If F-d3922c7fe0ec returns nonzero forever, the function does not reach its normal return. No iteration limit is visible.

### Completion

After F-d3922c7fe0ec returns zero, the function changes its local result from zero to one and returns it.

For a matching selector, zero is not an explicit error return. The visible function has no branch that converts a failed indirect call, failed item operation, or invalid receiver into zero.

## Side effects

### Directly established effects

The function writes its own local result and cursor variables. It passes the address of the cursor variable to F-d3922c7fe0ec.

No explicit allocation, deallocation, external-memory store, lock operation, or resource release appears in the visible body. External effects can nevertheless occur through every called operation.

On a normally completing matching invocation that produces exactly *n* nonzero cursor results followed by zero:

- F-9d0ecc5f6adf is called once.
- Two indirect dispatch calls occur.
- F-d3922c7fe0ec is called *n* + 1 times.
- F-008d77a63c85 is called *n* times.

Here *n* is a descriptive count, not a counter maintained by the function.

### Ownership and lifetime

No ownership transfer is established for any input, receiver, global-sourced operand, or character-data address.

The receiver is reused after the first indirect call. Normal completion therefore requires that later receiver accesses remain valid, but the visible code does not enforce that requirement or prove that the first call preserves it.

The cursor variable has invocation-local lifetime. Its address is supplied to F-d3922c7fe0ec, but the evidence does not establish whether that dependency retains the address. Safe use beyond the invocation cannot be inferred.

The absence of an explicit release operation does not prove that dependencies neither acquire nor release resources.

### Errors and abnormal execution

There are no explicit checks for:

- A null or invalid receiver on the matching path.
- An invalid dispatch base or callable entry.
- Invalid global-sourced arguments.
- An invalid cursor state.
- Failure reported through a dependency’s ignored return value.

The resulting fault, exception, nonlocal exit, or other abnormal behavior is not defined by this evidence. No cleanup handler is visible.

## Dependencies

| Opaque ID or role | Established contract at this call site |
|---|---|
| F-9d0ecc5f6adf | Called once before selector handling, with no explicit arguments. Its result supplies the receiver for the matching path. |
| Unidentified indirect dependency | Resolved twice through the receiver at dispatch displacement 0x54. Receives the receiver, one global-sourced operand, and the third input. Its identity may differ between calls. |
| F-d3922c7fe0ec | Receives the address of the local cursor. Returns a 32-bit value; zero terminates iteration and every nonzero value is forwarded. |
| F-008d77a63c85 | Called once per nonzero cursor result with the third input, that result, and one. Its return value does not affect visible control flow. |

No dependency is assumed to preserve global state, preserve receiver contents, or own or release memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

### Nonmatching selector

Given a selector other than 0x7691, and assuming F-9d0ecc5f6adf returns normally:

- That dependency is called once.
- No receiver dispatch or iteration occurs.
- The result is the 32-bit value zero.

This holds even if the returned receiver would be unusable for dispatch, because this path does not explicitly dereference it.

### Matching selector with immediate termination

Given selector 0x7691, valid dispatch operations, and a first cursor result of zero:

- Both indirect calls occur before the cursor call.
- F-008d77a63c85 is not called.
- The result is one.

### Matching selector with two produced values

Given successive cursor results of 7, −2, and zero, with all calls returning normally:

- F-008d77a63c85 receives the third input, 7, and one.
- It later receives the third input, −2, and one.
- The zero result is not forwarded.
- The function returns one.

The negative value is included because termination depends only on equality with zero.

### Changed dispatch state

If the first indirect call changes the receiver’s dispatch entry, the second dispatch resolves through the updated state. An implementation that unconditionally caches the first target for both calls would not preserve the established behavior.

### Different fourth input

Changing only the fourth input does not change the explicitly selected iteration address. That input is copied locally and then replaced before iteration. No stronger claim about whole-program equivalence is made.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of selector 0x7691.
- The purpose of the third input and both global-sourced operands.
- The actual character-data contents, mutability, encoding, and bounds.
- The iterator’s decoding, advancement, and error conventions.
- Whether a nonzero cursor result can represent an error in the dependency’s own contract.
- The identities and semantics of the indirect callees.
- Pointer width, complete ABI, and exact global storage widths.
- Dependency ownership, retention, allocation, release, and global-state behavior.
- Thread safety, reentrancy, and synchronization requirements.
- Whether any dependency can throw, fault, or exit nonlocally.

Repeated item processing suggests sequence consumption, but that is inferred intent rather than an established semantic description.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence covers the selector constant, 32-bit recovered scalar values, unconditional initial call, two sequential dispatches at byte offset 0x54, one-byte cursor-address adjustment, zero-terminated iteration, argument forwarding, ignored call results, and final zero-or-one result selection.

The supplied material also warns of overlapping global storage. It does not establish the relevant overlap boundaries or justify treating the global operands as independent immutable objects.

No runtime tests have been performed. Acceptance scenarios describe consequences of the visible control flow under their stated conditions; they are not observations of execution.
