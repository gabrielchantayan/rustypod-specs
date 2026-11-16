<!-- Trusted provenance: F-bd9f0b30e08b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd9f0b30e08b

## Purpose

F-bd9f0b30e08b performs an ordered sequence of unresolved indirect calls, transforms their results, and then reads and modifies memory through fixed and computed addresses. Its last represented operation is an unresolved indirect transfer with four argument words.

No application-level purpose is established. The constants, low addresses, and recurring offsets do not establish formatting, serialization, allocation, object initialization, or device control.

This contract distinguishes:

- **Established static evidence:** accesses, transformations, and ordering explicitly represented in the supplied material.
- **Inferred intent:** interpretations that impose no behavioral requirements.
- **documented_only:** unresolved behavior that cannot safely be converted into an executable requirement.

The contract specifies established trace constraints without claiming a fully recovered implementation. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface accepts three 32-bit inputs and declares no result.

| Input | Represented interpretation | Established use |
|---|---|---|
| First | Opaque 32-bit word | No explicit consumption |
| Second | Opaque 32-bit word | No explicit consumption |
| Third | Unsigned 32-bit word | Supplies a field to an early indirect call |

The third input contributes bits 10–19, positioned in argument bits 6–15. The resulting contribution is the unsigned 10-bit field multiplied by 64: a multiple of 64 between 0 and 65,472 inclusive. Bits outside that field do not affect this particular contribution.

The lack of explicit consumption of the first two inputs does not establish their irrelevance to implicit machine state or unresolved calling conventions.

### Implicit inputs

Execution also depends on incoming control-transfer state, unresolved call targets, primary and secondary call-result channels, fixed storage, and computed-address memory.

Some results are represented as 64-bit quantities. Other values appear through a secondary 32-bit result channel whose precise producer and calling convention are unresolved. These channels must not be merged into an assumed single-result interface.

### Outputs and completion

There is no established returned value, status, error object, or owned resource.

Observable effects include indirect calls, memory accesses, and the final transfer. The representation shows a return after that transfer; whether the target returns, continues elsewhere, fails, or never completes is **documented_only**.

### Preconditions and ownership

To reach completion, every reached indirect target must support execution and every reached memory location must support its actual access width and permissions. Alignment requirements and the memory map are unknown.

No explicit validation of addresses, bounds, target identity, or accessibility is represented. No allocation, release, ownership transfer, reference-count operation, or lifetime guarantee is established. An address-bearing value is not thereby an owned object.

## Behavior

### Indirect-call phase

An extended sequential call phase precedes the first explicit direct data-memory write. Calls displaying no arguments remain calls with unresolved effects, not no-ops.

The first target is formed using incoming control-transfer state and a displacement of 4,094 bytes. Subsequent targets predominantly use embedded control-location values with that displacement. This does not establish shared target identity or interchangeable helper behavior.

Arguments include:

- Results shifted left by 8 or 15 bits.
- Separately supplied low and high words of represented 64-bit results.
- A shifted low word together with an unshifted high word.
- Shifted secondary result words.
- Explicit zeros in one or several positions.
- The same retained value in multiple positions.

Values can remain live across many intervening calls. Similar field extractions later in this phase operate on call-derived values, not necessarily on the third explicit input.

The full identities, signatures, and semantic ordering of individual dependency operations remain **documented_only**. No call may be assumed to preserve fixed storage, referenced memory, ownership, or other global state.

### Integer widths and arithmetic

The representation combines 32-bit words with selected 64-bit result operations. Unsigned masks retain fields including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits.

Unsigned right shifts discard low bits and introduce zeros. Whole-result unsigned 64-bit shifts by 34, 36, and 44 are distinct from 32-bit shifts: their results correspond to the high word shifted right by 2, 4, and 12 respectively.

Under a 32-bit truncating word interpretation:

- Left shifts discard bits above bit 31.
- Left shift by 8 followed by unsigned right shift by 8 retains the original low 24 bits.
- Successive left shifts cannot recover discarded bits; a combined displacement of at least 32 leaves zero when each intermediate operation is a truncating 32-bit shift.
- Address scaling by two, four, eight, or larger factors must respect intermediate widths rather than use unbounded arithmetic.

Some shifts, multiplications, and address additions involve signed quantities with no established range. Their overflow behavior is **documented_only**. These conditional word-arithmetic consequences do not establish universally defined signed arithmetic.

### Entry into direct memory mutation

The last call in the call phase supplies a represented two-word result. Other retained ingredients include a low result shifted left by 25, a result multiplied by 33,554,432, and a small field derived from another result.

The small field retains the low 15 bits after an 8-bit shift and then shifts right by 6. Under 32-bit word semantics it ranges from 0 through 508 in multiples of four. Doubling it produces a base from 0 through 1,016 in multiples of eight. These are actual address ingredients, not validated array indices.

The initial trace establishes these ordering constraints:

- A word write at offset 20 from the final result’s high word occurs before a byte write at offset 9 from the retained 25-bit-shifted address.
- A word write at offset 84 from the final result’s low word scaled by 256 occurs before a second byte write to that offset-9 destination.
- A zero word write at offset 20 of the multiplied-result base precedes an explicit byte read at offset 5 of that base.
- A zero word write at offset 20 of the small doubled base follows that byte read and precedes a word write of the zero-extended byte at offset 52 of the multiplied-result base.
- The multiplied-result offset-20 word is then read. Another zero word write to the small doubled base and two byte writes at its offset 21 follow.

Offset 21 overlaps the word starting at offset 20. No disjointness assumption permits these accesses to be reordered or collapsed.

### Fixed-storage input and interlinked updates

After those initial accesses, named fixed storage at address 5 is read and its value converted to an unsigned 32-bit quantity used as a base. The underlying storage width is **not established** by that conversion. This differs from the explicit byte read at a computed offset 5 described above.

The fixed-storage read precedes the assignment at fixed address 1 and the following computed writes. Its resulting base is retained while the function updates offset 21, reads and replaces an offset-84 word, and performs further interlinked mutations.

The wider memory phase includes:

- Explicit byte accesses at offsets 1, 4, 5, 8, 9, 13, 16, 17, 21, and 25.
- Explicit word accesses at offsets 4, 16, 20, 32, 36, 52, 64, 68, 84, 100, and 116.
- Explicit halfword writes through scaled addresses.
- Reads of words subsequently used as addresses.
- Zero-extended byte values used as address ingredients or stored words.
- Self-referential writes of address-derived values through related addresses.
- Repeated reads from the same computed offset separated by writes.

These offsets do not establish a structure layout, object extent, or field ownership.

One expression reads the word at a computed offset 116 both as stored data and as an address ingredient. The representation does not independently establish the evaluation order of the two reads within that expression. Their equality must not be assumed for externally changing or read-sensitive storage.

Assignments also target fixed addresses 21 and 84 during this phase. Their named-storage declarations are absent; the widths of the supplied values do not independently establish storage-access widths.

### Constants and retained global-derived base

A later region explicitly writes 58 as a byte, a halfword, and a word at separately scaled destinations.

Another destination receives four successive word writes: 32, 47, a zero-extended byte read from offset 4 of its associated derived base, and 32 again. That byte read occurs after the writes of 32 and 47, before the last two writes.

**Only after all four word writes** does the function load a base from separate fixed storage. The loaded value is retained through the repeated block; it is not reloaded for each group.

The fixed-storage load precedes an assignment of 32 at fixed address 96. Consequently:

- The four writes can affect the base-provider storage if they alias it.
- The address-96 assignment occurs after the base has been retained.
- Later writes aliasing the base-provider storage do not, by themselves, change the retained base.

The repeated region interleaves a word write at offset 84 of an eightfold-scaled base, a byte write at offset 25 of the retained global-derived base, a byte write at offset 17 of another derived base, and a word write at offset 52 of the global-derived base.

The introductory group differs from subsequent groups in both the offset-84 destination’s base expression and the value supplied to the offset-52 write. It must not be treated as an identical repetition.

The displayed groups preserve their internal order. Whether every displayed repetition corresponds one-for-one to executable accesses, and the independently confirmed executable multiplicity, remain **documented_only**. This uncertainty does not justify replacing the region with one final-state assignment.

### Late fixed-storage reads and ordering

After the repeated region, named fixed storage at address 169 is read into a retained 32-bit value. **That read precedes the following explicit word read at computed offset 116.** Assignment to a 32-bit retained value does not independently establish the width of the named fixed-storage read.

Next, the function assigns fixed storage at address 58. Its value is formed from a character-narrowed adjusted base multiplied by eight, then increased by 48. The named destination’s storage width and signed-character interpretation are unresolved.

**The address-58 assignment precedes both the subsequent computed offset-20 word read and the subsequent computed offset-4 word read, in that order.** Those reads retain values for later writes and the final transfer. If the address-58 assignment overlaps either location, the reads occur after that mutation, not against pre-assignment memory.

Further writes include byte value 108 at offset 8 and address-derived bytes at offsets 13 and 9, followed by a self-referential word write at offset 84 of a doubled derived base.

### Separate observations of address 103

The late trace contains three distinct reads of named fixed storage at address 103. They are not established copies of one saved byte.

First, the function assigns address 103 a value narrowed to eight bits from the earlier address-169 result. **An explicit computed offset-68 word read intervenes before address 103 is read for the assignment at fixed address 33.** The computed read is therefore between the address-103 write and this first read.

After the address-33 assignment, intervening mutations include:

- A byte write at computed offset 16.
- Word writes at offset 84 of the retained global-derived base.
- A byte write using a signed right-shift result.
- An explicit offset-64 word read.
- A word write of the earlier retained offset-4 value at offset 116.
- A word write of the offset-64 result at offset 16.
- A word write at offset 68 of the global-derived base.

Only then is address 103 read again, this time to supply an explicit byte write at offset 9 of the other derived base.

Before its third read, the function performs another computed byte write, a computed offset-68 word write carrying the retained address-169 value, and a computed offset-84 word read. The third address-103 read supplies another byte write to the same offset-9 destination.

Each observation uses the contents at its own read point. Aliasing mutations can change what later reads obtain. Named-storage access widths at addresses 103 and 33 remain unresolved; the two computed offset-9 destinations are explicitly byte-wide.

### Remaining updates and final transfer

One late byte write uses a signed right shift whose count is the low eight bits of a derived address. Counts can therefore exceed 31. Count masking, saturation, faults, and signed right-shift behavior are not established.

After the last address-103-derived byte write, the function continues with interleaved reads and writes, including:

- Bytes derived from retained words at offsets 13 and 17.
- Restoration of an earlier offset-20 value to a computed offset 84.
- A fresh computed offset-20 word read.
- Another overwrite of the offset-9 byte destination.
- A halfword write of 101 through an address obtained by doubling a newly loaded word.
- A word write of 100 at computed offset 84.
- Further reads at offsets 100 and 116.
- A final word write through an address obtained by doubling a retained loaded word.

The final transfer receives four argument words:

1. 114.
2. The word loaded from offset 100 of the base used for the halfword-write address derivation.
3. The word retained from the late offset-20 read immediately after the address-58 assignment.
4. The later word loaded at offset 116.

The final memory write precedes the transfer. No represented validation, recovery, or cleanup intervenes.

## Side effects

### Explicit widths versus named storage

Explicit pointer-based accesses establish 8-, 16-, or 32-bit widths as described above. Named fixed-storage accesses do not inherit a proven storage width merely because their values are narrowed or retained in a particular width.

| Fixed storage | Established observation | Storage width |
|---|---|---|
| Address 5 | Read, then converted to an unsigned 32-bit base | Unresolved |
| Addresses 1 and 21 | Assigned narrowed address-derived values | Unresolved |
| Address 33 | Assigned from a separate read of address 103 | Unresolved |
| Address 58 | Assigned after the address-169 and computed offset-116 reads | Unresolved |
| Address 84 | Assigned a doubled retained value | Unresolved |
| Address 96 | Assigned 32 after loading the repeated-block base | Unresolved |
| Address 103 | Assigned a narrowed value, then read at three separate points | Unresolved |
| Address 169 | Read and retained as a 32-bit value | Unresolved |
| Separate base-provider storage | Loaded after the four-write sequence; result retained | Unresolved |

No deployment memory map establishes these locations as ordinary process globals. Overlap, device semantics, and read-sensitive behavior remain possible but unproven.

### Aliasing and partial effects

Earlier writes can change subsequent reads, including fixed-storage reads. Retained values remain distinct from later memory observations.

Overlapping byte and word accesses require byte-order information to determine resulting numeric words. Endianness is unknown.

Every unresolved dependency may independently mutate memory or global state, retain or release resources, fail, or not return. No preservation or ownership behavior is implied.

No error result, rollback, exception translation, synchronization, or cleanup path is represented. Failure can leave prior effects visible. Atomicity, transactionality, reentrancy, thread safety, and idempotence are not established.

## Dependencies

The following are document-local opaque references, not recovered identities.

| ID | Role | Status |
|---|---|---|
| F-000000000001 | Initial target derived from incoming control-transfer state | documented_only |
| F-000000000002 | Family of subsequent unresolved indirect call sites | documented_only |
| F-000000000003 | Final unresolved four-argument transfer | documented_only |

F-000000000002 denotes multiple unresolved sites, not one interchangeable helper. Argument counts, result conventions, effects, and continuation behavior can differ.

No dependency is established to preserve globals or referenced memory, acquire ownership, or release ownership.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

### Third-input boundaries

| Third input | Contribution to the relevant early argument |
|---:|---:|
| 0 | 0 |
| 1,023 | 0 |
| 1,024 | 64 |
| 1,048,575 | 65,472 |
| 1,048,576 | 0 |
| 4,294,967,295 | 65,472 |

Matching contributions do not establish equivalent whole-function behavior.

### Explicit byte widening

If an explicit computed offset-5 byte read yields 255, its corresponding widened word is 255. This example does not classify the named-storage read at fixed address 5 as byte-wide.

### Four writes before base acquisition

If the four-write destination is externally observable, the required order is 32, 47, widened byte, 32. The repeated-block base is acquired afterward. If these writes alter its provider storage, the subsequent load uses the resulting storage state.

Once acquired, that base remains retained through the repeated block even if later writes alter its provider.

### Late read boundaries

If the address-58 assignment overlaps the subsequent offset-20 read, that read occurs after the assignment. The following offset-4 read also occurs afterward.

The address-169 observation must precede the computed offset-116 observation; they are not interchangeable snapshots.

### Address-103 rereads

If an intervening computed write overlaps address 103, a later address-103 observation may differ from the earlier one. Acceptance must preserve the separate read supplying address 33 and the two later reads supplying computed byte writes.

Even without intervening writes, read-sensitive storage prevents assuming that three observations equal one retained value.

### Failure and final transfer

If a dependency does not return, later calls and direct accesses are not reached; rollback is not promised.

If the final transfer is reached, its first argument is 114 and its other arguments are the retained values identified above. Its completion behavior remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and complete dependency interfaces.
- Hidden input and machine-state dependencies.
- Exact primary, secondary, and two-word result conventions.
- Named fixed-storage widths, including addresses 5, 103, and 169.
- Memory mapping, endianness, alignment, permissions, and object extents.
- Signed overflow, character signedness, and unconstrained shift semantics.
- Evaluation order of multiple reads within a single expression where not independently represented.
- Independently confirmed executable repetition multiplicity.
- Volatile, device, concurrent, or read-sensitive memory behavior.
- Pointer provenance, resource ownership, and lifetime.
- Final-transfer identity, return behavior, and failure recovery.

Hardware-oriented or deliberately transformed behavior is an **inferred interpretation**, not an established purpose or additional guarantee.

## Evidence

This contract is based only on the supplied static representation. Embedded comments, prior prose, and reviewer feedback were not treated as independent behavioral authority.

Established evidence includes the three-word interface, explicit third-input extraction, unresolved indirect calls, mixed-width computed accesses, retained bases and values, separate fixed-storage observations, and the final four-argument transfer.

The distinction between explicit access widths and unresolved named-storage widths follows from the represented accesses and absent global declarations. The ordering requirements follow from the positions of the corresponding reads and writes.

No external implementation inspection, dependency-body inspection, compilation, execution, or runtime testing has been performed.
