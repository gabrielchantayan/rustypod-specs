<!-- Trusted provenance: F-353caf38ed9a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-353caf38ed9a

## Purpose

F-353caf38ed9a derives two signed boundaries, compares them with resource-dependent thresholds, and conditionally issues operations using three supplied resources and an optional resource reference read from an object field.

**Established static behavior:** the function obtains four-word records, invokes opaque numeric dependencies, selects paths through signed comparisons, and calls operation dependencies. A first-resource operation receives exactly the same record pointer in two argument positions. Some paths perform another operation using a reference read from the object after an earlier operation returns.

**Inferred intent:** the records and operations resemble segmented rendering across an interval. Neither a graphics interpretation nor any particular visual result is established. “Start,” “end,” and “extent” below identify how fields participate in arithmetic and comparisons, not a complete coordinate system.

The reference obtained from the object is not established to be object-owned. Ownership of every resource remains unknown.

## Interface

The function accepts ten arguments and exposes no explicit return value or status.

| Position | Width | Established role |
|---|---:|---|
| 1 | 32 bits | Object reference passed to F-960363c07b32 and used for later conditional field reads. |
| 2 | 32 bits | Context forwarded to operation dependencies. |
| 3 | 32 bits | First resource reference or token. |
| 4 | 32 bits | Third resource reference or token. |
| 5 | 32 bits | Second resource reference or token. |
| 6 | Signed 32 bits | Adjustment used in integer arithmetic and numeric conversions. |
| 7–8 | Two 32-bit words | Ordered opaque operand for the first requested-boundary calculation. |
| 9–10 | Two 32-bit words | Ordered opaque operand for the second requested-boundary calculation. |

Resource names indicate their roles in this contract, not argument order.

### Record layout

Each record occupies 16 bytes and contains four consecutive 32-bit words.

| Byte offset | Use established here |
|---|---|
| +0 | Opaque word copied into destination records. |
| +4 | Signed start-like field. |
| +8 | Opaque word copied into destination records. |
| +12 | Signed end-like field. |

F-960363c07b32 receives the object and writable object-record storage. F-c24767c70ad0 receives writable record storage followed by a resource value.

The three initial resource queries reuse one scratch record. After each query returns, all four scratch words are copied into separate retained storage for that resource. These copies are distinct from the object record and destination record.

Numeric dependencies return opaque 64-bit values. When forwarded explicitly, each is split into its low 32-bit word followed by its high 32-bit word. This does not establish byte endianness or a floating-point encoding.

### Preconditions

Normal completion requires valid inputs and dependency behavior sufficient for all accesses on the selected path:

- The object must be acceptable to F-960363c07b32.
- All three supplied resources must be acceptable to F-c24767c70ad0, even if later operations do not use them.
- Query outputs must initialize the words subsequently consumed.
- Conditional object accesses require a readable byte at +0xE8 and, when enabled, a readable 32-bit field at +0x104.
- Numeric dependencies must accept the supplied arguments.
- Operation dependencies must accept the described records, including the exact pointer aliasing on first-resource operations.

The function performs no explicit null check, resource validation, allocation, release, or ownership-transfer protocol.

## Behavior

### Initial records and common numeric value

The first dependency call is F-960363c07b32, which receives the object and writable object-record storage.

After it returns, the function reads that storage’s +12 and +4 fields and forms the signed-width expression:

**object-record end − object-record start + adjustment × −2**

F-b3a54d881220 receives this expression. Its opaque 64-bit result is retained as the **common numeric value**.

The function then calls F-c24767c70ad0 in this order:

1. First resource, argument 3.
2. Second resource, argument 5.
3. Third resource, argument 4.

Each call uses the same scratch record. Immediately following each return, its four words are copied into the corresponding separate resource record.

All three queries occur before requested-boundary calculations and branch selection. An eventual first-resource-only path does not avoid these queries.

The object-record storage is not an immutable snapshot. Its address has escaped to F-960363c07b32. Subsequent dependencies might modify it through a retained pointer. Later reads of its +0 and +8 words therefore use the contents present at those later reads, not necessarily the initial query result.

### Requested boundaries and global operands

For the first requested boundary:

- F-c4b34006a13c receives the common numeric value as low/high words, followed by arguments 7 and 8.
- After that call returns, two 32-bit global words are read.
- F-1b22d7e17791 receives the preceding 64-bit result as low/high words, followed by those two global values in their respective operand positions.
- F-baa332e06e32 is called without explicit arguments.
- The adjustment is added to its signed 32-bit result.

The result is the requested **start**.

The same sequence then produces the requested **end**, using arguments 9 and 10. The two global words are read again after the second F-c4b34006a13c returns and before the second F-1b22d7e17791 call.

These are global reads, not embedded constants. Their values may differ between calculations because intervening dependencies may change them. The supplied representation establishes their operand positions but not a relative evaluation order between the two reads within one call’s argument preparation.

The explicit result of each boundary-calculation F-1b22d7e17791 call is not forwarded as an ordinary argument to F-baa332e06e32. Any connection through implicit machine state or another calling convention is unresolved.

After both boundaries have been calculated, the function derives:

- **First extent:** retained first-resource +12 minus +4.
- **Second extent:** retained second-resource +12 minus +4.

Both subtractions use signed 32-bit fields. No check enforces start ≤ end or positive extents.

### Leading operation

If start is less than first extent, the function prepares the destination record by copying the object record’s then-current +0 and +8 words and assigning start to +4.

If end is also less than first extent, destination +12 is set to end. F-945301cd184f receives:

1. First resource.
2. Context.
3. Destination-record pointer.
4. The identical destination-record pointer.
5. The 32-bit value 255.

The function then returns.

If end is at least first extent, destination +12 is instead set to first extent. The same operation, including exact pointer aliasing, occurs, but execution continues.

The first-resource record is not supplied as the source record in either case. Equality at start suppresses this leading operation; equality at end does not take the early return.

For continuing execution, **continuation start** is first extent when start is below first extent, and otherwise start.

### Second threshold

Every path not already returned performs the following numeric sequence:

1. F-b3a54d881220 converts the adjustment.
2. F-b3a54d881220 converts second extent.
3. F-1b22d7e17791 receives the converted adjustment followed by the common numeric value.
4. F-8a693c103d1c receives that result followed by the converted second extent.
5. F-1b22d7e17791 receives that result followed by the retained converted adjustment.
6. F-baa332e06e32 is called without explicit arguments.

Each explicitly forwarded 64-bit value is supplied as low/high words. The final signed 32-bit result is the **second threshold**.

The converted adjustment remains the operand used in both indicated positions despite intervening calls. The source does not establish the mathematical operators or rounding behavior of this sequence.

Before selecting the next operation path, destination +4 is assigned continuation start.

### End below second threshold

When end is less than second threshold, the function copies the object record’s then-current +0 and +8 words into the destination and assigns end to destination +12. It calls F-168345203dbe with:

1. Third resource.
2. Context.
3. Retained third-resource record pointer.
4. Destination-record pointer.
5. The 32-bit value 1.
6. The 32-bit value 0.

Only after this call returns does the function read the byte at object +0xE8.

If the byte is not exactly 1, the function returns. This includes both zero and other nonzero values.

If the byte is exactly 1, the function:

- Reads the destination record’s **then-current +12 word**.
- Subtracts two and writes the result to destination +4.
- Reads the 32-bit reference at object +0x104.
- Calls F-c24767c70ad0 with separate optional-resource record storage and that reference.
- Reads object +0x104 again.
- Calls F-945301cd184f with the second-read reference, context, optional-resource record pointer, destination-record pointer, and 255.

The subtraction uses the destination’s post-operation field, not necessarily the previously calculated end. F-168345203dbe may have changed destination +12.

No other destination word is restored on this path. Moreover, the subsequent query may modify previously escaped destination storage through unknown retained-pointer behavior. Consequently, even the newly assigned +4 value is not guaranteed to remain unchanged until the final operation.

The two object-field reads are distinct and separated by the query. The queried reference and final operation reference may differ.

### End at or above second threshold

When end is at least second threshold, the destination already has continuation start at +4. The function copies the object record’s then-current +0 and +8 words and sets destination +12 to second threshold.

F-168345203dbe receives the third resource, context, retained third-resource record, destination record, 1, and 0.

After that call returns, the function rebuilds the destination:

- +0 and +8 are copied again from the object-record storage as it exists at those later reads.
- +4 is assigned second threshold.
- +12 is assigned end.

It then calls F-945301cd184f with the second resource, context, retained second-resource record, rebuilt destination record, and 255.

These writes replace any destination changes made by the preceding operation. They do not guarantee restoration of the object record’s original values: that storage may itself have changed.

The object flag is not read on this path. Equality belongs here, so end equal to second threshold still produces both operations, with equal start and end in the final destination.

## Side effects

### Observable ordering

The following ordering is established:

- Object query precedes common-value conversion.
- Common-value conversion precedes all three resource queries.
- Each resource query precedes its four-word copy; resource query order is first, second, third.
- All initial queries precede requested-boundary calculations.
- Each boundary calculation reads the two globals after F-c4b34006a13c returns and before F-1b22d7e17791 is invoked.
- Any leading operation precedes second-threshold calculation.
- F-168345203dbe precedes the conditional object-flag read.
- On the optional path, the destination-end read and start write precede the first optional-reference read and query.
- The optional query separates the two reads of object +0x104.
- On the end-at-or-above-threshold path, rebuilding the destination occurs after F-168345203dbe and before the second-resource operation.

Destination preparation copies +0 before +8 in the supplied representation. The leading path then writes +4 and +12. The continuing paths assign +4 before their branch-specific copies and +12 assignment.

### Storage, aliasing, and ownership

The function directly writes temporary records, not object fields or resource contents. Dependencies may have additional effects.

First-resource operations receive exactly aliased source and destination pointers. Other operation calls receive distinct source-record and destination-record storage.

Object-record storage and shared query scratch storage have escaped to dependencies. Destination storage escapes to operation dependencies. Resource-record storage escapes when supplied to an operation. No dependency contract establishes whether these addresses are retained or later used for mutation.

Separate copied records prevent ordinary scratch reuse from directly overwriting an earlier resource record; this is not a general guarantee against dependency side effects.

No resource ownership, release responsibility, pointer-retention permission, or lifetime extension is established. Temporary storage is available during the invocation; safe use of retained pointers beyond that lifetime is not demonstrated.

### Errors and arithmetic

There is no explicit dependency-status check or error return. Faults, exceptions, nonreturning calls, and dependency-specific signaling remain unspecified.

The function does not reject negative adjustments, negative extents, zero-length intervals, reversed boundaries, or nonpositive common numeric inputs.

Integer expressions use 32-bit signed quantities where described. Exact overflow behavior is not established by the supplied representation. Examples requiring ordinary mathematical results therefore assume that every intermediate operation—including adjustment multiplied by −2, extent subtraction, boundary addition, and destination-end minus two—is representable.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-960363c07b32 | Receives object and writable object-record storage. |
| F-c24767c70ad0 | Receives writable record storage and a resource value; called three times initially and optionally once later. |
| F-b3a54d881220 | Receives a signed 32-bit expression and returns an opaque 64-bit value. |
| F-c4b34006a13c | Receives two ordered word pairs and returns an opaque 64-bit value. |
| F-1b22d7e17791 | Receives two ordered word pairs; some results are explicitly forwarded, while others precede argumentless extraction calls. |
| F-baa332e06e32 | Receives no explicit arguments; supplies a signed 32-bit result. |
| F-8a693c103d1c | Receives two opaque 64-bit operands as word pairs and returns another opaque 64-bit value. |
| F-168345203dbe | Receives resource, context, source record, destination record, 1, and 0. |
| F-945301cd184f | Receives resource, context, source record, destination record, and 255. |

These interfaces do not establish numeric operators, rendering modes, ownership, mutation limits, or failure behavior.

## Acceptance examples

These are conditional static examples, not runtime observations. Unless a case explicitly allows mutation, assume dependencies return normally, initialize required outputs, do not mutate previously supplied storage, and keep arithmetic representable.

1. **Entire interval below first extent.** With first extent 20, start 3, and end 12, exactly one operation occurs: F-945301cd184f on the first resource with destination boundaries 3 and 12 and identical source/destination pointers. All initial queries and both boundary calculations have already occurred. Second-threshold calculation does not occur.

2. **End equals first extent.** With first extent 20, start 3, and end 20, the leading operation uses 3 and 20, then execution continues. Equality does not trigger the early return.

3. **Start equals first extent.** With first extent 20 and start 20, no leading operation occurs. Continuation start is 20.

4. **Below second threshold, flag disabled.** With continuation start 20, end 40, and second threshold 60, F-168345203dbe receives boundaries 20 and 40. A subsequently read flag of 0 or 2 suppresses the optional operation.

5. **Optional operation without mutation.** Under the same boundaries with flag 1, the final destination boundaries are 38 and 40.

6. **Destination end changed by operation.** If F-168345203dbe changes destination +12 to 73 and the later query leaves the destination untouched, flag 1 produces final boundaries 71 and 73—not 38 and 40.

7. **Later query changes escaped destination.** If the optional query changes destination +4 after the function assigned it, that changed value reaches the final operation unless another dependency effect intervenes. The function does not reassign destination +4 after the query.

8. **Optional reference changes during query.** If object +0x104 contains reference A before the query and reference B afterward, the query receives A while the final operation receives B. No ownership relationship follows.

9. **End equals second threshold.** With continuation start 20 and end and threshold both 60, F-168345203dbe receives 20 and 60, followed by a second-resource operation with 60 and 60. No flag read occurs.

10. **Object-record mutation before rebuilding.** If the middle operation changes the escaped object-record +0 and +8 words, the end-at-or-above-threshold rebuild copies those changed words. It does not recover the initial query values.

11. **Globals differ between boundary calculations.** If the globals contain one pair after the first F-c4b34006a13c and a different pair after the second, the respective F-1b22d7e17791 calls receive those different pairs. No equality between the pairs is required.

12. **Reversed interval.** With first extent 20, start 10, and end 5, the early operation receives boundaries 10 and 5 unchanged. The function neither swaps nor suppresses them.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the object, context, resources, and record coordinates.
- Values, representation, stability, and update mechanisms of the two global words.
- Relative evaluation order of those two global reads within each operand pair.
- Numeric encodings, operators, rounding, conversions, and exceptional-value handling.
- Implicit state connecting F-1b22d7e17791 to F-baa332e06e32.
- Exact signed-overflow behavior.
- Dependency failure, mutation, pointer-retention, ownership, and release contracts.
- Stability of escaped object-record and destination storage across intervening calls.
- Concurrent changes to globals, object fields, or escaped records.
- Safety or visual meaning of empty, reversed, negative, or out-of-range records.

The inferred rendering interpretation does not resolve any of these unknowns.

## Evidence

Established claims derive from static inspection of the supplied function representation: argument widths and forwarding, record accesses and copies, global reads, signed comparisons, arithmetic expressions, branch structure, dependency-call ordering, exact pointer aliasing, and conditional object-field accesses.

Dependency implementations were not supplied. The purpose interpretation is inferred; unresolved dependency semantics remain documented_only.

No runtime tests, execution traces, or visual checks have been performed. The acceptance examples specify conditional consequences of the static control flow, not observed executions.
