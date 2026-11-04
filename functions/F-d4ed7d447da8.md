<!-- Trusted provenance: F-d4ed7d447da8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4ed7d447da8

## Purpose

F-d4ed7d447da8 conditionally obtains a two-component value associated with a selected index and reports whether an unsigned request is no greater than that value.

The result is a 32-bit Boolean value:

- **1** means the index is eligible, both returned components have valid low bytes, and the requested value is within the assembled limit.
- **0** means the index is ineligible, at least one component has the invalid-byte marker, or the request exceeds the assembled limit.

This is not established to be a pure query. Eligible calls invoke dependencies and an indirect operation whose other effects are unknown. An unsuccessful lookup invokes a failure dependency rather than explicitly returning 0.

**Inferred intent:** the two component bytes represent a high/low limit, with byte value 255 reserved as an invalid or unavailable marker. No particular domain, unit, or interpretation beyond the observed numeric comparison is established.

## Interface

### Inputs

The function accepts three 32-bit values, in this order:

| Input | Interpretation |
|---|---|
| First argument | Unused by the visible function body. Its value does not directly affect any visible decision, memory access, or dependency argument. |
| Index | Unsigned 32-bit selector. Only values 0 through 12 are eligible for further processing. |
| Request | Unsigned 32-bit quantity compared with a limit assembled from two bytes. |

The first argument is neither dereferenced nor passed to the visible dependencies. It therefore has no visible pointer-validity requirement, even if a caller happens to supply an address-shaped value.

The index is converted to a character-sized value and then to an integer for the lookup dependency. Because this conversion occurs only for indices 0 through 12, it preserves the numeric value regardless of whether the target character type is signed.

The request is not truncated to 16 bits. The comparison uses its complete unsigned 32-bit value.

### Output

On ordinary completion, the function returns exactly 0 or 1 in a 32-bit result.

No caller-provided output buffer exists. The function instead creates two temporary 32-bit output words for an indirect dependency. Their addresses are passed out during the call, but those addresses are not returned to the caller.

### Required environmental state

For an eligible index, the environment must provide:

- A readable global base reference and a readable 32-bit enable mask at byte offset 40 from that base.
- A callable context-provider dependency.
- A lookup dependency that can consume a context address displaced by four bytes and the selected index.
- Either a suitable object returned by lookup or a failure path that prevents subsequent use of a null object.
- For a non-null object, a valid dispatch structure and a callable entry at byte offset 24 from that structure.
- An indirect operation compatible with an object argument followed by pointers to two writable 32-bit words.

These are memory-safety and call-compatibility preconditions, not checks performed by this function.

## Behavior

### Eligibility and early return

The function initially selects result 0.

It first compares the unsigned index with 13. If the index is 13 or greater, it returns 0 without reading the enable mask and without invoking any visible dependency.

For an index from 0 through 12, it reads the 32-bit enable mask from the global structure at byte offset 40. The bit whose position equals the index controls eligibility:

- A clear bit causes an immediate result of 0.
- A set bit permits dependency processing.

The shift used to select the bit is safe for every index that reaches it: the shift count lies between 0 and 12. Other mask bits do not affect the outcome for this invocation.

The mask is checked only at this stage. The visible body does not revalidate eligibility after any dependency executes.

### Preparation and lookup

For an eligible index, the function prepares two separate temporary 32-bit words:

1. The first component word is initialized to numeric value 255.
2. The second component word is initialized to numeric value 255.

These are whole-word initializations to 0x000000FF, not initializations to an all-ones 32-bit value.

The function then invokes the context provider, identified below as F-unresolved-01. It adds a byte displacement of four to the returned address-like value and supplies that adjusted value to F-9266831caef7. The lookup’s other argument is the eligible index after the character-sized conversion described under Interface.

The visible body does not check the provider’s result before performing the adjustment and lookup. In particular, a zero provider result is not handled as an early-return condition here.

### Null lookup result

If F-9266831caef7 returns a null object pointer, the function invokes F-a817d8d58c10.

The supplied representation annotates that dependency as non-returning, but its implementation is unavailable. Therefore:

- It is established that the null-result branch invokes F-a817d8d58c10.
- It is not independently established whether that dependency terminates, throws, transfers control elsewhere, or has another non-returning effect.
- There is no visible recovery return after that invocation.
- If the dependency unexpectedly returns normally, execution proceeds toward dispatch through the null object. No defined Boolean failure result can be promised for that case.

A null lookup result must not be specified as equivalent to returning 0.

### Indirect operation and object layout

For a non-null lookup result, the function obtains a dispatch-table address from the beginning of the object, then obtains the operation address from byte offset 24 within that table.

It invokes that operation, identified here as F-unresolved-02, with:

1. The object pointer.
2. The address of the first temporary component word.
3. The address of the second temporary component word.

The two component pointers refer to distinct writable 32-bit words. The first is later interpreted as the high component; the second is later interpreted as the low component.

This establishes the layout needed for this dispatch, but not the object’s complete layout, the dispatch table’s other entries, or ownership of either structure.

The indirect operation produces a 32-bit return value. That value does not determine the function’s ordinary Boolean result:

- If the first component is valid, the operation’s return value is replaced by the extracted second component before the comparison.
- If the first component is invalid, the final acceptance condition fails regardless of the operation’s return value.

Thus the operation’s return value is not an alternate limit, status override, or substitute for an unwritten output.

### Component interpretation

After the indirect operation completes, the function masks the first component word to its least significant eight bits and writes the masked value back into that temporary word.

If the resulting first byte is 255, the function returns 0. In that case, the second component is not needed for the visible acceptance decision.

If the first byte is not 255, the function extracts the least significant eight bits of the second component word. If that second byte is 255, it returns 0.

Only the low byte of each component word matters. Upper bits do not make an otherwise valid byte invalid and do not contribute to the assembled limit.

Because both words begin with low byte 255, leaving either relevant word unchanged causes rejection. The function does not independently require that the indirect operation write both words; it evaluates their resulting contents.

### Limit construction and comparison

When both extracted bytes are valid, the limit is:

**256 × first byte + second byte**

Equivalently, the first byte occupies bits 8–15 and the second byte occupies bits 0–7.

This is a numeric assembly rule. It does not establish the in-memory byte order used when the indirect operation writes a component word.

Each valid byte is in the range 0 through 254. Consequently:

- The minimum valid limit is 0.
- The maximum valid limit is 65,278, or 0xFEFE.
- Some values below that maximum are unrepresentable because a component byte of 255 is reserved.
- The shift and combination cannot overflow a 32-bit unsigned value.

The function returns 1 exactly when the unsigned 32-bit request is less than or equal to this limit. Equality is accepted. Requests larger than 65,278 cannot be accepted by any valid component pair.

## Side effects

### Effects established by the visible body

The function can:

- Read a global base reference and an enable-mask word.
- Initialize two temporary 32-bit words.
- Invoke the context provider and lookup dependency.
- Invoke the failure dependency on a null lookup result.
- Read an object’s dispatch reference and a dispatch entry.
- Pass the object and temporary output addresses to an indirect operation.
- Normalize the first temporary word to its low byte after the operation returns.

For eligible calls, the important visible order is:

1. Read and evaluate eligibility.
2. Initialize the first temporary word, then the second.
3. Call the context provider.
4. Call the lookup dependency.
5. Handle a null result, if present.
6. Resolve and invoke the indirect operation.
7. Interpret the first component, conditionally interpret the second, and decide the result.

The visible body performs no explicit allocation, deallocation, reference-count adjustment, or persistent-state write. This does not establish that its dependencies perform none.

### Ownership and lifetime

The component storage belongs to the current invocation. The indirect operation receives temporary writable access; no ownership transfer is visible.

The evidence does not establish whether that operation retains the addresses, publishes them, or writes through other aliases. Retained use after the invocation’s storage lifetime is not supported by this interface and would require an additional external contract.

The function does not visibly release the returned object or context. No ownership acquisition, borrowed-reference guarantee, or release obligation can be inferred for those values.

### Dependency effects and concurrency

Dependencies may affect globals or other state. Their preservation behavior is unknown.

The earlier mask decision is not repeated even if a dependency changes the mask. There is no visible synchronization, locking, or atomic-access contract. Behavior under concurrent mutation requires external guarantees and remains undocumented here.

## Dependencies

| Opaque function ID | Established role | Unknowns |
|---|---|---|
| F-9266831caef7 | Receives the provider result displaced by four bytes and the eligible index; returns an object pointer or null. | Lookup semantics, state changes, allocation, ownership, and failure causes. |
| F-a817d8d58c10 | Invoked when lookup returns null. | Actual control-transfer behavior and all other effects. Non-returning behavior is annotated but not independently verified. |
| F-unresolved-01 | Supplies the address-like value used to form the first lookup argument. | Identity, result validity, ownership, and side effects. |
| F-unresolved-02 | Indirect operation resolved through the object’s dispatch table at byte offset 24; receives the object and two output-word addresses. | Identity, output-writing guarantees, state changes, address retention, and failure behavior. |

F-unresolved-01 and F-unresolved-02 are document-local opaque labels. No supplied stable dependency IDs identify these two callable roles.

## Acceptance examples

The examples below are conditional static consequences, not runtime observations. Unless an example concerns eligibility or lookup failure, assume the selected mask bit is set, lookup succeeds, dispatch is valid, and the indirect operation returns normally.

| Conditions | Required ordinary result |
|---|---|
| Index is 13, irrespective of mask contents or request. | 0; no mask read or dependency call. |
| Index is the largest unsigned 32-bit value. | 0; no mask read or dependency call. |
| Index is 12 and mask bit 12 is clear. | 0; no provider, lookup, or indirect-operation call. |
| First component is 0 and second component is 0; request is 0. | 1. |
| First component is 0 and second component is 0; request is 1. | 0. |
| First component is 1 and second component is 2; request is 258. | 1, because the assembled limit is 258. |
| The same component pair; request is 259. | 0. |
| First word is 0x12340001 and second word is 0xABCD0002; request is 258. | 1; upper word bits are ignored. |
| First word has low byte 255, even if its upper bits or the operation’s return value suggest success. | 0. |
| First byte is valid and second word has low byte 255. | 0. |
| The indirect operation leaves both initialized words untouched. | 0. |
| Both component bytes are 254; request is 65,278. | 1. |
| Both component bytes are 254; request is 65,279. | 0. |
| Both component bytes are 254; request is 0xFFFFFFFF. | 0; the request is not truncated. |
| Lookup returns null. | F-a817d8d58c10 is invoked; no ordinary result is established. |

Changing only the first argument has no direct visible effect when the remaining inputs and execution environment are otherwise identical.

## Uncertainties

The following items are **documented_only**:

- The domain meaning and units of the index, components, and request.
- The concrete global-state layout beyond the enable-mask access.
- The context provider’s identity and valid result range.
- Whether the address adjustment can encounter an invalid or overflowing address. No explicit guard is visible, and the supplied representation does not establish a portable arithmetic outcome for such cases.
- Object lifetime, ownership, dispatch-table extent, and dependency thread safety.
- Whether the indirect operation writes whole words or narrower portions of them.
- Whether the indirect operation can mutate or invalidate the object, mask, context, or other global state.
- Whether the failure dependency actually prevents normal return.
- The machine-level ABI details beyond the visible 32-bit scalar operations and byte offsets.

The 32-bit widths of the input values, temporary component words, mask, and result are supported by the supplied representation. A complete pointer-width and address-arithmetic specification is not independently established.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence covers the index bound, mask gate, temporary-word initialization, dependency-call order, null-result branch, indirect-dispatch offsets, component-byte extraction, sentinel checks, limit assembly, and unsigned comparison.

The interpretation as a component-derived limit query is inferred intent. Dependency implementations, runtime memory validity, ownership guarantees, and exceptional behavior are unavailable.

No runtime tests have been performed.
