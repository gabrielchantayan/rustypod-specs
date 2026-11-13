<!-- Trusted provenance: F-c4a99fa364dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4a99fa364dc

## Purpose

This operation interleaves indirect dependency calls with byte-sized and 32-bit memory accesses. It transforms explicit inputs and dependency results into stored values, subsequent call arguments, and addresses.

No application-level purpose is established. Repeated offsets suggest structured storage, but the evidence does not establish objects, tables, devices, allocations, or a valid memory schema. Some accesses explicitly target very low addresses.

The supplied representation contains one visible execution sequence ending at a bad-instruction boundary, not a normal return. This contract describes that represented prefix. It does not guarantee that the prefix completes or establish execution beyond its final boundary.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First input | 32 bits, interpretation unspecified | Not explicitly consumed in the visible body |
| Second input | Unsigned 32 bits | First dependency argument, full-word store, and later byte derivation |
| Third input | 32 bits, interpretation unspecified | Two low-byte stores and one full-word store |

The first input’s apparent non-use does not prove irrelevance to the actual calling convention. Indirect-call signatures and hidden argument conventions remain unresolved.

### Additional entry state and preconditions

Two incoming machine-state values are consumed without a visible definition:

- A control-related value participates in determining the first indirect-call target.
- An address-like value supplies the base of an early byte store at offset 1.

Their origins and ABI roles are unknown. Execution also requires the selected call targets and accessed memory to support the represented operations. No validation establishes those conditions.

### Outputs

The displayed interface has no explicit return value. Observable effects are dependency invocations, memory accesses, dependency-generated effects, and possible arrival at the terminal boundary.

No success status, ordinary return, error code, exception type, or recovery mechanism is established.

### Widths and notation

A **word** is 32 bits; a **byte** is 8 bits. A **pair result** is a represented 64-bit result with low and high 32-bit halves.

- **low8(value)** extracts the least significant eight bits, after evaluating the expression inside it.
- Right shifts described as logical are unsigned shifts.
- Unsigned 32-bit operations retain their low 32 bits.
- Some scales, additions, and left shifts use represented signed intermediates. Their exact overflow behavior is **documented_only**; this contract does not silently replace them with portable modulo arithmetic.

Addresses below reproduce the represented address relationships. Actual address width, alignment requirements, and byte order are unresolved. A word store does not establish any particular physical byte order.

Descriptive names below identify captured values in the visible dataflow. They do not assert that unresolved dependencies preserve machine registers, globals, or memory.

## Behavior

### Dependency notation and ordering

Opaque IDs **F-U01-01**, **F-U01-02**, and so forth identify individual call sites. Different IDs are not proof of different target functions.

A range denotes separate invocations in ascending order. A call without listed arguments has no explicit arguments in the representation; its actual ABI is not established. Results are unused unless explicitly captured below.

Each stage follows the preceding stage. Reads and writes are separate observable events, including repeated accesses to the same address.

### 1. Capture an initial index

Before any dependency call, save the second input logically shifted right by 2.

Invoke **F-U01-01** with the second input. Its target depends on incoming control-related state. Invoke **F-U01-02** through **F-U01-05**, capturing the last result as the **initial index**.

Store the full second input as a word at four times the initial index plus 20.

Invoke **F-U01-06** with four times the initial index, then **F-U01-07** through **F-U01-10**. Capture the last result as the **first returned word**.

### 2. Write and overwrite an early word location

Form the **eightfold index** as eight times the initial index. In order:

1. Store low8(eightfold index) at the incoming address-like base plus 1.
2. Store the first returned word shifted left by 2 as a word at the eightfold index plus 52.

Invoke **F-U02-01** with the first returned word shifted left by 2. Invoke **F-U02-02**, capturing its word result as the **forwarded word**.

Retain the first returned word’s low 30 bits as the **masked base**.

Invoke **F-U02-03** with the forwarded word, then **F-U02-04** through **F-U02-07**. Capture the last result as a pair. In order:

1. Store low8(eightfold index) at the masked base plus 1.
2. Store the pair’s high word at the eightfold index plus 52.

The second word store overwrites the earlier word location, subject to intervening effects. The earlier store remains independently observable.

Invoke **F-U02-08** with the pair’s low word shifted left by 2, then **F-U02-09** through **F-U02-12**. Capture the last result as the **next scaled-call word**.

### 3. Publish third-input bytes

Form the **sixteenfold index** as sixteen times the initial index. In order:

1. Store low8(sixteenfold index) at the masked base logically shifted right by 20, plus 17.
2. Store low8(third input) at the sixteenfold index plus 13.

Invoke **F-U03-01** with two equal arguments: the next scaled-call word shifted left by 2.

Invoke **F-U03-02** and capture its word result. Invoke **F-U03-03** with that result and that result logically shifted right by 3.

Invoke exactly four further calls, **F-U03-04** through **F-U03-07**. Capture **F-U03-07** as the **later scaled-call word**. Then:

1. Store low8(sixteenfold index) at absolute address 17.
2. Store low8(third input) again at the sixteenfold index plus 13.

Invoke **F-U03-08** with the later scaled-call word shifted left by 2. Invoke **F-U03-09** through **F-U03-12**, capturing **F-U03-12** as the **record selector**.

There are no additional calls between **F-U03-07** and the two stores, or between **F-U03-12** and the next stage’s first store.

### 4. Derive and populate a record-like address

Store the full third input as a word at the masked base plus 20. Then store low8(masked base shifted left by 1) at the masked base plus 13.

Derive these captured values from the record selector:

- **Record base:** record selector multiplied by 32.
- **Record payload:** record selector shifted left by 7.
- **Small selector:** the record selector masked to its low 27 bits, then logically shifted right by 17.

The small selector ranges from 0 through 1023.

Invoke **F-U04-01** through **F-U04-03**. Capture the last result as the **record-call word**. In order:

1. Store the record payload as a word at the record base plus 20.
2. Store low8(record-call word shifted left by 2) at the record base plus 13.

Invoke **F-U04-04** with the record-call word shifted left by 2, then **F-U04-05** through **F-U04-08**.

### 5. Use fresh byte reads as address selectors

Store a byte at the small selector plus 7. Its value is:

**low8((second input logically shifted right by 2) shifted left by 1).**

The right shift operates on the full unsigned input; byte extraction occurs only after both shifts. This uses the right-shifted value captured before the first dependency call.

Read an unsigned byte at the record base plus 1. Store low8(small selector) at the read byte’s value plus 17.

Invoke **F-U05-01** with three equal arguments, each the record-call word shifted left by 21.

Invoke **F-U05-02**, capturing its word result as the **saved shift source**. Invoke **F-U05-03** with that value, then **F-U05-04** through **F-U05-07**. Capture the last result as the **clear-following call word**.

Read another unsigned byte at the record base plus 1 and retain it as the **saved byte**. Store zero at the saved byte plus 17.

These are distinct reads separated by calls. Their results need not agree. Each byte-selected destination lies between addresses 17 and 272 inclusive.

### 6. Publish the saved byte and invoke a constant-argument dependency

Invoke **F-U06-01** with the clear-following call word shifted left by 2, then **F-U06-02** through **F-U06-05**.

Derive:

- **Doubled-byte base:** twice the saved byte, ranging from 0 through 510.
- **Saved shifted byte:** low8(saved shift source logically shifted right by 3).

Store the saved shifted byte at the record base plus 13. Then store the saved byte at absolute address 13.

Invoke **F-U06-06** with three arguments, in order: the doubled-byte base, 152, and 608.

Invoke **F-U06-07**, then **F-U06-08** with an unresolved extra 32-bit machine-result value logically shifted right by 3. The representation does not establish that value’s producer or a dependable high-level result convention.

Invoke **F-U06-09** through **F-U06-12**. Invoke **F-U06-13** with the last result shifted left by 2, capturing its result as the **forwarding result**.

### 7. Replace low-address state and follow a word link

Store the saved shifted byte at the doubled-byte base plus 13. Store byte value 192 at absolute address 13.

Invoke **F-U07-01** with the forwarding result shifted left by 2, then **F-U07-02** through **F-U07-05**. Capture the last result as the **link payload source**.

Store the link payload source shifted left by 2 as a word at the doubled-byte base plus 36.

Read a word at the doubled-byte base plus 116 and capture it as the **linked base**. Retain the link payload source shifted left by 5 as the **scaled link payload**.

Invoke **F-U07-06** with, in order:

1. The linked base.
2. The linked base shifted left by 1.
3. The scaled link payload.
4. The linked base masked to its low 31 bits, then logically shifted right by 2.

Invoke **F-U07-07** through **F-U07-10**.

### 8. Update the linked location and consume pair results

Store the scaled link payload as a word at the linked base plus 36.

Read a word at the linked base plus 116 and pass it to **F-U08-01**. Invoke **F-U08-02** through **F-U08-05**, capturing the last result as the **first linked pair**.

Retain the first linked pair’s low word shifted left by 2 as the **retained linked payload**.

Store the saved shift source logically shifted right by 3 as a word at four times the first linked pair’s high word, plus 20. This store uses the full shifted word, not the saved shifted byte.

Invoke **F-U08-06** with three copies of the retained linked payload, followed by the first linked pair’s high word logically shifted right by 3.

Invoke **F-U08-07**, capturing its result as the **second linked pair**. Invoke **F-U08-08** with:

1. The second linked pair’s low word.
2. Its high word.
3. Its low word again.
4. Its high word logically shifted right by 3.

### 9. Publish the retained payload and perform two selected byte stores

Invoke **F-U09-01** through **F-U09-04**. Capture the last result as the **destination index**.

Store the retained linked payload as a word at four times the destination index, plus 20.

Invoke **F-U09-05** with four times the destination index, then **F-U09-06** through **F-U09-09**. Capture the last result as the **final selector**. Retain its left shift by 2 as the **final scaled selector**.

Read an unsigned byte at the linked base plus 1. Store low8(final scaled selector) at that byte’s value plus 17.

Invoke **F-U09-10** through **F-U09-12**, capturing the last result as the **final auxiliary word**.

Read another unsigned byte at the linked base plus 1. At that newly read byte’s value plus 17, store:

**low8((final selector masked to its low 30 bits) logically shifted right by 20).**

Masking and shifting precede byte extraction. Extracting the selector’s low byte first would describe a different operation.

### 10. Perform two separate final clears

Invoke **F-U10-01** with the final auxiliary word shifted left by 2, then **F-U10-02** through **F-U10-05**. Capture the last result as the **final call word**.

Read an unsigned byte at the final scaled selector multiplied by 1024, plus 1. Store zero at that byte’s value plus 17.

Invoke **F-U10-06** with:

1. The final scaled selector shifted left by 3.
2. The same value again.
3. The final scaled selector.
4. The final call word shifted left by 2.

Invoke **F-U10-07** through **F-U10-10**, capturing the last result.

Read the selector byte again at the final scaled selector multiplied by 1024, plus 1. Store zero at the newly read byte’s value plus 17.

Invoke **F-U10-11** with the result of **F-U10-10** shifted left by 2, then **F-U10-12** through **F-U10-15**.

The representation ends at a bad-instruction boundary. It supplies no ordinary return path.

## Side effects

### Memory and aliasing

Direct accesses include byte stores at absolute addresses 13 and 17, byte-selected stores within addresses 17–272, computed byte and word stores, and word reads supplying later addresses or arguments.

A byte store replaces only the addressed byte. A word access transfers a represented 32-bit quantity.

No non-aliasing guarantee exists. Computed locations may overlap each other or absolute-address locations. Earlier stores can therefore affect later reads. Dependencies may also observe or mutate those locations.

Consequently, the contract specifies an ordered event sequence, not merely final field values. Repeated stores and clears cannot be merged, and repeated reads cannot be replaced with cached values.

### Ownership and lifetime

No explicit allocation, deallocation, reference counting, ownership transfer, or lifetime validation is visible.

Address-like values do not establish valid allocations or access permissions. Passing a value to a dependency does not establish transfer of ownership. No dependency is assumed to preserve globals, preserve pointed-to storage, or release memory.

### Errors and interruption

No visible branch validates inputs, checks bounds or null values, handles dependency errors, or rolls back earlier effects.

Invalid accesses, invalid targets, dependency behavior, or the terminal condition may interrupt the sequence. Their actual platform effects are unknown. Earlier effects must not be assumed undone.

## Dependencies

All **F-U…** identifiers are document-local opaque call-site IDs. Target expressions derive from incoming control state or address-derived values. Function identities and possible shared targets remain unresolved.

The explicit argument lists contain up to four arguments. Selected results are represented as words or 64-bit pairs; their actual ABI interpretation remains unverified.

Unused results do not make calls removable. Every listed invocation retains its position relative to surrounding reads and writes. No functional role, hidden argument list, global-state preservation guarantee, failure convention, or memory-management responsibility is established.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes that the relevant operations complete and that unstated interference does not invalidate the example’s premises.

1. **Call count around the repeated third-input store.** After **F-U03-03**, exactly **F-U03-04** through **F-U03-07** execute before the two stores. The shifted argument to **F-U03-08** derives from **F-U03-07**. The record selector derives from **F-U03-12**.

2. **Full-word versus byte storage.** For third input `0x123456AB`, both stores at the sixteenfold index plus 13 carry byte `0xAB`. The word store at the masked base plus 20 carries `0x123456AB`.

3. **Second-input truncation precedence.** For second input 1023, the initial logical right shift produces 255; shifting left once produces 510; final byte extraction produces 254. Truncating the original input to a byte before shifting would instead produce 126 and is not acceptable.

4. **Observable overwrite.** For initial index 3 and first returned word 5, the first word store at address 76 carries 20. If the later pair’s high word is 9, the second store there carries 9. The intervening calls remain observable.

5. **Fresh selection.** If the first record-base byte read yields 4 and the later read yields 9, the small-selector store targets address 21 and the subsequent clear targets address 26.

6. **Small-selector boundary.** A record selector with all low 27 bits set produces small selector 1023. Its second-input-derived byte store targets address 1030.

7. **Saved value and aliasing.** Saved byte 255 produces doubled-byte base 510, so the later saved-shifted-byte store targets address 523. Saved byte zero instead makes that store target address 13; the immediately following constant store overwrites it with 192.

8. **Byte versus word use of a saved source.** Saved shift source `0x00001238` produces saved shifted byte `0x47`, but the later word store derived from that source carries `0x00000247`.

9. **Final-selector precedence.** Final selector `0x123000AB` produces byte `0x23` for the second selected store in stage 9. Masking and shifting an already extracted low byte would incorrectly produce zero.

10. **Independent final clears.** If the final selector-byte reads yield 2 and 7 respectively, the two zero stores target addresses 19 and 24. Reaching the last dependency does not establish normal return.

## Uncertainties

### Established static evidence

The supplied representation establishes displayed widths, explicit computations, call-site ordering, argument relationships, memory-access ordering, and the absence of a visible ordinary return.

### Inferred intent

Repeated scales and offsets suggest related storage layouts. This does not establish a record schema, object model, protocol, or business operation.

### Documented-only unknowns

The following remain **documented_only**:

- Accuracy and completeness of recovered instructions and control flow.
- Architecture, ABI, address width, byte order, and alignment requirements.
- Signed-overflow, signed-conversion, and signed-left-shift behavior.
- Dependency identities, hidden arguments, clobbers, result conventions, and errors.
- Origins and validity of incoming machine-state values.
- Producer and meaning of the extra machine-result word.
- Memory mappings, permissions, concurrency, and memory-mapped effects.
- Whether the terminal boundary corresponds to an actual invalid instruction, incomplete recovery, or missing control flow.

These unknowns are not assumptions of safety or successful completion.

## Evidence

This contract derives exclusively from static examination of the supplied representation for **F-c4a99fa364dc**. Prior prose, comments, and reviewer feedback are not independent evidence of execution.

No implementation access, dependency inspection, external ABI verification, or runtime testing has been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed runtime results.
