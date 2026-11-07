<!-- Trusted provenance: F-d19f8fa90ae4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d19f8fa90ae4

## Purpose

F-d19f8fa90ae4 routes a four-byte payload to one of three operations on a resolved object, according to a one-byte selector. Selectors 0, 1, and 2 choose distinct indirect operations. Every other selector follows a separate path involving a global byte-addressed region and an initially acquired context object.

This is a source-facing behavioral contract based solely on the supplied static representation. It specifies the visible operations and their ordering without assigning domain-specific meaning to objects, payloads, or global data.

**Established:** The function obtains a context before inspecting the selector, returns no explicit value, and does not directly write memory.

**Inferred intent:** The three recognized selector values identify alternative operations on the resolved object. The remaining path may perform reporting or auxiliary processing, but its actual purpose is not established.

**Verification status:** No runtime tests have been performed.

## Interface

### Inputs

The function has three positional inputs:

| Input | Representation | Observable use |
|---|---|---|
| First input | Four-byte value | Not referenced by the visible body and not forwarded to any visible dependency. |
| Second input | Integer represented as 32 bits | On recognized-selector paths, an address-like value at an offset of 40 bytes is passed to F-7d964f24a044. Otherwise unused by the visible body. |
| Third input | Pointer to a record | A selector byte is read at byte offset 4. For recognized selectors only, the four-byte payload at offset 0 is subsequently read. |

The third-input record has this minimum accessed layout:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Opaque payload passed to the selected operation. |
| 4 | 1 byte | Selector. |
| 5 onward | Not accessed directly | No behavior established. |

The payload is an uninterpreted four-byte value. It may represent a number, address, handle, or other encoding; the visible function does not establish which.

The selector is read as a character-sized value. Its recognized byte encodings are exactly 0, 1, and 2. Whether other encodings are interpreted as positive or negative character values does not affect branch selection: all follow the unrecognized-selector path.

### Output

The function has no explicit return value. Results from selected operations are not exposed to the caller as a return value. One indirect operation on the unrecognized-selector path produces a result that controls a subsequent call.

### Preconditions

Successful execution depends on the validity of every address and indirect target reached on the selected path:

- The context returned by F-5b24edfb22c0 must remain usable when subsequently needed.
- The selector byte must be readable when accessed, after that dependency returns.
- For selectors 0, 1, and 2, the payload must be readable, F-7d964f24a044 must return a usable object, and the selected indirect operation must be callable.
- For other selectors, the global byte must be readable and any reached context-table entry must be callable.
- If the final global callback is reached, its stored target and supplied arguments must be usable according to that callback’s contract.

There are no visible null checks or explicit validation of table pointers, payload meaning, address arithmetic, or indirect targets.

## Behavior

### 1. Acquire a context, then inspect the selector

The first visible action is a call to F-5b24edfb22c0 with no explicit arguments. Its result is retained as the context object.

Only after this call returns does the function read the selector byte at offset 4 in the third-input record.

This ordering is significant. The contract does not assume that F-5b24edfb22c0 preserves the record, global state, or memory reachable from either. The selector used for dispatch is the byte observed after that call, not necessarily the byte present on entry.

The selector is read once in the visible body. Later actions do not reselect a branch if the record changes.

### 2. Recognized selectors: resolve an object and dispatch

For any selector in the set {0, 1, 2}, the visible sequence is:

1. Form the second-input value plus 40 bytes and pass that value to F-7d964f24a044.
2. Retain the returned object.
3. Read the four-byte payload at offset 0 in the third-input record.
4. Read the returned object’s first pointer-sized field as an operation-table address.
5. Read the selected callable target from that table.
6. Invoke that target with the resolved object, the captured payload, and the previously acquired context, in that argument order.
7. Return if the operation returns.

The operation-table offsets are byte offsets:

| Selector | Selected table offset |
|---:|---:|
| 0 | 360 |
| 1 | 364 |
| 2 | 356 |

These are three distinct entries. The numerical order of the table offsets does not match the selector order.

F-7d964f24a044 is called exactly once on a recognized-selector path. The function does not first read the payload and then resolve the object: payload acquisition occurs after resolution. Consequently, if resolution changes the payload memory, the later value is the one passed to the operation.

The captured selector and payload have different observation times. A dependency may change the selector during resolution without changing the already selected branch, while a payload change during that same call affects the value subsequently captured.

After the payload is captured, the visible body resolves the indirect target from the returned object. Once that target is captured, it is called without a further visible lookup. No global-byte fallback processing is performed on these paths.

### 3. Unrecognized selectors: inspect global data

For any selector other than 0, 1, or 2, the function does not call F-7d964f24a044 and does not directly read the payload at record offset 0.

It reads the first byte of a global data region.

#### Global first byte is zero

If that byte is zero:

- The context’s first pointer-sized field supplies an operation-table address.
- The callable target at byte offset 12 is invoked with the context as its sole explicit argument.
- The function returns if the operation returns.

F-5cb57bbb2e0e and the later global callback are not reached on this branch.

The operation’s name, purpose, and ownership effects are unknown. In particular, this call must not be described as releasing or destroying the context without additional evidence.

#### Global first byte is nonzero

If the byte is nonzero:

1. Call F-5cb57bbb2e0e with the address of the global data region.
2. Form an integer value one greater than its result.
3. Resolve the context operation at table byte offset 8.
4. Invoke it with the context, the incremented value, and zero, in that argument order.
5. Inspect that operation’s result.

If the result is zero, the function returns without invoking the global callback.

If the result is nonzero, the function obtains a callable target from a separate global function-pointer location and invokes it with:

- the nonzero operation result; and
- the address of the global data region.

It then returns if that callback returns.

The initial global-byte observation is not repeated. Neither F-5cb57bbb2e0e nor the context operation is assumed to preserve the global region. The final callback receives the same region address, but its contents may have changed.

Likewise, the global callback target is used after the context operation has returned. This contract does not treat it as a target captured before that operation.

### 4. Integer and address calculations

The supplied representation uses four-byte values and 32-bit integer results. Relevant calculations are:

- second input plus 40;
- F-5cb57bbb2e0e result plus 1;
- fixed byte offsets used to locate operation-table entries.

There are no visible overflow checks or arithmetic-error branches.

The static representation does not establish a portable signed-overflow contract or conclusively resolve source-language integer semantics versus underlying machine arithmetic. Ordinary in-range calculations are established. Behavior at overflow boundaries is **documented_only** until the execution model is confirmed.

The literal zero passed to the context operation is an integer zero in the visible representation. A null-pointer, flag, or mode interpretation is not established.

## Side effects

The visible body performs memory reads and external calls. It contains no direct assignment to the input record, resolved object, context object, or global data.

Observable side effects may nevertheless occur through:

- initial context acquisition;
- object resolution;
- the selected recognized-selector operation;
- F-5cb57bbb2e0e;
- either context operation; and
- the global callback.

No dependency is assumed to preserve global state, preserve pointed-to memory, or retain object validity for unrelated future operations.

The function visibly performs no allocation, copying of a buffer, reference-count update, or explicit deallocation. Passing a pointer or payload does not establish ownership transfer. Conversely, absence of an explicit release does not establish that dependencies leave ownership unchanged.

No explicit error object, status return, exception handler, retry, or recovery mechanism is present. Zero from the context operation at table offset 8 suppresses the final callback; it is not established as an error code. Nonzero permits the callback; it is not established as successful allocation or resource creation.

If a dependency faults, throws, exits, or does not return, completion of later steps is not guaranteed. The visible body supplies no local handling for those outcomes.

## Dependencies

### F-5b24edfb22c0

Called once at the start, with no explicit arguments, before selector inspection. Returns the context subsequently passed to recognized-selector operations or used for unrecognized-selector dispatch. Its acquisition and ownership semantics are unknown.

### F-7d964f24a044

Called only for selectors 0, 1, and 2. Receives the second input plus 40 and returns the object whose first field supplies the operation table. The input’s domain meaning and the object’s lifetime are unknown.

### F-5cb57bbb2e0e

Called only for an unrecognized selector when the first global byte is nonzero. Receives the global region’s address. Its integer result is incremented before use by the context operation.

A length-like role is plausible but not established. No string format, termination rule, or region extent may be assumed.

### Indirect dependencies

No opaque function IDs are supplied for the indirect callees. They are identified only by their dispatch locations:

| Dispatch location | Explicit arguments |
|---|---|
| Resolved-object table, offset 360 | Resolved object, payload, context |
| Resolved-object table, offset 364 | Resolved object, payload, context |
| Resolved-object table, offset 356 | Resolved object, payload, context |
| Context table, offset 12 | Context |
| Context table, offset 8 | Context, incremented helper result, zero |
| Global function-pointer location | Nonzero context-operation result, global-region address |

Their names, broader contracts, calling conventions, and ownership effects remain unknown.

## Acceptance examples

These are conditional acceptance criteria, not executed test results. Each assumes valid memory, callable targets, returning dependencies, and in-range arithmetic unless stated otherwise.

1. **Selector 0:** After context acquisition, the observed selector is 0. The function resolves the object, then captures the payload, and invokes the resolved-object table entry at offset 360 with object, payload, and context. It does not inspect the fallback global byte.

2. **Selector 1:** With an observed selector of 1, the selected table offset is 364, not 360 or 356.

3. **Selector 2:** With an observed selector of 2, the selected table offset is 356.

4. **Selector changed during acquisition:** If context acquisition changes the selector from 0 to 2 before returning, dispatch follows selector 2 because the selector read occurs afterward.

5. **Record changed during resolution:** If selector 0 is captured and object resolution then changes both the selector and payload, dispatch remains at table offset 360, but the payload passed is the value read after resolution.

6. **Unrecognized selector with zero global byte:** With selector 3 and a zero first global byte, the context operation at offset 12 is called once. No payload read, object resolution, helper call, or final global callback occurs in the visible body.

7. **Unrecognized high-valued byte:** A selector byte encoded as 255 follows the same unrecognized-selector branching as selector 3. Character signedness does not make it a recognized selector.

8. **Nonzero global byte and zero operation result:** If the helper returns 7, the context operation receives context, 8, and zero. If it returns zero, the final callback is skipped.

9. **Nonzero operation result:** Under the preceding conditions, if the context operation instead returns a nonzero value, the global callback receives that exact result and the global-region address.

10. **Global data changed by a dependency:** If the helper or context operation changes the global region, the function does not restore it or repeat the original byte test before the final callback.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the payload, selector alternatives, context, resolved object, and global data.
- Whether the global data is textual, terminated, mutable, or bounded.
- Whether the helper result denotes a length, count, index, status, or something else.
- Ownership, retention, destruction, and allocation behavior of every dependency.
- Exact calling conventions and executable target validity.
- Alignment requirements and byte order for four-byte and pointer-sized accesses.
- Exact overflow behavior and address validity near arithmetic boundaries.
- Whether indirect transfers have ordinary returning-call semantics in the executable. The supplied representation models them as calls followed by return but does not independently establish that lowering.
- Concurrency guarantees, synchronization, and behavior under unsynchronized memory changes.

## Evidence

The supplied static body establishes:

- unconditional initial context acquisition;
- selector access at record byte offset 4;
- recognized selector values and their exact dispatch offsets;
- object resolution before payload capture;
- a four-byte payload read at record offset 0;
- the global-byte split for all other selectors;
- the context-operation offsets, explicit argument ordering, and zero-result gate;
- absence of visible direct memory writes and explicit return values.

The contract does not infer dependency behavior from suggestive call shapes or treat annotations about indirect transfers as a complete executable model. There is no runtime evidence; the acceptance examples remain unexecuted conditions.
