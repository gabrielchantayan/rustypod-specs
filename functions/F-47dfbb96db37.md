<!-- Trusted provenance: F-47dfbb96db37; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-47dfbb96db37

## Purpose

F-47dfbb96db37 conditionally updates a caller-supplied mutable context, invokes dependencies in a defined sequence, and may replace the context’s first three 32-bit words when a computed unsigned size satisfies a range-and-growth condition.

Its normal result is a packed 64-bit value. The upper half is obtained from a local working word during return construction; the lower half is read from context offset 0x18 during return construction.

The established contract concerns memory accesses, arithmetic, call order, and result composition. Interpreting the bounded operations as text copies, or the conditional replacement as allocation, is inferred intent rather than established behavior.

## Interface

### Inputs

The function takes five arguments:

| Argument | Established role |
|---|---|
| Context address | Identifies mutable storage accessed at fixed byte offsets. Its initial first byte selects the active or inactive path. |
| Second argument | Opaque 32-bit value. Initializes the first local working word and supplies an argument to the first bounded operation. |
| Third argument | Opaque 32-bit value. Initializes the second local working word on the active path and supplies an argument to the second bounded operation. |
| Fourth argument | Opaque 32-bit value. Written to context offset 0x2D8 and used to initialize the third local working word on the active path. |
| Fifth argument | Opaque 32-bit value passed with the context address to F-9e81aa419c67. |

The semantic types of the four 32-bit arguments are not established. They must not be assumed to be addresses, handles, lengths, or flags merely from their uses here. The context address’s machine width is not established by the supplied body.

### Directly accessed context layout

All offsets are byte offsets from the context address.

| Offset | Width or address use | Established access |
|---|---|---|
| 0x00 | 1 byte | Read once for the entry gate. |
| 0x00 | 32 bits | Written from the first working word on the replacement path. |
| 0x04 | 32 bits | Written from the second working word, then read later for a dependency argument. |
| 0x08 | 32 bits | Written from the third working word, then read later for a dependency argument. |
| 0x0C | Derived address | Passed to F-ffcea234129d; not directly dereferenced by this function. |
| 0x14 | 32 bits | Read as an unsigned addition operand, then read separately as the unsigned comparison operand. |
| 0x18 | 32 bits | Read during return construction. |
| 0x1C | 32 bits | Read for the size addition, with a signed interpretation before unsigned arithmetic conversion. |
| 0x2C4 | Derived address | Passed to the first F-7cfd4da96bb8 invocation. |
| 0x2D8 | 32 bits | Written with the fourth argument. |
| 0x2E8 | Derived address | Passed to the second F-7cfd4da96bb8 invocation. |
| 0x317 | 1 byte | Written with the byte-sized result obtained from F-9e97fde8a8f2. |

The addresses at 0x0C, 0x2C4, and 0x2E8 designate positions within the context; they are not pointer values loaded from those positions.

Both bounded operations receive numeric argument 0x14, or 20. Neither the unit of that argument nor the actual access extent is established.

### Result

On normal return:

- Bits 63–32 contain the first local working word’s value as used during return construction.
- Bits 31–0 contain the context word read at offset 0x18 during return construction.

These are bit positions, not a byte serialization. Target byte order and the external return-value ABI are not independently established.

### Preconditions

Every normal invocation requires readable storage for the first context byte and the 32-bit word at offset 0x18.

The active path additionally requires valid storage for its direct reads and writes, and arguments satisfying the contracts of every dependency reached. The replacement path requires writable leading words and valid inputs for its additional dependencies.

There is no direct null check, context-size check, alignment repair, or explicit validation of the opaque arguments. Dependency-specific validity requirements remain documented_only.

## Behavior

### Entry gate

The first working word is initialized from the second argument before the context’s first byte is examined.

If that byte is zero:

- No dependency is called.
- No context write is directly performed.
- The context word at offset 0x18 is still read for the result.
- The upper result half is the second argument.

If the byte is nonzero, the active path executes. This is an entry-only decision. The function does not recheck the byte after dependencies or later writes may have changed it.

### Active-path preparation

The following sequence occurs:

1. Store the fourth argument as a 32-bit word at context offset 0x2D8.
2. Initialize the second and third working words from the third and fourth arguments, respectively.
3. Invoke F-7cfd4da96bb8 with the context address plus 0x2C4, numeric argument 0x14, and the original second argument.
4. Invoke F-7cfd4da96bb8 with the context address plus 0x2E8, numeric argument 0x14, and the original third argument.
5. Invoke F-9e81aa419c67 with the context address and fifth argument.
6. Invoke F-9e97fde8a8f2 without explicit arguments.
7. Store its byte-sized result at context offset 0x317.
8. Invoke F-e8a4cb4f2634 with the context address.
9. Invoke F-8adab8bbb840 with the context address.

No dependency result is explicitly tested for failure. The result of F-9e97fde8a8f2 is consumed by the byte store; results of the other preparation calls are not consumed.

The fields used by the subsequent size decision are read only after this sequence. Their entry values do not determine that decision if preceding dependencies change them.

### Size calculation and separate comparison read

After preparation, the function performs three relevant context reads:

- One 32-bit unsigned read at offset 0x14 for the addition.
- One 32-bit read at offset 0x1C for the addition.
- A separate, later 32-bit unsigned read at offset 0x14 for the strict-growth comparison.

The relative order of the two addition-operand reads is not established. Both supply the completed sum before the separate comparison read. No dependency call is shown between these reads.

The operand from 0x1C has a signed interpretation in the supplied body, but its addition to the unsigned 32-bit operand produces an unsigned 32-bit result. The sum wraps modulo 2³².

The replacement path is selected exactly when:

- The wrapped sum is strictly greater, under unsigned comparison, than the value obtained by the **second read of offset 0x14**; and
- The wrapped sum is below 0x3000, or its unsigned difference from 0x3000 is below 0xC10.

The range clause is equivalent to requiring the wrapped sum to be below 0x3C10. That equivalence does not require stable context contents.

Consequently, the general condition is: the wrapped sum exceeds the separately read comparison value and is at most 0x3C0F. It must not be reduced unconditionally to “the sum exceeds the base used in its addition.”

There is no established atomic snapshot covering these accesses. If the two reads of offset 0x14 observe different values, the later value controls the strict comparison. The source establishes the additional read, but supplies no concurrency or synchronization contract explaining when such differences can occur.

### Consequences when the two base reads agree

Only under the assumption that both reads of offset 0x14 return the same value may the condition be described as strict unsigned growth over the addition base, together with the upper bound.

Under that stability assumption:

- A zero additional operand fails strict growth.
- A positive addition that wraps fails strict growth.
- A sum of 0x3000 is accepted if it exceeds the base.
- A sum of 0x3C0F is accepted if it exceeds the base.
- A sum of 0x3C10 is rejected.
- Every additional operand whose signed sign bit is set fails the combined condition.

The last consequence follows from both possible unsigned addition outcomes. Without wrap, a sign-bit-set operand produces a sum of at least 0x80000000, outside the permitted range. With wrap, the sum is below the unchanged base, failing strict growth.

There is no explicit signed-negative rejection branch. Rejection of all such operands is a consequence of the arithmetic and the stable-base condition. If the comparison read changes, that conclusion need not hold.

### Conditional replacement

If the condition fails, the function proceeds directly to return construction without further dependency calls.

If it passes:

1. Invoke F-b602fe3da75b with the address of the first working word, the wrapped sum plus one, and an opaque value obtained from a fixed data reference.
2. Store the then-current first working word at context offset 0x00.
3. Store the then-current second working word at context offset 0x04.
4. Store the then-current third working word at context offset 0x08.
5. Invoke F-db4a6f92f3d8 without explicit arguments.
6. Read the words at context offsets 0x04 and 0x08, and pass their values with the context address plus 0x0C to F-ffcea234129d.
7. Invoke F-8adab8bbb840 with the context address again.

The accepted sum cannot be zero, because it must exceed an unsigned comparison value. Its permitted range is therefore 1 through 0x3C0F. The numeric argument formed by adding one is between 2 and 0x3C10 and cannot overflow 32 bits.

The three working words occupy consecutive four-byte local storage positions. Only the first word’s address is explicitly passed to F-b602fe3da75b. A three-word output convention is consistent with this arrangement, but its actual access extent is unknown. No success test precedes the three context stores.

F-db4a6f92f3d8 executes after those stores and before the field reads supplying F-ffcea234129d. Those arguments are therefore the field values read after that intervening call, not guaranteed copies of the second and third working words. The relative order of the two argument-field reads is not established.

### Return construction and working-storage exposure

Return construction follows all calls on the selected path.

On paths without replacement, the first working word retains the second argument under the direct local-state behavior shown; its address has not been explicitly passed to a dependency.

On the replacement path, F-b602fe3da75b receives that word’s address. Its value immediately after that call is **not necessarily its value during return construction**. The dependency may expose the address to later dependencies, and preservation across those calls is not established.

Accordingly, the upper result half uses the working word’s value during return construction, including any intervening mutation permitted by the unknown dependency behavior. It is not guaranteed to equal either the value immediately after F-b602fe3da75b or the value previously stored at context offset 0x00.

The lower half uses the read at offset 0x18 after the final executed dependency. No relative evaluation order between the upper-word access and that context read is established.

## Side effects

### Direct writes

Every active invocation directly performs:

- One 32-bit store at offset 0x2D8.
- One byte store at offset 0x317.

The replacement path additionally performs three 32-bit stores at offsets 0x00, 0x04, and 0x08, in that order.

The inactive path performs no direct writes. Later dependencies may overwrite directly stored fields, so these stores do not independently guarantee final context contents.

### Dependency effects and errors

Dependencies may affect context, globals, shared state, or other reachable storage. No-argument calls are not evidence of purity.

There is no explicit rollback, checked failure branch, or alternate error result. If a dependency fails to return normally, behavior beyond that point is not established by this body. Earlier writes and effects have already occurred; no compensating action is shown.

### Ownership and lifetime

No independently established ownership API appears in the supplied evidence. Allocation, release, retention, and ownership transfer remain unknown for every dependency.

F-b602fe3da75b receives an address into invocation-local storage. Whether it retains or publishes that address is unknown. Such exposure could permit mutation during later calls; valid use after the invocation ends is not established.

## Dependencies

| Opaque ID | Established invocation |
|---|---|
| F-7cfd4da96bb8 | Twice per active invocation, with distinct context-relative targets, numeric argument 0x14, and the original second or third argument. |
| F-9e81aa419c67 | Once per active invocation, with the context and fifth argument. |
| F-9e97fde8a8f2 | Once per active invocation, without explicit arguments; supplies the byte stored at 0x317. |
| F-e8a4cb4f2634 | Once per active invocation, with the context. |
| F-8adab8bbb840 | Once before the size decision; again after replacement processing if selected. |
| F-b602fe3da75b | Only on replacement; receives local storage, the sum plus one, and an opaque fixed-data value. |
| F-db4a6f92f3d8 | Only on replacement, after the leading-word stores and before the subsequent argument-field reads. |
| F-ffcea234129d | Only on replacement; receives context plus 0x0C and the later-read words at 0x04 and 0x08. |

No dependency implementation or error contract is supplied.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Active-path examples assume preparation returns normally and the stated operand values are observed afterward. Unless explicitly stated otherwise, both reads of offset 0x14 return the same value.

1. **Inactive path.** With a zero first byte, second argument 0x12345678, and return-time word 0x89ABCDEF at offset 0x18, the result is 0x1234567889ABCDEF. No dependency is called and no direct context write occurs.

2. **Zero increment with stable base.** With both base reads equal to 100 and additional operand zero, the sum is 100. Replacement is skipped. F-8adab8bbb840 is called once.

3. **Boundary at 0x3000.** With both base reads equal to 0x2FFF and additional operand 1, replacement executes. F-b602fe3da75b receives numeric argument 0x3001.

4. **Upper accepted boundary.** With both base reads equal to 0x3C0E and additional operand 1, the sum is 0x3C0F and the numeric argument is 0x3C10.

5. **First rejected upper boundary.** With both base reads equal to 0x3C0F and additional operand 1, the sum is 0x3C10. Replacement is skipped.

6. **Wrapped addition with stable base.** With both base reads equal to 0xFFFFFFFF and additional operand 2, the sum is 1. The range clause passes, but strict growth fails.

7. **Signed-negative operand with stable base.** With both base reads equal to zero and operand bits 0x80000000, the sum exceeds the permitted range. With both base reads equal to 0x80000001 and the same operand, the wrapped sum is 1 and fails strict growth. More generally, every sign-bit-set operand is rejected when the base reads agree.

8. **Different comparison read.** If the addition reads 0x80000001 at offset 0x14 and operand bits 0x80000000 at offset 0x1C, the wrapped sum is 1. If the later comparison read at 0x14 instead returns zero, replacement executes with numeric argument 2. This example requires different observed reads; it does not establish a supported mutation mechanism.

9. **Intervening context mutation.** If the replacement call leaves working words X, Y, and Z, those values are stored at 0x00, 0x04, and 0x08. If F-db4a6f92f3d8 then changes the latter fields to U and V, the following dependency receives U and V.

10. **Later working-word mutation.** If F-b602fe3da75b leaves the first working word as X, the store at 0x00 writes X. If a later dependency can reach that local storage and changes it to W before return construction, the upper result half is W, not X.

11. **Return-time context value.** If dependencies change offset 0x18, the lower result half reflects the value read during return construction, not its entry value.

## Uncertainties

The following remain documented_only:

- Context semantic type, complete extent, and alignment requirements.
- Meaning of opaque arguments and fields.
- Bounded-operation semantics and access extents.
- Type and meaning of the fixed-data argument.
- F-b602fe3da75b’s actual output footprint and address-retention behavior.
- Global-state preservation, resource ownership, and release behavior of dependencies.
- Dependency error signaling and non-returning behavior.
- Concurrency, synchronization, and guarantees about stability between repeated reads.
- Operand evaluation orders not explicitly sequenced.
- Target byte order and concrete return ABI.

A context refresh followed by size-gated replacement of a three-word resource description is plausible inferred intent, not required semantics.

## Evidence

The supplied static body establishes the entry gate, argument widths, direct access offsets and widths, dependency sequence, wrapped unsigned arithmetic, separate reread of offset 0x14, ordered leading-word stores, and packed return construction.

The stable-base signed-negative rejection rule follows from that arithmetic. Possible later mutation through exposed local storage is an unresolved dependency behavior, not an observed event.

No dependency bodies, external object definitions, ownership contracts, or runtime traces were supplied. No runtime tests have been performed.
