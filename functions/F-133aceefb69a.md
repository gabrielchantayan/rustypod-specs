<!-- Trusted provenance: F-133aceefb69a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-133aceefb69a

## Purpose

F-133aceefb69a gates a sequence of dependency calls using object state, classification, and a table entry. If eligible, it offers temporary storage to an acquisition dependency, selects a block size, constructs header fields for nominal fragments, and invokes intermediate-submission and final-completion dependencies.

“Acquisition,” “fragment,” “submission,” and “completion” describe inferred roles. The established behavior is the visible ordering of calls, arguments, memory accesses, arithmetic, and branches. In particular, the dependency receiving source and destination addresses is **not established to copy memory**. Its effects and transfer extent remain unverified.

The function reports a small set of control-flow statuses. It does not visibly verify the final-completion result or establish external protocol success.

## Interface

### Inputs and object layout

The visible interface accepts:

- One 32-bit value not referenced by the body.
- A pointer to an object with the following visible accesses.

| Object offset | Width | Access |
|---|---:|---|
| 0 bytes | 32 bits | Read before the first dependency call and saved for subsequent arguments. |
| 12 bytes | 16 bits | Read after F-5f5f14cedda3 returns; must be zero to continue. |

The object pointer is also passed to F-5f5f14cedda3 and F-dece790ddc92. The complete object layout, ownership, and allowable mutations are unknown.

The saved first word is not subsequently refreshed from the object. Thus, a dependency may change the object’s first word without changing the value this function explicitly saved.

### Return values

Assuming execution reaches an ordinary return:

| Value | Condition |
|---:|---|
| 0 | Acquisition returned zero, no non-returning branch was taken, and final completion returned. |
| 2 | Acquisition returned nonzero. |
| 4 | The object field at offset 12 was nonzero, or classification was 6 but the selected table byte was not 2. |
| 7 | Classification was not 6. |

Successful acquisition with a zero low-16-bit length still reaches final completion and returns 0 if that call returns.

A separate failure path invokes F-a817d8d58c10, which is marked non-returning. That path has no ordinary return status.

### Temporary storage and safety requirements

The visible temporary layout includes:

- A 2,048-byte acquisition region.
- A four-byte header immediately preceding a separate 2,048-byte outgoing region.
- A 32-bit size/result slot.
- Scalar state for a source position, block size, final extent, and byte-sized count and ordinal.

The header is initialized from a global 32-bit value. Neither 2,048-byte region is visibly initialized in full.

There are no visible null checks, table-index bounds checks, or comprehensive checks relating lengths and block sizes to these capacities. Safe execution requires valid object and table addresses, valid pointer arithmetic, and dependency memory accesses compatible with the supplied storage. These are environmental requirements, not validations performed here.

## Behavior

### Initialization and eligibility

The function first reads the global initializer into the temporary header, then reads and saves the object’s first 32-bit word. It calls F-5f5f14cedda3 with the object pointer and only afterward reads the 16-bit field at offset 12.

A nonzero field returns 4. Changes made to that field during F-5f5f14cedda3 therefore affect eligibility.

For a zero field, U-01 is called without visible arguments. Its result and the result of F-5f5f14cedda3 are passed, in that order, to F-d5cded701dac. Any classification other than 6 returns 7 without reading the table entry.

For classification 6, one byte is read at a global base plus 36 times the index returned by F-5f5f14cedda3, plus 28 bytes. Only byte value 2 permits further processing; every other value returns 4.

The index is represented as a signed 32-bit integer. Negative and excessive indices are not rejected. The source establishes the address expression but does not establish a valid address range or safe behavior for overflowing index/address arithmetic.

### Block-size selection

After eligibility succeeds, the size/result slot is set to 2,048 and the source position is set to the acquisition region’s beginning. F-1a836ac93559 is then called with the saved object word and zero.

Its result is treated as unsigned 32-bit:

- Results from 0 through 128 select block size 128.
- Result 1,016 also selects 128.
- Every other result above 128 selects that result unchanged.

There is no visible upper cap of 2,048. In particular, 1,015 and 1,017 remain unchanged, while 1,016 maps to 128.

### Acquisition and initial length decision

F-479b289100a4 receives the acquisition-region address and the address of the 32-bit size/result slot. The slot was initialized to 2,048 before the size-query call.

A nonzero acquisition return produces status 2, without intermediate submission or final completion. A zero return permits further processing.

The slot’s address has now escaped. The source does not establish that F-479b289100a4 discards that address, or that later dependencies cannot alter the slot through retained or indirect access. Later slot reads therefore obtain the value present at each read, not necessarily a value preserved from acquisition.

After a zero acquisition return, the function reads the slot’s low 16 bits to decide whether to prepare fragments:

- Zero skips all count selection, header-field preparation, U-02 calls, and intermediate submissions.
- Nonzero enters count selection.

On the zero-length branch, the internal final extent remains zero. Final completion still occurs. A locally selected status 2 is overwritten with 0 after completion returns; it is not the externally returned result for this branch.

### Count selection and header representation

For a nonzero effective length, F-870c4fcf7295 receives that low-16-bit length minus one and the selected block size. The first argument is therefore between 0 and 65,534 at this call.

Only eight bits of its result are retained. This retained value controls the intermediate-iteration count. No division rule or consistency relationship between count, length, and block size is established.

The ordinal begins at zero. Each header preparation writes:

| Header bits | Value written |
|---|---|
| 0–15 | Preserved from the header’s current value. |
| 16–23 | Current ordinal. |
| 24–31 | Retained count. |

These are numeric bit assignments, not an established wire-byte order. Preserving the current lower half is not equivalent to restoring the initializer: dependencies may have modified the header.

F-870c4fcf7295 runs after the slot address has escaped. Its return does not establish that the slot still contains the length used to construct its arguments.

### Intermediate iterations

An intermediate iteration occurs while the ordinal is less than the retained count. Because both values are unsigned bytes, the visible loop permits at most 255 iterations, using ordinals 0 through 254.

Within an iteration, the function first installs the ordinal and count in the header. It then calls U-02 with three visible arguments:

1. The outgoing-region address.
2. The current acquisition-region source position.
3. The selected 32-bit block size.

No copying, transfer length, successful initialization of the outgoing region, or other memory effect is established by this call alone.

After U-02 returns, the function advances the source position by the block size. It then **reads the size/result slot afresh**, takes that read’s low 16 bits, subtracts the block size using unsigned 32-bit arithmetic, and stores the result back into the slot.

Consequently, this subtraction uses the slot value **after U-02**, not necessarily the acquisition value or the value left by the preceding iteration. Underflow wraps modulo \(2^{32}\); later low-half reads discard the upper 16 bits.

The function next calls F-cd7e1905512f with:

- The saved object word.
- Two zeros.
- Operation value 27.
- The header address.
- An extent derived from the selected block size.

The extent is block size plus four, with 32-bit unsigned wraparound, followed by clearing bit 16 using mask `0xFFFEFFFF`. This is neither ordinary alignment nor general truncation to 16 bits.

The header writes precede U-02. The source does not guarantee that dependencies preserve those writes until submission reads the header.

A zero submission result immediately invokes F-a817d8d58c10. The source-position advance and slot update have already occurred. For a nonzero result, F-256e8a77aa59 receives that result and zero; after it returns, the ordinal increments.

Both submission and result processing occur between this iteration’s slot update and any later slot read. Preservation of the slot across either call is not established.

### Final validation and preparation

After the intermediate loop, the function reads the slot’s low 16 bits for final validation. This occurs after F-870c4fcf7295 when there were no iterations, or after the last intermediate dependencies when there were iterations.

A zero low half invokes F-a817d8d58c10. If nonzero, the low half is read for comparison against the block size; a value greater than the block size invokes the same dependency. No opaque call intervenes between these validation tests.

If validation passes, the function writes the final ordinal and retained count into the header’s upper two bytes, preserving its current lower half. It then calls U-02 with only two visible arguments: the outgoing-region address and the current source position.

There is **no visible third length argument** on this invocation. Neither a remaining-length transfer nor any other copying behavior may be assumed.

After this final U-02 returns, the function **reads the size/result slot again**. The internal final extent is calculated from this new read’s low 16 bits, plus four, with bit 16 cleared by `0xFFFEFFFF`.

The validated remainder and the value used for this calculation are therefore not guaranteed to match. There is no second validation after U-02. For example:

- A post-call low half of zero produces extent 4.
- Post-call low halves 65,532 through 65,535 produce extents 0 through 3.

Those calculations apply even if the newly read value would not have passed the preceding validation.

### Final completion

Whenever acquisition returned zero and execution has not taken the non-returning path, the function stores the internal final extent into the size/result slot. It then calls F-dece790ddc92 with:

- The object pointer.
- The saved first word.
- Two zeros.
- Operation value 27.
- The header address.

No extent argument is visible. Storing the extent in the slot does not establish that final completion receives or reads it. Conversely, prior escape of the slot address prevents assuming that no dependency can access it.

No completion result is checked. If the call returns, this function returns 0.

## Side effects

### Established direct effects

The function directly:

- Reads the global header initializer, object fields, and conditionally a global table byte.
- Initializes and subsequently updates the local size/result slot.
- Writes header fields and advances local bookkeeping.
- Passes temporary-region addresses to dependencies.
- Passes the slot’s address to acquisition and the header’s address to submission and completion.

Dependency-induced writes are separate from these established direct effects. F-479b289100a4 receives addresses through which it could affect the acquisition region and slot, but its actual implementation is unavailable. U-02 receives addresses and, on intermediate calls, a block-size argument; **whether it copies, what it accesses, and how much memory it changes are unverified**.

### Lifetime, ownership, and shared state

The temporary storage belongs to this invocation. Its addresses do not provide valid storage after the invocation ends. The visible body performs no dynamic allocation or explicit release.

No ownership transfer, retention permission, or preservation guarantee is established by a pointer argument alone. In particular, escaped access to the size/result slot may affect later reads while this invocation is active.

F-256e8a77aa59’s handling of a nonzero submission result is unknown; receiving that result does not prove release, acknowledgment, or resource ownership.

Dependencies may also modify globals, object contents, the header, or other reachable state. F-a817d8d58c10’s concrete non-returning action is undocumented.

## Dependencies

| Opaque ID | Visible interaction |
|---|---|
| F-5f5f14cedda3 | Receives the object pointer; returns the index used in classification and table addressing. |
| U-01 | Called without visible arguments; supplies the first classification argument. |
| F-d5cded701dac | Receives U-01’s result and the index; only result 6 continues. |
| F-1a836ac93559 | Receives the saved word and zero; supplies the unsigned block-selection value. |
| F-479b289100a4 | Receives acquisition storage and the slot address; zero permits final-completion processing. |
| F-870c4fcf7295 | Receives effective length minus one and block size; eight retained result bits control iteration. |
| U-02 | Receives outgoing and source addresses; intermediate calls additionally show block size. Memory semantics are unknown. |
| F-cd7e1905512f | Receives intermediate header arguments and extent; zero triggers non-returning failure. |
| F-256e8a77aa59 | Receives each nonzero intermediate result and zero; its return is not used. |
| F-a817d8d58c10 | Marked non-returning; reached for zero intermediate results or invalid final validation. |
| F-dece790ddc92 | Receives final-completion arguments without an explicit extent; its result is not checked. |

U-01 and U-02 are document-local opaque identifiers for dependencies without supplied mappings.

## Acceptance examples

These are conditional static cases, not observed executions. They assume valid memory and normally returning dependencies except where non-returning behavior is specified. Header examples describe this function’s writes, not guaranteed header contents after intervening dependencies.

1. **Object rejection.** If the offset-12 field is nonzero when read, return 4. F-5f5f14cedda3 has already run; classification and acquisition do not occur.

2. **Classification rejection.** With a zero field and classification result 5, return 7 without reading the table entry or calling acquisition.

3. **Table rejection.** With classification 6 and selected table byte 1, return 4 without acquisition.

4. **Block-selection boundary.** Eligible invocations receiving size-query results 128, 129, 1,016, and 1,017 select block sizes 128, 129, 128, and 1,017 respectively.

5. **Acquisition failure.** A nonzero acquisition return produces 2. There is no U-02 invocation, intermediate submission, or final completion.

6. **Empty low half.** If acquisition returns zero and the subsequent length read sees `0x00010000`, count selection and fragment preparation are skipped. The internal final extent remains zero, is stored into the slot before completion, and the function returns 0 if completion returns. No ordinal or count is installed by this function.

7. **One intermediate iteration with preserved slot.** Suppose acquisition leaves effective length 200, block size is 128, and the retained count is 1. **If every dependency after acquisition preserves the slot**, the intermediate U-02 call has third argument 128, the subsequent subtraction stores 72, and intermediate extent is 132. The function writes ordinal 0/count 1 before that U-02 call. After successful submission processing, final validation accepts 72. The final header writes ordinal 1/count 1. If final U-02 also preserves the slot, the internal final extent is 76. Neither call establishes copied bytes, and the final call has no visible length argument.

8. **No intermediate iteration.** With effective length 100, block size 128, retained count 0, and slot preservation through count selection, validation accepts 100. The final header writes ordinal 0/count 0. Extent 104 additionally requires preservation through final U-02.

9. **Invalid remainder without intervening mutation.** With effective length 200, block size 128, retained count 0, and slot preservation through count selection, final validation invokes F-a817d8d58c10 because 200 exceeds 128.

10. **Slot change during count selection.** If count selection returns retained count 0 but directly or indirectly changes the escaped slot to zero, final validation invokes the non-returning dependency even though the earlier acquisition-length test was nonzero.

11. **Slot change after validation.** Suppose validation accepts 100 with block size 128. If final U-02 directly or indirectly leaves the slot’s low half at 65,535, the next read produces internal final extent 3. No repeated validation rejects that value.

12. **Intermediate submission failure.** A zero intermediate result invokes F-a817d8d58c10 after the source advance and slot subtraction. F-256e8a77aa59 and final completion are not reached for that branch.

## Uncertainties

The following remain **documented_only**:

- Object, table, initializer, operation-27, and payload meanings.
- The actual count calculation and its consistency with lengths.
- U-02’s operation, memory effects, transfer extent, and final-call argument contract.
- Slot retention and modification by dependencies after its address escapes.
- Header preservation across dependencies.
- Whether or how final completion obtains an extent.
- Resource handling and ownership effects of submission-result processing.
- Target byte ordering and external wire representation.
- Validation inside dependencies and behavior outside valid memory bounds.
- Effects of excessive block sizes, inconsistent counts, and pointer-arithmetic overflow.
- The concrete action of the non-returning dependency.

## Evidence

Established behavior derives only from the supplied static body: access widths, temporary-storage declarations, visible arguments, branch conditions, header bit assignments, and arithmetic.

Fragmentation terminology is inferred intent. Dependency copying, slot preservation, ownership, and external protocol semantics are not established.

No runtime tests have been performed. No dependency implementation, runtime memory-safety property, or protocol conformance has been independently verified.
