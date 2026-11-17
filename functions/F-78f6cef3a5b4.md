<!-- Trusted provenance: F-78f6cef3a5b4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-78f6cef3a5b4

## Purpose

F-78f6cef3a5b4 performs a long ordered sequence of unresolved indirect calls, transforms values obtained across those calls, and then modifies memory through calculated addresses and absolute global locations. Its last represented action is an indirect invocation with four argument words.

**Established static evidence:** The supplied representation identifies arithmetic, call ordering, memory-access widths, capture points, overlapping accesses, and repeated stores. It does not establish an application-level operation.

**[INFERENCE]** The shifts and masks suggest packed-field processing; the offset-based accesses suggest relationships between memory regions. Neither interpretation establishes a record format, collection type, serialization protocol, or ownership model.

This contract describes represented behavior, not a verified executable implementation. Environmental semantics and unresolved dependency behavior remain **documented_only** where the supplied evidence does not determine them.

## Interface

### Explicit inputs

The represented interface has four 32-bit arguments and no declared result.

| Argument | Established explicit use |
|---|---|
| First | No subsequent explicit reference appears. |
| Second | No subsequent explicit reference appears. |
| Third | Shifted left by four before an early indirect invocation. |
| Fourth | Shifted left by 28 before that invocation; the captured transformed word is subsequently shifted left by four. |

The absence of explicit references to the first two arguments does not establish their irrelevance to actual execution. Unresolved calling conventions may expose incoming register or stack state to dependencies.

### Implicit inputs

Execution additionally depends on:

- Incoming control-link state determining the first indirect target.
- Primary results and secondary words exposed across indirect calls.
- Represented 64-bit dependency results.
- Existing memory at calculated and absolute addresses.
- An address-bearing global word captured before the repeated-store region.
- The unresolved final target.

The explicit arguments alone are insufficient to determine behavior or safe execution.

### Outputs, completion, and errors

No return value, status code, output buffer, or length result is established. Observable outputs are dependency invocations, memory accesses, and dependency effects.

Normal return requires every preceding invocation and access to complete and the final target to return. No explicit validation, error conversion, rollback, or recovery appears. If execution stops partway through, completed effects are not represented as being undone.

Executable targets, accessible memory, supported alignment, and suitable memory mappings are environmental preconditions. No argument range independently guarantees these conditions.

## Behavior

### Widths and interpretation

The principal computational width is 32 bits. Represented paired results are 64 bits; byte, halfword, and word accesses are respectively eight, 16, and 32 bits.

A 64-bit result supplies distinct low and high 32-bit words. Right shifts by 34, 36, or 44 positions operate on the full unsigned 64-bit quantity before narrowing. They must not be replaced with oversized shifts of its low word.

The conditional **32-bit bit-vector interpretation** used below retains the low 32 bits of left shifts, multiplication, and address arithmetic. This permits useful deductions but does not independently establish the target semantics of signed overflow, signed left shifts, address wrapping, or invalid operations.

Explicit unsigned right shifts zero-fill. Explicit widening of an unsigned byte yields a word from 0 through 255. Narrowing to a byte retains the low eight bits.

Byte order is not established. Consequently, a byte write overlapping a word can affect its later value, but the affected word bits cannot be specified without an additional byte-order assumption.

### Indirect-call phase

Three indirect calls precede the first represented invocation carrying transformed explicit arguments. The first target depends on incoming control-link state; subsequent targets are derived from embedded control locations with displacement 0xffe.

The early argument-bearing invocation receives, in order:

1. The third argument shifted left by four.
2. A secondary word exposed by an earlier call.
3. The fourth argument shifted left by 28.

The remaining call phase is represented as straight-line ordered invocations, without an explicit loop or conditional branch. Dependencies may nevertheless branch, mutate memory or globals, fail, transfer control, or not return.

Argument construction repeatedly uses shifted primary results, high and low words of paired results, secondary words, masked fields, zeros, and duplicated values. Captures are sometimes reused across intervening calls. Distinct call sites are not interchangeable merely because their argument shapes match.

Meaningful arithmetic distinctions include:

- A low word shifted left by eight may later undergo another left shift by seven.
- High words are forwarded both unchanged and shifted.
- Masking before a right shift differs from shifting the complete word.
- Values explicitly reduced to zero remain zero through subsequent represented shifts.
- A previously captured paired result may remain relevant after later calls produce other results.

Under the conditional 32-bit interpretation, shifting the fourth argument left by 28 and then shifting that captured result left by four yields zero. The earlier shift-by-28 argument still depends on the original low nibble.

An invocation represented without explicit arguments is not established to receive no machine-level inputs. Secondary words are not established to be conventional return values rather than otherwise unrecovered machine state.

### Initial direct accesses

The following descriptive labels identify values, not source identifiers or object types:

| Contract label | Definition |
|---|---|
| Primary base | A late dependency result multiplied by \(2^{25}\). |
| Shifted base | A captured low result word shifted left by 25. |
| Small index | Another result shifted left by eight, masked to 15 bits, then shifted right by six. |
| Small base | Twice the small index. |
| Earlier shifted word | A separately captured dependency result shifted left by eight. |

Under the conditional interpretation, the primary and shifted bases depend only on their originating low seven bits. The small index is a multiple of four from 0 through 508; the small base is a multiple of eight from 0 through 1016. None is thereby a valid pointer.

The first direct accesses establish this ordering:

- A word at offset 0x14 from the last paired result’s high word receives the shifted base.
- A byte at shifted-base offset 9 receives the low byte of that high word.
- A word at the last paired result’s low word multiplied by 256, plus 0x54, receives the primary base.
- The byte at shifted-base offset 9 receives the low byte of the captured shifted base.

These are two distinct byte stores to the same destination, not necessarily two unequal values. Under the conditional interpretation, the second byte is zero because the shifted base has 25 trailing zero bits. The first byte may also be zero. The intervening word store remains a separate effect and may alias other locations.

The next access ordering is specifically:

1. Clear the word at primary-base offset 0x14.
2. Read the byte at primary-base offset 5.
3. Clear the word at small-base offset 0x14.
4. Write the captured byte, zero-extended, to the word at primary-base offset 0x34.
5. Read the word at primary-base offset 0x14.

The byte read occurs between the clears; moving it after both would change the represented ordering.

Under the conditional 32-bit interpretation and ordinary, stable memory semantics, the last word read is zero. The small-base clear either addresses the same word or does not overlap it: the small base is restricted to a small multiple of eight, whereas the primary base is a multiple of \(2^{25}\). The primary-base offset-0x34 store does not overlap its offset-0x14 word.

The represented stores therefore do not justify a nonzero value for this particular read through aliasing. Device-memory behavior, concurrent modification, address translation, and other environmental effects remain unresolved.

After that capture, the small-base offset-0x14 word is cleared again, and its offset-0x15 byte is written twice with the captured word’s low byte.

### Early absolute globals and dependent state

The routine next captures the value read at absolute address **0x05**, widening it to an unsigned word, **before** writing the low byte of the small base to absolute address **0x01**. The width of the original 0x05 global access is not independently established by a supplied declaration; its unsigned-word conversion is explicit.

The captured value from 0x05 subsequently serves as an address base. It is not reloaded merely because later writes might overlap that global.

The ensuing state changes include:

- A byte store at captured-global-base offset 0x15.
- A byte at primary-base offset 0x11 receiving the earlier shifted word’s low byte and later being cleared.
- A word store at offset 0x54 from the captured primary-base offset-0x14 value.
- Capture of the word at captured-global-base offset 0x54 before that location receives the captured primary-base offset-0x14 value.

Call the word captured from this last offset-0x54 location the **seed word**. It supplies subsequent scaled addresses and data.

Further dependent reads follow words at offsets 0x74 and 0x64, create self-referential word stores, and combine byte stores with word stores. Repeated stores remain distinct accesses.

Within this sequence, absolute address **0x54** receives a 32-bit store of the seed word shifted left by one. This occurs after an initial scaled-value store to small-base offset 0x14 and before that field receives the seed word twice. Later, after two byte stores at small-base offset 0x15 using the low byte of four times the seed word, absolute address **0x15** receives the low byte of the small base.

These absolute accesses are not merely annotations for calculated fields. They participate in the access sequence and may alias calculated destinations. Under ordinary memory and the conditional zero deduction above, an earlier calculated offset-0x54 store also reaches absolute 0x54; it precedes the explicit seed-derived absolute store.

### Mixed-width memory relationships

The accessed offsets include:

| Offsets | Represented widths |
|---|---|
| 1, 5, 8, 9, 0x0d, 0x11, 0x15, 0x19 | Eight bits |
| 4 | Eight or 32 bits, depending on the access |
| 0x10 | Eight and 32 bits |
| 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 | 32 bits |

This is an access inventory, not a complete layout or size declaration.

Words loaded from offsets 0x64 and 0x74 become bases for later accesses. Some address chains instead begin with an unsigned byte read at offset 5, followed by a word read at that byte value plus 0x74. The byte remains an address component from 0 through 255, not a recovered full pointer.

The region includes successive distinct values at the same word location, repeated clears, repeated self-reference stores, and companion byte writes. General aliasing remains relevant here; the earlier zero deduction applies only to its specifically bounded access interval.

### Constants and repeated-store region

A later dependent word supplies destinations for separate eight-bit and 16-bit stores of 0x3a and a 32-bit store of 0x3a at a related offset-0x20 location.

For the following region, define:

- **Adjusted base:** four times the seed word, minus 0x96.
- **Working base:** four times the adjusted base.
- **Publication value:** four times the dependent word used for the preceding constant stores.

All scaling uses the conditional width model where required.

The word at working-base offset 0x20 receives 0x20, then 0x2f, then the zero-extended byte read at working-base offset 4, then 0x20 again. The byte read belongs between the second and third stores.

Next, the routine captures the 32-bit address-bearing global at **0x087fbbbc**, before writing the word 0x20 to absolute address **0x60**. The captured global value supplies the **publication base** throughout the repeated stores; no reload is represented within that run.

Each publication group has four ordered stores:

| Position | Effect |
|---|---|
| First | Word at a scaled-base offset 0x54 receives the publication value. |
| Second | Byte at publication-base offset 0x19 receives the working base’s low byte. |
| Third | Byte at working-base offset 0x11 receives the publication base’s low byte. |
| Fourth | Word at publication-base offset 0x34 receives the scaled base. |

The first group uses eight times the seed word as its scaled base. Subsequent groups use eight times the adjusted base.

No progress update, loop condition, synchronization, or atomic-publication guarantee is represented. Individual stores must not be collapsed solely because their values repeat. The executable fidelity and executable repetition count of this unusually long region remain **documented_only**.

### Final captures, global reloads, and invocation

The final region begins by capturing the 32-bit word at absolute **0xa9**, then a word at publication-value offset 0x74. The former is the **captured absolute word**; the latter supplies a later narrowed byte.

Absolute **0x3a** then receives an eight-bit value formed from eight times the adjusted base plus 0x30, narrowed to eight bits.

A doubled dependent base retained from the preceding memory chain supplies two important captures: its offset-0x14 word, preserved for the final invocation, and later its offset-4 word. The working-base offset-4 word is also captured before later stores reuse related locations.

After several calculated byte stores and a self-reference word store, absolute **0x67** receives the low byte of the captured absolute word. The routine then reads a word at doubled-base offset 0x44 before reading byte **0x67** and transferring that byte to absolute **0x21**.

This transfer is a read from 0x67, not an unconditional reuse of the captured 0xa9 word’s low byte. The intervening operation is a read; its environmental effects are unresolved.

Subsequent calculated writes include:

- An eight-bit write at working-base offset 0x10.
- Two successive word stores at publication-base offset 0x54.
- A byte store involving a signed right shift whose count is the working base’s low eight bits.
- A word captured from working-base offset 0x40 and then written at offset 0x10, overlapping the earlier byte destination.
- A word store at publication-base offset 0x74 using the previously captured working-base offset-4 value.

The routine subsequently reads byte **0x67** for a store at working-base offset 9. After additional writes, including a word store of the full captured absolute word at working-base offset 0x44 and a word read at offset 0x54, it reads byte **0x67 again** for another store at working-base offset 9.

These later reads must remain reloads. Intervening calculated writes may alias absolute 0x67, so neither read is unconditionally equal to the byte initially assigned there. The full captured 0xa9 word, by contrast, remains a captured value when used as word-store data.

The word loaded from working-base offset 0x54 becomes another address base. Later updates include a word store of 100 at its offset 0x54 and a byte store derived from a separately captured working-base offset-0x14 word.

A word loaded from doubled-base offset 4 supplies twice its value as the destination of a 16-bit store of 0x65. Its offset-0x64 and offset-0x74 words are then captured, with intervening writes respected. A further doubled-base offset-0x64 read contributes to the last dependent word stores.

The final unresolved invocation receives four represented words:

1. 0x72.
2. The late offset-0x64 word obtained through the value loaded from doubled-base offset 4.
3. The earlier captured doubled-base offset-0x14 word.
4. The late offset-0x74 word obtained through that same offset-4-derived base.

No result is explicitly consumed.

## Side effects

Visible effects comprise indirect invocations, eight-, 16-, and 32-bit memory accesses, absolute-global accesses, overlapping writes, and dependent reads after mutation.

No allocation or release operation is identified. Ownership transfer, borrowing, retention, and release obligations are **documented_only**. Address-like values do not establish ownership.

Unknown dependencies may modify globals or memory subsequently accessed. No preservation assumption applies. Purity, reentrancy, thread safety, idempotence, transactionality, and rollback are not established.

Represented access ordering does not independently establish hardware ordering, atomicity, observer visibility, or exact bus-access granularity.

## Dependencies

The following opaque IDs are documentation handles, not resolved target identities:

| ID | Boundary | Unresolved properties |
|---|---|---|
| F-D001 | Entry target derived from incoming control state | Signature, implicit inputs, results, effects, termination |
| F-D002 | Ordered family of subsequent indirect call sites | Identities, equivalence, conventions, secondary words, preservation |
| F-D003 | Final indirect target | Argument meanings, effects, ownership, results, return behavior |

F-D002 is a family, not one interchangeable function. No dependency is assumed to preserve globals, own memory, or release memory.

## Acceptance examples

These are conditional static examples. **No runtime tests have been performed.**

### Input truncation

Under 32-bit truncating shifts, a third argument of 0x10000001 contributes 0x00000010 to the early invocation. A fourth argument with low nibble 1 contributes 0x10000000 initially and zero after the additional left shift by four.

### Paired-result extraction

For a represented paired result of 0x123456789abcdef0, unsigned right shift by 36 followed by narrowing yields 0x01234567. Shifting only its low word is not equivalent.

### Initial byte stores

If the last paired high word ends in 0x00, both stores at shifted-base offset 9 may store zero. Acceptance still requires two represented store operations with the intervening word store preserved.

### Intervening byte read

An access trace must place the primary-base offset-5 byte read after the primary-base clear and before the small-base clear. A trace with both clears first does not match.

Under ordinary stable memory and the conditional width interpretation, the subsequent primary-base offset-0x14 word read returns zero, including when both clears address the same word.

### Absolute captures

The 0x05 capture precedes the byte write at 0x01. The publication-base global capture precedes the word write at 0x60. Aliasing does not retroactively change either captured value.

### Global reload versus captured word

Suppose the captured 0xa9 word ends in 0x12. The initial byte assignment at 0x67 stores 0x12. If an intervening calculated write changes byte 0x67 to 0x34, a later reload used for working-base offset 9 supplies 0x34. A word store using the captured 0xa9 value still uses the original complete word.

### Ordered overwrites and failure

The offset-0x20 sequence retains all four stores and its intervening byte read. Final-memory equality alone does not validate that access sequence.

If a dependency does not return, later effects are not required to occur. No established requirement undoes earlier writes after a later fault.

## Uncertainties

The following remain **documented_only**:

- Application purpose and complete memory layouts.
- Actual calling conventions, implicit arguments, and secondary-word meanings.
- Resolved indirect targets and dependency effects.
- Original access width of the global at 0x05.
- Signed arithmetic, oversized variable shifts, and address wrapping.
- Alignment, byte order, mapping validity, device semantics, and concurrency.
- Executable fidelity and repetition count of the long repeated region.
- Ownership, error signaling, exceptional control flow, and final return behavior.
- Hardware access ordering and externally observable granularity.

These unknowns prevent a complete portable implementation contract; they do not erase explicit capture-versus-reload distinctions or represented access order.

## Evidence

Evidence is limited to the supplied static representation. Embedded comments, prior descriptions, and reviewer statements are not independent proof.

Established observations include represented widths, explicit arithmetic, indirect-call order, direct-access ordering, absolute-global sites, capture points, reloads, overlapping widths, repeated stores, and final argument arrangement.

Zero deductions, address-range constraints, and numeric examples are conditional arithmetic conclusions. Application-level intent is explicitly separated as inference.

No target resolution, executable inspection beyond the supplied material, execution, memory tracing, or runtime testing has been performed.
