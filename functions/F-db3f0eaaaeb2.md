<!-- Trusted provenance: F-db3f0eaaaeb2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db3f0eaaaeb2

## Purpose

F-db3f0eaaaeb2 performs signed-index processing using caller-supplied context and seed values. It delegates initial bound adjustment to F-80474946e6fd, then conditionally invokes F-3593620aad0a and F-5ad1202cca6f for successive indices.

Admission to each iteration depends on whether the current signed index is less than or equal to the **current contents of writable upper-bound storage**. The upper bound is not established as a fixed snapshot. F-80474946e6fd receives its address and may retain it; later dependencies may modify that still-live storage through the retained address.

A nonzero result from F-5ad1202cca6f stops processing. A zero result causes an index increment followed by another admission check, subject to the signed-overflow uncertainty described below.

The 64-bit result contains the final working word in its upper 32 bits. Its lower 32 bits are zero on admission-check exhaustion, or a global word sampled after the callback that causes early termination.

This establishes index traversal with callback-controlled stopping and a mutable admission bound. A more specific interpretation, such as searching for a match or reporting an error, is not established.

## Interface

### Inputs

The function accepts four arguments, in this order:

| Position | Representation | Use |
|---|---|---|
| 1 | Signed 32-bit value used as a context address | Passed to F-80474946e6fd; used as the base for a 32-bit read at byte offset `0xBC` on every admitted iteration |
| 2 | Uninterpreted 32-bit seed word | Initializes working storage and resets it before each callback |
| 3 | Signed 32-bit initial upper bound | Initializes writable upper-bound storage |
| 4 | Signed 32-bit initial lower bound | Initializes writable lower-bound storage |

The upper-bound argument precedes the lower-bound argument. The adjustment dependency receives pointers in the opposite semantic order: lower-bound storage first, upper-bound storage second.

The context value is used as an address, but the supplied representation does not establish a portable pointer type, address-space interpretation, or context object layout beyond the observed field access.

### Result

The return value is 64 bits:

| Bit positions | Contents |
|---|---|
| 63–32 | Final 32-bit working-word contents |
| 31–0 | Zero on failed admission; otherwise the global sample associated with the stopping callback |

Packing preserves both words’ bit patterns. The evidence does not establish signed interpretation of the packed result, a pointer interpretation, or a named aggregate type.

Consequences:

- If the first admission check fails, the upper word remains the initialized seed and the lower word is zero.
- If processing ends through a later failed admission check, the upper word contains the last callback’s working-word result and the lower word is zero.
- If a callback returns nonzero, the upper word contains its resulting working word and the lower word contains the subsequent global sample.
- The callback’s return value controls stopping but is not itself packed into the result.
- A zero lower result word does not distinguish exhaustion from early termination with a zero global sample.

### Preconditions and lifetime

For defined accesses and normal completion:

- F-80474946e6fd must accept the context and pointers to writable 32-bit lower- and upper-bound storage.
- Every admitted iteration requires a valid 32-bit read at context offset `0xBC`.
- F-3593620aad0a must accept the field value and current signed index.
- F-5ad1202cca6f must accept the lookup result and a pointer to writable 32-bit working storage.
- The designated global location must support a 32-bit read after every returning callback.
- Any indirect access through retained local-storage pointers must occur while that storage remains valid.

The bounds need not initially describe a nonempty range. Initial emptiness does not guarantee that processing is skipped, because adjustment occurs before the first admission check.

No visible validation enforces these conditions. Upper-bound stability is **not** a general precondition: mutation can affect traversal. Stability is an additional assumption only where a fixed-range expectation is stated.

## Behavior

### Initialization and adjustment

Before calling a dependency, the function initializes three 32-bit storage locations:

- Working storage receives the seed.
- Upper-bound storage receives the third argument.
- Lower-bound storage receives the fourth argument.

It calls F-80474946e6fd with the context, the address of lower-bound storage, and the address of upper-bound storage, in that order.

The dependency’s return value is not used. Its resulting bound contents are used. The visible function does not independently clamp, swap, validate, or normalize them.

After this call, the current index is initialized from lower-bound storage. That storage is not visibly reread during traversal. By contrast, upper-bound storage is consulted at each admission check.

### Admission and mutable upper bound

Before each admission comparison, the prospective lower result word is set to zero. The current index is then compared with the current upper-bound contents using signed 32-bit ordering.

If the index exceeds that bound, processing ends without another context-field read, lookup, callback, or global sample.

If the index is less than or equal to that bound, one iteration is admitted.

The address of upper-bound storage escapes to F-80474946e6fd. No supplied evidence establishes that the dependency discards the address. Later dependency calls may therefore cause writes to the still-live upper-bound storage through a retained pointer.

Such writes affect the next admission check:

- Raising the bound can admit indices beyond the value left by initial adjustment.
- Lowering the bound can stop processing before reaching that initially adjusted value.
- A change made during an already admitted iteration does not trigger an immediate second admission check. The admitted iteration proceeds through its visible operations unless a dependency fails to return.
- Mutation of lower-bound storage after index initialization does not, by itself, change the current index.

Consequently, a zero-returning callback does not guarantee that the next index will be processed. It guarantees only progression to the increment and subsequent admission check, provided those operations have a defined outcome.

No fixed iteration count follows merely from valid accesses, normally returning dependencies, and the bounds present immediately after adjustment.

### Per-index ordering

For each admitted index, the visible sequence is:

1. Read the 32-bit field at context byte offset `0xBC`.
2. Call F-3593620aad0a with that field value and the current signed index.
3. Reset working storage to the original seed.
4. Call F-5ad1202cca6f with the 32-bit lookup result and the address of working storage.
5. Read the designated global 32-bit word.
6. Test whether the callback’s signed 32-bit return is nonzero.

The context field is read afresh on every iteration. A dependency’s earlier change to that field can therefore affect a subsequent lookup.

The seed reset occurs **after** lookup returns. Each callback starts with the seed in the supplied working storage, rather than automatically inheriting the previous callback’s result. A lookup-side modification of that working storage through a previously retained pointer would also be overwritten by this reset before the next callback.

The callback may modify the working word. The function does not require it to replace the entire word; whatever 32-bit contents remain contribute to the eventual upper result word.

The global read occurs after every returning callback, including a callback that returns zero. It is not conditional on early termination.

### Stopping and exhaustion

If the callback returns nonzero:

- The function stops after the global sample and callback-result test.
- No index increment occurs.
- No further admission check occurs.
- The sampled global word becomes the lower result word.

Positive and negative nonzero callback results have the same visible stopping effect.

If the callback returns zero:

- The current index is incremented by one.
- Another admission check resets the prospective lower result word to zero and compares against the then-current upper-bound contents.

A global sample from a continuing iteration is therefore not preserved as the lower result word on later admission-check exhaustion.

Exhaustion means failure of the current admission comparison. It does not necessarily mean that the upper bound produced by initial adjustment has been processed.

### Widths and arithmetic limits

The bounds, current index, and callback result are represented as signed 32-bit integers. The seed, working word, context field, lookup result, and global sample are 32-bit quantities without established signed semantics.

Signed comparison permits negative indices. This function does not visibly reject them; dependency support remains a separate requirement.

If index `2,147,483,647` is processed and the callback returns zero, the next visible operation attempts a signed increment beyond the representable range. The supplied representation does not establish a portable result for that overflow. This remains **documented_only**.

Changing the upper bound during that callback does not visibly avoid the increment: the increment precedes the next admission check. A nonzero callback result does avoid it.

The context-relative address calculation likewise depends on the target address model. The observed offset is `0xBC` and the read width is 32 bits; address-overflow behavior is not established.

## Side effects

### Direct effects

The visible function:

- Initializes and updates invocation-local storage.
- Exposes writable bound-storage addresses to F-80474946e6fd.
- Exposes writable working-storage addresses to F-5ad1202cca6f.
- Reads the context field once per admitted iteration.
- Reads the designated global word once after each returning callback.
- Returns a packed 64-bit value.

No direct heap allocation, release, context-field write, or global write is shown.

### Delegated mutation and ownership

Dependency bodies are unavailable. This contract does not assume that they preserve globals, preserve the context, or refrain from allocating or releasing memory.

Pointer retention matters during the invocation, not only after return. In particular, retained access to upper-bound storage can change traversal while that storage is still live. The function does not copy the adjusted upper bound into an independently fixed traversal limit.

The bound and working pointers refer to invocation-local storage. No ownership transfer or durable lifetime is established. Retaining an address does not establish that later dereference after the invocation is valid.

The lookup result is forwarded as an opaque 32-bit value. Its ownership, pointer interpretation, and release requirements are unknown. No explicit release appears in the visible function.

No synchronization is shown around the context field, bound storage, working storage, or global sample. Concurrency, reentrancy, and access atomicity are **documented_only**. Possible synchronous mutation by returning dependencies is sufficient to explain the mutable-bound behavior; concurrent execution need not be assumed.

### Errors and abnormal completion

No explicit error return, null check, exception handler, or cleanup branch is shown.

A nonzero callback result establishes early termination, not necessarily failure. The sampled global word is not established as an error code.

If a dependency does not return normally, subsequent visible operations are not established. Recovery from invalid accesses, dependency failures, or arithmetic overflow is unknown.

## Dependencies

| Opaque function ID | Inputs supplied | Outputs used |
|---|---|---|
| F-80474946e6fd | Context; pointer to lower-bound storage; pointer to upper-bound storage | Bound contents after return; possible retained access to those locations |
| F-3593620aad0a | Freshly read 32-bit context field; current signed index | A 32-bit value passed to the callback |
| F-5ad1202cca6f | Lookup result; pointer to seed-reset working storage | Working-word contents; signed 32-bit continuation decision |

There is also a global 32-bit data dependency, sampled immediately after each returning callback.

No dependency is assumed to preserve that global. The relevant sample is the value read after the callback, not a value present before lookup or callback execution.

The dependency interface establishes possible access to bound storage but not a particular retention mechanism or mutation policy. Actual retention and subsequent mutation remain unknown; their possibility prevents unconditional fixed-range guarantees.

## Acceptance examples

These are conditional expectations from static evidence, not executed tests. Unless stated otherwise, they assume valid accesses and normally returning dependencies. Fixed-count examples additionally state the necessary upper-bound stability condition.

### Empty first admission

Suppose the first admission check sees lower-derived index `9` and upper bound `8`, with seed `0x12345678`.

Expected behavior:

- Range adjustment has run.
- No lookup, callback, or per-iteration global read occurs.
- The result is `0x1234567800000000`.

The original argument bounds do not determine this outcome; the values used at the first admission check do.

### Single-element exhaustion with stable upper bound

Suppose adjustment leaves lower bound `4` and upper bound `4`, and upper-bound storage remains `4` throughout traversal. The callback writes `0xAABBCCDD`, returns zero, and the following global sample is `0x11223344`.

Expected behavior:

- Exactly one lookup and callback occur, at index `4`.
- The global read occurs despite the zero callback result.
- The increment produces index `5`; the next admission check fails.
- The result is `0xAABBCCDD00000000`.

Without upper-bound stability, exactly one iteration is not guaranteed.

### Extension through retained upper-bound access

Suppose adjustment leaves bounds `4` through `4`. During the index-`4` callback, a dependency uses retained access to change upper-bound storage to `5`. The callback returns zero.

Expected behavior:

- The current iteration completes, including its global sample.
- The index increments to `5`.
- The next check sees upper bound `5` and admits index `5`.

This is consistent with the visible interface. It is not evidence that any actual dependency performs such a write.

### Early stop despite bound mutation

Suppose index `2` is admitted. Its callback writes `0x01020304`, returns a negative nonzero value, and the following global sample is `0x55667788`.

Expected behavior:

- No later index is processed.
- The result is `0x0102030455667788`.
- Upper-bound changes during that iteration do not override the nonzero stopping result.
- The callback’s negative return value is not packed into the result.

### Working-word reset

Suppose the seed is `0x00000010`. The callback at index `0` writes `0x00000020` and returns zero. The next admission check permits index `1`.

Expected behavior:

- After the index-`1` lookup returns, working storage is reset to `0x00000010`.
- The second callback starts with that seed, not the preceding callback’s output.

This example conditions the second callback on admission; the initially adjusted bounds alone do not guarantee it occurs.

### Negative indices with stable upper bound

Suppose adjustment leaves bounds `-2` through `0`, upper-bound storage remains `0` throughout traversal, and all callbacks return zero.

Expected behavior:

- Indices are processed in order: `-2`, `-1`, `0`.
- The next index is `1`, whose admission fails.
- The lower result word is zero.

Dependency acceptance of negative indices remains required.

If instead the first callback changes the upper bound to `-2` and returns zero, the next index is `-1`, whose admission fails. Neither `-1` nor `0` is then processed.

### Maximum signed index

Suppose index `2,147,483,647` is admitted.

- A nonzero callback result stops processing without an increment.
- A zero result reaches the signed-overflow boundary, even if the callback lowers the upper bound.

No ordinary-exhaustion result is specified for the latter case.

## Uncertainties

The following remain **documented_only**:

- Context layout beyond the observed field access.
- Range-adjustment policy and dependency return semantics not used here.
- Whether dependencies retain bound or working-storage pointers.
- Whether, when, or how later dependencies mutate upper-bound storage.
- Lookup-result meaning and ownership.
- Callback interpretation of the seed and working word.
- Domain meaning of callback termination and the global sample.
- Additional dependency effects on context, globals, allocation, and release.
- Post-return use of invocation-local addresses.
- Signed-increment overflow and context-address arithmetic limits.
- Concurrency, reentrancy, synchronization, and atomicity guarantees.

The upper-bound mutation policy is unknown. The rule that admission consults current upper-bound contents is established and does not depend on resolving that policy.

## Evidence

Established static evidence supports the argument widths and ordering, local initialization, adjustment-pointer order, one-time index initialization from the lower bound, repeated signed comparison against upper-bound storage, context-field offset and read width, dependency-call order, per-callback seed reset, unconditional post-callback global read, nonzero stopping rule, increment placement, and two-word return packing.

Possible later mutation of upper-bound storage follows from its escaped address and the absence of dependency behavior establishing non-retention or preservation. It is a permitted conditional behavior, not an observed dependency action.

Inferred intent is limited to range-like processing with callback-controlled stopping. Domain semantics remain unknown.

No runtime tests have been performed. All acceptance examples are conditional static expectations.
