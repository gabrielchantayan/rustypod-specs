<!-- Trusted provenance: F-5a6bb19fe33d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5a6bb19fe33d

## Purpose

F-5a6bb19fe33d performs an extended sequence of unresolved indirect calls, transforms their results and an explicit integer input, updates memory through fixed and computed addresses, and reaches a final indirect transfer with four explicit arguments.

The supplied static representation establishes an effectful operation, not an application-level purpose. It does not establish formatting, serialization, object construction, device configuration, or a particular data-structure invariant.

This contract distinguishes:

- **Established static evidence:** operations, widths, dependencies, and ordering represented in the supplied material.
- **Inferred intent:** interpretations compatible with those operations but not established by them.
- **documented_only:** unresolved behavior requiring dependency, platform, or executable evidence.

All execution descriptions are conditional on earlier operations completing. No runtime tests have been performed.

## Interface

### Explicit interface

| Property | Contract |
|---|---|
| Argument | One signed 32-bit integer |
| Direct interpretation | Arithmetic input; no explicit dereference through this argument |
| Declared result | No returned value |
| Observable outputs | Dependency interactions, memory effects, and the final indirect transfer |
| Validation | No explicit input-range, address, lifetime, or dependency-status checks |

The explicit input undergoes two successive left shifts by two positions before reaching an early two-argument dependency boundary. Under the conditional 32-bit bit-vector model defined below, this produces the input multiplied by 16 modulo \(2^{32}\). It does not establish that the input represents a count or offset.

### Implicit inputs and preconditions

Execution also depends on incoming control context, ordinary and secondary dependency-result channels, existing memory, and an unresolved final destination. The explicit argument alone cannot determine behavior.

Successful execution requires every reached destination to support the represented indirect transfer and every reached memory location to support its access width. These are environmental requirements, not validations performed by the function.

Computed addresses include scaled integers and zero-extended bytes used directly as numerical bases. Fixed accesses include very low addresses. Ordinary heap allocation and conventional process mappings are therefore insufficient assumptions.

Address width, integer-to-address conversion, byte order, alignment requirements, and access-fault behavior are **documented_only**.

## Behavior

### Execution structure

The represented body contains:

1. A long sequence of indirect calls interspersed with scalar transformations.
2. A direct-memory phase with reads, mixed-width writes, and repeated updates.
3. A final indirect transfer.

No explicit input-controlled branch or loop appears in the body. Repetition is represented as repeated operations, not traversal over a known collection. Dependencies may themselves branch, modify state, fail, or never return.

The ordering below describes the static representation. It is not an independently verified hardware memory-ordering or atomicity guarantee.

### Indirect-call phase

The first indirect destination derives from incoming return-control context plus displacement 0xFFE. Subsequent destinations derive from embedded code-location values using the same displacement. Their executable identities are unresolved.

The opening interaction includes:

- A result from the first dependency shifted left eight positions and supplied to another dependency.
- Calls with no represented explicit arguments.
- Calls receiving explicit zeros.
- A later two-argument call receiving a dependency result shifted left eight positions and the explicit input shifted twice by two positions.
- Subsequent reuse of the shifted dependency result after additional, separately applied shifts.

A missing explicit argument list does not imply absence of implicit inputs. An unused ordinary result does not imply absence of effects.

Later interactions distinguish ordinary 32-bit results, apparent 64-bit results, and secondary result channels. Apparent 64-bit results are repeatedly separated into low and high 32-bit portions. The low portion may be shifted independently while the high portion is passed unchanged, transformed separately, or retained across intervening calls.

The representation also extracts portions above bit 31 using unsigned 64-bit right shifts by 34, 36, and 44 positions. These operations do not establish whether the original value is an application-level integer or a paired calling-convention result.

Masks and shifts select fields of several widths, including 24, 22, 20, 19, 18, 17, 14, 13, 12, and 11 bits. Represented transformations include selecting bits 4–13, selecting bits 10–19, shifting masked low-22-bit fields by 3, 7, or 8, scaling a low-19-bit field by four, and positioning a selected bit at bit 16 or bit 31. These are transformations, not established bounds checks.

Some intermediates are explicitly zeroed before further shifts or masks. Their derived arguments remain zero under bit-vector arithmetic. The surrounding calls remain effectful boundaries with unresolved semantics.

### Arithmetic and conversion rules

The following rules apply **only under a conditional 32-bit bit-vector interpretation**, not as an unconditional portable-language guarantee:

- A 32-bit left shift discards bits above bit 31.
- An unsigned right shift inserts zeros.
- A byte store retains eight bits; a halfword store retains 16; a word store retains 32.
- Represented 32-bit multiplication truncates to 32 bits **before** conversion to an address.
- A wider address model does not widen that multiplication intermediate.
- Address width and integer-to-address conversion remain separate platform questions.

Successive shifts retain their individual widths. A 32-bit value shifted left 25 positions and then left another 25 positions becomes zero in this model; it does not retain a 50-bit intermediate.

Several operations instead use signed 32-bit values. Signed overflow, signed left-shift validity, and out-of-range conversions are not resolved by imposing bit-vector semantics.

A late operation applies right shift to a **signed 32-bit value**, with its count taken from the low eight bits of another quantity. Two independent uncertainties remain:

- For a negative operand and a count below 32, sign propagation is not established without a platform contract.
- For counts of 32 or greater, effective shift behavior is not established.

This contract does not classify that operation unconditionally as an arithmetic right shift.

### Direct-memory phase: terminology

The following descriptive labels identify captured numerical quantities, not recovered objects or implementation identifiers:

| Label | Meaning |
|---|---|
| Seed base | A dependency result multiplied by \(2^{25}\) using represented 32-bit arithmetic |
| Carrier | An earlier dependency result shifted left eight positions |
| Compact value | A result shifted left eight positions, masked to 15 bits, then unsigned-shifted right six |
| Compact base | Twice the compact value |
| Initial byte-derived base | A zero-extended byte read from fixed address 0x05 |
| Captured link value | A word read from the initial byte-derived base at offset 0x54 |
| Quadrupled link | Four times the captured link value |
| Adjusted link | The quadrupled link minus 0x96 |
| Repeated data base | Four times the adjusted link |
| Repeated owner base | A word captured from fixed address 0x087FBBBC |

All numerical scaling remains subject to the arithmetic and address-conversion qualifications above. Labels do not establish disjoint addresses or valid lifetimes.

### Entry writes and initial fixed-memory accesses

The direct-memory phase opens with a word store at offset 0x14 through the high portion of the last apparent paired result. It then writes a byte at offset 9 through an earlier derived base, stores the seed base at offset 0x54 through the last low portion scaled by 256, and overwrites the same offset-9 byte with another value.

A zero word is then stored at seed-base offset 0x14 before a byte is read from seed-base offset 5. After clearing compact-base offset 0x14, the function stores that zero-extended byte at seed-base offset 0x34 and rereads seed-base offset 0x14. Another clear of compact-base offset 0x14 and two byte writes at compact-base offset 0x15 follow.

The seed-base reread is not a promise to obtain zero. Earlier accesses may alias it, and the memory environment is unresolved.

The first fixed-memory boundary is ordered as follows:

- The byte at **0x05 is read first**, supplying the initial byte-derived base.
- The low byte of the compact base is then written at **0x01**.
- Subsequent accesses use the already captured byte as a numerical base, including an offset-0x15 byte write and an offset-0x54 word read.

Thus the read at 0x05 is not a later observation following the write at 0x01.

### Link-derived updates and the next fixed writes

The function writes and overwrites seed-base offset 0x11, updates word destinations reached through previously read values, reads the initial byte-derived base at offset 0x54, and replaces that word. The word captured before replacement becomes the captured link value.

Further accesses use seed-base and compact-base offset 0x74 words as addresses. They include self-referential-looking word writes, repeated writes at a derived offset 0x24, repeated offset-9 byte stores, and replacements of compact-base offset 0x14.

Within this sequence:

- A **32-bit write at fixed address 0x54** stores the captured link value shifted left one position.
- After intervening derived writes, a **byte write at fixed address 0x15** stores the low byte of the compact base.

The fixed word write at 0x54 therefore precedes the fixed byte write at 0x15. These fixed destinations are not substitutes for the similarly numbered offsets through computed bases.

The remaining link-derived work uses word offsets 4, 0x14, 0x44, 0x54, 0x64, and 0x74, with byte stores at offsets including 9, 0x0D, and 0x15. Some byte reads at offset 5 supply the numerical base for an offset-0x74 word read. This is not established array indexing.

A byte at offset 5 is read more than once with intervening writes. These observations cannot generally be collapsed into one captured value.

### Constant writes and entry to the repeated region

For the last word obtained through a byte-derived offset-0x74 read, the function forms doubled and quadrupled quantities. The doubled quantity is called the **late link base** below; the quadrupled quantity is the **repeated payload**.

Before the repeated region, the function writes 0x3A at three destinations and widths: a byte through the quadrupled quantity, a halfword through the corresponding eightfold quantity, and a word at offset 0x20 through the quadrupled quantity.

The adjusted link is formed by subtracting 0x96 without an explicit underflow check. At repeated-data-base offset 0x20, the represented word-write sequence is:

- 0x20;
- 0x2F;
- the zero-extended byte read from repeated-data-base offset 4;
- 0x20 again.

The byte read belongs between the 0x2F write and its byte-derived replacement.

**After the final offset-0x20 write**, the function reads the word at **0x087FBBBC**, capturing the repeated owner base. **After that read**, it writes the word 0x20 at fixed address **0x60**. The base capture must not be moved before the preceding writes or after the 0x60 write without preservation assumptions.

The repeated region uses ordered four-write groups:

1. A word store of the repeated payload at a scaled destination’s offset 0x54.
2. A byte store of the repeated data base’s low byte at repeated-owner-base offset 0x19.
3. A byte store of the repeated owner base’s low byte at repeated-data-base offset 0x11.
4. A word store of a scaled numerical value at repeated-owner-base offset 0x34.

The first group uses eight times the captured link value for its scaled destination and final stored value. Subsequent groups use eight times the adjusted link.

These distinctions survive even if some resulting bit patterns coincide. Repetition does not establish traversal, and replacement by final-state-only writes is not justified for aliased or externally observed memory. Exact multiplicity is retained as **documented_only** rather than assigned an unverified count.

### Post-repetition fixed-memory ordering

The post-repetition phase begins with three ordered accesses:

| Order | Access |
|---|---|
| First | Read a 32-bit word at fixed address **0xA9** |
| Second | Read a 32-bit word at repeated-payload offset **0x74** |
| Third | Write a byte at fixed address **0x3A**, derived from eight times the adjusted link plus 0x30, truncated to eight bits |

The captured 0xA9 word is retained for later use. The offset-0x74 word supplies later byte updates; it is not a substitute for the fixed-memory word.

Next, words are captured from late-link-base offset 0x14 and repeated-data-base offset 4. Several derived writes follow, including the byte constant 0x6C and a self-address-like word at late-link-base offset 0x54.

The accesses involving **0x67** are distinct memory events:

- A byte write at 0x67 stores the low byte of the captured 0xA9 word.
- A word is then read from late-link-base offset 0x44.
- A subsequent **read of 0x67** supplies the byte written at fixed address **0x21**.
- After intervening derived writes, a later **read of 0x67** supplies a byte store at repeated-data-base offset 9.
- After more derived writes and a word read at repeated-data-base offset 0x54, another **read of 0x67** supplies a second byte store at that same derived offset 9.

The three reads of 0x67 are not interchangeable with the originally written byte. Intervening accesses may overlap 0x67, and preservation or device-memory behavior is not established.

### Final derived updates and transfer

Between the 0x67 observations, the function performs the signed-operand right shift described above and stores its low byte at late-link-base offset 1.

It also captures a word from repeated-data-base offset 0x40, writes the earlier offset-4 capture at repeated-owner-base offset 0x74, and then writes the offset-0x40 capture as a word at repeated-data-base offset 0x10. That word can overlap preceding byte writes at offsets 0x10 and 0x11.

After the second derived offset-9 store supplied by 0x67, further updates include:

- Byte writes derived from the captured repeated-payload offset-0x74 word.
- Restoration of late-link-base offset 0x54 from its earlier offset-0x14 capture.
- A new word read at repeated-data-base offset 0x14.
- A later overwrite of repeated-data-base offset 9.
- A halfword store of 0x65 at twice a value read from late-link-base offset 4.
- A word store of decimal 100 at offset 0x54 through another loaded base.
- A final word store at twice another loaded numerical value.

The final indirect transfer receives, in order:

1. 0x72.
2. A word loaded at offset 0x64 through the value read from late-link-base offset 4.
3. The earlier captured word from late-link-base offset 0x14.
4. A word loaded at offset 0x74 through the value read from late-link-base offset 4.

A return is represented afterward. The destination, whether it returns, and whether the transfer corresponds to an ordinary call remain **documented_only**.

## Side effects

### Widths, layout, and aliasing

Represented accesses include:

| Width | Access evidence |
|---|---|
| 8-bit | Offsets 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19; fixed and scaled destinations |
| 16-bit | Selected doubled and eightfold numerical destinations |
| 32-bit | Offsets 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74; fixed cells |

These are observed access locations, not a recovered structure layout. A byte containing an address-like value’s low eight bits is not established as a complete pointer.

Fixed cells and computed destinations share a potentially overlapping address space. Neither different labels nor different widths establish independence. Later reads use memory at their represented observation point, not an initial snapshot.

### Ownership, failure, and preservation

No allocation, release, ownership transfer, reference-count protocol, synchronization, rollback, or transaction boundary is explicitly established.

Dependencies are not assumed to preserve globals, retain address validity, or own or release memory. Earlier captured values receive no explicit lifetime validation.

No structured error result is represented. Unsupported accesses or unsuccessful transfers have environment-dependent consequences. Earlier effects are not promised to be undone.

## Dependencies

Only F-5a6bb19fe33d is resolved by the supplied identity mapping.

| Opaque dependency ID | Role | Status |
|---|---|---|
| F-U001 | Initial indirect destination derived from incoming control context | documented_only |
| F-U002 | Ordered family of subsequent indirect destinations | documented_only |
| F-U003 | Final destination receiving four explicit arguments | documented_only |

These IDs identify unresolved roles, not proven distinct executable functions. Similar call forms do not establish destination equality. No dependency is assumed pure, state-preserving, lifetime-preserving, or guaranteed to return.

## Acceptance examples

These are conditional review criteria, not executed tests.

1. **Input transformation.** Under 32-bit bit-vector semantics, input 1 supplies 16 in the early input-derived argument position. Inputs differing only in their highest four bits supply the same value there, without establishing whole-function equivalence.

2. **Paired-result separation.** For high portion 0x11223344 and low portion 0x00ABCDEF, a boundary using the shifted low portion and unchanged high portion exposes 0xABCDEF00 and 0x11223344 separately.

3. **Multiplication before address conversion.** Under the conditional model, multiplying 0x40000001 by four produces the 32-bit numerical value 4 before address conversion. A wider address space must not turn this into an untruncated 0x100000004 intermediate. The resulting address still requires a conversion contract.

4. **Early fixed order.** A trace preserving the represented ordering observes the byte read at 0x05 before the byte write at 0x01, and the word write at 0x54 before the byte write at 0x15.

5. **Repeated-base capture.** The word read supplying the repeated owner base occurs after the final repeated-data-base offset-0x20 write and before the word write at 0x60.

6. **Post-region order.** The 0xA9 word read precedes the repeated-payload offset-0x74 word read, which precedes the byte write at 0x3A.

7. **Fresh 0x67 observations.** If an intervening derived write changes 0x67, a later read supplying an offset-9 store must observe that later state. Reusing the originally captured byte is acceptable only under an explicit preservation assumption.

8. **Signed right shift.** For a negative operand, this contract alone supplies no sign-propagation result, even when the count is below 32. Counts of 32 or greater require a separate rule.

9. **Partial execution.** A non-returning F-U001 or F-U002 destination prevents any requirement to perform later writes. An unsupported reached access carries no rollback promise.

10. **Final boundary.** Reaching F-U003 exposes 0x72 as the first explicit argument; it does not establish normal completion.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid input domain.
- Dependency identities, full interfaces, secondary result channels, and preservation rules.
- Whether apparent paired results are genuine 64-bit returns.
- Signed overflow, signed left shifts, and out-of-range conversions.
- Negative signed right-shift behavior, independently of excessive-count behavior.
- Address width and integer-to-address conversion after 32-bit arithmetic.
- Byte order, alignment, low-address mappings, aliasing, and external memory observation.
- Ownership, lifetimes, and dependency-induced invalidation.
- Exact repeated-region multiplicity and executable fidelity of suspicious repetitions or low-address accesses.
- Final destination and return behavior.

Link-like updates suggest structured-state manipulation; character-like constants suggest possible text-related effects. Both are inferred intent only.

## Evidence

This contract uses the supplied static representation and identity mapping. Prior prose, comments, warnings, and reviewer interpretations are not independent behavioral authority.

Established evidence consists of represented inputs, widths, transformations, call boundaries, memory-access ordering, fixed-memory observations, repeated writes, and final arguments. Arithmetic examples explicitly state their conditional model.

No dependency inspection, executable instruction verification, implementation modification, or runtime testing has been performed. Unresolved platform and dependency behavior is preserved rather than replaced with assumed semantics.
