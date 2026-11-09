<!-- Trusted provenance: F-6eacc8d7bd2e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6eacc8d7bd2e

## Purpose

F-6eacc8d7bd2e conditionally dispatches a counted sequence of 8-bit or 16-bit elements through callbacks configured in an object. Its input storage is identified either by a direct address or by an offset added to a dependency-supplied base.

The function checks the count and an object flag, resolves the input address, validates the element format, and invokes preparation. If preparation returns nonzero, it dispatches each element, evaluates an optional completion callback, and replaces two callback descriptors from globally referenced template storage.

Certain rejection paths attempt to record an error in the object's first 32-bit word. Other paths return without a direct write. Error recording preserves an existing nonzero error.

These are findings from the supplied static representation. The higher-level meanings of the object, elements, callbacks, and descriptor replacement are not established.

## Interface

### Inputs and result

The function accepts five inputs and has no explicit return value.

| Input | Representation | Role |
|---|---|---|
| Object | 32-bit address | Provides error storage, flags, address-resolution state, and callback descriptors. |
| Preparation argument | Opaque 32-bit value | Forwarded unchanged to F-f4bb5a4884b6. |
| Count | Signed 32-bit integer | Negative values select error handling; nonnegative values permit further checks. |
| Format | 32-bit value | `0x1401` selects 8-bit elements; `0x1403` selects 16-bit elements. |
| Input address or offset | 32-bit address-sized value | Used directly or added to a base, according to object state. |

The maximum nonnegative count is **2,147,483,647**. There is no explicit success result or processed-element count.

No object-validity check is shown. The object and other addresses must support the accesses actually reached. Some rejection paths do not access the object: a negative count with a captured error value of zero returns after the entry global reads.

### Relevant object layout

Offsets are byte offsets from the object address.

| Offset | Width | Observed role |
|---|---:|---|
| `0x000` | 32 bits | Sticky error word. |
| `0x14C` | 8 bits | Bit 0 enables further processing. |
| `0xA9C` | 32 bits | Per-element direct target or encoded table offset. |
| `0xAA0` | 32 bits | Per-element receiver adjustment and indirect-dispatch flag. |
| `0xAA4` | 32 bits | Completion direct target or encoded table offset. |
| `0xAA8` | 32 bits | Completion receiver adjustment and indirect-dispatch flag. |
| `0xAC8` | Embedded region; size unknown | Its address is passed to F-a01d4dfb5f88. |
| `0xADC` | 32 bits | Nonzero selects base-relative input addressing. |

The complete object size and layout are unknown.

### Global inputs

Two 32-bit global error values are read at entry, in this order:

1. The error used when the enable gate fails or the initial base read is zero.
2. The error used for a negative count.

Both reads precede count testing and object-flag access. Their numeric values are unknown, and either may be zero.

A separate global template reference is read three times during descriptor replacement. Those reads have distinct uses; the function does not establish one coherent template snapshot.

## Behavior

### Entry checks and sticky error recording

After reading the two global errors, the function tests the count.

A negative count selects the captured negative-count error. The enable byte is not read. Address resolution, format validation, preparation, callbacks, and descriptor replacement do not occur.

For a nonnegative count, the captured gate-failure error is selected and the byte at object offset `0x14C` is read. If bit 0 is clear, execution enters error recording. If it is set, address resolution follows.

Error recording has three cases:

- A selected error of zero causes return without accessing the object's error word.
- A nonzero selected error causes a 32-bit read at object offset `0x000`.
- The selected error is written there only if the value read was zero.

A preexisting nonzero error does not independently prevent processing. The error word is inspected only on a reached error-recording path.

### Input-address resolution

After the enable gate passes, the function reads the 32-bit word at object offset `0xADC`.

If that word is zero, the fifth input is the effective input address.

If it is nonzero, the function invokes F-a01d4dfb5f88 with the address of the embedded region at object offset `0xAC8`. It then reads a 32-bit value through the returned pointer.

- If this first value is zero, execution enters sticky error recording with the captured gate-failure error.
- Otherwise, the function reads another 32-bit value through the same returned pointer and adds this second value to the fifth input.

The first read supplies the zero check; the second supplies the addition operand. Their agreement is not guaranteed. A zero second value does not independently select the base-error path.

The dependency-returned pointer is dereferenced without a null check. A null or unreadable pointer is not converted into a handled error.

Address formation uses the represented 32-bit arithmetic without bounds or overflow checks. Behavior outside valid address-arithmetic ranges is not established.

The resulting effective address is checked for zero. A zero address causes a silent return before format validation and preparation. Consequently:

- A null direct input returns silently even if the format is unsupported.
- A zero offset can identify valid input in base-relative mode.
- A zero first base read selects error handling even when the count is zero.
- A nonzero first base read followed by a zero second read uses the fifth input as the effective address, subject to the subsequent zero-address check.

### Format validation and preparation

With a nonzero effective address:

- `0x1401` selects one-byte elements.
- `0x1403` selects two-byte elements.
- Every other format selects error `0x500` and enters sticky error recording.

For either supported format, F-f4bb5a4884b6 receives the object and the opaque preparation argument.

A zero preparation result causes immediate return. No element callback, completion callback, or descriptor replacement follows. Preparation's own effects are not rolled back.

Any nonzero preparation result permits processing, including a negative result.

Preparation runs even for count zero. The effective input address is already resolved, but input contents and callback descriptors are read afterward. The supplied representation does not establish that preparation preserves the object, input storage, globals, or memory lifetimes.

### Per-element processing

For positive counts, each iteration resolves a callback and receiver, reads one input element, and invokes that callback with the receiver and element value. After normal callback return, it decrements the remaining count and advances the input cursor.

| Format | Element read | Cursor advance |
|---|---:|---:|
| `0x1401` | 8 bits | 1 byte |
| `0x1403` | 16 bits | 2 bytes |

Element addresses increase by the applicable stride, assuming valid, nonoverflowing address arithmetic. The cursor is advanced after the last callback as well, although that final cursor value is not subsequently used for another element read.

No byte swapping is shown. The concrete byte order for halfword reads is not established. The narrow element representations do not establish signedness or exact extension to the callback's argument width.

Callback resolution precedes the input-element read in each iteration. Neither descriptors nor input contents are snapshotted for the sequence. Subject to valid storage and supported mutation semantics, a callback's changes to later input contents or object descriptors can affect later iterations.

### Per-element callback resolution

Each iteration reads the target descriptor at `0xA9C`, then the adjustment at `0xAA0`. The receiver address is the object address plus the signed 32-bit adjustment shifted right by one.

The adjustment field is read again to test its low bit. Receiver calculation and dispatch-form selection therefore use separate reads, not an atomic descriptor snapshot.

- **Low bit clear:** the descriptor is the direct callback target.
- **Low bit set:** the function reads a 32-bit table base from the adjusted receiver, then reads the callback target at that table base plus the descriptor with its low two bits cleared.

The low-two-bit mask applies to an indirect table offset, not to a direct target.

There is no per-element null-target guard. Direct dispatch requires a callable target. Indirect dispatch additionally requires a readable table-base word and table entry.

The adjustment is shifted as a signed 32-bit value. Exact handling of negative right shifts is target-dependent. Receiver and table-address calculations have no explicit bounds or overflow checks.

### Completion dispatch

After all element callbacks return normally—or immediately after successful preparation for count zero—the function reads the completion descriptor at `0xAA4`, then the adjustment at `0xAA8`.

Completion is attempted when either the descriptor is nonzero or the adjustment's low bit is set.

A zero descriptor with an even adjustment skips completion. A zero descriptor with an odd adjustment requests indirect dispatch at table offset zero.

For direct completion, the descriptor supplies the target. For indirect completion, the receiver address is calculated from the object address and the signed adjustment shifted right by one. A 32-bit word at that receiver supplies the table base; the target is read at the table base plus the descriptor with its low two bits cleared.

The supplied representation shows no explicit completion-call arguments. In particular, the receiver used for indirect lookup is not explicitly passed as an argument. No completion return value is examined.

### Descriptor replacement

After completion returns normally, or when completion is skipped, replacement proceeds in the following observable order:

1. Read and retain the global template reference.
2. Read the global template reference again, then read its 32-bit field at `0xAC`.
3. Read the global template reference a third time, then read its 32-bit field at `0xA8`.
4. Write the value obtained from `0xA8` to object offset `0xA9C`.
5. Write the value obtained from `0xAC` to object offset `0xAA0`.
6. Read the 32-bit field at `0x5C` through the initially retained reference.
7. Read the 32-bit field at `0x58` through that same retained reference.
8. Write the value obtained from `0x58` to object offset `0xAA4`.
9. Write the value obtained from `0x5C` to object offset `0xAA8`.

The per-element descriptor pair can obtain its two words from different template references. The completion pair uses the initially retained reference, but its fields are still read separately.

Replacement is not atomic. All three reference reads occur after completion handling, so preparation or callbacks may affect the values subsequently observed. No earlier-return path performs replacement.

## Side effects

Direct writes are limited to:

- The error word at `0x000`, when sticky error recording permits a write.
- The four descriptor words at `0xA9C`, `0xAA0`, `0xAA4`, and `0xAA8`, after successful preparation and normal completion handling.

Direct reads include global errors, object fields, dependency-returned base storage, input elements, indirect-dispatch tables, and template references and fields.

No direct allocation, deallocation, retention, or input-buffer write is shown. Dependencies and callbacks may perform such actions; their ownership behavior is unknown. Every address must remain valid when actually used.

For stable, contiguous input storage and nonoverflowing arithmetic, a positive count requires:

| Format | Required byte span | Maximum span |
|---|---:|---:|
| `0x1401` | Count bytes | 2,147,483,647 bytes |
| `0x1403` | Twice the count | 4,294,967,294 bytes |

The maximum halfword span is **two bytes smaller** than the 4,294,967,296-byte unsigned 32-bit address space. It does not exceed that address-space size.

Total span and address overflow are distinct. Even a span smaller than the address space can cross its upper boundary when placed at a nonzero starting address. The last accessed byte is at the starting address plus the span minus one; the final cursor advance forms the starting address plus the span. Neither calculation is protected by an explicit bounds or overflow check. A representable final accessed byte therefore does not necessarily imply a representable final cursor.

The function does not calculate or validate an aggregate input size before dispatch. Halfword and word alignment requirements are target-dependent. No synchronization, fault recovery, exception recovery, or rollback is shown. If a dependency or callback does not return normally, later operations are not established to occur.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-a01d4dfb5f88 | Called only in base-relative mode with the address at object offset `0xAC8`. Its returned pointer supplies the initial base check and, on the nonzero branch, a separate read for address addition. |
| F-f4bb5a4884b6 | Called with the object and preparation argument after address and format checks. Zero stops processing; nonzero permits it. |
| F-375c8199e7c2 | Associated with the shared error-recording destination. The supplied representation establishes shared control flow, not an independent call interface. |

Element and completion targets are selected at runtime; no stable opaque IDs are supplied for them.

Dependency internals, retention, release, global preservation, and failure contracts are unknown beyond these observed interactions.

## Acceptance examples

These are conditional expectations from static evidence, not runtime test results. Unless otherwise stated, accessed storage is valid, relevant values remain stable, arithmetic is valid, and callees return normally.

1. **Negative count and entry reads.**  
   The gate-failure error global is read before the negative-count error global. A negative count then selects the latter without reading the enable byte. If the selected error is zero, the object's error word is not accessed.

2. **Disabled object.**  
   A nonnegative count with bit 0 clear at `0x14C` selects the captured gate-failure error. Address resolution is not invoked. A preexisting nonzero error remains unchanged.

3. **Null direct input takes precedence over format rejection.**  
   With the gate enabled and direct addressing selected, a zero fifth input returns without selecting error `0x500`, even for an unsupported format.

4. **Zero initial base.**  
   A zero first read through the address-resolution result selects the captured gate-failure error. No second base read or preparation call follows.

5. **Different base reads.**  
   If the first base read is nonzero, the second supplies the addition operand. A zero second value does not itself select base-error handling.

6. **Unsupported format.**  
   With a nonzero effective address, an unsupported format selects `0x500`. A zero error word becomes `0x500`; an existing nonzero word is preserved.

7. **Zero count with successful preparation.**  
   Preparation runs for a supported format and nonzero effective address. No input element is read. Completion is evaluated, then descriptor replacement occurs.

8. **Preparation returns zero.**  
   No element callback, completion callback, or replacement follows. Preparation's own effects remain unrolled back.

9. **Element reads and ordering.**  
   Three byte elements are read at offsets zero, one, and two. Two halfword elements are read at offsets zero and two. Each read follows its iteration's callback resolution and precedes that callback invocation.

10. **Mutation affects a later dispatch.**  
    If the first callback changes the per-element descriptor, the next iteration reads that changed field rather than an entry-time snapshot.

11. **Zero indirect completion descriptor.**  
    A zero completion descriptor with an odd adjustment resolves table offset zero and attempts the resulting target.

12. **Distinct template references.**  
    If the three replacement reference reads yield different valid references, the first supplies fields `0x5C` and `0x58`, the second supplies `0xAC`, and the third supplies `0xA8`.

13. **Maximum halfword span.**  
    A count of 2,147,483,647 implies 4,294,967,294 input bytes, not more than the 32-bit address-space size. This arithmetic alone does not establish that such storage is available or that its placement is safe.

14. **Address boundary differs from span size.**  
    For that maximum halfword count, a start address of `0x00000002` places the final halfword at `0xFFFFFFFE`, covering through `0xFFFFFFFF`. The subsequent cursor advance reaches mathematical address `0x100000000`, outside the unsigned 32-bit range. No explicit check handles this boundary; overflow behavior remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Numeric values and meanings of the two global errors.
- Higher-level object type, element meaning, and preparation semantics.
- Stability of repeated base reads, adjustment reads, and template-reference reads.
- Ownership and lifetime of input, dependency-returned, and template storage.
- Whether dependencies or callbacks retain addresses, mutate globals, or release memory.
- Exact callback ABI, narrow-argument extension, and implicit completion-call state.
- Concrete byte order, alignment requirements, signed-shift semantics, and behavior outside valid address-arithmetic ranges.
- Concurrency guarantees and the validity of unsynchronized external mutation.
- Whether replacement resets handlers, restores them, or performs another lifecycle transition.

The descriptor pairs resemble receiver-adjusted dispatch information, and replacement resembles installing template handlers. These are inferred intentions, not additional guarantees.

## Evidence

This contract uses the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

The representation establishes the entry-read order, branch precedence, separate base reads, object offsets and access widths, element strides, callback-resolution forms, sticky error behavior, and replacement access order.

The maximum-span calculation follows from the signed 32-bit nonnegative count limit and two-byte halfword width: 2,147,483,647 multiplied by two is 4,294,967,294, which is less than the 4,294,967,296-byte address space. That calculation does not establish safe address placement or checked arithmetic.

Dependency internals, complete object layout, ownership guarantees, coherent snapshots, and higher-level API identity are not established.

No runtime tests have been performed.
