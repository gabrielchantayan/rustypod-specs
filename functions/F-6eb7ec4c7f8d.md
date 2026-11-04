<!-- Trusted provenance: F-6eb7ec4c7f8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6eb7ec4c7f8d

## Purpose

F-6eb7ec4c7f8d obtains a primary object address, performs partial initialization using shared state and constants, invokes an indirect dependency with an interior address of that object, then obtains and processes two additional values. If the entire sequence returns normally, the function returns the original primary object address.

The established contract concerns evaluations, calls, memory accesses, arguments, and ordering. Constructor-like initialization and acquisition of two resources are inferred intent only. The evidence does not establish allocation semantics, attachment semantics, ownership transfer, or a destruction protocol.

## Interface

### Explicit and implicit inputs

The function has no explicit parameters.

Its implicit inputs are:

- The result and effects of F-796ba019ae1d.
- Two ordered evaluations of the same shared 32-bit value after that dependency returns.
- The result and effects of F-df3f6f4fe2dc.
- A dispatch-table address and callable entry reached through that result.
- Two separately obtained results from F-8157a180d901.
- Any additional state observed by the dependencies.

The first shared-value evaluation is retained for two later additions. The second evaluation supplies the write at primary-object offset `0x00`. Equality between these observations requires an additional stability assumption; it is not an unconditional guarantee.

These are two evaluations in the supplied representation. The evidence does not establish a physical machine-load count, atomicity, volatility, synchronization, or a memory-consistency model.

### Result and representation

On normal completion, the result is the same primary object address returned by F-796ba019ae1d. Neither result from F-8157a180d901 replaces that return value.

The represented operations use four-byte integer words and 32-bit addresses. All object offsets in this contract are byte offsets from the retained primary address. A byte is one byte; a word is four bytes.

The public ABI, calling convention, concrete object type, and byte order are not established. Word values below describe four-byte values, not a specified byte serialization.

### Preconditions for normal execution

The function performs no local checks enforcing these conditions:

- The primary address must support each direct write when it occurs.
- Both evaluations of the shared value must be valid.
- The address returned by F-df3f6f4fe2dc must support a four-byte read at offset zero.
- The resulting dispatch-table address must support reading a callable entry at byte offset `0x98`.
- That entry must accept the two represented arguments.
- Dependencies must return normally for subsequent operations to occur.
- Dependency effects must leave later accesses valid if normal completion is expected.

The highest direct object write covers offsets `0xE4` through `0xE7`. This establishes required writable locations, not a complete object size or the memory-access requirements of dependencies. No null check, bounds check, lifetime check, or alignment check is visible.

## Behavior

### Primary address and shared-state observations

The first operation is a call to F-796ba019ae1d without explicit arguments. Its result is retained as the primary object address throughout the visible sequence.

The function next evaluates the shared 32-bit value and retains that first observation. It then evaluates the same shared value again to supply the four-byte write at primary offset `0x00`.

The two computed words at offsets `0xCC` and `0xD0` use the retained **first** observation, not the second evaluation and not a reread of offset `0x00`.

No dependency call intervenes between the two shared-value evaluations. Nevertheless, the supplied representation does not establish that the shared value is stable, or that the two evaluations necessarily produce equal values.

### Initial direct writes

The initial writes occur in the following order:

| Order | Object byte offset | Width | Value |
|---|---:|---:|---|
| 1 | `0x00` | 4 bytes | Second shared-value observation |
| 2 | `0xCC` | 4 bytes | First shared-value observation plus `0x16C` |
| 3 | `0xD0` | 4 bytes | First shared-value observation plus `0x178` |
| 4 | `0xD4` | 1 byte | Zero |
| 5 | `0xD5` | 1 byte | Zero |
| 6 | `0xD8` | 4 bytes | Signed 32-bit −1 |
| 7 | `0xE0` | 1 byte | Zero |
| 8 | `0xE1` | 1 byte | Zero |
| 9 | `0xE2` | 1 byte | Zero |

The addition constants are 364 and 376 decimal. The represented arithmetic is 32-bit signed integer arithmetic. The evidence does not resolve behavior when a mathematical sum exceeds the signed 32-bit range. Wrapping, trapping, or another overflow interpretation must not be imposed as an established contract.

This is partial initialization. In particular:

- Offsets `0xD6` and `0xD7` receive no direct write.
- Offset `0xE3` receives no direct write.
- The four-byte field at `0xDC` is written later.
- The four-byte field at `0xE4` is written later.
- No whole-object clearing operation is visible.

“Receives no direct write” does not mean that a location retains its original contents: a dependency may change it.

### Indirect invocation

After the nine initial writes, the function calls F-df3f6f4fe2dc without explicit arguments.

From its returned address, the function reads a four-byte value at offset zero and uses that value as a dispatch-table address. It then reads the callable entry at dispatch-table byte offset `0x98` and invokes it with:

1. The address returned by F-df3f6f4fe2dc.
2. The retained primary object address plus `0xCC`.

The second argument is an interior address. It is not the word stored at that address and is not a copy of the surrounding fields.

That address identifies the location **previously written** with the first shared observation plus `0x16C`. The following word and selected nearby bytes were also previously initialized as listed above. Their contents at indirect-call entry are not unconditional guarantees: F-df3f6f4fe2dc has already run, and its mutation behavior is unknown.

The indirect call’s result is unused. Its effects on object contents, shared state, retained addresses, or memory lifetime are not established. The visible sequence continues using the retained primary address if the call returns.

### First additional value

After the indirect invocation returns, the function calls F-8157a180d901 with one explicit argument, `0x2C`, equal to 44 decimal.

The function then:

- Stores the returned 32-bit value at primary-object offset `0xDC`.
- Calls F-861a66c91b48 with four arguments, in order: the retained primary address, that returned value, zero, and zero.

There is no local test of the returned value before the store or subsequent call. The second argument to F-861a66c91b48 comes from the request result itself; it is not obtained by rereading the field at `0xDC`.

The result of F-861a66c91b48 is unused. Its effects are unknown.

### Second additional value

After the first call to F-861a66c91b48 returns, the function calls F-8157a180d901 again with `0x2C`.

It stores this second returned 32-bit value at primary-object offset `0xE4`, then calls F-861a66c91b48 with the retained primary address, the second returned value, zero, and zero.

This is a separate request and processing sequence. The function does not compare the two results, require distinct results, deduplicate equal results, or reject zero.

The second F-861a66c91b48 result is also unused.

### Return and interrupted execution

After the second processing call returns, the function returns the original primary address. It performs no final validation, restoration of earlier fields, result-status conversion, or explicit cleanup.

If a dependency fails to return normally, the visible sequence establishes no continuation past that call. If a required memory access is invalid, no local recovery behavior is shown. A specific exception, fault, error code, or termination mechanism is not established.

Earlier writes are not rolled back by any visible operation. They also do not guarantee final field contents, because intervening dependencies may alter those locations.

## Side effects

The function directly performs eleven object writes: nine initial writes and two later four-byte result stores.

Its explicit call order is:

1. F-796ba019ae1d.
2. F-df3f6f4fe2dc.
3. The indirect dispatch target.
4. F-8157a180d901.
5. F-861a66c91b48.
6. F-8157a180d901.
7. F-861a66c91b48.

The two shared-value evaluations occur after the first call and before the initial computed-field writes. The first observation supplies both additions; the second supplies the write at offset `0x00`. This description specifies represented evaluation order, not a machine-load count.

Other explicit reads obtain the dispatch-table address and its callable entry. The function does not explicitly reread either stored request result before passing it onward.

The visible body contains no direct assignment to the shared-value location. This does not guarantee that the location remains unchanged: dependencies may modify it, and the evidence supplies no non-aliasing guarantee between that location and other addressed storage.

No direct release, deallocation, or rollback operation is visible. This does not establish that dependencies refrain from releasing memory or assuming ownership. In particular, successful completion cannot be taken as proof that the returned address still denotes a live object under an unknown dependency contract.

Call order is material. Effects of the first processing call may influence the second request. Combining, reordering, or removing the repeated calls is not justified by the supplied evidence.

## Dependencies

| Opaque dependency | Established use | Unresolved contract |
|---|---|---|
| F-796ba019ae1d | Called once without explicit arguments; result retained as the primary address and eventual return value | Allocation, initial contents, size, ownership, failure behavior |
| F-df3f6f4fe2dc | Called once after initial writes; result supplies the indirect receiver and dispatch lookup | Mutation, receiver lifetime, dispatch validity, failure behavior |
| F-8157a180d901 | Called twice with `0x2C`; each result is stored and passed onward | Meaning of the argument, result type beyond its represented width, failure sentinel, ownership |
| F-861a66c91b48 | Called twice with the primary address, the corresponding request result, and two zeros | Processing or attachment semantics, mutation, retention, release, error reporting |

The indirect dependency is identified only by the entry at byte offset `0x98` in the table reached through F-df3f6f4fe2dc’s result. No separate opaque function ID is supplied for that target. Its identity and semantic contract remain documented_only.

No dependency is assumed to preserve globals, preserve primary-object fields, retain or release memory, or leave memory lifetimes unchanged.

## Acceptance examples

These examples are conditional static acceptance criteria. They are not executed tests.

### Stable shared state and non-mutating dependencies

Assume both shared-value evaluations produce `0x1000`, all arithmetic is in range, all accesses are valid, every dependency returns normally, and dependencies leave the relevant fields unchanged.

At return:

- Offset `0x00` contains `0x1000`, from the second observation.
- Offset `0xCC` contains `0x116C`, computed from the first observation.
- Offset `0xD0` contains `0x1178`, computed from the first observation.
- Offsets `0xD4`, `0xD5`, `0xE0`, `0xE1`, and `0xE2` contain zero bytes.
- Offset `0xD8` contains signed 32-bit −1.
- Offsets `0xDC` and `0xE4` contain the first and second request results, respectively.

The returned address equals the result of F-796ba019ae1d.

### Different shared-state observations

Assume the first shared-value evaluation produces `0x1000` and the second produces `0x2000`.

The initial writes must place `0x2000` at offset `0x00`, `0x116C` at `0xCC`, and `0x1178` at `0xD0`. Using the second observation for either addition would not match the supplied behavior.

This example does not assert that a particular runtime permits such differing observations; that depends on the unresolved shared-state and execution model.

### Observation at indirect-call entry

Assume F-df3f6f4fe2dc returns normally. At indirect-call entry, the second argument is the retained primary address plus `0xCC`, and all nine initial writes have already occurred. Neither request-result store has yet occurred.

If F-df3f6f4fe2dc leaves the relevant fields unchanged, those fields still contain the initial values. If it changes them, the changed contents are permitted; the function performs no restoration before the indirect invocation.

### Equal or zero request results

If both requests return the same value, both requests, both stores, and both processing calls still occur.

If either request returns zero, zero is stored in the corresponding field and passed as the second argument to its processing call. No local rejection occurs. Whether the dependency accepts zero remains unknown.

### Mutation and non-return

If an indirect or processing dependency changes an initialized field and returns normally, that earlier field value is not restored by this function unless a later explicit store targets the same location.

If a dependency does not return normally, later operations are not reached through the visible sequence. No local cleanup or alternative return is established.

## Uncertainties

The following remain documented_only:

- Concrete object type, complete layout, alignment, byte order, and public ABI.
- Meaning and stability rules of the shared value.
- Physical load count and concurrency semantics of its two evaluations.
- Intended interpretation and overflow behavior of the computed words.
- Meaning of the zero bytes and the −1 word.
- Indirect-target identity, access range, and retention behavior.
- Whether `0x2C` is a size or another request parameter.
- Ownership, validity, and lifetime of all returned values.
- Dependency mutation, release, failure, exception, and error-reporting behavior.
- Aliasing, synchronization, thread safety, and reentrancy.

These unknowns preclude an unconditional final-state, resource-acquisition, or safe-destruction guarantee.

## Evidence

Established facts derive solely from static inspection of the supplied representation: two ordered shared-value evaluations and their distinct uses, direct write widths and offsets, call order, dispatch lookup, argument values, unused call results, and return of the retained primary address.

Object construction and resource attachment are inferred intent, not established semantics. Dependency implementations and runtime memory behavior are unavailable.

No runtime tests have been performed. No execution trace, allocator contract, object definition, or observed failure behavior supports stronger claims.
