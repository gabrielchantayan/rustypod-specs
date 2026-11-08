<!-- Trusted provenance: F-e9e4be5dad47; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e9e4be5dad47

## Purpose

F-e9e4be5dad47 selects a 32-bit integer result for a caller-supplied key and conditionally updates a cache in a caller-supplied state object.

The low bit of a control word selects between two paths:

- **Exact-key mode:** return an active entry with an equal key; otherwise call F-92c35313a71c and cache its result if nonzero.
- **Range mode:** return an active range entry containing the key; otherwise enumerate through F-e155b614573e and, if any candidate is accepted, cache the last accepted candidate.

“Cache,” “key,” “boundary,” and “candidate” describe inferred intent. The established contract consists of the accesses, comparisons, calls, writes, and return construction below. The application meaning of the selected integer is unknown.

The return is a packed 64-bit value. Its lower 32 bits contain the selected result or zero. Its upper 32 bits are path-dependent and are not a uniform status indicator.

## Interface

### Inputs

Three inputs are visible:

| Input | Representation | Established use |
|---|---|---|
| State address | 32-bit address | Base for state-field accesses |
| Request key | 32-bit pointer-typed value | Compared and passed as a value; never dereferenced here |
| Control word | Unsigned 32-bit integer | Bit 0 selects mode; the full word initializes range scratch storage |

A zero request key is normalized to the bit pattern `0x00000001`. The normalized value is used for comparisons, the exact-mode dependency argument, and an exact-mode cache replacement. Cache-hit returns instead place the **original**, unnormalized request in their upper half.

Control bit 0 clear selects exact-key mode. Bit 0 set selects range mode. Other control bits do not select a branch, but remain significant because the entire control word initializes the range boundary scratch word.

### State layout

Offsets are byte offsets from the supplied state address.

| Offset | Width | Role established by use |
|---|---:|---|
| `0x18` | 32 bits | Context argument read for each dependency invocation |
| `0x2c` | 8 bits | Active flag; any nonzero byte is active |
| `0x30` | 32 bits | Exact cached key or range lower boundary |
| `0x34` | 32 bits | Cached integer result |
| `0x38` | 16 bits | Stored selector-derived value |
| `0x3a` | 8 bits | Range-valid flag; any nonzero byte enables range reuse |
| `0x3c` | 32 bits | Range upper boundary |
| `0x40` | 32 bits | Integer result associated with the upper boundary |

This function writes, but does not read, the fields at `0x38` and `0x40`.

No null check, bounds check, alignment check, or state validation is visible. Memory required by the executed path must support its accesses. A state object supporting every path must extend through offset `0x43`. Platform-specific alignment and fault behavior are not established.

### Output

The packed return has these components:

| Path | Lower 32 bits | Upper 32 bits |
|---|---|---|
| Exact-key cache hit | Cached result | Original request |
| Range-cache hit | Cached result | Original request |
| Exact-key dependency path | Dependency result, including zero | Address of a local 32-bit selector word |
| Range-enumeration path | Last accepted result, or zero | Address of a local 32-bit boundary word |

The local addresses are not durable output objects. No valid post-return dereference, retained-storage guarantee, or ownership transfer is established for them. The upper half must not be interpreted uniformly as the request, a success flag, or a usable output pointer.

The lower half is a 32-bit integer bit pattern. A nonzero value may have its most significant bit set; it is not restricted to positive signed integers.

## Behavior

### Initialization and comparison rules

Before mode dispatch, the function:

- Normalizes the request.
- Initializes a local 32-bit selector to `1`.
- Reads the active byte at `0x2c`.
- Initializes enumeration bookkeeping with no accepted result.
- Initializes a local 32-bit boundary word from the full control word.

The active byte is captured once for the initial cache decision. The function does not reread it during that decision.

Exact-key matching uses equality of all 32 bits. Range comparisons interpret the normalized request and boundaries as signed 32-bit integers. Consequently, values with bit 31 set compare below nonnegative values.

Each dependency invocation evaluates two distinct argument sources: the 32-bit context field at `0x18`, and a shared global value expression. Both are evaluated anew for that invocation. The displayed source does **not** establish which of those two argument evaluations occurs first.

### Exact-key mode

If the captured active byte is zero, the function proceeds directly to the dependency path without reading the cached key.

If active, it reads the key at `0x30`. Equality with the normalized request produces a hit: the function reads `0x34`, returns that result with the original request in the upper half, and performs no explicit state writes.

Otherwise, it stages the address of the local selector word for the upper return half and calls F-92c35313a71c. The four visible argument positions are:

1. The context value read from state offset `0x18`.
2. Zero.
3. The value obtained by evaluating the shared global expression for this invocation.
4. The normalized request.

Argument position does not establish evaluation order between the context read and the global read.

A zero dependency result causes no explicit state writes. The function returns zero in the lower half and the staged local selector address in the upper half.

For a nonzero dependency result, writes occur in this order:

1. Write the result to `0x34`.
2. Write the normalized request to `0x30`.
3. Write the low 16 bits of the local selector to `0x38`.
4. Write byte `1` to `0x2c`.
5. Write byte `0` to `0x3a`.

The function then returns the dependency result in the lower half.

This path does not clear `0x3c` or `0x40`. Those fields retain whatever values they have after the dependency call, despite range reuse being disabled by the explicit write to `0x3a`.

The selector starts at `1`. No visible dependency argument supplies its address. Whether an undisclosed mechanism can change it is unresolved; if it remains unchanged, the stored selector is `1`.

### Range-cache reuse

Range mode reads the range-valid byte at `0x3a` only when the captured active byte is nonzero.

A hit requires all of the following:

- The captured active byte is nonzero.
- The range-valid byte is nonzero.
- The signed normalized request is at least the signed lower boundary at `0x30`.
- The signed normalized request is strictly below the signed upper boundary at `0x3c`.

The interval is lower-inclusive and upper-exclusive.

The conditional read order is significant:

- If either flag is absent, neither boundary is needed for the hit decision.
- If both flags are present, the lower boundary is read first.
- The upper boundary is read only if the lower-bound comparison passes.
- The cached result at `0x34` is read only after all hit conditions pass.

A hit returns the cached result in the lower half and the original request in the upper half, without explicit state writes.

Exact mode ignores the range-valid byte. It can therefore reuse a range entry when the entry’s lower boundary equals the normalized request. Range mode cannot reuse an entry whose range-valid byte is zero, even if its lower boundary equals the request.

### Range enumeration

If range reuse fails, the function repeatedly calls F-e155b614573e. At enumeration entry, the selector is `1`, the boundary scratch word contains the control word, and no result has been accepted.

For each invocation, the function stages the boundary scratch address as the upper return half. The four visible arguments are:

1. The newly read context at `0x18`.
2. Zero.
3. The newly evaluated shared global value expression.
4. The selector’s low 16 bits interpreted as a signed 16-bit integer and widened to signed 32 bits.

The shared global location is the same across invocations. Its **observed value is not established as constant**. A dependency may change that global, or other state affecting later calls. Neither the global value nor the context is snapshotted once for the enumeration.

After a call, enumeration stops if the result is zero. In that case, the boundary comparison is not evaluated.

For a nonzero result, the function compares the signed request with the signed boundary scratch value observed after the call:

- If the request is below the boundary, enumeration stops without accepting that result.
- Otherwise, the result and boundary become the most recently accepted pair, the selector advances, and enumeration continues.

Equality is accepted. There is no visible check that successive boundaries increase, no iteration bound, and no cancellation check. The algorithm retains the **last accepted pair in invocation order**, not necessarily the pair with the greatest boundary.

The boundary word is initialized from the control word, but no visible call argument supplies its address. Any mechanism by which the dependency changes that word remains unresolved. Conditional examples involving changing boundaries describe observed scratch values, not an established output-parameter interface.

### Enumeration completion and commit

If no result was accepted, the lower return half is zero and no explicit state writes occur.

If at least one result was accepted, the function writes:

1. Last accepted result to `0x34`.
2. Last accepted boundary to `0x30`.
3. Selector-derived 16-bit value to `0x38`.
4. Byte `1` to `0x2c`.
5. Byte `1` to `0x3a`.

It then writes the upper-bound fields according to the terminating call:

- **Zero terminating result:** write `0xffffffff` to `0x3c`, then zero to `0x40`.
- **Nonzero terminating result with boundary above the request:** write the terminating boundary to `0x3c`, then the terminating result to `0x40`.

The lower return half is the last accepted result, not the terminating result.

The sentinel `0xffffffff` is signed `−1` in later range comparisons. It is not positive infinity. In particular, an upper boundary of `−1` cannot satisfy the strict upper-bound check for a nonnegative normalized request.

### Selector arithmetic

The selector is a 32-bit unsigned scratch value. Its call argument uses only its signed low 16 bits.

On each acceptance, advancement masks the current selector to 16 bits and then adds one. The arithmetic result is therefore between `1` and `65536`, inclusive. With no external changes to the scratch word:

- The initial call receives `1`.
- Selector `32767` is passed as `32767`.
- Selector `32768` is passed as `−32768`.
- Selector `65535` is passed as `−1`.
- Selector `65536` is passed as `0`.
- Acceptance at selector `65536` advances it to `1`.

For a range commit, the terminating selector’s signed low-16-bit value is widened, one is subtracted, and the result is stored in 16 bits. The subtraction fits signed 32-bit arithmetic; the final store retains the low 16-bit pattern. This does not establish an unbounded or always-nonnegative ordinal.

## Side effects

The function’s explicit persistent writes are limited to the state fields described above. Cache hits do not invoke dependencies and make no explicit state writes. Failed dependency or enumeration paths do not explicitly clear or invalidate an existing entry.

“No explicit writes” does not mean the state is necessarily unchanged across a dependency call. Dependency effects on state, global storage, scratch storage, and other memory are unknown.

The shared global expression is an observable argument source on every dependency invocation. Its declaration and access width are not supplied. The source does not establish an ordering between that global read and the context read. Changes caused by an earlier dependency invocation may affect either argument on a later invocation.

Commit writes are sequential rather than atomic as a group. No synchronization or reentrancy protection is visible. Concurrent mutation may invalidate the comparisons or expose partially updated entries.

No allocation, release, reference-count adjustment, or request-key dereference is explicit. Dependency results are 32-bit integers; their semantic meaning, lifetime implications, and any ownership responsibilities remain unknown.

## Dependencies

### F-92c35313a71c

Used on an exact-key miss.

Established at this call site:

- Four visible arguments, as specified above.
- A 32-bit integer result.
- Zero prevents an explicit cache commit.
- Nonzero is cached and returned.
- The third argument comes from a global expression evaluated for that call, not an established constant.

Scratch-word mutation, memory effects, and any wider calling convention are **documented_only**.

### F-e155b614573e

Used repeatedly after a range-cache miss.

Established at this call site:

- Four visible arguments, including a signed-16-bit-derived selector.
- A 32-bit integer result.
- Zero terminates enumeration.
- Nonzero is accepted only if the post-call boundary is no greater than the request under signed comparison.
- Every invocation reevaluates both the context argument and the shared global argument.

Boundary production, monotonicity, eventual termination, and result ownership are **documented_only**.

### Shared global argument source

Both dependencies use the same global expression in the third argument position. This establishes a shared source location, not equal values across calls.

Its declaration, access width, contents, purpose, mutability constraints, and ownership significance are **documented_only**. Dependency effects must not be assumed to preserve it.

## Acceptance examples

These are conditional static expectations, not executed tests. Examples involving dependency effects assume the stated observations occur without other effects that alter the described outcome.

1. **Zero request on an exact hit.** With an active entry whose key is `1` and result is `R`, an exact-mode request of zero returns lower half `R`, upper half zero, and invokes neither dependency.

2. **Exact replacement.** If exact reuse fails and F-92c35313a71c returns nonzero `R`, the explicit stores are result, normalized key, selector low 16 bits, active byte `1`, and range-valid byte `0`, in that order. If the selector remains unchanged, its stored value is `1`.

3. **Exact failure preserves the entry explicitly.** If F-92c35313a71c returns zero, the lower half is zero and this function performs no cache writes. The upper half is a local selector address, not the request. Dependency-side mutations remain possible.

4. **Range endpoints.** With both flags set and signed boundaries `10` and `20`, requests `10` and `19` hit. Request `20` enters enumeration.

5. **Signed range membership.** With both flags set, lower boundary `0x80000000`, and upper boundary zero, request `0xffffffff` hits because signed `−1` lies in that interval.

6. **First boundary too high.** If the first enumeration call returns nonzero and the observed boundary exceeds the signed request, no candidate is accepted. The function returns lower-half zero without an explicit commit.

7. **Accepted then rejected candidate.** For request `15`, suppose one call returns `A ≠ 0` with observed boundary `10`, and the next returns `B ≠ 0` with boundary `20`. The committed result is `A`, lower boundary `10`, upper boundary `20`, and upper-bound result `B`.

8. **Zero termination after acceptance.** After an accepted candidate, a zero result commits that candidate and writes upper boundary `0xffffffff` and associated result zero. A later nonnegative request cannot hit that signed upper bound.

9. **Global value changes between calls.** Suppose an enumeration invocation evaluates the shared global as `G₁`, and its dependency call changes that global. A later invocation evaluates it again and may receive `G₂`. The same global location does not require `G₁ = G₂`. No relative ordering with either invocation’s context read is implied.

10. **Signed selector transition.** If enumeration reaches selector `32768`, the dependency receives `−32768`. At selector `65536`, it receives zero.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning and ownership obligations of the 32-bit results.
- The full dependency calling convention and any mechanism affecting local scratch words.
- Whether dependencies preserve state, globals, or other memory.
- The declaration and access width of the shared global expression.
- Boundary ordering and eventual enumeration termination.
- Whether callers consume the upper return half and why it carries local addresses.
- Platform-specific memory faults and dependency errors beyond the visible zero/nonzero distinction.
- Any external synchronization or reentrancy requirements.

If no hidden mechanism changes the boundary scratch word, its value remains the initial control word. Repeated nonzero results with that boundary not exceeding the request can therefore continue indefinitely. The supplied evidence does not justify replacing this uncertainty with an assumed boundary-output interface.

## Evidence

This contract uses only the supplied static function body and opaque dependency mapping.

Established evidence includes state offsets and access widths, short-circuit read conditions, signed range comparisons, per-invocation context and global argument evaluation, explicit store order, selector truncation and arithmetic, and packed return construction.

The supplied expression establishes a global read for each dependency invocation, but supplies neither its declaration nor its access width. Argument placement does not establish evaluation order relative to the context read.

No dependency bodies or runtime observations were supplied. No runtime tests have been performed. Inferred intent and unresolved behavior are identified separately above.
