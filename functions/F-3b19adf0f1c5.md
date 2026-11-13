<!-- Trusted provenance: F-3b19adf0f1c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b19adf0f1c5

## Purpose

The supplied static representation describes a sequence of indirect calls interleaved with byte-sized and 32-bit memory writes, several memory reads, and address calculations derived from inputs, ambient register state, and call results.

No application-level purpose is established. In particular, the representation does not establish that these operations initialize an object, manipulate a particular device, allocate storage, or implement a recognized protocol.

The represented path ends at an invalid-instruction boundary rather than a normal return. An explicit warning also identifies unreliable instruction recovery. Consequently, this contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Conditional interpretation:** the behavior those operations describe if the recovered path, widths, and calling conventions are accurate.
- **documented_only unknowns:** behavior that cannot be established without additional evidence.

The detailed behavior below is conditional on execution reaching each operation and every preceding dependency returning.

## Interface

### Explicit inputs

| Input label | Represented width | Role established by the representation |
|---|---:|---|
| Input A | 32 bits, unsigned | Initial stored value; source of logical right shifts; dependency argument. |
| Input B | 32 bits | No explicit use in the represented body. |
| Input C | 32 bits | Source of both byte writes and one full-word write. |
| Input D | 32 bits, represented as signed | Source of a left-shifted byte write. |

“No explicit use” does not establish that Input B is irrelevant to the actual machine execution. Recovered indirect-call signatures and implicit argument registers are incomplete evidence.

### Ambient inputs

The representation also consumes three entry-state quantities not supplied through the explicit interface:

- **Ambient scale:** a register-derived quantity used to form doubled and quadrupled address bases.
- **Ambient address:** a register-derived address used for the first byte write.
- **Ambient call base:** a register-derived quantity used to select the first indirect-call target.

Their source, validity, preservation requirements, and association with any object are unknown. A caller cannot establish a safe invocation from the four explicit inputs alone.

A later dependency argument uses an additional register result whose production is not explicitly represented as an ordinary return value. Its meaning and calling-convention status are also unknown.

### Output and termination

The recovered signature declares no return value. The represented path does not reach a normal return; it ends at an invalid-instruction stop.

There is no established success value, error value, exception contract, or recoverable failure mode. Memory writes and dependency effects preceding an interruption are not shown being rolled back.

### Width and notation conventions

The following labels describe values for this contract; they are not source identifiers.

- **Low byte** means the least significant eight bits.
- **Low word** and **high word** mean the low and high 32-bit portions of a represented 64-bit result.
- Right shifts of unsigned 32-bit quantities are logical.
- A byte read produces an unsigned value from 0 through 255.
- Address offsets are byte offsets.
- “Word write” means a 32-bit write, not a machine-dependent native-width write.

For compactly describing address arithmetic, the behavior section uses the intended 32-bit bit-vector interpretation: left shifts and multiplication retain their low 32 bits. This is a **conditional interpretation**, not proof of language-defined signed arithmetic. Several recovered expressions have signed types; overflow and left shifts of negative values cannot safely be assigned portable language semantics.

Word byte order, alignment requirements, address-space mapping, and fault behavior are undocumented.

## Behavior

### Ordering rule

The numbered stages follow the represented execution order. Dependencies within each stage are listed in invocation order. No-argument calls mean **no arguments explicitly represented**, not proof that other machine registers are unobserved.

Dependency IDs below identify call occurrences for this contract. They do not assert distinct underlying functions.

### 1. Initial stores and first wide result

Form a **doubled base** from twice Ambient scale.

1. Write its low byte at Ambient address plus 1.
2. Write Input A as a word at doubled base plus `0x34`.
3. Invoke F-a001 with Input A.
4. Invoke F-a002 with no represented arguments, retaining its 32-bit result.
5. Derive the **quarter-input base** by logically shifting Input A right by 2.
6. Invoke F-a003 with the result of F-a002.
7. Invoke F-a004, F-a005, and F-a006, each with no represented arguments.
8. Invoke F-a007 with no represented arguments, retaining a represented 64-bit result.

F-a001’s target depends on Ambient call base. No validation of that target is shown.

### 2. Replacement store and quadrupled base

1. Write the low byte of doubled base at quarter-input base plus 1.
2. Write the high word of F-a007’s result at doubled base plus `0x34`.
3. Invoke F-b001 with the low word of that result shifted left by 2.
4. Invoke F-b002, F-b003, and F-b004 without represented arguments.
5. Invoke F-b005 without represented arguments, retaining its 32-bit result.
6. Form the **quadrupled base** from four times Ambient scale.
7. Write its low byte at address `(Input A logically shifted right by 22) + 0x11`.
8. Write Input C’s low byte at quadrupled base plus `0x0d`.
9. Invoke F-b006 with two identical arguments, each the result of F-b005 shifted left by 2.

The word write in this stage addresses the same location as the initial Input A word write. Dependencies occur between the writes and may observe or modify that location.

### 3. Repeated low-byte setup

1. Invoke F-c001 without represented arguments, retaining an unsigned 32-bit result.
2. Invoke F-c002 with that result and its logical right shift by 3.
3. Invoke F-c003, F-c004, and F-c005 without represented arguments.
4. Invoke F-c006 without represented arguments, retaining a 32-bit result.
5. Write the low byte of quadrupled base at absolute address `0x11`.
6. Again write Input C’s low byte at quadrupled base plus `0x0d`.
7. Invoke F-c007 with the result of F-c006 shifted left by 2.
8. Invoke F-c008, F-c009, and F-c010 without represented arguments.
9. Invoke F-c011 without represented arguments, retaining its unsigned 32-bit result as the **scaling result**.

The repeated write cannot be removed from the observable contract merely because the same value was written earlier.

### 4. Derived region and field writes

1. Write Input C as a word at quarter-input base plus `0x14`.
2. Write the low byte of twice quarter-input base at quarter-input base plus `0x0d`.
3. Derive three quantities from the scaling result:
   - **Region base:** scaling result multiplied by 32.
   - **Region payload:** scaling result shifted left by 7.
   - **Selector:** scaling result masked with `0x07ffffff`, then logically shifted right by 17.
4. Invoke F-d001 and F-d002 without represented arguments.
5. Invoke F-d003 without represented arguments, retaining its 32-bit result as the **region index**.
6. Write region payload as a word at region base plus `0x14`.
7. Write the low byte of region index shifted left by 2 at region base plus `0x0d`.
8. Invoke F-d004 with region index shifted left by 2.
9. Invoke F-d005, F-d006, F-d007, and F-d008 without represented arguments.

Selector ranges from 0 through 1023 before any address-space interpretation.

### 5. Selector write and first byte-indexed destination

1. Write the low byte of Input D shifted left by 1 at Selector plus 7.
2. Derive a **wide index** by shifting region index left by 21.
3. Read one unsigned byte at region base plus 1.
4. Write Selector’s low byte at the address equal to that byte plus `0x11`.
5. Invoke F-e001 with three identical arguments, each equal to wide index.
6. Invoke F-e002 without represented arguments, retaining its unsigned 32-bit result as the **retained word**.
7. Invoke F-e003 with retained word.
8. Invoke F-e004, F-e005, and F-e006 without represented arguments.
9. Invoke F-e007 without represented arguments, retaining a 32-bit result.

The byte-indexed write targets the inclusive address range `0x11` through `0x110`.

### 6. Clear through a fresh byte read

1. Read one unsigned byte at region base plus 1, retaining it as the **saved byte**.
2. Write zero at saved byte plus `0x11`.
3. Invoke F-f001 with the result of F-e007 shifted left by 2.
4. Invoke F-f002, F-f003, F-f004, and F-f005 without represented arguments.
5. Form the **byte-derived base** as twice saved byte.
6. Retain the low byte of retained word logically shifted right by 3 as the **retained byte**.
7. Write retained byte at region base plus `0x0d`.
8. Write saved byte at absolute address `0x0d`.
9. Invoke F-f006 with byte-derived base, `0x98`, and `0x260`.

The new byte read is not interchangeable with the read in stage 5. Intervening dependencies may have changed the source byte.

### 7. Additional register result and absolute-byte replacement

1. Invoke F-g001 without represented arguments.
2. Invoke F-g002 with an additional register result logically shifted right by 3.
3. Invoke F-g003, F-g004, and F-g005 without represented arguments.
4. Invoke F-g006 without represented arguments, retaining a 32-bit result.
5. Invoke F-g007 with that result shifted left by 2, retaining its 32-bit result.
6. Write retained byte at byte-derived base plus `0x0d`.
7. Write `0xc0` at absolute address `0x0d`.
8. Invoke F-g008 with F-g007’s result shifted left by 2.
9. Invoke F-g009, F-g010, and F-g011 without represented arguments.
10. Invoke F-g012 without represented arguments, retaining its 32-bit result as the **link payload**.

The additional register result’s provenance is documented_only. It must not be assumed equal to F-g001’s ordinary return value.

### 8. Word-linked state update

1. Write link payload shifted left by 2 as a word at byte-derived base plus `0x24`.
2. Read a word at byte-derived base plus `0x74`, retaining it as the **linked base**.
3. Derive the **expanded payload** by shifting link payload left by 5.
4. Invoke F-h001 with four arguments, in order:
   - linked base;
   - linked base shifted left by 1;
   - expanded payload;
   - linked base masked with `0x7fffffff`, then logically shifted right by 2.
5. Invoke F-h002, F-h003, F-h004, and F-h005 without represented arguments.
6. Write expanded payload as a word at linked base plus `0x24`.
7. Read a word at linked base plus `0x74` and pass that value to F-h006.
8. Invoke F-h007, F-h008, and F-h009 without represented arguments.
9. Invoke F-h010 without represented arguments, retaining a represented 64-bit result.

The read at linked base plus `0x74` occurs after the preceding calls and word write. It is not an entry-state snapshot.

### 9. Wide-result-derived writes and calls

1. Derive the **retained offset** by shifting F-h010’s low word left by 2.
2. Write retained word logically shifted right by 3 as a word at four times F-h010’s high word plus `0x14`.
3. Invoke F-i001 with retained offset three times, followed by F-h010’s high word logically shifted right by 3.
4. Invoke F-i002 without represented arguments, retaining another represented 64-bit result.
5. Invoke F-i003 with that result’s low word, high word, low word again, and high word logically shifted right by 3.
6. Invoke F-i004, F-i005, and F-i006 without represented arguments.
7. Invoke F-i007 without represented arguments, retaining a 32-bit result.
8. Write retained offset as a word at four times F-i007’s result plus `0x14`.
9. Invoke F-i008 with four times F-i007’s result.
10. Invoke F-i009, F-i010, and F-i011 without represented arguments.
11. Invoke F-i012 without represented arguments, retaining its unsigned 32-bit result as the **terminal seed**.

The word written in step 2 contains the full shifted retained word, not merely retained byte.

### 10. Repeated indirect byte updates and terminal boundary

1. Form the **terminal offset** by shifting terminal seed left by 2.
2. Read a byte at linked base plus 1. At that byte’s value plus `0x11`, write terminal offset’s low byte.
3. Invoke F-j001 and F-j002 without represented arguments.
4. Invoke F-j003 without represented arguments, retaining a 32-bit result.
5. Read a byte again at linked base plus 1. At that newly read byte’s value plus `0x11`, write the low byte of terminal seed masked with `0x3fffffff` and logically shifted right by 20.
6. Invoke F-j004 with F-j003’s result shifted left by 2.
7. Invoke F-j005, F-j006, and F-j007 without represented arguments.
8. Invoke F-j008 without represented arguments, retaining a 32-bit result.
9. Read a byte at terminal offset multiplied by `0x400`, plus 1. Write zero at that byte’s value plus `0x11`.
10. Invoke F-j009 with terminal offset shifted left by 3 twice, terminal offset, and F-j008’s result shifted left by 2.
11. Invoke F-j010, F-j011, and F-j012 without represented arguments.
12. Invoke F-j013 without represented arguments, retaining a 32-bit result.
13. Repeat the byte read and zero write described in step 9.
14. Invoke F-j014 with F-j013’s result shifted left by 2.
15. Invoke F-j015, F-j016, F-j017, and F-j018 without represented arguments.
16. Reach the represented invalid-instruction boundary.

Both repeated byte reads remain observable operations. Their resulting destinations need not match.

## Side effects

The directly represented effects include:

- Byte writes to computed addresses and absolute addresses `0x0d` and `0x11`.
- Word writes at offsets `0x14`, `0x24`, and `0x34` from derived bases.
- Byte reads used as addresses for further writes.
- Word reads at offset `0x74` from two derived bases.
- Numerous indirect calls with unknown effects.
- Abnormal termination at the represented endpoint.

No allocation, release, ownership transfer, synchronization, or rollback is explicitly established. Passing a value to a dependency does not establish whether it is an address, whether it denotes owned storage, or whether the dependency retains or releases it.

Aliasing is unrestricted by the supplied evidence. A byte destination may overlap a later word field, an absolute address, or memory used by a dependency. Therefore, final memory contents cannot generally be reduced to a simple list of independent field assignments.

There are no represented null checks, bounds checks, alignment checks, or dependency-failure checks. Safe execution conditionally requires valid access to every address actually reached and callable targets for every invocation. Whether low addresses identify ordinary storage, special-purpose mappings, or inaccessible memory is unknown.

## Dependencies

F-a001 through F-j018, as individually enumerated above, are opaque call-occurrence IDs. They are not additional established API names. The supplied dependency map establishes only the identity of F-3b19adf0f1c5 itself; it does not provide trusted identities for the indirect targets.

For every dependency:

- Only explicitly represented arguments and consumed results are described.
- Unused results are not evidence of a void return type.
- An absent argument list is not evidence that ambient registers are irrelevant.
- Global state preservation, memory preservation, ownership behavior, termination, and error behavior remain unknown.
- A dependency that faults, does not return, or changes relevant memory can prevent or alter later observable effects.

The three represented 64-bit results may reflect paired machine-register results rather than a verified high-level 64-bit return convention.

## Acceptance examples

These examples are conditional arithmetic and ordering checks, not runtime observations.

1. **Initial-byte truncation.** If Ambient scale is `0x123`, doubled base is `0x246`, and the first byte write stores `0x46`. The initial word write stores all 32 bits of Input A.

2. **Replacement ordering.** If F-a007 returns high word `0x12345678`, stage 2 writes that word at the same address that initially received Input A. A dependency between those writes may observe the initial value; final-state equality alone is insufficient acceptance evidence.

3. **Unsigned selector extraction.** If scaling result is `0xffffffff`, Selector is `0x3ff`. Its stage-5 byte destination is `0x406`, and the stored byte derived from Input D depends only on that input’s low seven bits under the bit-vector interpretation.

4. **Byte-indexed destination.** A source byte of `0xff` selects destination `0x110`, not an address based on sign-extending the byte.

5. **Fresh reads across calls.** If region base plus 1 contains `0x02` during stage 5 and `0x09` during stage 6, the first byte-indexed write targets `0x13`, while the subsequent zero write targets `0x1a`. Acceptance must not require these destinations to coincide.

6. **Retained byte versus retained word.** For retained word `0x12345678`, its logical right shift by 3 is `0x02468acf`. The retained byte is `0xcf`; stage 9’s word write instead stores `0x02468acf`.

7. **Zero is not skipped.** A linked base of zero still leads, if execution continues, to a word write at `0x24` and a word read at `0x74`. No represented null guard suppresses them.

8. **Terminal behavior.** Reaching the represented end does not establish successful return. Acceptance of this static contract requires retaining the invalid-instruction boundary as an uncertainty, not substituting normal completion.

## Uncertainties

The following remain **documented_only**:

- Accuracy and completeness of the recovered instruction path.
- Whether the invalid-instruction boundary is executable behavior or a recovery artifact.
- Actual calling convention, entry-state requirements, and implicit arguments.
- Production of the additional register result used in stage 7.
- Concrete identities and contracts of indirect dependencies.
- Address width, signed-arithmetic semantics, alignment, and byte order.
- Memory mapping and meaning of absolute low addresses.
- Ownership, lifetime, aliasing restrictions, concurrency, and device-memory semantics.
- Whether any dependency returns normally or reports errors through unrepresented state.

No inferred application intent is promoted to a requirement.

## Evidence

The sole evidence is the supplied static representation: its declared input widths, explicit memory operations, arithmetic expressions, indirect-call ordering, consumed return fragments, and invalid-instruction warnings.

No runtime tests have been performed. No dependency implementations, machine instructions, external interface definitions, or independent memory-layout documentation were available. All acceptance examples are conditional deductions from that static evidence.
