<!-- Trusted provenance: F-6d7cdf4ae0e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6d7cdf4ae0e7

## Purpose

F-6d7cdf4ae0e7 conditionally invokes a dependency after checking a descriptor and global values. If the dependency returns zero, the function transfers the requested number of bytes to a destination derived from the descriptor and a call-local offset. Each transferred byte is multiplied by −89, retaining the low eight bits.

The destination’s unsigned numerical ordering relative to the source selects transfer direction. For valid, ordinary, nonwrapping memory ranges, this direction supports overlapping transfers without overwriting source bytes before they are consumed.

After transfer, the function optionally publishes the then-current contents of the local offset object. This is not necessarily the value left by the dependency: the dependency receives that object’s address, and no visible restriction prevents the destination from overlapping it.

**Established static behavior:** staged descriptor gates, separately evaluated product-bit tests, dependency argument order, zero-only transfer, post-call destination calculation, directional byte transformation, and trailing optional offset publication.

**Inferred intent:** the dependency prepares storage or identifies an insertion offset. Allocation, capacity, ownership, and preservation guarantees are not established.

## Interface

### Positional inputs

| Position | Role | Representation |
|---|---|---|
| 1 | Opaque value forwarded to the dependency | 32-bit value |
| 2 | Requested byte count | Unsigned 32-bit integer |
| 3 | Opaque value forwarded to the dependency | 32-bit value |
| 4 | Descriptor address | Nullable pointer to signed 32-bit words |
| 5 | Opaque value forwarded to the dependency | 32-bit value |
| 6 | Opaque value forwarded to the dependency | 32-bit value |
| 7 | Source start address | Unsigned 32-bit address value |
| 8 | Optional offset-output address | Nullable pointer to a signed 32-bit integer |

The forwarded opaque values have no established interpretation within this function. The count is passed by value; changes to the function’s working count do not update a caller-owned count object.

### Descriptor layout

| Byte offset | Accessed width | Direct use |
|---|---|---|
| 0 | 32 bits, signed interpretation | Initial nonzero gate; reread after dependency success as the destination base |
| 8 | 32 bits, signed interpretation | Descriptor operand in two separate product-bit tests |

The word at byte offset 4 is not directly accessed. The complete descriptor layout and size are unknown.

The first word is used as a numerical destination base. It is not dereferenced as a pointer to another descriptor.

### Global inputs

Four global symbols supply values with these roles:

- **Initial status:** captured into the unsigned 32-bit return variable before descriptor validation.
- **First-check status:** captured into that return variable before the first product-bit test.
- **Second-check status:** captured into that return variable before the second product-bit test.
- **Multiplier:** used in each product evaluation.

The supplied representation contains no declarations for these globals. Their storage widths and declared types are therefore unresolved. In particular:

- A status captured into the unsigned 32-bit return variable is not evidence that the corresponding global itself has unsigned 32-bit storage.
- The multiplier’s signedness and width are not established.
- Although the descriptor operand is signed 32-bit, the multiplication’s effective type depends on the multiplier’s type and the applicable arithmetic conversions.

The status names identify roles, not numerical values or error categories.

### Local offset and outputs

A signed 32-bit local offset object is initialized to zero. Its address is passed to the dependency.

Two subsequent observations must remain distinct:

- **Destination offset:** read after dependency success to calculate the destination.
- **Publication offset:** read after transfer when the optional output pointer is nonnull.

Transfer writes can alter the local object through aliasing, so these observations need not agree.

The return value is unsigned 32-bit: a captured global status or the dependency result as captured into the return variable. Returning zero does not by itself establish that the dependency ran; a selected global status may convert to zero.

### Memory and arithmetic preconditions

Ordinary completion requires:

- Every reached descriptor read to address readable storage.
- The descriptor’s first word to remain readable after dependency success.
- Every reached source-byte read and destination-byte write to address suitable live storage.
- A nonnull output pointer to support the final 32-bit store.
- Reached arithmetic and address conversions to have defined semantics in the target environment.
- Dependency effects not to invalidate storage subsequently accessed.

No explicit capacity, pointer-validity, alignment, or overflow check is visible. A zero count removes byte-transfer accesses, but does not remove successful-path destination derivation, comparison, or optional publication.

## Behavior

### Initialization and descriptor gate

The local offset is initialized to zero. The initial-status value is then read and captured into the unsigned 32-bit return variable.

If the descriptor pointer is null, that captured value is returned. No descriptor access, dependency call, transfer, or direct output store follows.

Otherwise, the descriptor’s first word is read. If it is zero, the captured initial status is returned. The descriptor word at byte offset 8 is not read on this path.

### Two eligibility stages

With a nonnull descriptor and a nonzero first word:

1. The first-check status is captured into the return variable. A product involving the signed descriptor word at byte offset 8 and the multiplier global is evaluated. If the result of testing that product with the unsigned mask 1 is nonzero, the captured first-check status is returned.
2. If the first test passes, the second-check status is captured. A separate product involving the descriptor word and multiplier is evaluated. If testing that product with the unsigned mask 2 gives a nonzero result, the captured second-check status is returned.

Passing both stages reaches the dependency.

These are separate product evaluations, not two tests of one retained product. Within each product expression, the relative evaluation order of the descriptor operand and multiplier operand is not established. The first stage precedes the second.

The masks select product bits 0 and 1 respectively, subject to the expression’s arithmetic conversions. Their unsigned 32-bit representation does **not** establish that the preceding multiplication is unsigned or 32-bit.

If operands remain stable and both product evaluations have defined semantics, passing both tests means the tested product’s lowest two bits are zero. This conditional equivalence does not remove the separate reads and evaluations.

### Dependency invocation

F-5c8c9e1af152 is invoked once with these arguments, in order:

1. Descriptor pointer.
2. Input 1.
3. Input 5.
4. Address of the initialized local offset.
5. Input 6.
6. Requested byte count from input 2.
7. Input 3.

Neither the source address nor the optional output pointer is explicitly passed.

The dependency result is captured into the unsigned 32-bit return variable. **Only a captured result of zero enables destination calculation, transfer, and publication.**

Every nonzero captured result, including 1 and values above 1, skips those operations and is returned unchanged. Any dependency side effects remain; the caller performs no rollback.

### Destination calculation

After a zero dependency result, the function rereads the descriptor’s first word. It does not reuse the value from the initial gate and does not repeat the nonzero check.

It then reads the local offset and adds 20 bytes. The destination start is:

**post-call descriptor base + destination offset + 20 bytes**

The offset-plus-20 calculation and the addition of the base to that displacement are explicitly signed 32-bit operations in the supplied representation. Their defined range must not be replaced by an assumption of universal modular wrapping. This issue is distinct from the unresolved effective type of the eligibility products.

The calculated destination is compared with the source as an unsigned 32-bit address. The base and displacement used for transfer have already been captured; later writes to the descriptor or local offset do not redirect the loop.

No explicit negative-offset rejection occurs. Even with a zero count, the descriptor reread, offset read, destination arithmetic, and unsigned comparison are reached.

### Directional byte transfer

The unsigned comparison selects traversal order:

- Destination numerically below source: indices increase from zero.
- Destination equal to or above source: indices decrease from count minus one.
- Equal starts therefore select descending traversal.
- Zero count produces no byte iterations in either branch.

Each iteration reads its source byte before writing the corresponding destination byte. The stored byte is the source byte multiplied by −89, retaining the low eight bits. Equivalently, the transformation is multiplication by 167 modulo 256.

Whether the loaded byte is interpreted as signed or unsigned does not change those low eight result bits.

There is no complete source snapshot. Each iteration reads source storage at that time. Aliasing and external changes can therefore affect later reads.

For ordinary, valid, nonwrapping ranges, the direction protects overlapping source data from premature overwrite. Identical source and destination ranges are transformed in place in descending order. These properties do not establish behavior for wrapping addresses, invalid storage, or device-specific memory.

### Offset aliasing and publication

The dependency has the local offset’s address. No visible check prevents it from arranging a destination that overlaps that object.

If transfer writes change the offset object:

- The destination remains based on the values captured before transfer.
- Publication can observe the modified offset contents.

After all byte iterations, the function checks the optional output pointer. If null, it performs no publication store. Otherwise, it reads the local offset’s then-current signed 32-bit value and stores that value through the pointer.

Publication does not add the fixed 20-byte displacement. It does not use an independently preserved copy of the dependency-provided offset.

The output store may overlap transferred data, the descriptor, or other reachable storage. Its source value is obtained before the store. If it overlaps destination data, it can replace bytes just produced by the transfer.

The function then returns the captured zero dependency result.

## Side effects

### Direct accesses

The function directly performs:

- Initialization of the local offset.
- An unconditional initial-status read and capture.
- Conditional descriptor, status-global, and multiplier-global reads.
- A post-success descriptor-base reread and local-offset read.
- One source-byte read and one destination-byte write for each transfer iteration.
- A post-transfer local-offset read and 32-bit output store when publication is enabled.

The widths of the descriptor, offset, byte, and output accesses follow the supplied representation. The storage widths of global accesses cannot be established without their declarations.

No explicit global assignment, allocation, or release operation appears. Aliased destination or output stores can nevertheless modify globals or other objects.

### Dependency effects and ownership

F-5c8c9e1af152 receives the descriptor and a writable pointer to call-local storage. Its implementation is unavailable.

It may change the descriptor, offset object, globals, reachable storage, or ownership state. This function provides no preservation guarantee on its behalf. The absence of an explicit release in this function does not prove that buffers remain live after the dependency returns.

Whether the dependency retains the offset pointer is unknown. Passing that pointer does not establish a valid right to access the local object after this function returns.

### Errors and partial completion

No visible recovery handles invalid accesses, arithmetic faults, or dependency-induced invalidation.

Interruption during transfer may leave some destination bytes modified, including bytes of the local offset if aliased. The optional publication store occurs only after every byte iteration completes.

There is no transactional completion or rollback mechanism.

## Dependencies

### F-5c8c9e1af152

**Established relationship:** receives the seven arguments listed above. Its result is captured into an unsigned 32-bit variable.

**Established caller interpretation:**

- Captured zero enables destination calculation, transfer, and optional publication.
- Captured nonzero skips those direct operations and is returned.

**documented_only — unresolved contract:**

- Allocation, growth, validation, or release of storage.
- Capacity checks or interpretation of the requested count.
- Selection of the offset and whether it is written on every path.
- Modification of descriptor fields or globals.
- Creation of aliases involving the local offset.
- Pointer retention, lifetime guarantees, and ownership changes.
- Meaning of individual results beyond the caller’s zero/nonzero distinction.

No allocator or buffer-preparation convention is assumed.

## Acceptance examples

These are conditional static acceptance criteria, not runtime results. Except where discussing rejection, they assume readable and writable ordinary memory, defined arithmetic, and no interfering external mutation.

1. **Null descriptor:** the initial-status value is captured into unsigned 32-bit form and returned. No dependency call or direct output store occurs. The global’s storage width remains unspecified.

2. **Zero initial base:** a readable descriptor with a zero first word returns the captured initial status. No read at descriptor byte offset 8 is reached.

3. **First-stage rejection:** if the first product-bit test is nonzero, the captured first-check status is returned. The second stage and dependency are not reached.

4. **Second-stage rejection:** if the first test passes and the second product-bit test is nonzero, the captured second-check status is returned without invoking the dependency.

5. **Stable eligible product:** if the effective multiplication semantics are defined, both evaluations use stable operands, and the tested product has its lowest two bits clear, both eligibility tests pass. This criterion does not prescribe the multiplier’s signedness or width.

6. **Nonzero dependency result:** a captured result of 1 or 7 is returned without direct transfer or publication. Dependency effects are not undone.

7. **Changed destination base:** if the dependency succeeds, sets the descriptor base to B, and leaves the offset at 12, the destination starts at B + 32, provided the signed additions are defined. A nonnull output receives 12 only if the local offset remains 12 until its publication read.

8. **Transfer overwrites the offset:** if the dependency arranges destination overlap with the local offset, transfer continues using the previously calculated destination. If the writes leave a readable offset value Q, publication stores Q rather than necessarily publishing the dependency’s original value.

9. **Byte transformation:** source bytes 0, 1, 2, 128, and 255 produce bytes 0, 167, 78, 128, and 89 respectively, assuming those source values remain present until their reads.

10. **Higher overlapping destination:** a destination beginning inside the source range at a higher address selects descending traversal. For valid nonwrapping ranges, overlapping writes do not destroy source bytes before consumption.

11. **Equal starts:** source and destination at the same address select descending traversal and transform the range in place.

12. **Zero count after success:** no source or destination byte access occurs. Destination derivation and comparison still occur. A nonnull output receives the offset value present at publication.

13. **Output overlaps destination:** the publication store follows every transfer iteration and can overwrite transformed destination bytes.

14. **Zero-valued rejection status:** a rejected gate whose captured status is zero returns zero without calling the dependency or transferring bytes.

## Uncertainties

- **documented_only — global declarations:** no declarations establish the four globals’ storage widths or declared types. Status capture establishes unsigned 32-bit return values, not global storage representation.
- **documented_only — product arithmetic:** the descriptor operand is signed 32-bit, but the multiplier’s type is unknown. Effective multiplication width, signedness, conversions, and overflow behavior cannot be fixed from the masks alone.
- **documented_only — destination arithmetic:** the represented offset-plus-20 and base-plus-displacement calculations use signed 32-bit intermediates. Behavior outside their defined range requires target-level evidence; universal wrapping is not established.
- **documented_only — address model:** the source and comparison use unsigned 32-bit address values. Wider pointers, wrapping ranges, and unusual mappings are unresolved.
- **documented_only — offset representation:** exact results of partial offset overwrites depend on byte order and integer representation.
- **documented_only — concurrency:** no synchronization is visible. The source establishes separate evaluations, not a general concurrent-memory contract.
- **documented_only — dependency and descriptor contracts:** complete layout, capacity, lifetime, ownership, and preservation guarantees are unavailable.
- **Inferred intent:** the fixed 20-byte displacement may reserve a prefix before payload data. Its meaning is unknown.

## Evidence

The supplied static function representation establishes the conditional control flow, descriptor accesses, unsigned 32-bit status captures, separate product-bit tests, dependency argument order, zero-only transfer, signed destination intermediates, traversal direction, byte transformation, and trailing optional offset publication.

The distinction between destination offset and publication offset follows from the dependency receiving the local object’s address, destination calculation preceding transfer, and publication reading the object after transfer. No preserved publication copy or alias-exclusion check is visible.

Global declarations and the dependency implementation were not supplied. The prior draft and reviewer feedback are not independent evidence of behavior.

No runtime tests have been performed.
