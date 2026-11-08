<!-- Trusted provenance: F-ec10021fad7a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec10021fad7a

## Purpose

F-ec10021fad7a dispatches a 32-bit selector using separately read external dispatch values. Depending on the selected path, it acknowledges a selector, writes a byte, invokes a dependency, forwards an indirect callback’s result, or returns an unhandled-selector word.

The established behavior is dispatch and delegation. Application-level meanings of the selectors, referenced objects, and dependency results are unknown. “Context,” “payload,” and “anchor” describe their roles here, not established application types.

Most recognized paths return the 32-bit word `0`. The fallback returns `0xFFFFFFCE`, equivalent to −50 under signed 32-bit interpretation. The callback path returns the callback’s 32-bit result unchanged.

Dispatch does **not** establish a single snapshot of the external values. In particular, the anchor observation used to calculate a displacement is separate from the observations used for equality and ordering.

## Interface

### Arguments

The function receives three arguments:

| Argument | Width and role |
|---|---|
| Context address | A 32-bit address-bearing value passed to dependencies and used as the base of one byte write at offset `0x42`. |
| Selector | A 32-bit integer used in equality comparisons, signed ordering comparisons, subtraction, and conditional addition. |
| Payload address | An address of storage accessed in four-byte element increments, with one path writing only a single byte. |

The directly accessed payload locations are:

- Offset `0`: a 32-bit element, interpreted as an object address on dependency and callback paths.
- Offset `4`: a 32-bit element used as a callback argument or result destination; a separate path writes only its first byte.
- Offset `8`: a 32-bit callback argument.

The object layout used by particular paths includes:

| Object offset | Access |
|---|---|
| `0x0C` | Read a 32-bit argument for F-fcc54fe9487b. |
| `0x190`, decimal 400 | Read the callable address for an indirect invocation. |
| `0x194` | Read a distinct 32-bit callback argument. |

These offsets do not establish a complete object layout or a universal payload format.

### External dispatch storage

Three external 32-bit values participate:

- **Anchor A:** read for the initial displacement, then separately for equality and, conditionally, ordering.
- **Anchor B:** read only after entry to the secondary group, with separate displacement, equality, and conditional ordering observations.
- **Adjustment C:** read only on the final adjustment path.

Their configured values, initialization, mutability, and relationships are unknown.

### Result and preconditions

The result is a 32-bit word: zero, the callback result, or `0xFFFFFFCE`.

Required memory validity depends on the selected path:

- Every invocation requires the initial Anchor-A reads and arithmetic.
- The context-byte path requires a writable byte at context offset `0x42`.
- The payload-byte path requires a writable byte at payload offset `4`.
- Dependency paths that consume the first payload element require that element to be readable.
- The callback path requires readable payload elements, readable object fields, and a valid callable target.
- The successful preparation path requires a readable object field at offset `0x0C` and a writable 32-bit payload element at offset `4`.

Dependencies may impose additional requirements not established here. No argument null checks, bounds checks, object validation, or callback validation are visible.

Paths with no direct argument-memory access do not, on this evidence alone, require usable context or payload storage. They still read dispatch storage. Arithmetic-boundary behavior remains unresolved as described below.

## Behavior

### Source-visible dispatch reads and precedence

The following sequence is established by the supplied static representation. Each named observation denotes a separate source-visible read, not a cached anchor value.

1. Read Anchor A and calculate a 32-bit displacement as selector minus that observation. This occurs **before** the equality test and occurs even when the equality test subsequently succeeds.
2. Read Anchor A again for an equality comparison with the selector.
   - If equal, select the context-byte update and return.
3. If equality failed, read Anchor A again for a signed ordering comparison.
   - If this observation is less than the selector, enter the upper group.
   - Otherwise, enter the secondary group.
4. On entry to the secondary group, read Anchor B and calculate a new displacement as selector minus that observation.
5. Read Anchor B again for equality with the selector.
   - If equal, select the payload-byte update and return.
6. If that equality failed, read Anchor B again for a signed ordering comparison.
   - If this observation is less than the selector, enter the secondary displacement group.
   - Otherwise, enter the final adjustment path.
7. Only on the final adjustment path, read Adjustment C and add it to the selector.

An equality match therefore uses its equality-test observation, not necessarily the value previously used for subtraction.

Likewise, entry to the upper group does not establish that its displacement was computed from the same Anchor-A value that passed the ordering test. The equivalent distinction applies to Anchor B.

When an ordering test fails, its current anchor observation may equal the selector even though the preceding equality test failed against a different observation. Without stability, the secondary group cannot unconditionally be described as “selector below Anchor A,” nor the final path as “selector below both anchors.”

Under stable anchor values, those descriptions are valid shorthand: equality is checked before ordering, so a failed equality followed by a failed greater-than test establishes strict less-than. All displacement descriptions below refer to the values calculated at the specified subtraction reads.

### Anchor-A equality action

When the selector equals the Anchor-A observation used for equality:

- Write the byte value `4` at context offset `0x42`.
- Return zero.

No payload access or dependency call occurs on this path. The preceding Anchor-A subtraction and separate equality read still occur.

### Upper group

Entry requires failure of Anchor-A equality followed by a true signed comparison of selector greater than the ordering observation of Anchor A.

This group uses the displacement already calculated from the **first** Anchor-A observation. It does not recalculate that displacement using the equality or ordering observations.

The function first recognizes displacement `0x610` and returns zero without direct argument-memory access.

Otherwise, it compares the displacement with `0x611` using signed ordering.

For a displacement less than `0x611`:

| Displacement | Behavior |
|---|---|
| `0x20C` | Return zero. |
| `0x408` | Read the first payload element and call F-724e1e872fe7 with it. A nonzero result produces outer result zero; a zero result produces fallback `0xFFFFFFCE`. |
| `0x40C` | Perform the indirect callback invocation described below. |
| `0x608` | Return zero. |
| Anything else | Return `0xFFFFFFCE`. |

For a displacement not less than `0x611`:

- Displacement `0x700` calls F-fd61d89535ad with the context address and a zero second argument, then returns zero.
- Otherwise, the representation calculates displacement minus `0x908` and checks whether the result is zero.
- If that result is nonzero and the displacement is not `0x910`, it calculates displacement minus `0xA00`.
- It returns zero if the displacement-minus-`0x908` result was zero, the displacement equals `0x910`, or the remaining calculated result equals `2`.
- Otherwise, it returns `0xFFFFFFCE`.

Away from unresolved arithmetic boundaries, these latter acknowledgments correspond to displacements `0x908`, `0x910`, and `0xA02`. They perform no direct argument-memory access and make no dependency calls.

#### Indirect callback invocation

At displacement `0x40C` in the upper group:

1. Read the first payload element as an object address.
2. Obtain the callable address from object offset `0x190`.
3. Obtain the second and third payload elements and the 32-bit field at object offset `0x194`.
4. Invoke the callable with these four positional arguments:
   - object address;
   - second payload element;
   - third payload element;
   - object field at offset `0x194`.
5. Return its 32-bit result unchanged.

The initial object-address read precedes the invocation’s use of that object. The exact relative evaluation order of the callable-field read and the remaining callback-input reads is not established.

The callable slot and the additional argument occupy distinct adjacent fields. No direct write outside callback effects is visible on this path.

### Secondary group

Entry requires failed Anchor-A equality and a false result from the subsequent signed comparison of selector greater than the separately observed Anchor A.

Before checking Anchor-B equality, the function calculates the secondary displacement using a fresh Anchor-B read. It then reads Anchor B separately for equality.

If that equality succeeds:

- Write byte value `1` at payload offset `4`.
- Return zero.

This writes one byte, not the 32-bit integer value `1`. The other three bytes of the second payload element are not directly written.

If equality fails, the function reads Anchor B again for signed ordering.

#### Secondary displacement group

When the selector is greater than the Anchor-B ordering observation, the function dispatches using the displacement calculated **before** Anchor-B equality:

| Displacement | Behavior |
|---|---|
| `0x105` | Read the first payload element, call F-921bd1d4bbbc with it, and return zero. |
| `500` decimal, or `0x1F4` | Return zero without direct argument-memory access. |
| `0x2FA` | Perform preparation and delegation. |
| `0x5F5` | Return zero without direct argument-memory access. |
| Anything else | Return `0xFFFFFFCE`. |

A displacement match does not establish the ordering observation’s value or equality observation’s value.

#### Preparation and delegation

At secondary displacement `0x2FA`:

1. Read the first payload element.
2. Call F-84f8a0152639 with that value and the address of a 516-byte automatic scratch region.
3. If the dependency returns nonzero, return zero without directly assigning the second payload element.
4. If it returns zero:
   - Read the first payload element again.
   - From the object identified by this later value, read the 32-bit field at offset `0x0C`.
   - Call F-fcc54fe9487b with the context address, that field value, and the same scratch-region address.
   - After the call returns, store its 32-bit result at payload offset `4`.
   - Return zero.

The scratch region is not visibly initialized before the first dependency receives it.

The second payload-object read is significant. F-84f8a0152639 may affect reachable memory or shared state; preservation of the first payload element or the originally referenced object is not established. The later object-field read uses the later payload value.

The delegated result is a payload output, not the enclosing function’s return value.

### Final adjustment path

This path is reached only after:

- Anchor-A equality failed;
- the Anchor-A ordering test was false;
- Anchor-B equality failed; and
- the Anchor-B ordering test was false.

Only then does the function read Adjustment C and calculate selector plus that value.

It returns zero if the adjusted 32-bit value equals `0`, `0x0B`, `0xFF`, or `0x208`. Otherwise, it returns `0xFFFFFFCE`.

There are no direct argument-memory accesses or dependency calls on this path.

With stable anchors, this is the “below both anchors” case. With changing observations, that description is not guaranteed.

## Side effects

### Direct memory writes

The only explicit branch-dependent writes to caller-accessible storage are:

- One byte, value `4`, at context offset `0x42`.
- One byte, value `1`, at payload offset `4`.
- One 32-bit dependency result at payload offset `4`, after successful preparation and return from F-fcc54fe9487b.

No direct dispatch-global write is visible.

These writes occur on alternative paths. Dependencies and the callback may perform additional writes not characterized here.

### Ordering, aliasing, and lifetime

The dispatch-global read sequence is specified above. It must not be replaced with a single source-level snapshot assumption.

Preparation completes before the later payload-object read. The object-field read supplies the subsequent dependency call; that call completes before its result is written to the payload.

No non-aliasing guarantee is supplied among context, payload, object storage, and dispatch storage. A direct write may therefore affect another view of the same bytes. Callees may introduce further alias-mediated changes.

There is no visible heap allocation, release, retention operation, or explicit ownership transfer by this function. Passing an address to a dependency does not establish whether that dependency preserves, retains, or releases the referenced object.

The scratch region belongs to the invocation and is unavailable after it returns. Dependency retention of its address is unknown; continued validity after return is not promised.

### Failure behavior

Returned dependency values are handled differently:

- F-724e1e872fe7: nonzero acknowledges; zero falls back.
- F-84f8a0152639: nonzero skips further processing but still returns outer zero.
- F-fcc54fe9487b: its result is stored without visible validation.
- F-fd61d89535ad and F-921bd1d4bbbc: returned values do not control the outer result.
- Indirect callback: its result becomes the outer result.

Invalid memory, invalid callback targets, non-returning callees, and failures outside these returned values have no visible recovery path.

## Dependencies

| Dependency | Established interaction | Unresolved behavior |
|---|---|---|
| F-724e1e872fe7 | Receives the first payload element; zero/nonzero controls fallback versus acknowledgment. | Meaning, object effects, ownership, shared-state effects. |
| F-fd61d89535ad | Receives context and zero. | Meaning of arguments and effects. |
| F-921bd1d4bbbc | Receives the first payload element. | Inspection, mutation, retention, or release of referenced storage. |
| F-84f8a0152639 | Receives the first payload element and scratch address; zero permits delegation. | Scratch format, initialized extent, memory and ownership effects. |
| F-fcc54fe9487b | Receives context, the later object field at `0x0C`, and scratch address; its result is stored. | Result meaning, scratch use, retention, shared-state effects. |
| Indirect callback, identity unresolved | Target comes from object offset `0x190`; receives four arguments and supplies the outer result. | Calling-convention details, ownership, memory and global effects. |

These are observed call relationships, not complete callee contracts.

## Acceptance examples

These are conditional deductions, not executed tests. Assume valid required storage, returning callees, and arithmetic away from unresolved overflow boundaries. Anchor stability is assumed only where stated.

1. **Equality after a different subtraction observation.** Suppose selector is `100`. The first Anchor-A observation is `90`, and its equality observation is `100`. The initial displacement is `10`, but equality selects the context-byte write. The function writes `4` at context offset `0x42` and returns zero without an Anchor-A ordering read.

2. **Upper displacement from an earlier observation.** Suppose selector is `10000`, the first Anchor-A observation is `8456`, the equality observation is `9000`, and the ordering observation is `9500`. The displacement is `1544`, or `0x608`; equality fails and ordering selects the upper group. The result is zero. The displacement is not recalculated as `500` from the ordering observation.

3. **A false ordering test need not establish strict less-than.** Suppose Anchor-A equality fails, but its later ordering observation equals the selector. The ordering test is false, so the secondary group is entered. The selector is not strictly below that ordering observation.

4. **Stable-anchor predicate distinction.** With stable anchors selecting upper displacement `0x408`, F-724e1e872fe7 returning `7` produces outer zero; returning zero produces `0xFFFFFFCE`.

5. **Callback forwarding.** On the callback path, a returned word `0x12345678` becomes the enclosing result unchanged.

6. **Secondary equality after a different displacement observation.** After secondary entry, suppose selector is `1000`, the subtraction observation of Anchor B is `900`, and the equality observation is `1000`. The function writes one byte at payload offset `4` and returns zero, despite the previously calculated displacement of `100`. No Anchor-B ordering read follows.

7. **Byte-width preservation.** If the payload-byte path is selected and bytes at offsets `4` through `7` initially contain `AA BB CC DD`, the direct write produces `01 BB CC DD`, independent of integer byte order.

8. **Preparation declines.** At secondary displacement `0x2FA`, nonzero preparation returns outer zero without the direct result assignment. Dependency-induced payload changes remain possible.

9. **Preparation changes the object.** If preparation returns zero after changing the first payload element to another valid object address, the later field read uses that new object. A subsequent result `0x89ABCDEF` is stored at payload offset `4`; the outer result is zero.

10. **Conditional adjustment read.** When both equality tests and both greater-than tests fail, Adjustment C is read. An adjusted result `0xFF` yields zero; `0x100` yields fallback. No Adjustment-C read is visible on earlier-returning paths.

## Uncertainties

- **documented_only — Arithmetic boundaries:** Operations use 32-bit integer representations and signed ordering. Exact machine behavior for overflowing additions or subtractions is not independently established. Offset descriptions must not silently assume unbounded arithmetic or guaranteed wrapping.
- **documented_only — External configuration:** Actual anchor values, adjustment value, initialization, and reachable selector cases are unknown.
- **documented_only — Concurrency and physical reads:** Source-visible sequencing and separate anchor observations are established. Atomicity, synchronization, hardware access realization, and behavior under concurrent mutation are not. The changing-observation examples describe selection conditional on those observations, not a concurrency-safety guarantee.
- **documented_only — ABI and byte order:** Four-byte words, four-byte payload spacing, and address-bearing values are supported by the representation. Complete ABI details and endianness are not independently established.
- **documented_only — Scratch validity:** Scratch format, required initialized extent, and dependency guarantees after a zero preparation result are unknown.
- **documented_only — Callee behavior:** No preservation of globals, objects, payload contents, ownership, or scratch pointers may be assumed.
- **Inferred intent:** The recognized selectors and delegated actions resemble an event-dispatch interface. No domain-specific event meanings are established.

## Evidence

The evidence is the supplied static function representation and opaque dependency mapping.

It establishes the preceding subtraction reads, separate equality and ordering reads, conditional Adjustment-C read, branch precedence, explicit access widths and offsets, call arguments, result handling, and preparation sequencing.

It does not establish dependency implementations, configured dispatch values, application object types, synchronization guarantees, or runtime reachability.

No runtime tests have been performed. Acceptance examples are conditional static deductions, not observed executions.
