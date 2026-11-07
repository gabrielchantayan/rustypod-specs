<!-- Trusted provenance: F-f9f8cdbaee8c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f9f8cdbaee8c

## Purpose

F-f9f8cdbaee8c gates access to an object, conditionally initializes one field, and uses a selector to record a request bit and copy an indirectly referenced 32-bit value into selector-dependent object storage. A newly recorded selector 1 also triggers conditional dependency calls.

The return value is path-dependent. It may be zero, the gate result, an initialization-derived result, a computed mask bit, an indirectly fetched value, or the literal value 1. It is not a uniform success indicator.

“Object,” “root,” “table,” “request mask,” and “initialization” describe structural roles inferred from the accesses. Their application-level meanings are not established.

This contract distinguishes established static behavior from arithmetic and dependency behavior that remain **documented_only**. No runtime tests have been performed.

## Interface

### Inputs and output

The function receives:

- **Object:** a pointer to storage containing 32-bit fields and a byte-sized condition.
- **Selector:** a 32-bit unsigned value, also converted to a signed 32-bit value for a branch decision.

The result is 32 bits, interpreted as unsigned.

The representation uses 32-bit integers and pointer-bearing fields in a 32-bit address model. It does not establish a native layout for wider-pointer environments. Signed conversions, pointer conversions, and address validity require the represented target’s conventions; they are not a promise of unrestricted portability.

### Object layout

Offsets below are byte offsets from the object base.

| Offset | Width | Role |
|---|---:|---|
| `0x000` | 32 bits | Root pointer-bearing field |
| `0x004` | 32 bits | Receives a retained nonzero F-94e50f72111d result |
| `0x104` | 32 bits | Request mask |
| `0x108` | 32 bits | Initialization field |
| `0x10C + 4 × selector` | 32 bits | Destination for a newly fetched value |
| `0x15C` | 8 bits | Additional selector-1 condition |

The selector-dependent destination formula describes the nominal byte offset when the indexing and address calculations are valid. It does not establish a bounded array or permission to access every such offset.

Aliasing between roles is possible. Selector 20 places its 32-bit destination at `0x15C`, overlapping the byte later examined during selector-1 processing. Which part of that stored value occupies the byte depends on target byte order.

### Referenced storage

| Location | Width | Role |
|---|---:|---|
| Root + `0x008` | 32 bits | Table pointer-bearing field |
| Root + `0x020` | 8 bits | Second fallback-call argument |
| Table + `0x018 × selector + 0x014` | 32 bits | Pointer to the ordinary fetched value |
| Table + `0x01C` | 32 bits | Selector-1 condition; subsequently reloaded as a pointer on fallback |
| Fallback referenced structure + `0x004` | 32 bits | Pointer to the fallback value |
| Ordinary or fallback value pointer | 32 bits | Value read through that pointer |

A conditional path also evaluates an opaque global and passes its value to F-2022b0f73f3b. The global’s width, type, ownership, and meaning are not established. The argument is its evaluated value, not an established constant or an established address of the global itself.

### Preconditions

There is no explicit validation of object size, pointer validity, alignment, selector bounds, or referenced storage.

Normal completion requires every reached access to be valid when it occurs, including accesses after dependency calls. No dependency is assumed to preserve object fields, globals, referenced storage, or memory lifetimes.

A zero gate result avoids this function’s direct object accesses. It does not prove that a null object is generally supported or constrain the gate’s own effects.

Defined selector-mask behavior requires a valid shift count and a representable signed shift result. Counts 0–30 have the ordinary single-bit interpretation. Count 31 is unresolved, as are counts 32–255. Valid address calculations and storage are separate requirements even when the shift is defined.

## Behavior

### Gate and initial object reads

The first operation is a call to F-9b3ea8470521 without explicit arguments.

If it returns zero, this function returns zero immediately. It makes no direct object access and invokes no other dependency.

If it returns nonzero, the function reads:

1. The 32-bit initialization field at object offset `0x108`.
2. The 32-bit root field at object offset `0x000`.

Both reads occur before any initialization dependency call. The captured root is retained for the ordinary lookup.

### Conditional initialization and signed arithmetic

A nonzero initialization field skips initialization.

A zero field causes F-887ba2404a49 to be called with the gate result and the integer 91. Its represented result is 64 bits.

The initialization calculation converts that result to a signed 32-bit integer and adds one using signed arithmetic. The resulting bits are then held as an unsigned 32-bit value and written to object offset `0x108`.

This is **not established as universally wrapping unsigned addition**:

- If the converted signed value is between −2,147,483,648 and 2,147,483,646 inclusive, adding one is representable.
- If the converted value is 2,147,483,647, corresponding to low bits `0x7FFFFFFF` under the represented low-word interpretation, adding one overflows the signed range. The static representation does not establish a wrapping result, a subsequent store, or a normal return for that case.
- With low bits `0xFFFFFFFF` interpreted as −1, adding one produces zero without signed-addition overflow. Thus zero can be stored in the initialization field.

The representation uses the narrowed result; it does not subsequently use the high 32 bits to determine behavior. Exact portability of a conversion from an out-of-range 64-bit value to signed 32 bits is not independently established. Examples involving such bit patterns are conditional on the target’s narrowing interpretation.

When initialization validly produces zero, the function does not replace it with a nonzero sentinel. A later invocation can therefore initialize again.

The ordinary lookup uses the root captured before F-887ba2404a49. A dependency change to the object’s root does not redirect that lookup. Other changes to storage reachable through the captured root can still affect it.

### Selectors taking the signed-negative branch

When the selector’s signed interpretation is negative, request-mask processing and selector-dependent accesses are bypassed.

Under the represented 32-bit signed interpretation, this covers selectors with the high bit set. The function returns:

- The nonzero gate result when initialization was skipped.
- The initialization-derived 32-bit result when initialization completed with defined arithmetic.

The earlier initialization-field and root reads still occur. Consequently, this is not a uniform invalid-selector rejection. The result can be zero after initialization from a narrowed value of −1.

The mapping of unsigned high-bit values to negative signed values follows the represented target interpretation; unrestricted portability of that conversion is not established.

### Mask processing and its boundaries

For a selector taking the signed-nonnegative branch, the function reads the 32-bit request mask at object offset `0x104`.

The shift count is the selector’s low eight bits. The shifted operand is a signed integer value of one. Only afterward is the result held as an unsigned 32-bit bit value.

Accordingly:

| Low-eight-bit count | Established interpretation |
|---|---|
| 0–30 | Defined positive single-bit result |
| 31 | **documented_only:** the mathematical result is not representable as signed 32-bit |
| 32–255 | **documented_only:** count is outside the 32-bit operand width |

No five-bit count masking, processor-specific behavior, or unsigned-shift substitution is established.

The reduced count applies only to the shift. Table and destination calculations use the full selector. For example, selector 256 has shift count zero, but its table and destination addresses are not those for selector zero. A defined shift therefore does not establish an overall safe selector range.

### Already-recorded selector

The function intersects the mask with the computed bit value.

If the intersection is nonzero, it returns that bit value. It does not update the mask, read the selected table entry, read or rewrite the destination, or perform selector-1 special processing.

The gate, initial reads, and any initialization have already occurred. For selector 1, this path returns 2, not the previously stored value.

### Newly recorded selector

If the intersection is zero, the function writes the mask combined with the computed bit value to object offset `0x104`.

It then performs these ordered accesses:

1. Reads the table pointer from offset `0x008` of the root captured before initialization.
2. Reads the value pointer at table offset `0x018 × selector + 0x014`.
3. Reads the 32-bit value addressed by that pointer.
4. Writes the value to the selector-dependent object destination.

The table stride is 24 bytes; the destination stride is four bytes. The table arithmetic involving the unsigned selector is represented at 32-bit unsigned width, so arithmetic wrap can affect its numeric address. Such wrap does not establish a valid pointer or valid storage. Destination indexing and pointer arithmetic likewise carry no bounds guarantee.

The mask write precedes the indirect fetch and destination write. No rollback or recovery path is shown if a subsequent access cannot complete.

For a selector other than 1, the function returns the fetched 32-bit value.

### Newly recorded selector 1

The ordinary fetch and destination write at object offset `0x110` complete before special processing begins.

The function rereads the object’s root field, retaining this newly captured root for the special sequence. It then reads the table pointer from that root’s offset `0x008` and the 32-bit condition at table offset `0x01C`.

- A nonzero table condition causes return zero. The object byte at `0x15C` is not read.
- A zero table condition causes the byte at `0x15C` to be read.
- A nonzero byte causes return zero.
- If both conditions are zero, F-94e50f72111d is called with the captured root, zero, zero, and 2000, in that order.

The meaning and unit of 2000 are unknown.

### Nonzero special dependency result

If F-94e50f72111d returns a nonzero signed 32-bit result:

1. The opaque global’s value is evaluated.
2. F-2022b0f73f3b is called with the object and that value.
3. After the call returns, the retained F-94e50f72111d result is written to object offset `0x004`.
4. The function returns 1.

Any nonzero result takes this path, including a negative signed result. The stored field preserves its 32-bit representation.

The global is evaluated after F-94e50f72111d returns, so changes made by that dependency can affect the argument. The final object-field write occurs after F-2022b0f73f3b and overwrites any change that call made to offset `0x004`, provided the object remains valid.

### Zero special dependency result

If F-94e50f72111d returns zero, F-5c4fede89f8c is called.

Its first argument is a 32-bit value obtained by:

1. Reloading the table pointer through offset `0x008` of the root retained for special processing.
2. Reloading the pointer-bearing field at table offset `0x01C`.
3. Reading a pointer from offset `0x004` of that referenced structure.
4. Reading the 32-bit value addressed by the latter pointer.

Its second argument is the byte at offset `0x020` of the same retained root. The relative evaluation order of these two argument expressions is not established. Within the first expression, address dependencies require the nested reads in the stated order.

The object’s root is not reread after F-94e50f72111d. Changing that object field alone does not redirect fallback accesses; changing the table or pointers reached through the retained root can.

The earlier table condition was zero, but fallback reloads it and uses its new value as a pointer. The wrapper neither establishes nor checks how that chain becomes valid. Normal completion requires the later chain to be valid when accessed.

The fallback dependency’s result is ignored. The function returns zero.

## Side effects

Direct writes occur in this path-dependent order:

| Location | Condition |
|---|---|
| Object + `0x108` | Initialization is required and its calculation completes |
| Object + `0x104` | The tested request bit is absent |
| Selector-dependent destination | The indirect ordinary fetch completes |
| Object + `0x004` | The special dependency returns nonzero and the following dependency returns |

These are access roles, not necessarily disjoint locations. Selector-dependent storage can overlap other fields.

There is a conditional global read but no direct global write in the wrapper. Dependency effects remain unconstrained.

No explicit allocation, release, ownership transfer, locking, diagnostic reporting, or rollback is shown. Passing an object or global-derived argument establishes neither borrowing nor transfer semantics.

No atomicity, concurrency, or reentrancy guarantee is established. The request bit can become visible before the corresponding value is fetched or stored. Dependency effects can also invalidate postconditions about earlier writes.

## Dependencies

| Opaque ID | Invocation and result use |
|---|---|
| F-9b3ea8470521 | First call, no explicit arguments. Zero returns immediately; nonzero permits processing. |
| F-887ba2404a49 | Called when the initialization field is zero. Receives the gate result and 91. Its represented 64-bit result is narrowed to signed 32 bits before signed addition of one. |
| F-94e50f72111d | Called for newly recorded selector 1 when both special conditions are zero. Receives the retained special root, zero, zero, and 2000. |
| F-2022b0f73f3b | Called after a nonzero special result. Receives the object and the subsequently evaluated opaque global value. Its result is unused. |
| F-5c4fede89f8c | Called after a zero special result. Receives the indirectly resolved 32-bit value and root-relative byte. Its result is unused. |

These entries establish call conditions, arguments, and result use only. Callee ownership, preservation, lifetime, and error behavior remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless specified otherwise, reached arithmetic and accesses are valid, the gate succeeds, and dependencies do not modify state relevant to the example.

1. **Gate failure.** A zero F-9b3ea8470521 result produces zero without direct object access or any other dependency call.

2. **Ordinary initialization.** A zero initialization field and a dependency result narrowing to signed 40 produce a stored value of 41. With selector `0x80000000` under the represented signed interpretation, the function then returns 41 without reading the mask.

3. **Initialization to zero.** A dependency result narrowing to −1, represented by low bits `0xFFFFFFFF`, produces zero at `0x108`. A signed-negative selector then returns zero. This is defined signed addition of one to −1, not evidence of general wraparound.

4. **Initialization overflow boundary.** A result narrowing to `0x7FFFFFFF` requires adding one beyond the signed maximum. The contract does not require a store of `0x80000000`, a particular return, or normal completion.

5. **Already-recorded selector.** Selector 3 with bit 3 present returns 8, without reading the selected table entry or destination.

6. **Highest established shift count.** Selector 30 computes `0x40000000`. If that bit is present, it returns that value. Selector 31 has no corresponding guaranteed result: its signed shift boundary is **documented_only**.

7. **New ordinary selector.** Selector 2, mask `0x00000001`, and fetched value `0x12345678` produce a mask write of `0x00000005`, followed by a destination write at `0x114`, and return `0x12345678`.

8. **Blocking selector-1 condition.** A newly recorded selector 1 writes the mask and destination at `0x110`. A nonzero special table condition then causes return zero without reading byte `0x15C`. If instead the table condition is zero and that byte is nonzero, return is still zero without the special dependency call.

9. **Changed global and successful special operation.** Both conditions are zero. F-94e50f72111d changes the opaque global from A to B and returns `0x1234`. With no intervening change, F-2022b0f73f3b receives B. After that call, `0x1234` is stored at `0x004`, and the function returns 1.

10. **Fallback.** Both conditions are zero, and F-94e50f72111d returns zero after making the reloaded pointer chain valid. The fallback receives the resolved value and root byte. Its result does not change the final zero return.

11. **Repeated selector 1.** A later call returns 2 only if its gate succeeds, initialization arithmetic completes if needed, and bit 1 is still present when the mask is read. Gate failure instead returns zero; a cleared bit takes the newly recorded path.

## Uncertainties

The following remain **documented_only**:

- Shift count 31 because the shifted value is signed, and counts 32–255 because they exceed its width.
- Initialization behavior when the narrowed signed value is `0x7FFFFFFF`.
- Exact out-of-range signed-narrowing conventions and unsigned-to-signed selector conversion portability.
- Legal selector range, pointer conversions, address validity, alignment, faults, and destination-index limits.
- Byte order and special memory-access requirements.
- Domain meanings of fields, values, constants, and the global.
- The global’s width, type, ownership, and lifetime.
- The target-level convention represented by the 64-bit dependency result.
- Dependency mutation, allocation, retention, release, and error effects.
- Concurrency and reentrancy.
- Relative evaluation order of fallback arguments.
- How the fallback pointer chain becomes valid after its earlier zero condition.

Interpreting the mask as a cache marker or the special sequence as resource initialization is inferred intent, not an additional guarantee.

## Evidence

The supplied static representation establishes the branches, access widths and offsets, retained and reloaded roots, dependency arguments, direct-write order, conditional global evaluation, and branch-specific return values.

Its arithmetic distinguishes a signed shift operand from an unsigned result holder, and a signed initialization increment from an unsigned result holder. Those assignments do not establish defined wrapping behavior at the signed boundaries.

No implementation execution, runtime tests, or smoke checks have been performed. Acceptance examples state conditional consequences of static evidence, not observed results.
