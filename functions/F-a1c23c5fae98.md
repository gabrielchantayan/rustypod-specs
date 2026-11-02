<!-- Trusted provenance: F-a1c23c5fae98; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1c23c5fae98

## Purpose

F-a1c23c5fae98 conditionally replaces a two-byte field in a caller-supplied object, then invokes two dependencies in a fixed order.

An initial dependency always runs before the update decision. The update occurs only when:

- The initial dependency returns a nonzero result.
- The requested value, interpreted as an unsigned 32-bit integer, is less than three.
- The requested value differs from a comparison value obtained by numerically narrowing the current four-byte temporary to signed 16 bits, then applying the integer conversions required for comparison.

The comparison is not a read of a two-byte portion of the temporary. Under the represented target’s ordinary narrowing semantics, the low-order 16 bits of the temporary’s current value determine the signed comparison value, independently of byte order. The initial dependency may change that value before the conversion.

The object field’s meaning is not established. Describing this operation as a mode change or state transition is inferred intent, not a contractual requirement.

## Interface

### Arguments

| Position | Argument | Representation and use |
|---|---|---|
| 1 | Object address | A signed 32-bit value used as an address, forwarded to dependencies, and used to form addresses at byte offsets `0x184` and `0x48` |
| 2 | Requested value | Unsigned 32-bit integer; only zero, one, and two can reach the update path |
| 3 | Auxiliary word | Opaque four-byte value passed twice to the initial dependency |
| 4 | Initial word | Opaque four-byte value used to initialize a temporary and also passed separately to the initial dependency |

No pointer, handle, flag, or domain-specific meaning is established for either opaque word.

### Result

The function has no return-value contract. It returns no success indicator and does not distinguish a skipped update from a completed update through its return interface.

Observable behavior consists of dependency invocations, their effects, and the function’s conditional object-field access.

### Relevant storage and values

| Item | Location or derivation | Contractual use |
|---|---|---|
| Temporary word | Four bytes of function-local storage | Initialized from the initial word; its address is passed to the initial dependency |
| Comparison value | Numeric conversion of the temporary’s current four-byte value | Narrowed to signed 16 bits, promoted to signed 32 bits, then converted as required for comparison with the unsigned request |
| Object field | Two bytes at object address plus `0x184` | Read as a raw two-byte representation and then overwritten on the update path |
| Derived address | Object address plus `0x48` | Passed to the final dependency; not directly dereferenced by this function |
| Shared datum | Shared storage referenced for the final call | Supplied as the final dependency’s second argument after the preceding dependency returns |

The temporary and object field are distinct storage. The guard does not compare the requested value against the object field.

The comparison value has no independently addressed two-byte location in the contract. In particular, it must not be described as the first two bytes of the temporary or as a byte-order-dependent choice between its halves.

### Preconditions

For execution along a particular path:

- Each invoked dependency must accept its supplied arguments.
- The object field must be readable and writable for two bytes if the update path is reached.
- Address arithmetic and resulting addresses must be valid under the represented execution model.
- Any storage accessed by a dependency must satisfy that dependency’s requirements.

The function performs no explicit validation of nullness, object size, lifetime, alignment, or address-arithmetic overflow. The signed 32-bit address representation does not establish a general guarantee of wrapping address arithmetic.

Skipping the direct field access does not make an invalid object address safe: the initial dependency always receives it.

## Behavior

### Temporary initialization and initial call

The function initializes a four-byte local temporary with the initial word. It then invokes F-569945c32ec5 with five arguments in this order:

1. Object address.
2. Address of the temporary.
3. Auxiliary word.
4. Initial word.
5. Auxiliary word again.

The fourth argument is the original initial word passed by value. It is not obtained by rereading the temporary after the dependency has modified it.

The dependency can access the temporary through its address. Its behavior may also affect the object, shared state, or other memory; those effects are not specified by the supplied function body.

No request-range check precedes this call.

### Ordered guard evaluation

If F-569945c32ec5 returns normally, its signed 32-bit result is checked for nonzero. Both positive and negative nonzero results satisfy this condition.

The remaining conditions are evaluated with short-circuit behavior:

1. A zero result ends the update decision.
2. Otherwise, the unsigned 32-bit requested value is checked against three.
3. Only if the request is less than three is the temporary’s current value converted and compared with the request.

Failure of any condition prevents this function’s object-field read, object-field write, and both post-write dependency calls.

Thus:

- An out-of-range request still causes the initial dependency call.
- A zero initial result prevents evaluation of the request-range condition.
- A zero initial result or an out-of-range request prevents evaluation of the temporary comparison.

These statements concern the function’s own accesses and calls. They do not constrain accesses already performed by the initial dependency.

### Numeric comparison semantics

The third condition operates on the current four-byte temporary value, including changes made by F-569945c32ec5.

The conversion sequence is:

- Narrow that value numerically to signed 16 bits.
- Promote the signed 16-bit result to signed 32 bits, preserving its signed value.
- Convert that signed 32-bit value to unsigned 32 bits for comparison with the unsigned requested value.

Under the represented target’s ordinary narrowing semantics, let the low-order 16-bit pattern have an unsigned value between zero and 65,535:

- Patterns from zero through 32,767 produce the corresponding nonnegative signed value.
- Patterns from 32,768 through 65,535 produce that value minus 65,536.

The low-order 16 bits determine this result regardless of whether the target stores multibyte values in little-endian or big-endian order. No selection of the first two addressed bytes is involved.

For a negative signed result, conversion to unsigned 32 bits adds 4,294,967,296. For example, signed negative one becomes unsigned 4,294,967,295. It therefore differs from every permitted request.

Since the range check has already restricted the request to zero, one, or two, the comparison blocks the update only when the narrowed signed value is exactly the requested nonnegative number.

Consequences include:

- A temporary value of 65,537 narrows to one and blocks request one.
- A temporary value of 65,535 narrows to negative one and does not block any permitted request.
- Differences solely in the temporary’s upper 16 bits do not change the comparison under these narrowing semantics.

These consequences apply to the temporary’s value after the initial dependency, not necessarily its initial contents.

### Saving and replacing the field

When all guard conditions pass, the function performs these operations in order:

1. Reads the existing two-byte representation at object address plus `0x184`.
2. Saves that representation for the final dependency call.
3. Writes the requested value into the same field using a signed 16-bit representation.

The accepted requests zero, one, and two are exactly representable as signed 16-bit values. Narrowing the request for this write does not change its numeric value.

The saved field representation is obtained after the initial dependency returns. If that dependency changed the field, the saved representation reflects the later read rather than the field’s entry value.

There is no equality check against this field. A write and both post-write calls can occur even when the field already contains the requested value.

### Post-write calls

After writing the field, the function invokes F-09b9dd966794 with the object address. Its return value is not used.

If that dependency returns normally, the function invokes F-ebd0ee53699f with:

1. Object address plus `0x48`.
2. The shared datum supplied at this call site.
3. Object address.
4. The saved two-byte field representation.
5. The original unsigned 32-bit requested value.

The shared datum is supplied after F-09b9dd966794 returns. It must not be treated as a value captured before that call or assumed unchanged by it. The supplied evidence does not establish the relative evaluation order of individual arguments within the final call.

The object field is not reread between the post-write calls. If F-09b9dd966794 changes it, the final call still receives the saved pre-write representation and original request. The object’s current contents may differ from both.

The saved representation is two bytes wide. Its extension or interpretation at the final call boundary is not established.

## Side effects

### Direct effects and ordering

The function directly establishes:

- Initialization of a four-byte temporary.
- One unconditional dependency invocation.
- Conditional evaluation of the temporary’s current four-byte value and its numeric narrowing.
- Conditional reading and writing of the two-byte object field.
- Two conditional post-write dependency invocations.
- Conditional access to the shared datum for the final call.

The temporary comparison is a value-conversion operation, not an independently established two-byte memory read. The supplied representation does not establish the machine instruction width used to obtain that value.

On the update path, the observable operation order is:

1. Temporary initialization.
2. F-569945c32ec5 invocation and normal return.
3. Guard evaluation, including numeric conversion when reached.
4. Existing field read.
5. Field write.
6. F-09b9dd966794 invocation and normal return.
7. Shared-datum argument supply and F-ebd0ee53699f invocation.

No atomicity, synchronization, or concurrent-visibility guarantee is established.

### Indirect effects, ownership, and failures

A rejected update is not necessarily side-effect-free because F-569945c32ec5 has already run.

A normally completed update path does not guarantee that the field still contains the requested value. Either post-write dependency may change accessible state.

No explicit allocation, deallocation, ownership transfer, reference-count operation, or rollback appears in this function. This does not establish that dependencies preserve or release any supplied resource.

The temporary has local lifetime. Its address is supplied to the initial dependency during that call; the function does not provide storage with a lifetime extending beyond its invocation. Whether the dependency retains that address is unknown, and later use after the storage ceases to be valid is not supported by this interface.

No dependency-specific error protocol is available. A zero initial result controls the guard but is not established as an error code. If a dependency does not return normally, subsequent operations in the stated sequence are not guaranteed. No visible recovery undoes an already completed field write.

## Dependencies

| Opaque ID | Invocation condition | Established interaction |
|---|---|---|
| F-569945c32ec5 | Always, after temporary initialization | Receives five arguments, including the temporary’s address; returns the signed 32-bit result used by the guard |
| F-09b9dd966794 | After a qualifying field write | Receives the object address; its result is unused |
| F-ebd0ee53699f | After normal return from F-09b9dd966794 | Receives the derived address, shared datum, object address, saved field representation, and unsigned request |

Their internal behavior, ownership rules, failure mechanisms, and global-state effects are unavailable. None is assumed to preserve shared data or object contents.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless otherwise stated, storage and arguments are valid and invoked dependencies return normally.

### Zero initial result

For request one and an initial dependency result of zero:

- The initial dependency runs.
- Neither the request-range condition nor temporary comparison is evaluated.
- This function does not access the object field or invoke either post-write dependency.

### Unsigned range boundary

For a nonzero initial result and request three:

- The initial dependency has already run.
- The unsigned range check fails.
- No temporary comparison, direct field access, or post-write call occurs.

The same applies to request 4,294,967,295. It is not treated as negative one for the range check.

### Numeric narrowing blocks an update

For a nonzero initial result, request one, and a current temporary value of 65,537:

- The low-order 16-bit pattern represents one.
- Signed narrowing and subsequent conversions produce comparison value one.
- The update is skipped regardless of the object field’s contents.

This result is independent of byte order.

### Negative narrowed value permits an update

For a nonzero initial result, request two, and a current temporary value of 65,535:

- Narrowing produces signed negative one.
- Promotion preserves negative one.
- Conversion for the unsigned comparison produces 4,294,967,295.
- The comparison differs from request two, so the field update and post-write calls proceed.

### Initial dependency changes the temporary

Suppose the initial word is zero, but F-569945c32ec5 changes the temporary to 65,538 and returns nonzero.

For request two, the update is skipped because the current temporary narrows to two. Comparing against the original initial word would be incorrect.

### Field already equals the request

For a nonzero initial result, request one, current temporary value zero, and object field already containing one:

- The guard passes.
- The field is read and written with one.
- Both post-write dependencies run.
- The saved representation supplied to the final dependency represents one.

### Intervening mutation

For an eligible request two, suppose the saved field representation represents seven. After the write, F-09b9dd966794 changes the field to zero and changes the shared datum.

The final dependency receives the saved representation of seven and request two. Its shared-datum argument is supplied after the intervening call. This function performs no corrective field write afterward.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the object, field, opaque words, and shared datum.
- Dependency effects, ownership, retention, failure behavior, and nonlocal exits.
- Shared-datum width, declared type, and interpretation.
- Extension of the saved two-byte representation at the final call boundary.
- Exact calling convention, alignment requirements, synchronization, and machine access widths.
- Behavior for invalid addresses or invalid dependency arguments.
- Portability of out-of-range signed narrowing to execution models other than the represented target’s ordinary semantics.

Byte order is not supplied, but it does not select which half participates in this numeric narrowing. The unsupported first-two-bytes interpretation is not retained as an uncertainty.

An inferred intent is that the initial dependency obtains a current value and the field update initiates a transition. Neither interpretation is required for conformance.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping.

It supports the four-argument interface, four-byte temporary initialization, unconditional initial five-argument call, ordered short-circuit guard, unsigned request bound, numeric narrowing to signed 16 bits, promotion to signed 32 bits, unsigned comparison conversion, two-byte field access, byte offsets, and post-write call order.

The comparison uses the temporary’s current four-byte value. The representation does not support describing it as a dereference of the temporary’s first two bytes or making its numeric result depend on byte order.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. Acceptance examples specify required behavior under their stated conditions; they do not report observed outcomes.
