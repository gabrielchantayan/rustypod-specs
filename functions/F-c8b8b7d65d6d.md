<!-- Trusted provenance: F-c8b8b7d65d6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8b8b7d65d6d

## Purpose

F-c8b8b7d65d6d performs a position-dependent extraction through an object’s indirect operations. It first requests an absolute position eight bytes beyond a supplied two-word position, obtains a four-byte discriminator, and selects one of two layouts. Both layouts request a four-byte value for one caller-provided destination. The selected layout then supplies either a four-byte value expanded into two words or an eight-byte value delivered directly to a second destination.

The function returns a 32-bit status: zero when the selected sequence reports all required results, and one otherwise.

The interpretation as positioning and reading is **inferred intent**, supported by argument shapes and result checks. The directly established behavior is a sequence of indirect calls, comparisons, conditional branches, and output stores. The supplied evidence does not define the implementations of those indirect operations.

## Interface

### Inputs

The function receives six arguments:

| Argument | Contract |
|---|---|
| Context word | A 32-bit value not explicitly used by this function. Its value does not select a branch or appear in a visible dependency call. |
| Operation object | A pointer to an object whose first 32-bit word supplies an operation-table address. The function resolves indirect call targets from this table. |
| Position low word | An unsigned 32-bit value supplying the low part of the initial position. |
| Position high word | An unsigned 32-bit value supplying the high part of the initial position. It also initializes a temporary four-byte storage location before any dependency call. |
| Four-byte destination | A caller-supplied address passed to a dependency with a requested transfer size of four bytes. |
| Two-word destination | A caller-supplied address treated as storage for two consecutive 32-bit words. Depending on the branch, this storage is either explicitly assigned by this function or passed to a dependency with a requested transfer size of eight bytes. |

The observed representation uses four-byte words and operation-table entries. The two-word output occupies eight bytes, with the second word located four bytes after the first.

No explicit validation checks reject null pointers, invalid operation tables, inaccessible destinations, misalignment, or overlapping storage.

### Return value

The return value is a 32-bit word:

- **0:** Every checked dependency result in the selected path matched its required value, and the path reached its success return.
- **1:** A dependency returned normally but its checked result did not match the required value.

There are no distinct returned error codes for positioning failure, a short transfer, a negative dependency result, or another unexpected result.

A fault, exception, nonlocal exit, or failure to return from a dependency is not converted into status one by any visible mechanism.

### Preconditions

For defined completion, the object, operation table, indirect targets, and relevant storage must remain usable at every access that the function actually performs.

The four-byte destination must support the behavior of a dependency receiving that address and a four-byte request. The two-word destination must support either two explicit four-byte stores or an eight-byte dependency request, according to the selected branch.

The function visibly allocates and releases no caller-owned resource. It does not visibly retain either destination after returning. Retention, ownership changes, or memory management performed by dependencies remain unknown.

## Behavior

### 1. Prepare and request the initial position

A temporary unsigned 32-bit word is initialized from the position high word.

The function then calls the positioning dependency with:

- The operation object.
- A function-pointer value obtained from the positioning operation’s table entry.
- An adjusted low position word.
- An adjusted high position word.
- A final selector value of zero.

The adjustment adds eight to the supplied two-word position:

- The low word is the supplied low word plus eight, reduced modulo \(2^{32}\).
- A carry of one is added to the supplied high word exactly when the original low word is at least `0xfffffff8`.
- The high-word addition also wraps modulo \(2^{32}\).

Consequently, the pair implements addition of eight modulo \(2^{64}\), when interpreted as low word followed by high word. No overflow error is reported.

The indirect call must return exactly zero for processing to continue. Any other normal return value causes an immediate return of one.

The final selector’s interpretation as an absolute-position mode is inferred, not established by a dependency definition.

### 2. Request the discriminator

After successful initial positioning, the function calls the transfer dependency with:

- The operation object.
- The address of the temporary four-byte word.
- A requested size of four.
- A final selector value of one.

Only an exact return value of four permits further processing. Any other normal return causes an immediate return of one.

The branch decision uses the temporary word’s value after this call. The test is numeric: it checks whether the word’s upper eight bits are zero.

This is not independently a test of a particular byte’s address in memory. The correspondence between the upper eight bits and a byte in a serialized representation depends on byte order and dependency behavior, neither of which is established here.

A return value of four is accepted without independently verifying that four bytes were written. If a dependency reports four while leaving some or all of the temporary storage unchanged, the branch uses whatever value that storage then contains. Its initial value was the position high word.

### 3. Resolve the branch positioning operation

After the discriminator request returns four, the function resolves the positioning operation again from the object’s then-current table.

That resolved function-pointer value is used both as the indirect call target and as an explicit argument in the upcoming positioning call.

The next action depends on the upper eight bits of the temporary word.

### 4. Path with a zero upper byte

If the discriminator word satisfies the upper-byte-zero test, the function requests an additional positioning operation with:

- The operation object.
- The resolved positioning function-pointer value.
- A low offset word of eight.
- A high offset word of zero.
- A final selector value of one.

The selector and positive offset suggest relative forward movement, but that meaning remains inferred.

The result must be exactly zero. Otherwise, the function returns one without requesting either output transfer.

After successful positioning:

1. The transfer dependency receives the four-byte destination, a requested size of four, and selector one.
2. Only an exact return value of four permits continuation.
3. The transfer dependency then receives the temporary word’s address, a requested size of four, and selector one.
4. Only an exact return value of four permits continuation.
5. The function explicitly stores the temporary word into the first word of the two-word destination.
6. It explicitly stores zero into the second word of that destination.
7. It returns zero.

The first output store precedes the second output store.

Under the inferred read interpretation, this path obtains a 32-bit value and represents it as a zero-extended two-word value. The zero extension is established by the explicit second-word store, not by assumptions about the dependency.

### 5. Path with a nonzero upper byte

If any of the discriminator word’s upper eight bits are set, the function instead requests an additional positioning operation with:

- The operation object.
- The resolved positioning function-pointer value.
- A low offset word of sixteen.
- A high offset word of zero.
- A final selector value of one.

The result must be exactly zero.

After successful positioning:

1. The transfer dependency receives the four-byte destination, a requested size of four, and selector one.
2. Only an exact return value of four permits continuation.
3. The transfer dependency receives the two-word destination directly, a requested size of eight, and selector one.
4. Only an exact return value of eight permits success.
5. The function returns zero.

This path performs no explicit normalization, word rearrangement, or zero extension of the eight-byte output.

### 6. Short-circuit failure behavior

The first mismatching result ends the normal sequence immediately. There is no retry, alternate-layout fallback, cleanup call, or position restoration.

In particular:

- Failed initial positioning prevents the discriminator request.
- A discriminator result other than four prevents both branch positioning and output requests.
- Failed branch positioning prevents output requests.
- A first output result other than four prevents the final transfer.
- A failed final transfer produces status one even though the earlier four-byte destination may already have changed.

The function does not provide transactional output semantics.

## Side effects

### Object and external state

The function explicitly reads the object’s table address and operation entries. It does not explicitly assign an object field or a global variable.

Dependencies can nevertheless modify the object, replace its table or entries, modify globals, change an underlying position, or perform other externally visible work. Their behavior is not supplied.

Later transfer operations resolve their targets from the object again rather than relying on a transfer function captured at entry. Accordingly, mutations made by an earlier dependency may affect which function receives a later call.

The positioning operation used for branch-specific movement is resolved after the discriminator transfer. The evidence does not establish a general snapshot or consistency guarantee for operation-table reads.

### Output visibility and failure

The four-byte destination is exposed to the transfer dependency before the final output operation completes. Its contents may therefore change on either a successful or a failing invocation.

In the zero-upper-byte path, the function’s explicit stores to the two-word destination occur only after the final temporary-word request reports four. Those stores are not reached if that request reports another result. This does not establish that the destination was otherwise untouched: dependencies and aliases can affect it.

In the nonzero-upper-byte path, the two-word destination is passed directly to the final dependency. A return other than eight does not imply that the destination is unchanged. Partial writes are neither detected beyond the return comparison nor rolled back.

### Aliasing

No visible check requires the destinations to be disjoint.

If the four-byte destination overlaps the two-word destination, later output operations may overwrite earlier output bytes. For example, when both destinations start at the same address in the zero-upper-byte path, the explicit two-word stores occur after the four-byte transfer and replace the overlapping storage.

Aliasing destinations with the operation object or its table is more hazardous: writes can alter subsequent dispatch reads. The function does not prevent this or restore corrupted dispatch state.

## Dependencies

The following opaque IDs are specification-local labels for indirect operation roles. They are not recovered identities of concrete implementations. Different resolutions of one role may produce different callable targets.

### F-6d9a30e17bc2

The positioning role, resolved through the operation-table entry at byte displacement twenty.

Observed call contract:

- Five arguments: object, function-pointer value, unsigned 32-bit low word, unsigned 32-bit high word, and a selector.
- Required result: exactly zero.
- Initial invocation: adjusted input position and selector zero.
- Branch invocation: either `(8, 0)` or `(16, 0)` as the position-word pair, with selector one.

**Inferred intent:** positioning with absolute and relative modes.

**Documented_only unknowns:** actual selector meanings, units, pointer-argument purpose, state changes on failure, validity constraints, and ownership effects.

### F-a28f519d0ce4

The transfer role, resolved through the operation-table entry at byte displacement sixteen.

Observed call contract:

- Four arguments: object, destination address, requested byte count, and selector one.
- Requested counts are four or eight.
- Progression requires a result exactly equal to the requested count.

**Inferred intent:** reading bytes into the supplied destination.

**Documented_only unknowns:** whether the selector denotes reading, whether the result is formally a byte count, byte order, partial-transfer guarantees, position advancement, memory retention, and side effects outside the destination.

The function does not verify the semantic truth of either dependency’s reported result.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Ordinary zero-upper-byte success

Given that initial positioning reports zero, the discriminator request reports four and leaves the temporary word equal to `0x00abcdef`, and all later requests report their required results:

- The branch positioning request uses offset words `(8, 0)` and selector one.
- The four-byte destination is requested before the final temporary-word transfer.
- If that final transfer leaves the temporary word equal to `0xfedcba98`, the explicit two-word output becomes first word `0xfedcba98`, second word zero.
- The function returns zero.

### Nonzero-upper-byte success

Given a discriminator value of `0x01000000` and all required successful results:

- The branch positioning request uses `(16, 0)` and selector one.
- The four-byte output request precedes the direct eight-byte output request.
- The function returns zero without explicitly rewriting either word of the eight-byte destination.

### Upper-byte boundary

A discriminator of `0x00ffffff` selects the eight-byte positioning offset. A discriminator of `0x01000000` selects the sixteen-byte positioning offset.

The lower twenty-four bits do not otherwise affect branch selection.

### Initial-position carry

For position words `(0xfffffff7, 7)`, the initial positioning request receives `(0xffffffff, 7)`.

For `(0xfffffff8, 7)`, it receives `(0, 8)`.

For `(0xffffffff, 0xffffffff)`, it receives `(7, 0)` because both word additions wrap.

### Reported transfer failure

If the four-byte destination request returns three, the function returns one and does not make the final transfer request. Any writes already performed by that request remain observable; the function performs no rollback.

### Eight-byte short result

In the nonzero-upper-byte path, if the final dependency returns seven, the function returns one. It does not clear the two-word destination or undo the prior four-byte output.

## Uncertainties

The following are **documented_only**:

- Concrete dependency identities and implementations.
- Precise meaning of the positioning and transfer selectors.
- Serialized format, field names, and units.
- Platform byte order and the interpretation of direct eight-byte output.
- Whether reported transfer counts guarantee corresponding writes.
- Dependency effects on globals, object state, ownership, and memory lifetime.
- Behavior under concurrent mutation of the object, table, or destinations.
- Fault and exception behavior for invalid storage or invalid indirect targets.

The evidence supports two layout paths, but does not establish that they correspond to particular format versions, record types, or semantic size classes.

## Evidence

This contract is based solely on the supplied static function representation.

**Established static evidence:** six arguments; an unused first word; four-byte word arithmetic; operation-table displacements; indirect-call arguments; exact result comparisons; the upper-byte discriminator; short-circuit ordering; explicit two-word output stores; and zero/one return values.

**Inferred intent:** a seek-like operation, a read-like operation, and selection between narrow and wide output layouts.

**Verification status:** no runtime tests have been performed. The acceptance examples describe conditions an implementation should satisfy under the stated dependency behavior; they are not observed execution results.
