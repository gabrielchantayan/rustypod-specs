<!-- Trusted provenance: F-1d0a81f00daa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1d0a81f00daa

## Purpose

F-1d0a81f00daa performs an ordered sequence of unresolved calls, calculated memory accesses, fixed-location storage accesses, and a terminal indirect transfer. Its declared interface has three 32-bit inputs and no scalar return value.

The application-level purpose is **documented_only**. The supplied representation does not establish formatting, serialization, object construction, device control, or collection manipulation. Constants resembling characters do not establish a text-output contract.

This specification distinguishes:

- **Established static evidence:** represented operations, explicit access widths, ordering, constants, and value dependencies.
- **Inferred execution model:** interpretations requiring assumptions about machine arithmetic, address calculation, or calling conventions.
- **documented_only:** behavior not established by the available representation.

All behavior is conditional on execution reaching the relevant operation. Successful completion is not established for arbitrary inputs or memory states.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No direct consumption appears in the body. |
| Second | 32 bits | No direct consumption appears in the body. |
| Third | Unsigned 32 bits | Supplies an early extraction of bits 10–19 inclusive. |

The extracted field ranges from 0 through 1023. Bits 0–9 and 20–31 do not participate in that extraction.

Lack of direct references to the first two inputs does not prove whole-function independence from them. Unresolved call interfaces and implicit machine state prevent that conclusion.

### Implicit inputs and output

Execution also depends on incoming control state, dependency results, additional post-call register values, existing memory contents, fixed storage, and an unresolved terminal destination.

There is no explicit output-buffer parameter or declared scalar result. Observable output consists of calls, memory accesses, state changes, and the final control transfer. A represented return follows that transfer; whether it is reached is **documented_only**.

### Preconditions

Normal progress through a reached stage requires:

- Executable call destinations and a compatible actual calling convention.
- Readable or writable storage for every reached access, with suitable alignment.
- Valid address calculations, including scaled values and very low addresses.
- Valid production and interpretation of post-call values.

No visible bounds check, null check, argument rejection, or address validation establishes these conditions. Inputs are not identified as lengths, handles, or owned pointers.

## Behavior

### Ordering and arithmetic

The represented main sequence has no explicit input-controlled loop or conditional branch. It consists of a long call phase, a direct-memory phase, and a terminal indirect transfer. Dependencies may themselves branch, alter storage, block, fault, or fail to return.

Ordering is part of the contract. Repeated stores and distinct reads must not be replaced by a final-state description. Calculated addresses may alias one another or fixed storage.

Explicit calculated accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Byte stores select the low eight bits; unsigned byte reads widen to values from 0 through 255. Represented 64-bit call results are sometimes divided into low and high 32-bit halves.

Unsigned field operations have explicit widths. Many other calculations involve represented signed words, left shifts, and multiplication. Interpreting those uniformly as arithmetic modulo \(2^{32}\) is an **inferred execution model**, not an independently verified platform guarantee. Address wraparound, signed overflow, and byte order remain unresolved.

### Input-field preparation and calls

The initial dependency is invoked before the third-input field extraction. Its represented result is shifted left by eight bits and forwarded.

Early calls subsequently receive the ten-bit input field shifted left by 8, 15, and 16 bits. Their maximum values are respectively 261,888, 33,521,664, and 67,043,328, all within 32 bits.

The call phase also forwards:

- Dependency-produced words shifted by various amounts.
- Identical transformed values in multiple argument positions.
- Low and high halves of represented 64-bit results.
- High halves both unchanged and shifted.
- Explicit zero arguments.
- Masked fields and logically right-shifted values.
- Values attributed to additional post-call register state.

For a represented unsigned 64-bit result, right shifts by 34, 36, and 44 correspond to logical right shifts of its high 32-bit half by 2, 4, and 12. They are not shifts of the low half.

Masks retain low fields of several widths, including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Mask placement matters: masking before shifting can exclude bits that would otherwise affect an argument.

Later ten-bit extractions consume dependency-derived values, not necessarily the third input. The input field does not control a visible iteration count or select a visible early exit.

No call is established as allocation, release, arithmetic assistance, I/O, or synchronization. Unused results do not make calls unobservable.

### Entry to the direct-memory phase

The last preparation result supplies two represented 32-bit halves. The first explicit store writes a saved dependency-derived word at offset 0x14 from the high half used as a base.

A byte at another calculated base plus 9 is then written with the low byte of that high half. A word store follows at offset 0x54 from a base formed by scaling the low half by 256. The earlier byte destination is then written again, using a saved word’s low byte.

An initial dependency-derived base has its offset-0x14 word cleared before its offset-5 byte is read. Another destination, formed by doubling a bounded quantity and adding 0x14, is cleared before the byte read is widened into a word store at offset 0x34 of the initial base.

The initial base’s offset-0x14 word is subsequently read. The separate clear is repeated, followed by two byte stores to offset 0x15 of its doubled base. If intervening destinations overlap the cleared word, the reload need not yield zero.

### Fixed-location reads and early propagation

Storage at **0x05 is read before the fixed-location write at 0x01**. Its value is converted into an unsigned 32-bit quantity and used as a base. The supplied global declaration is absent, so this conversion does not establish the original storage-read width.

The write at 0x01 receives the low byte of the bounded doubled base. Subsequent accesses include:

- A byte store at offset 0x15 from the base obtained through 0x05.
- Two writes to an initial base’s offset-0x11 byte, the second writing zero.
- A word read at offset 0x54 from the base obtained through 0x05, followed by a word store to that same address.
- A word store at offset 0x34 from the bounded doubled base.
- Offset-0x74 word reads whose results become further bases.

The word loaded through the base obtained from 0x05 is preserved for later address formation and scaling. It is not simply discarded when its source location is overwritten.

Further effects include repeated self-valued word stores at offset 0x24, repeated byte stores at offset 9, word updates at offset 0x14, a word-valued fixed write at 0x54, and a byte-valued fixed write at 0x15. These do not establish a particular object structure.

### Two distinct reads of one offset-5 byte

A middle portion reads a word at offset 4 from a fourfold calculated base. The loaded word becomes a base for **two separate unsigned byte reads at offset 5**.

For this portion, “fourfold base” means four times the earlier saved word. “Doubled target” means twice a word loaded through an earlier offset-0x74 access. “Earlier indirect target” denotes another word previously obtained through a byte-derived offset-0x74 access.

The first offset-5 byte read supplies a zero-extended word store at offset 0x44 of the earlier indirect target. Between that first byte read and the second, the representation orders five stores:

1. A byte at offset 9 of the fourfold base, receiving the doubled target’s low byte.
2. The first byte’s widened value at offset 0x44 of the earlier indirect target.
3. A word at offset 0x14 of the fourfold base, receiving eight times the saved word.
4. A byte at offset 0x15 of the doubled target, receiving the loaded base’s low byte.
5. A word at offset 0x44 of the doubled target, receiving the fourfold base.

The offset-5 byte is then read again from the loaded base. After a further byte store at offset 0x15 of the fourfold base, this **second** byte value supplies the base for a 32-bit read at offset 0x74.

The first value supplies stored data; the second determines a later read address. They need not agree. Intervening stores may alias the source byte, and no nonaliasing guarantee is established.

### Constants, fixed-base loading, and repeated stores

A later group stores 0x3A as an 8-bit value at a fourfold destination, as a 16-bit value at an eightfold destination, and as a 32-bit value at offset 0x20 from the fourfold destination.

Another calculated offset-0x20 word destination receives, in order:

- 0x20.
- 0x2F.
- A zero-extended byte read from offset 4 of the same base.
- 0x20 again.

**After those four stores, and before the fixed-location write at 0x60, a separate fixed-location cell is loaded.** Its represented value is a 32-bit signed word used as a base throughout the following region. The actual storage-access width is not independently established without its global declaration. This load is distinct from the earlier read at 0x05.

The fixed-location write at 0x60 receives the word value 0x20. A long unrolled region then repeatedly performs a word store at offset 0x54, a byte store at offset 0x19, a byte store at offset 0x11, and a word store at offset 0x34.

The first group uses an unadjusted scaled destination and value. Later groups use the adjusted quantity formed after subtracting 0x96 from the preceding fourfold quantity. Intermediate writes remain observable even if later groups overwrite the same locations.

Exact total multiplicity is **documented_only** in this contract. The region must not be described as one assignment or as an input-controlled loop.

### Tail fixed reads and alias-sensitive reloads

Immediately after the repeated-store region, storage at **0xA9** is read into a saved represented 32-bit word. A calculated offset-0x74 word is read next, before the fixed-location write at 0x3A.

The saved 0xA9 value later supplies both the low-byte value assigned to fixed storage at 0x67 and a full-word store at a calculated offset 0x44. Missing global declarations prevent independently confirming the physical width of the 0xA9 access beyond its represented word-valued use.

For the remaining description:

- **Working base** is the calculated base whose offset-0x20 word received the four stores above.
- **Auxiliary base** is twice the word obtained through the second offset-5 byte read’s offset-0x74 access.
- **Fixed-loaded base** is the base loaded from separate fixed storage before the write at 0x60.

The tail saves words from the auxiliary base plus 0x14 and the working base plus 4. Calculated stores then include byte updates at offsets 0x0D, 8, and 9, including 0x6C at working-base offset 8, and a self-valued word store at auxiliary-base offset 0x54.

The fixed write at 0x67 is followed by a **32-bit calculated read at auxiliary-base offset 0x44**. Only after that read does a distinct read of storage at 0x67 supply the fixed write at 0x21. The calculated read may itself be externally observable; the later fixed read is not merely propagation of a cached truncation.

Between that first read of 0x67 and its next read, effects include:

- Byte and word stores using the auxiliary offset-0x44 result.
- A signed-right-shift-derived byte store at auxiliary-base offset 1.
- Two word stores at fixed-loaded-base offset 0x54.
- A word read at working-base offset 0x40.
- Word stores at fixed-loaded-base offset 0x74, working-base offset 0x10, and fixed-loaded-base offset 0x44.

The word read at working-base offset 0x40 supplies the word written at offset 0x10 of that same base, replacing an earlier byte write there.

A **second distinct read of 0x67** supplies a byte store at working-base offset 9. A byte store at auxiliary-base offset 9 follows, then a word store of the saved 0xA9 value at working-base offset 0x44. A word is then read at working-base offset 0x54. Only afterward does a **third distinct read of 0x67** supply another byte store at working-base offset 9.

Calculated stores between these reads may overlap fixed storage. Consequently, neither later read is guaranteed to return the value originally written to 0x67.

### Final effects and transfer

The remaining accesses include restoring an earlier saved word at auxiliary-base offset 0x54, reading working-base offset 0x14, further byte updates at offsets 0x11 and 9, and a word update at working-base offset 0x44.

A word read at auxiliary-base offset 4 becomes a base. A 16-bit store writes 0x65 at twice that loaded value. Later operations read words at offsets 0x64 and 0x74 from that base, write decimal 100 at another calculated offset 0x54, and load another word through auxiliary-base offset 0x64.

The final calculated word store uses twice the word loaded from offset 0x64 as its destination. The terminal dependency receives four represented arguments:

1. 0x72.
2. The loaded word used in that final doubled-address calculation.
3. The earlier word saved from auxiliary-base offset 0x14.
4. The near-tail word read at offset 0x74.

The signed right shift earlier in the tail uses a count derived from the working base’s low eight bits. Counts exceeding 31 are possible in the representation; their actual machine interpretation is **documented_only**.

## Side effects

### Memory layout and state

Explicit calculated accesses use these recurring offsets:

| Width | Representative offsets |
|---|---|
| 8 bits | 0x01, 0x04, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Scaled destinations, including doubled and eightfold addresses |
| 32 bits | 0x04, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

These are access layouts, not recovered object definitions. A word at offset 0x14 overlaps a byte at offset 0x15. Byte order determines the resulting word value.

Fixed-location effects include reads at 0x05 and 0xA9, a separate fixed-base load, repeated reads at 0x67, and writes at 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67. Fixed assignments carry byte-valued or word-valued expressions, but absent global declarations leave some actual access widths unresolved. Explicitly typed calculated accesses provide stronger width evidence.

No general preservation of global state is established.

### Ownership, failures, and concurrency

No allocation, release, ownership transfer, borrowing duration, retention rule, or reference-counting behavior is established. Unknown callees may modify globals or manage memory; this contract assigns them no preservation or ownership guarantee.

There is no visible error code, rollback, validation failure path, or cleanup path. Faults and nonreturning dependencies can leave earlier effects in place. Atomicity, retry safety, reentrancy, and thread safety are **documented_only**.

## Dependencies

Only the subject has a resolved opaque function identity. The following are contract-local opaque references, not recovered callee identities:

| Reference | Role |
|---|---|
| F-U001 | Initial indirect dependency, with destination derived from incoming control state. |
| F-U002 | Family of unresolved preparation dependencies consuming transformed arguments and supplying represented words, two-word results, and additional post-call values. |
| F-U003 | Terminal indirect destination receiving four arguments after the direct-memory effects. |

F-U002 represents multiple destinations, not one established function. Complete call identities, signatures, preservation rules, errors, and memory effects are **documented_only**.

## Acceptance examples

These are conditional static review criteria, not executed tests.

### Input-field boundaries

Third inputs 0 through 1023 produce an early field of zero. Input 1024 produces one; 0x000FFFFF produces 1023; 0x00100000 produces zero. Inputs differing only outside bits 10–19 produce the same early field, but need not have identical whole-function behavior.

### Width conversion

A represented value 0x123456AB written through an explicit byte destination stores 0xAB. An unsigned byte read of 0xFE widened into a word store produces 254.

### Fixed-read ordering

The read at 0x05 precedes the write at 0x01. The separate fixed-base load follows all four offset-0x20 stores and precedes the write at 0x60. The 0xA9 read follows the repeated-store region and precedes the next calculated offset-0x74 read.

Acceptance must not move these reads earlier merely because their addresses are fixed.

### Repeated offset-5 reads

If the first read returns 0x12 and an intervening aliasing store changes that byte to 0x34, the first widened store uses 18. The subsequent byte-derived offset-0x74 word read uses base 52, giving address 0xA8 under ordinary nonwrapping address arithmetic.

Reusing the first byte would instead read at 0x86 and would violate the represented dependency.

### Reloads at 0x67

If an intervening calculated store changes storage read through 0x67, the next fixed read observes the applicable current value. The first, second, and third reads may differ. Assuming all equal the low byte originally assigned from the saved 0xA9 word requires additional nonaliasing and memory-stability assumptions.

### Bounded address calculation

Under 32-bit bit-vector semantics, shifting a word left by eight, retaining its low 15 bits, and logically shifting right by six yields a multiple of four from 0 through 508. Doubling yields a base from 0 through 1016.

This bounded quantity must not be treated as an unrestricted caller-provided pointer.

### Final-state conditions

For ordinary stable memory with no intervening overlap, a cleared word reloads as zero, and the four offset-0x20 stores leave 0x20. Observable-access semantics still require the intermediate operations; these final values alone are insufficient acceptance evidence.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid invocation context.
- Callee identities, actual interfaces, side effects, ownership, and return behavior.
- Additional post-call register production.
- Target byte order, alignment, address mapping, and volatility.
- Actual widths of fixed-symbol accesses lacking declarations.
- Exact repeated-store multiplicity in this contract.
- Signed-overflow, address-wraparound, and oversized shift-count semantics.
- Terminal-transfer behavior and normal completion.
- Object boundaries, lifetimes, concurrency, and recovery rules.

Under an inferred 32-bit bit-vector model, successive left shifts whose total exceeds 31 can discard every original bit. For example, two shifts by 28 yield zero. That conditional arithmetic consequence is not an unconditional platform claim.

## Evidence

This document uses the supplied static representation as evidence, not as executable validation. Prior prose, comments, and reviewer feedback are not independent proof.

The strongest evidence is explicit calculated-access typing, input extraction, constants, data dependencies, and ordered access sequences. In particular, the representation distinguishes the two middle offset-5 byte reads, places fixed reads relative to surrounding stores, and shows three separate late reads of 0x67.

Missing dependency bodies, global declarations, platform semantics, and runtime observations limit stronger claims. No runtime tests have been performed.
