<!-- Trusted provenance: F-d897597fcebf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d897597fcebf

## Purpose

F-d897597fcebf performs a fixed sequence of indirect calls, computed-address memory accesses, and updates to apparent global storage. The supplied representation contains no explicit conditional branch or loop.

Its application-level purpose is unresolved. Four-byte scaling, upper-bit extraction, and repeated field copies suggest encoded-reference or packed-state processing. That is **inferred intent**, not an established data model.

The established behavior is operational: forward an explicit input, consume call results and additional register state, perform ordered byte and word accesses, construct a final global word, clear a halfword, and attempt a call to address zero. Normal return is possible only if all preceding operations, including that final call, return normally.

## Interface

### Inputs and result

The explicit interface accepts one 32-bit value. Its meaning and signedness are unspecified. The first indirect call receives it unchanged; no other explicit use is visible.

The routine has no explicit return value. Four internal calls expose 64-bit results whose lower and upper 32-bit portions are consumed separately.

Execution also depends on:

- Incoming link-register state, used to derive the first call target.
- An incoming register-held base used for a byte store and word load.
- An incoming 32-bit payload written to two destinations.
- Additional register outputs from dependencies, including slot selectors, repeated arguments, and a terminal byte pointer.
- Two temporally distinct samples of an apparent global source word.
- Repeated samples of an apparent byte at address 1.
- An apparent global destination word.
- An apparent three-byte source at address 9.
- Memory reached through computed addresses.

The explicit signature alone therefore does not establish a sufficient invocation contract.

### Widths, layouts, and arithmetic

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. A low-byte store retains only the least significant eight bits. For a 64-bit result, “low word” and “high word” mean its lower and upper 32 bits.

The recurring address forms are:

| Description | Address calculation |
|---|---|
| Ordinary slot | 20 plus four times a selector |
| Wide slot | 20 plus sixteen times a selector |
| Field base | Four times a selector |
| Byte-selected destination | 17 plus an unsigned byte loaded from another address |

These are address calculations, not evidence of arrays, validated structures, or allocation boundaries. A byte-selected destination lies between 17 and 272 inclusive.

Unsigned right shifts are zero-filling. Unsigned 32-bit arithmetic retains only the representable 32-bit result. Some scaling and left shifts operate on values represented as signed 32-bit integers; overflowing or negative-operand cases are **documented_only**, because their intended machine behavior is not independently established. Pointer validity, alignment, and address-calculation overflow likewise require an execution-environment contract.

### Preconditions and ownership

Reaching any operation requires all earlier calls and accesses to complete. Successful execution additionally requires callable indirect targets, suitable register relationships, and accessible memory of the indicated widths. The routine does not visibly validate these conditions.

No ownership transfer is established. Neither a passed address nor a call-produced pointer proves allocation ownership, lifetime extension, or responsibility for release.

## Behavior

All phases below are ordered. Unknown calls may change accessible memory, including apparent globals, and may fail or never return. They cannot be removed or moved across accesses merely because their results appear unused.

Dependency IDs identify unresolved call classes, not recovered function identities. Different occurrences of one class may have different targets.

### 1. Input forwarding and incoming-base access

The first call, F-00000001, uses a target derived from the incoming link value plus 4094 and receives the explicit input unchanged.

Four calls with no explicit arguments follow; the fourth exposes a 64-bit result. The routine then writes the low byte of the incoming base at that base plus 9. After this store, it reads a word at the same base plus 52.

The next dependency receives, in order:

- The low word of the 64-bit result shifted left two bits.
- Its high word.
- The word read at base plus 52.

Four further calls with no explicit arguments precede the next explicit store. The supplied representation also exposes an additional register output used as the next slot selector.

### 2. Payload stores and chained reads

The incoming payload is written to the ordinary slot selected by that additional output. A word is then read at the **unscaled** selector plus 20. A dependency receives the unscaled selector.

A no-explicit-argument call produces a result that the following call receives. Four further calls then produce another selector. Its ordinary slot receives the same incoming payload.

The next word read uses address 20 plus the first saved word shifted right 22 bits. A dependency receives four times the new selector.

Four more calls produce a result that is shifted left two bits. Using the last word read shifted right one bit as a selector, the routine writes this shifted result to an ordinary slot. It then reads a word at the **unshifted** last word plus 20 and calls a dependency with the shifted-right selector.

A further result-producing call, forwarding call, and four-call acquisition produce a new selector. Its ordinary slot receives the same shifted result. The apparent global source word is sampled, and a dependency receives four times this selector.

The read through the unshifted word occurs after the preceding slot store. Aliasing can therefore make that store affect the read.

### 3. Extracted fields and zero-valued stores

After four further calls, the routine extracts:

- The latest chained-read word shifted right 22 bits.
- The first global-source sample shifted right 23 bits.

An additional register output selects an ordinary slot receiving the chained-word extraction. The following call receives the selector twice, then the global-source extraction, then the chained-word extraction, in that order.

The next no-explicit-argument call produces a result. Before that result is forwarded, the already-extracted global value is shifted right another 23 bits. This second extraction is necessarily zero: the first extraction is at most 511.

Four further calls produce a selector whose ordinary slot receives the chained-word extraction. A dependency receives four times that selector. Another four calls precede an ordinary-slot store of zero selected by an additional register output, followed by a call receiving that output.

A result-producing call, its forwarding call, and four more calls produce another selector. Its ordinary slot receives the twice-shifted global sample, hence zero.

Before the next call, the routine reads a word at address 20 plus the earlier additional output masked with `0x1fffffff` and shifted right 19 bits. The following dependency receives four times the latest selector.

### 4. Byte-selected writes and the immediate zero-valued call

Four calls precede extraction of the last word read by an unsigned right shift of 22 bits.

The routine reads an unsigned byte at address 1. This address follows from the previously established zero-valued double extraction. At 17 plus that byte’s value, it writes the low byte of the newly extracted word.

The next dependency receives an additional register output three times, followed by zero.

**Exactly one call with no explicit arguments follows. Immediately after it, the next dependency receives one explicit argument whose value is zero.** There is no intervening call sequence between these two calls. This is a call with a zero-valued argument, not a call with no arguments.

Only afterward do four further calls produce another selector. Before calling a dependency with that selector shifted left two bits, the routine reads an unsigned byte at four times the saved extraction plus 1 and writes zero at 17 plus the byte’s value.

Four more calls produce another result. Its value shifted left two bits is passed twice to a dependency. A subsequent word-producing call is followed by a call receiving that word, then one call with no explicit arguments.

### 5. Low-address clears and the first global-destination update

The apparent byte at address 1 is sampled again. Zero is written at 17 plus its unsigned value.

An additional register output is saved for later shifts. A dependency receives that value three times. The next call produces a result, which a following dependency receives together with the previously forwarded word shifted right 23 bits.

Four calls produce another selector. The apparent byte at address 1 is then sampled afresh and saved as a numerical base. Zero is written at that base plus 17. A dependency receives the selector shifted left two bits.

Four more calls produce a result whose value shifted left two bits is passed twice to a dependency. A subsequent word-producing call supplies the first explicit write to the apparent global destination word. That same result is then passed to another dependency.

After one further call, the routine reads a word at the saved byte-derived base plus 116 and writes its low byte at that base plus 1.

The repeated address-1 samples are distinct reads. Unknown dependencies may change their values.

### 6. Repeated word-to-byte field copies

A dependency receives the saved additional register output shifted left 11 bits. The next call exposes a 64-bit result. A following dependency receives its low word, its high word, and the word previously written to the global destination shifted right 23 bits.

Four calls produce a selector. Four times that selector forms a field base. A word is read at base plus 116, its low byte is written at base plus 1, and a dependency receives the base.

Four further calls produce a new selector. Four times this new selector is saved separately. Before the next call, however, the routine repeats the word read and low-byte store at the **previous** field base. This is a fresh read, not reuse of the earlier word.

The next dependency receives, in order:

- The saved additional register output shifted left 12 bits.
- Four times the new selector.
- That same scaled value again.
- The new selector shifted left 18 bits.

Two result-producing calls, each followed by a forwarding call, precede four more calls that produce another selector.

At four times that selector, the routine reads a full word at offset 116, saves the complete word, writes its low byte at offset 1, and calls a dependency with the field base.

### 7. Full-word propagation and further slot stores

Four calls produce a selector whose **wide slot** receives the complete saved field word.

The following dependency receives the earlier saved four-times-selector quantity shifted right two bits and then left 16 bits. This preserves the visible operation order; it does not assume the original selector’s upper bits survived scaling.

Two result-producing calls, each followed by a forwarding call, and four further calls produce another selector. Its ordinary slot receives the same full saved field word. A dependency receives four times this selector.

Four calls produce a result that is shifted left two bits. The shifted result is stored at address 20 plus the earlier saved four-times-selector quantity masked with `0xfffffffc`. A dependency receives the newly shifted result twice.

Another pair of result-producing and forwarding calls, followed by four calls, produces a selector whose ordinary slot receives zero. A dependency receives four times that selector.

Four further calls culminate in a 64-bit result. The routine shifts its low word left two bits and writes it to the ordinary slot selected by its high word. It then samples the apparent global source word again.

The next dependency receives the shifted low word, the high word, and the shifted low word twice more.

### 8. Global-derived word and final byte-selected updates

Four calls produce a selector whose ordinary slot receives the same shifted low word. The routine then reads a word at the **second global-source sample plus 20**, without multiplying that sample by four. A dependency receives four times the selector.

Four calls culminate in the fourth 64-bit result. Before forwarding its components, the routine:

- Reads an unsigned byte at 1 plus the saved full field word shifted right 23 bits.
- Writes the low byte of the global-derived word at 17 plus that byte’s value.

The next dependency receives the 64-bit result’s high word three times, followed by its low word shifted left two bits.

A result-producing call and its forwarding call precede four more calls. Before calling a dependency with the latest selector shifted left two bits, the routine reads an unsigned byte at 1 plus the saved low word masked with `0x3fffffff` and shifted right 21 bits. At 17 plus that byte’s value, it writes the low byte of the global-derived word shifted right 22 bits.

Four calls produce another result; that result shifted left two bits is passed twice to a dependency.

The remaining calls before terminal processing are, in order: one with no explicit arguments, another with no explicit arguments whose target calculation uses displacement 176, a call forwarding the latter result, and the final byte-producing call.

### 9. Terminal stores and address-zero call

The final byte-producing call also exposes an additional pointer output. The routine then performs, in order:

1. A byte store of zero through that pointer.
2. A read of the apparent three-byte source at address 9.
3. A write to the apparent global destination word, using the returned byte as bits 0–7 and the three-byte source as bits 8–31.
4. A halfword store of zero at the saved low word from phase 8 masked with `0x3fffffff` and shifted right 7 bits.
5. A call to address zero with three zero arguments.

The halfword address ranges from zero through `0x007fffff` and need not be aligned. The composite word’s bit arrangement is established; its physical byte layout depends on endianness.

If the address-zero call returns normally, the routine returns without an explicit result.

## Side effects

Explicit accesses include byte, halfword, and word stores; word reads; unsigned selector-byte reads; and a three-byte source read. They are not confined to a demonstrated allocation.

Notable state changes are:

- Two stores of the incoming payload.
- Multiple ordinary-slot stores and one wide-slot store.
- Repeated zero stores through low-address selectors.
- Repeated word-to-low-byte field copies.
- Two writes to the apparent global destination word.
- Terminal byte and halfword clears.

All accessed regions may alias unless an external contract proves otherwise. Global storage may overlap smaller symbols or computed destinations. The terminal pointer clear can therefore affect the subsequent three-byte source read.

No explicit allocation, release, reference-count update, locking, or ownership transfer is visible. Dependencies may perform such operations.

There is no explicit error-status handling or recovery branch. **No explicit rollback is visible in this routine.** This does not guarantee that earlier writes remain intact after failure: dependencies may modify or restore previously written storage before failing or not returning.

## Dependencies

| Opaque ID | Unresolved role |
|---|---|
| F-00000001 | Initial target derived from incoming link state; receives the explicit input. |
| F-00000002 | Calls with no explicit arguments shown, including calls exposing additional register outputs. |
| F-00000003 | Calls receiving one forwarded value, selector, scaled value, or zero. |
| F-00000004 | Calls receiving multiple values, repeated arguments, or extracted fields. |
| F-00000005 | Four call sites exposing consumed 64-bit results. |
| F-00000006 | Calls distinguished by less-common target displacements. |
| F-00000007 | Final byte-producing call with an additional pointer output. |
| F-00000008 | Address-zero target receiving three zero arguments. |

These classes overlap. Most visible target calculations use displacement 4094; other displacements include 3896, 4076, 3976, 4016, 3988, 3996, 3984, and 176.

No dependency’s memory effects, global preservation, ownership rules, hidden inputs, failure behavior, or termination are established. “No explicit arguments” describes the representation, not a complete calling convention.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Input preservation.** An explicit input of `0x12345678` reaches F-00000001 unchanged.

2. **Incoming-base access.** With incoming base `0x1000`, the initial byte store writes zero at `0x1009`; the following word read uses `0x1034`.

3. **Distinct slot layouts.** Selector 3 identifies ordinary-slot address 32 and wide-slot address 68.

4. **Extraction and argument order.** If the first global sample is `0xff800000` and the chained word is `0xffc00000`, their extractions are 511 and 1023. The phase-3 four-argument call receives the additional selector twice, then 511, then 1023. The later double extraction is zero.

5. **Correct phase-4 call ordering.** After the call receiving a repeated additional output and zero, the observable call order is one no-explicit-argument call, then the call receiving zero, then the four-call acquisition. A dependency effect between these calls must follow that order.

6. **Byte selection.** A selector byte of 255 selects destination 272; zero selects destination 17. No pointer or word is loaded from the selector location.

7. **Fresh retained-base read.** If dependencies change the word at the retained field base plus 116 between its two copies, the second byte store uses the changed word’s low byte.

8. **Full-word preservation.** A saved field word of `0x89abcdef` supplies byte `0xef` to the immediate field store, but supplies the entire word to later ordinary and wide slots.

9. **Terminal aliasing.** If the terminal clear pointer overlaps the three-byte source, the subsequent source read observes memory after that clear, subject to the environment’s access semantics.

10. **Failure and restoration.** A dependency that restores an earlier destination before failing does not contradict this contract. Conversely, the routine itself shows no explicit rollback operation.

11. **Address-zero failure.** If the final call traps, the terminal stores were reached before that attempted call. This does not assert their indefinite persistence against dependency or external effects.

## Uncertainties

The following remain **documented_only**:

- Application purpose and structural meaning of the address patterns.
- Complete calling convention and interpretation of additional register outputs.
- Actual indirect target identities and contracts.
- Validity, alignment, lifetime, and ownership of accessed memory.
- Overflow behavior for signed scaling, signed left shifts, and target-address calculations.
- Endianness and physical layout of the final composite word.
- Aliasing among apparent globals, low-address storage, and computed destinations.
- Concurrency, atomicity, synchronization, and reentrancy.
- Whether address zero is callable.
- Whether all represented target and register relationships faithfully describe execution.

Unknown callees are not assumed to preserve globals, leave memory unchanged, or retain or release ownership.

## Evidence

**Established static evidence:** the supplied representation shows the explicit input forwarding, ordered call sites, access widths, masks, shifts, address calculations, repeated global samples, terminal stores, and address-zero call described above. It shows one no-explicit-argument call immediately before the phase-4 call receiving zero. It shows no explicit branch, loop, bounds check, error return, or rollback operation.

**Inferred intent:** scaling and bit extraction may serve encoded-reference or packed-state processing. No behavioral obligation depends on that interpretation.

**Unknown:** dependency semantics, runtime memory validity, unresolved arithmetic behavior, and normal completion.

No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence, not observed execution results.
