<!-- Trusted provenance: F-84fc449f1feb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-84fc449f1feb

## Purpose

F-84fc449f1feb conditionally populates a caller-supplied byte-addressed output region using three opaque dependencies. It returns a 32-bit success indicator.

Two distinct behaviors are selected by a 32-bit mode input:

- Mode exactly **5** obtains and stores two bytes, with a validation call after each store.
- Every other mode obtains an initial control byte and, when that byte is nonzero, processes a sequence of tagged items. Selected tags cause 32-bit values to be stored at fixed output offsets. A zero control value ends processing successfully.

The established behavior is a sequence of dependency calls, output writes, and conditional checks. Interpreting the context as a stream, the initial control byte as a count, or the tags as a particular external format is not established. In particular, the initial control byte is **not decremented**.

## Interface

### Inputs

| Input | Established representation | Role |
|---|---|---|
| Context | Opaque 32-bit value | Passed unchanged as an argument to every dependency call. Its interpretation is unknown. |
| Output | Address of byte-addressable storage | Receives individual bytes and, on selected paths, 32-bit values. |
| Mode | 32-bit integer | Selects the two-byte path only when equal to 5. All other values select tagged processing. |
| Initial width/state | Opaque 32-bit value | Initializes a mutable 32-bit cell used by the variable-value dependency and subsequent validation calls. |

The function also creates a separate mutable 32-bit state cell initialized to zero. Every dependency receives access to this same cell. Its meaning and permitted values are unknown.

The initial width/state cell is initialized on both paths, but the mode-5 path never passes it to a dependency.

### Return value

The return value is a 32-bit word:

- **1:** the selected path reaches its successful termination condition.
- **0:** a validation dependency returns zero.

The function does not expose the last dependency status directly. Any nonzero validation result permits processing to continue; it need not equal 1.

A successful return does not establish that every output field was populated. A failed return does not roll back prior writes.

### Output layout

Offsets below are byte offsets from the supplied output address.

| Offset | Width | Direct use by this function |
|---|---:|---|
| 0 | 8 bits | First byte on the mode-5 path |
| 1 | 8 bits | Second byte on the mode-5 path |
| 2 | 8 bits | Initial control byte on every other mode |
| 3 | 8 bits | Most recently obtained tag |
| 4–7 | 32 bits | Value selected by tag 1 or 3 |
| 8–11 | 32 bits | Value selected by tag 2 |
| 12–15 | 32 bits | Value selected by tag 3 or 6 |
| 16–19 | 32 bits | Value selected by tag 4 |

These are direct stores to caller storage, not a fully initialized result record. Fields not written on the executed path retain their prior contents unless a dependency changes them through some other access path.

For ordinary memory-safe use, the caller must provide storage valid for every access reached. Two writable bytes cover the direct accesses of mode 5; a 20-byte region covers all direct accesses of the other path. Target-specific alignment requirements for the 32-bit stores must also be met. No explicit size, null, or alignment checks are established.

The byte order of the 32-bit fields is not specified by the supplied evidence.

## Behavior

### Shared state and dependency ordering

At entry, the function initializes:

- A shared 32-bit state cell to zero.
- A mutable 32-bit width/state cell to the fourth input.

The shared cell is not reset between calls or loop iterations. Likewise, the width/state cell retains changes made by the variable-value dependency.

Every acquisition is followed by a direct output store and then a validation call. Consequently, validation failure occurs **after** the corresponding output bytes have already been written.

No independent interpretation of acquisition results as errors is established. In particular, zero returned by an acquisition dependency is a value to store, not automatically a failure. The validation dependency controls the explicit failure paths.

### Mode exactly 5

The function performs the following ordered operations:

1. Calls F-d54421325734 with the context, the value 1, and access to the shared state cell.
2. Stores its 8-bit result at output offset 0.
3. Calls F-8a872239bf2c with the context, the value 1, and access to the shared state cell.
4. If that validation returns zero, returns 0 immediately.
5. Otherwise, calls F-d54421325734 again with the same argument forms.
6. Stores the second 8-bit result at output offset 1.
7. Calls F-8a872239bf2c again with the context, the value 1, and access to the shared state cell.
8. Returns 1 if the second validation is nonzero; otherwise returns 0.

There is no direct access to output offsets 2–19 on this path.

If the first validation fails, the second acquisition and second output store do not occur. If the second validation fails, both output bytes have already been stored.

### Every mode other than 5

The function first calls F-d54421325734 with the context, the value 1, and access to the shared state cell. It stores the resulting byte at offset 2, then calls F-8a872239bf2c with the context, the value 1, and access to the shared state cell.

If this validation returns zero, the function returns 0. The offset-2 write remains observable.

If validation succeeds, the function reads offset 2 into a local 8-bit loop-control value. This read occurs **after** validation, so the byte used for control is the value then present in caller storage, not necessarily the originally acquired byte.

A zero loop-control value causes an immediate return of 1. No tag is acquired in that case.

A nonzero loop-control value begins an iteration:

1. F-2b01710262ca is called with the context and access to both mutable 32-bit cells.
2. The low 8 bits of its 32-bit result are stored at offset 3.
3. F-8a872239bf2c is called with the context, the **current** width/state value, and access to the shared state cell.
4. A zero validation result ends processing with return value 0.
5. Otherwise, the function evaluates the tag-dependent field groups below, in their listed order.
6. After all applicable field groups complete successfully, it reads offset 3 into the local loop-control value.
7. If that captured byte is zero, it returns 1; otherwise it begins another iteration.

The initial control byte therefore controls only whether the first tag acquisition occurs. It does not bound the number of iterations, and the function does not decrement it or reread offset 2 for subsequent iterations.

### Tag-dependent field groups

The groups are evaluated in this order:

| Evaluation order | Current tag condition | Destination |
|---:|---|---|
| 1 | Tag equals 1 or 3 | Offset 4, 32 bits |
| 2 | Tag equals 2 | Offset 8, 32 bits |
| 3 | Tag equals 3 or 6 | Offset 12, 32 bits |
| 4 | Tag equals 4 | Offset 16, 32 bits |

For each matching group, the function:

- Calls F-2b01710262ca with access to the existing width/state and shared-state cells.
- Stores the full 32-bit result at that group’s destination.
- Calls F-8a872239bf2c with the current width/state value and access to the shared-state cell.
- Returns 0 immediately if validation returns zero.

A failed field validation prevents every later field group and every later iteration. The just-written field is not restored.

These conditions are separate, sequential checks against output offset 3. They are not established as one immutable tag dispatch. The function rereads the output tag for later checks after intervening dependency calls. If a dependency can modify the output through an alias, later groups may observe a different tag.

Under the additional condition that dependencies do not modify the tag byte:

- Tag 1 obtains one value for offset 4.
- Tag 2 obtains one value for offset 8.
- Tag 3 obtains two values, first for offset 4 and then for offset 12.
- Tag 4 obtains one value for offset 16.
- Tag 6 obtains one value for offset 12.
- Every other tag obtains no payload value in that iteration.
- Tag 0 terminates successfully after its own validation succeeds.
- An unrecognized nonzero tag leads to another tag acquisition.

For tag 3, the two fields come from distinct calls. The value stored at offset 4 is not reused for offset 12.

### Widths and arithmetic

The variable-value dependency supplies a 32-bit result. Tag storage discards all but its low 8 bits; payload storage retains all 32 bits.

The mutable width/state value supplied to validation is 32 bits. No range checks, clamping, or conversion to an 8-bit width are established. The value used is the one present after the immediately preceding variable-value call.

The function performs no visible count decrement, payload-length accumulation, or arithmetic on acquired values. Its direct output addressing uses fixed offsets through byte 19.

## Side effects

### Direct effects

The function writes only the output locations reached by the selected path and conditions. It may overwrite the tag byte and selected payload fields repeatedly across iterations. Thus the final buffer is not an append-only record of all items processed.

No output region is cleared at entry. No transactional commit or failure cleanup is established.

The function directly allocates no heap object, frees no object, closes no resource, and transfers no ownership. Those observations do not establish the behavior of its dependencies.

### Dependency-mediated effects

All dependencies receive the opaque context and access to the shared state cell. F-2b01710262ca additionally receives access to the mutable width/state cell.

Dependency effects on context-owned resources, globals, aliased output storage, resource lifetimes, or external state are unknown. They must not be assumed absent. Nor is it established that a validation call merely inspects state without changing it.

The addresses supplied for the two mutable cells refer to per-invocation storage. Retaining and using those addresses beyond the invocation has no established valid lifetime.

### Failure and termination

Explicit failure is represented solely by a zero result from F-8a872239bf2c. There is no established separate error code, diagnostic message, exception policy, or error-output field.

For tagged processing, there is no explicit iteration limit. If validation continues to succeed and the captured end-of-iteration tag never becomes zero, the function has no established successful termination.

## Dependencies

| Opaque ID | Contract visible at this call site | Unknowns |
|---|---|---|
| F-d54421325734 | Called with context, the value 1, and a reference to shared 32-bit state; supplies an 8-bit result. | Meaning of the result, state changes, acquisition source, external effects, and failure conventions beyond subsequent validation. |
| F-8a872239bf2c | Called with context, a 32-bit argument, and a reference to shared state; returns a value tested as zero versus nonzero. | Meaning of the argument, whether it advances or validates anything, other state changes, and ownership effects. |
| F-2b01710262ca | Called with context and references to the mutable width/state and shared-state cells; supplies a 32-bit result. | Encoding, interpretation of the mutable cells, external effects, and whether results represent signed values. |

The dependency names here are identifiers only. Descriptions such as “acquisition” and “validation” distinguish their roles in this function’s control flow, not independently established dependency semantics.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless noted, they assume dependencies do not modify output storage through aliases and return normally.

1. **Mode 5, both validations succeed.**  
   If the byte acquisitions produce 0x12 and 0xAB and both validations return nonzero, offsets 0 and 1 become 0x12 and 0xAB. The function returns 1. There are two acquisition calls and two validation calls.

2. **Mode 5, first validation fails.**  
   If the first acquired byte is 0x12 and the first validation returns zero, offset 0 is written, offset 1 is not directly written, and the function returns 0. No second acquisition occurs.

3. **Zero initial control byte.**  
   In any non-5 mode, an acquired initial byte of zero followed by successful validation produces return value 1 without calling F-2b01710262ca. Offset 3 and all payload fields remain directly untouched.

4. **Initial zero does not bypass validation.**  
   If that initial byte is zero but its validation returns zero, the result is 0, not 1.

5. **Tag 3 followed by termination.**  
   With a nonzero initial control byte, a validated tag 3 causes two payload acquisitions. Their 32-bit results are stored at offsets 4 and 12 in that order, each before its own validation. If all validations succeed and the next validated tag is zero, the function returns 1.

6. **Failure on the first payload of tag 3.**  
   Offset 4 is written before its failing validation. Offset 12 is not directly written during that iteration, and no next tag is acquired.

7. **Unrecognized nonzero tag.**  
   A validated tag 5 causes no payload acquisition and does not itself fail. Processing proceeds to another tag acquisition.

8. **Tag truncation.**  
   A tag-acquisition result of 0x00000103 stores tag 3, not a distinct tag 259. Subject to successful validation and unchanged output storage, it selects the two tag-3 payload fields.

9. **Width/state propagation.**  
   If a variable-value call changes the mutable width/state cell to 7, the following validation receives 7. The cell is not reset to the original input before the next variable-value call.

10. **Aliased tag mutation.**  
    If tag 1 selects the first payload group and an intervening dependency changes offset 3 to 2, the second group can also execute in that iteration. A fixed, single-dispatch interpretation would not preserve the established read ordering.

## Uncertainties

The following are **documented_only**:

- The context’s type, validity requirements, and ownership.
- The semantic meanings of both mutable 32-bit cells.
- The encoding, signedness, and units of payload values.
- The byte order and exact target alignment requirements of 32-bit output stores.
- Whether dependencies modify globals, output aliases, resources, or memory ownership.
- Whether dependencies can fail by means other than returning normally.
- Whether surrounding callers guarantee eventual zero-tag termination.
- Whether the initial control byte has an intended count-like meaning outside this function.

An inferred intent is that the function decodes either a compact two-byte form or a tagged collection into fixed output slots. That interpretation is not required for the behavioral contract and does not justify adding count-based termination, rejecting unknown tags, clearing fields, or rolling back failed writes.

## Evidence

The established claims above come from static inspection of the supplied function representation: argument widths, fixed output offsets, dependency argument forms, store-before-validation ordering, conditional branches, tag rereads, and return paths.

No dependency implementations, caller contracts, format documentation, target-layout specification, or runtime observations were provided. **No runtime tests have been performed.** The acceptance examples state consequences conditional on dependency behavior; they do not claim observed execution results.
