<!-- Trusted provenance: F-eaccb645fcdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eaccb645fcdf

## Purpose

F-eaccb645fcdf conditionally updates two one-byte fields in a caller-supplied state object after consulting an associated object, an in/out object selection, and a predicate dependency.

The established behavior has two update paths:

- A selected object whose flag byte has bit `0x08` set causes the state bytes at offsets `8` and `9` to be written with `0` and `127`, respectively.
- Otherwise, a non-null byte pointer obtained through indirect dispatch supplies a nonzero unsigned byte used to conditionally lower the state byte at offset `8` and conditionally raise the state byte at offset `9`.

Interpreting the two state bytes as lower and upper bounds is consistent with the ordinary path, but is inferred intent rather than an established interface definition. In particular, the flagged path overwrites both bytes without preserving any prior ordering or range.

The function has no explicit result value. Its observable results are memory writes, dependency calls, and any effects of those calls.

## Interface

### Inputs

The function receives two arguments:

1. **State address.** An address represented by a 32-bit integer in the supplied representation. The function treats it as the base of a memory object.
2. **Initial selected-object address.** A pointer to an object whose address initializes a local in/out selection slot.

The following layout is established by direct accesses:

| Object | Byte offset | Accessed representation | Use |
|---|---:|---|---|
| State object | `4` | 32-bit pointer | Associated dispatch object |
| State object | `8` | Unsigned 8-bit value | Conditionally lowered, or overwritten with zero |
| State object | `9` | Unsigned 8-bit value | Conditionally raised, or overwritten with `127` |
| Associated object | `0` | Pointer to a dispatch table | Locates the first indirect call |
| Associated object | `4` | 32-bit integer | Forms the first indirect call’s scalar argument |
| Selected object | `0` | Pointer to a dispatch table | Locates the second indirect call when required |
| Selected object | `4` | Unsigned 8-bit value | Tests flag bit `0x08` |
| Either dispatch table | `32` | Callable address | Indirect-call target |

These are accessed fields, not complete object definitions. Required object sizes, alignment rules, and unused fields are not established.

### Output and return

There is no explicit return value. The function may return without writing either state byte. A return without direct state-byte writes does not imply that memory is unchanged: dependencies execute before several early exits.

### Preconditions

For a defined execution, every memory access and indirect call actually reached must be valid. This includes:

- Readability of the state pointer field.
- Readability of the associated object’s dispatch pointer and integer field.
- A valid first indirect-call target.
- After the predicate accepts the selection, readability of the selected object’s flag byte.
- On the ordinary path, a valid second indirect-call target and readability of any non-null byte pointer it returns.
- Writability of the state bytes whenever a corresponding update is reached.

The function performs no explicit checks for null state or associated-object addresses. It also does not explicitly validate the selected-object address before passing it to the predicate or, following acceptance, inspecting it.

A null initial selection is not rejected locally before the first dispatch. Whether it is admissible depends on that dispatch and the predicate.

## Behavior

### 1. Derive a dispatch argument from associated state

The function first reads the associated-object pointer at state offset `4`, then reads the 32-bit integer at associated-object offset `4`.

It forms a predecessor value by subtracting one from that integer. It then reloads the associated-object pointer from state offset `4`.

The scalar argument for the first indirect call is selected as follows:

- If the predecessor equals the 32-bit representation of `-1`, use `0`.
- Otherwise, read the integer at offset `4` of the reloaded associated object. If subtracting one from that integer equals the earlier predecessor, use `2147483647`, represented by `0x7fffffff`.
- Otherwise, retain the integer obtained by the initial nested read.

The second comparison is not simply a comparison against a constant. It can depend on a later read of the associated pointer and integer field.

Under ordinary stable-memory assumptions and for subtraction values whose semantics are unambiguous:

- An initial integer of zero selects `0`.
- Any unchanged nonzero integer selects `2147483647`.
- The retain-original-value branch requires the later comparison to differ.

This stable-memory simplification does not replace the observed reads. The representation does not establish atomicity, synchronization, or permission for concurrent modification.

The accessed integer and selected scalar are 32 bits wide. The supplied representation does not establish whether predecessor arithmetic at the signed minimum follows defined wrapping machine arithmetic or reflects an overflow-sensitive source-language operation. That boundary remains documented_only.

### 2. Dispatch with an in/out selection slot

The function initializes a local pointer-sized slot with the initial selected-object address.

It obtains an indirect-call target from byte offset `32` of the dispatch table belonging to the reloaded associated object. The displayed call supplies five arguments, in order:

1. The reloaded associated-object address.
2. The scalar selected above.
3. The address of the local selection slot.
4. A callable address read from the same dispatch-table slot.
5. The state address.

The third argument gives the callee the ability to replace the selected-object address for subsequent processing.

The displayed call-target lookup and callable-address argument use the same table offset. Their exact relative evaluation order is not established, and identity under intervening mutation must not be assumed.

The call’s return value is not used. The subsequent path uses the local selection slot rather than automatically reverting to the initial argument.

### 3. Apply the predicate dependency

After the first indirect call returns, F-eaccb645fcdf calls F-eda6f7b141ff with the selected-object address from the local slot.

- A result equal to zero causes immediate return.
- Any nonzero result allows processing to continue.

No direct state-byte update precedes this predicate decision.

The function itself does not establish what the predicate validates. A nonzero predicate result is not independent evidence that every later address is safe.

### 4. Inspect the selected-object flag

On predicate acceptance, the function reads the byte at selected-object offset `4` and tests only bit `0x08`.

Other bits in that byte do not affect this branch.

If bit `0x08` is set:

1. Write `0` to state offset `8`.
2. Write `127`, or `0x7f`, to state offset `9`.
3. Return.

No second indirect call is made on this path. The writes are unconditional once this branch is reached, even if the old bytes already match or the old byte at offset `9` exceeds `127`.

### 5. Obtain an ordinary-path byte pointer

If bit `0x08` is clear, the function obtains another indirect-call target from byte offset `32` of the selected object’s dispatch table and invokes it.

The supplied representation shows no explicit arguments for this call and captures its result as a 64-bit quantity:

- The low 32 bits are interpreted as a byte pointer.
- The high 32 bits are initially extracted into an unsigned value.

If the low-word pointer is non-null, the function reads its first byte and replaces that extracted unsigned value with the byte, zero-extended to an unsigned integer.

Consequently, the high return word cannot independently authorize an update:

- A null low-word pointer causes return regardless of the high word.
- A non-null low-word pointer makes the first pointed-to byte decisive.

No returned length or buffer extent is checked. Only the byte at the returned address is directly read.

### 6. Apply ordinary-path updates

If the returned pointer is null, return without a direct update to either state byte.

If its first byte is zero, likewise return without a direct update to either state byte.

For a nonzero first byte:

1. Read the unsigned byte at state offset `8`.
2. If the first returned byte is strictly smaller, write that byte to state offset `8`.
3. Read the first byte through the returned pointer again.
4. Read the unsigned byte at state offset `9`.
5. If the newly read returned byte is greater than the byte at state offset `9`, write it to state offset `9`; otherwise return without that write.

All ordinary-path value comparisons are unsigned byte comparisons. The nonzero values admitted by the initial check range from `1` through `255`; they are not restricted to `127`.

The second read through the returned pointer matters. The two updates need not use the same byte if the pointed-to storage changes, including through aliasing with a field written by this function. The initial nonzero test applies to the first read, not independently to the second read.

Under stable, noninterfering memory, the ordinary path has the familiar effect of lowering offset `8` toward the supplied nonzero byte and raising offset `9` toward it. The actual contract preserves the separate reads and conditional writes.

## Side effects

### Direct writes

The only established direct writes outside local storage are to state offsets `8` and `9`.

Their order is:

- Flagged path: offset `8`, then offset `9`.
- Ordinary path: an optional offset-`8` write, followed by an optional offset-`9` write.

There is no rollback if an execution fails after an earlier write. Atomicity of the pair is not established.

### Dependency effects

The first dispatch receives both the state address and the address of the local selection slot. It can therefore affect subsequent selection and potentially other state.

The predicate and second dispatch may also have side effects. Their implementations are unavailable; they must not be assumed to preserve global memory, object fields, dispatch tables, or the state bytes.

State-byte comparisons occur after the dependency calls. They therefore use the bytes present at those later reads, not necessarily the entry values.

### Ownership and lifetime

No direct allocation, deallocation, reference-count operation, or ownership-transfer operation is visible.

This establishes only the absence of such operations in the shown body. It does not establish that dependencies preserve ownership or refrain from releasing objects.

Safe execution requires all objects and returned byte storage to remain valid through their final actual accesses. The lifetime of the local selection slot ends when this invocation returns; the supplied representation does not establish whether the first callee retains its address.

### Error handling

There is no explicit error report, status code, recovery mechanism, or exception handling in the shown body.

Predicate rejection, a null returned byte pointer, and an initial zero byte are normal early-return conditions. Invalid memory, invalid call targets, and dependency failures are not converted into defined local error results.

## Dependencies

| Dependency | Established use | Unknown contract |
|---|---|---|
| F-eda6f7b141ff | Receives the selected-object address; zero stops processing, nonzero permits it | Validation meaning, effects, ownership behavior, failure modes |
| First indirect dispatch | Associated-object table offset `32`; receives five displayed arguments, including an in/out selection-slot address | Concrete function ID, exact ABI, selection rules, effects, ownership |
| Second indirect dispatch | Selected-object table offset `32`; reached only after predicate acceptance with flag bit `0x08` clear | Concrete function ID, exact ABI, result semantics beyond observed consumption, storage lifetime |

The two indirect targets are not resolved to opaque function IDs by the supplied dependency map. Equal table offsets do not establish equal target identity or equal contracts.

## Acceptance examples

These examples are conditional on valid addresses, returning dependencies, stable indicated values, and no unmentioned dependency effects.

1. **Stable zero associated integer.**  
   If the initial associated integer is `0`, the first dispatch receives scalar `0`. Its selection and subsequent predicate result still determine whether any byte update occurs.

2. **Stable nonzero associated integer.**  
   If both associated-integer reads yield `5`, the first dispatch receives scalar `2147483647`, not `4` or `5`.

3. **Different associated-integer observations.**  
   If the initial integer is `5` and the later comparison reads `6`, the first dispatch receives the original value `5`.

4. **Selection replacement followed by rejection.**  
   If the first dispatch replaces the local selection and F-eda6f7b141ff returns zero for that selection, the function performs no direct state-byte writes and does not execute the second dispatch.

5. **Flagged selection.**  
   If the predicate accepts and the selected flag byte has bit `0x08` set, entry state bytes `20` and `240` become `0` and `127`. The upper byte decreases; this path is not a monotonic maximum update.

6. **Null ordinary-path pointer.**  
   If the second dispatch returns a null low-word pointer with a nonzero high word, neither state byte is directly updated.

7. **Zero ordinary-path byte.**  
   If the returned pointer is valid and its first byte is zero, neither state byte is directly updated.

8. **Lower-only update.**  
   With state bytes `50` and `100`, and both returned-byte reads yielding `30`, the final bytes are `30` and `100`.

9. **Upper-only update above `127`.**  
   With state bytes `50` and `100`, and both returned-byte reads yielding `200`, the final bytes are `50` and `200`.

10. **Two writes and equality boundaries.**  
    With state bytes `100` and `20`, and both returned-byte reads yielding `60`, both bytes become `60`. If both state bytes already equal `60`, neither conditional write occurs.

11. **Distinct returned-byte observations.**  
    With state bytes `100` and `20`, a first returned-byte read of `60` and a second read of `10` produce final bytes `60` and `20`. The upper comparison uses the second observation.

## Uncertainties

The following remain documented_only:

- **Source-language overflow semantics:** behavior of the predecessor calculation at the signed 32-bit minimum.
- **Calling conventions:** whether the displayed argument lists and 64-bit return capture fully describe the machine ABI.
- **Object definitions:** semantic field names, complete sizes, alignment, invariants, and valid initial byte relationships.
- **Indirect target identity:** no concrete opaque IDs are supplied for either table dispatch.
- **Predicate meaning:** nonzero acceptance does not establish a documented validity or ownership guarantee.
- **Dependency effects:** mutation, allocation, release, retained pointers, and nonlocal exits are unknown.
- **Returned storage:** allocation extent, mutability, ownership, lifetime, and aliasing relationships are unknown.
- **Concurrency:** synchronization, atomicity, and legal concurrent mutation are not established.
- **Evaluation order:** the exact ordering among reads used to form the first indirect call is not established beyond the explicit earlier scalar preparation.
- **Range interpretation:** treating the state bytes as bounds, or `0` and `127` as a special complete range, is inferred intent only.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency mapping.

Established evidence includes the field offsets, access widths, branch conditions, displayed calls, explicit direct-write order, early returns, and repeated read of the ordinary-path byte.

The range-maintenance interpretation is an inference. Dependency internals, full object types, ownership guarantees, and ABI details are not established.

No runtime tests have been performed.
