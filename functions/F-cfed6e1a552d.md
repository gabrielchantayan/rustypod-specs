<!-- Trusted provenance: F-cfed6e1a552d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfed6e1a552d

## Purpose

F-cfed6e1a552d conditionally handles a record according to a 32-bit discriminator at byte offset +4. Two discriminator values are recognized:

- **15:** Resolve an associated object through F-682adfd5a561, compare a pointer-sized field in that object with an interior address of the receiver, and conditionally invoke F-0098a8c2a3f5.
- **19:** Invoke an indirect operation selected through the receiver’s dispatch table, passing the receiver and two 32-bit fields from the record.

The function returns **1** after either recognized path completes normally, including when the value-15 path does not invoke F-0098a8c2a3f5. It returns **0** for an unrecognized discriminator.

This is a behavioral contract derived from the supplied static representation. Describing the inputs as a “receiver” and a “record,” and interpreting the return value as a handled indicator, are inferred roles rather than established application-level semantics.

## Interface

### Inputs

The function takes two inputs:

| Input | Represented role | Required accessible content |
|---|---|---|
| Receiver | Pointer to storage represented as an array of 32-bit integers | Requirements depend on the selected path |
| Record | Address of discriminator-bearing storage | A readable 32-bit field at byte offset +4; additional fields for discriminator 19 |

The supplied representation uses 32-bit integer and four-byte data types. Pointer operations are consistent with a 32-bit address model. An implementation or analysis using different pointer widths must preserve the represented four-byte layout rather than silently substituting native-width fields.

No length, type tag beyond the discriminator, ownership flag, or validity indicator is supplied.

### Relevant layouts

The contract requires only the following portions of each object. Other storage is not characterized.

**Record**

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| +4 | 4 bytes | Discriminator, compared with 15 and then, if necessary, 19 |
| +8 | 4 bytes | First forwarded value on the value-19 path |
| +12 | 4 bytes | Second forwarded value on the value-19 path |

**Receiver**

| Location | Width or extent | Meaning in this function |
|---|---:|---|
| +0 | Pointer-sized under the represented 32-bit model | Address of a dispatch table, read only on the value-19 path |
| +68 | Address only | Interior address used for comparison on the value-15 path |

The address at receiver +68 is computed by advancing 17 elements of a represented 32-bit integer array. The function does **not** read the contents at that address.

**Associated object returned by F-682adfd5a561**

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| +8 | Pointer-sized under the represented 32-bit model | Pointer value compared with receiver +68 |

**Dispatch table**

| Byte offset | Width | Meaning in this function |
|---|---:|---|
| +92 | Function pointer under the represented model | Indirect operation used for discriminator 19 |

These offsets do not establish full structure sizes or the meanings of unrelated fields.

### Output

On normal completion, the function produces a four-byte result whose value is either:

- **1:** A recognized discriminator path completed.
- **0:** Neither discriminator comparison selected a recognized path.

There is no separate error result in the visible behavior. Results produced by invoked operations are not forwarded to the caller.

## Behavior

### Discriminator selection

The first observable decision reads the 32-bit field at record +4 and compares it with **15**.

If that comparison succeeds, the function follows the value-15 path. It does not subsequently test for 19.

If the first comparison fails, the represented behavior reads the discriminator field again and compares that second observation with **19**. If this comparison also fails, the function returns 0 without making a dependency call or reading the other described record fields.

For stable ordinary memory, this behaves as a selection among 15, 19, and all other values. The supplied representation nevertheless contains two discriminator accesses rather than an explicit single captured discriminator. It does not establish synchronization, volatile-memory semantics, or validity under concurrent modification.

### Discriminator 15: resolve, compare, and conditionally act

When the first discriminator comparison equals 15:

1. F-682adfd5a561 is invoked with the record address and the constant **3328**, equivalently hexadecimal **0xD00**.
2. After that call returns, its result is used as the address of an associated object.
3. A pointer value is read from associated-object offset +8.
4. That pointer is compared with the address receiver +68.
5. If the addresses are equal, F-0098a8c2a3f5 is invoked with the receiver.
6. If the addresses differ, that invocation is omitted.
7. After the selected work completes normally, the function returns 1.

The comparison is address equality. It does not compare the contents of the receiver’s interior storage with the contents of the associated object.

The associated-object field is read **after** F-682adfd5a561 returns. Its value must not be modeled as a pre-call snapshot. The dependency may affect memory or external state; no preservation guarantee is supplied.

Once this path has been selected, there is no later discriminator check. In particular, a change to the record during F-682adfd5a561 does not cause this invocation to switch to the value-19 path.

A nonmatching pointer does not make the operation unhandled: it still returns 1. Conversely, 1 does not prove that F-0098a8c2a3f5 ran.

### Discriminator 19: indirect dispatch

When the first comparison fails and the second discriminator observation equals 19:

- The function obtains a dispatch-table address from receiver offset +0.
- It obtains a callable target from dispatch-table byte offset +92.
- It reads the four-byte values at record offsets +8 and +12.
- It invokes the selected target with three arguments, in this argument-position order:
  1. The receiver.
  2. The value from record +8.
  3. The value from record +12.
- If that invocation returns normally, the function returns 1.

The payload fields are forwarded as four-byte values. Their interpretation as signed integers, unsigned integers, pointers, handles, flags, or another representation is not established.

The supplied representation establishes the dependency chain needed to find the call target: the table address must be obtained before its +92 entry can be read. It does not establish a total evaluation order among the independent payload reads and target lookup. Argument position must not be confused with memory-read order.

There is no explicit check that the receiver, dispatch table, or selected target is non-null or otherwise valid.

### Other discriminator values

For a stable discriminator other than 15 or 19, the function returns 0.

On that path, it does not:

- Resolve an associated object.
- Inspect the receiver’s dispatch table.
- Read the payload fields at record +8 and +12.
- Invoke either named dependency or the indirect target.

Consequently, readable discriminator storage is required even for an unrecognized record, but the visible path does not require readable receiver contents or payload fields.

## Side effects

The function contains no explicit store to caller-provided storage or global storage in the supplied representation.

Its visible reads are path-dependent:

- Every path reads record +4.
- A failed first discriminator comparison leads to another read of record +4.
- The value-15 path reads associated-object +8 after calling F-682adfd5a561.
- The value-19 path reads receiver +0, dispatch-table +92, record +8, and record +12.

Calls may have side effects beyond those reads. The supplied material does not establish whether any invoked operation modifies the receiver, record, associated object, global state, or external resources.

There is no explicit allocation or release operation in F-cfed6e1a552d itself. That observation does **not** establish that its dependencies avoid allocation, release, ownership transfer, or invalidation.

The address returned by F-682adfd5a561 is dereferenced without an intervening validity check. Its lifetime must permit that read. Similarly, the indirect-dispatch path requires its table and call target to remain usable through invocation.

## Dependencies

### F-682adfd5a561

- Called exactly once on a selected value-15 path.
- Receives the record address and the constant 3328.
- Its returned value is used as an address.
- The caller reads a pointer field at returned-address +8.
- Its failure signaling, ownership rules, memory effects, and result lifetime are **documented_only**.

No null-result branch, alternative result interpretation, or cleanup call is visible.

### F-0098a8c2a3f5

- Called only on the value-15 path when the associated-object pointer equals receiver +68.
- Receives the receiver as its sole visible argument.
- Its result, if any, does not influence the enclosing function’s normal return value.
- Its effects and ownership behavior are **documented_only**.

### Receiver-selected indirect operation

The target’s opaque function ID is not supplied. Its identity is determined at runtime through the dispatch-table entry at byte offset +92.

It receives the receiver followed by the two four-byte payload values. The enclosing function does not inspect its result. Target-specific behavior, calling-convention details, ownership effects, and failure behavior are **documented_only**.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes valid required accesses and normal return from invoked operations.

1. **Unrecognized stable discriminator.**  
   With discriminator 7, the result is 0. No dependency is invoked, and neither payload field nor receiver contents are read by the visible path.

2. **Recognized value 15 with matching association.**  
   F-682adfd5a561 receives the record and 3328. If its returned object contains exactly receiver +68 at offset +8, F-0098a8c2a3f5 receives the receiver once. The final result is 1.

3. **Recognized value 15 with a different association.**  
   If the returned object’s +8 pointer differs from receiver +68, F-0098a8c2a3f5 is not invoked. The result remains 1.

4. **Interior address, not receiver base.**  
   A stored pointer equal to the receiver’s base address does not satisfy the comparison merely because it identifies the same containing object. Equality must be with the +68 address under valid, nonwrapping address arithmetic.

5. **Null associated pointer field.**  
   A null pointer stored at associated-object +8 is compared as a pointer value; this function does not dereference that field’s value. A null address returned by F-682adfd5a561 is different: the function attempts an offset read through the returned address, with no defined recovery established here.

6. **Recognized value 19.**  
   The target selected at dispatch-table +92 receives the receiver, the exact four-byte value at record +8, and the exact four-byte value at record +12. On normal target return, the result is 1.

7. **Payload bit preservation.**  
   If record +8 contains the bit pattern 0xFFFFFFFF and record +12 contains 0x80000000, those four-byte patterns are forwarded in their respective argument positions. No visible validation, normalization, or arithmetic transformation applies to them.

8. **Record mutation during resolution.**  
   If F-682adfd5a561 changes the discriminator after the value-15 path has begun, the function continues with the association comparison. It does not perform indirect dispatch based on the changed discriminator.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of discriminator values 15 and 19.
- Meaning of the constant 3328.
- Full object layouts, allocation sizes, alignment requirements, and byte order.
- Dependency side effects, ownership contracts, and lifetime guarantees.
- Behavior following invalid addresses, invalid call targets, or inaccessible memory.
- Exception propagation, nonlocal control transfer, and dependency nontermination.
- Concurrent or externally changing memory semantics.
- The precise runtime ordering of independent reads used to prepare the indirect call.

The address additions +4, +8, +12, +68, and +92 require valid address formation in the represented environment. No bounds check or overflow handling is visible. A portable implementation must not invent wrapping-address semantics or wider payloads to fill these gaps.

## Evidence

**Established static evidence:** the supplied representation shows the two discriminator comparisons, dependency calls and arguments, pointer equality test, layout offsets, indirect target lookup, forwarded four-byte fields, and constant normal return values.

**Inferred intent:** the function resembles a record handler with a handled/unhandled result and a receiver-based dispatch mechanism. Those labels explain the observed shape but do not establish domain semantics.

**Unknown:** callee implementations, complete types, ownership rules, runtime memory validity, and external effects were not supplied.

No runtime tests have been performed. The acceptance examples describe consequences of the static evidence under their stated preconditions; they are not claims of observed execution.
