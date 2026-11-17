<!-- Trusted provenance: F-e3bb8c9db174; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3bb8c9db174

## Purpose

F-e3bb8c9db174 performs an extended sequence of indirect invocations and integer transformations, then mutates memory through computed addresses and fixed numerical locations before a terminal four-argument indirect transfer.

**Established static evidence:** three explicit 32-bit inputs, no explicit return value, numerical splitting of 64-bit dependency results, accesses of multiple widths, repeated stores, and terminal arguments formed from retained reads.

**Inferred intent:** masks, shifts, and address-bearing fields suggest packed-value manipulation and interconnected state updates. They do not establish an application, protocol, object type, or encoding.

This contract describes the supplied static representation, not verified executable behavior. Unresolved matters are identified as **documented_only**. No runtime tests have been performed.

## Interface

### Inputs and output

| Element | Representation | Limitation |
|---|---|---|
| First input | Uninterpreted 32-bit value | No explicit use is visible. |
| Second input | Uninterpreted 32-bit value | No explicit use is visible. |
| Third input | Signed 32-bit integer | Initially shifted left by four; no validation is visible. |
| Return | No explicit value | No success indicator, error status, or returned resource is established. |

There is no declared output buffer, input length, or allocation interface.

Incoming return-link state participates in selection of the first indirect target. Later operations consume secondary machine-result values whose production is not fully expressed as ordinary returns. These are implicit environmental inputs, not established constants.

### Preconditions

Reaching the terminal transfer requires that:

- Every reached indirect target is executable under the actual calling convention.
- Dependencies needed for continuation return.
- Computed and fixed addresses permit the requested accesses.
- Applicable alignment requirements are satisfied.
- Consumed implicit machine results have valid definitions.

These are execution requirements, not checks performed by this function. No visible null, bounds, permission, overflow, or indirect-target validation protects the sequence.

### Widths and arithmetic

Explicit computed-memory operations include 8-bit reads and writes, 16-bit writes, and 32-bit reads and writes. Address-bearing values and most arithmetic intermediates are 32 bits. Some dependency results are 64 bits, divided numerically into low and high 32-bit portions.

Byte stores retain the low eight bits. Explicit unsigned byte reads widen to values from 0 through 255. Multi-byte memory byte order is **documented_only**.

Unsigned 32-bit arithmetic retains the low 32 bits where applicable. Signed left shifts and signed multiplication do not establish portable behavior for negative operands or overflow. Examples using truncating 32-bit arithmetic state that assumption explicitly.

Some fixed-location accesses use symbols without supplied declarations. Their numerical values may be consumed as 32-bit quantities without establishing the underlying access width. Such widths remain **documented_only** rather than being inferred solely from the destination of an assignment.

## Behavior

### Invocation and transformation region

The representation shows an expanded sequence without explicit loops or conditional branches. This does not constrain branching, blocking, failure, or non-returning behavior inside dependencies.

The first indirect invocation precedes the third input’s transformation. The next invocation receives zero, zero, and the third input shifted left by four.

A retained unsigned version of that shifted value is later masked to 20 bits and logically shifted right by ten. Under truncating 32-bit semantics, this extracts original input bits 6 through 15, giving a value from 0 through 1023. A later invocation receives that field shifted left by eleven, twice. Earlier delivery of the less-masked shifted input prevents treating this field as the function’s entire input dependency.

The dependency sequence repeatedly:

- Passes 32-bit results shifted left by eight.
- Retains shifted values across intervening invocations and supplies further shifted derivatives, sometimes twice in one invocation.
- Separates 64-bit results into numerical low and high portions.
- Passes unchanged high portions alongside independently shifted low portions.
- Mixes explicit zeros with ordinary and secondary machine results.
- Masks retained fields before further shifts.

Visible mask widths include 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Unsigned right shifts are logical. Whole unsigned 64-bit shifts by 34, 36, and 44 select the high portion shifted right by two, four, and twelve respectively; these are not oversized 32-bit shifts.

Intermediate truncation matters. Under a 32-bit bit-vector interpretation, left eight followed by logical right eight retains only the original low 24 bits; successive left shifts by 28 produce zero; and a value already shifted left by 16 has zero low 16 bits. These facts do not justify substituting zero for unrelated machine results or memory reads.

Calls with discarded ordinary results remain observable dependencies. Neither their removal nor their reordering is justified by this evidence.

### Memory layout and aliasing

No unified structure layout is established. Recurring computed offsets include:

| Offsets | Visible uses |
|---|---|
| +0x01 | Byte write from a variable right shift |
| +0x04 | Byte and 32-bit reads |
| +0x05 | Unsigned byte reads |
| +0x08, +0x09, +0x0d | Byte writes |
| +0x10 | Byte and 32-bit writes |
| +0x11, +0x15, +0x19 | Byte writes |
| +0x14 | 32-bit reads and writes |
| +0x20, +0x24, +0x34 | 32-bit writes |
| +0x40 | 32-bit read |
| +0x44, +0x54, +0x74 | 32-bit reads or writes |
| +0x64 | 32-bit address-bearing reads |

A word at +0x14 overlaps a byte at +0x15; a word at +0x10 overlaps +0x11. Different bases may also alias each other or fixed locations. Offsets do not establish distinct fields, allocations, or array elements.

### Entry into direct mutation

The first stores use the final dependency result’s high portion, its low portion multiplied by 256, and an earlier retained shifted base. A word store through the high-derived base precedes a byte store through the retained base. A word store through the low-derived base follows, then another store to the same byte destination.

For the following description:

- The **primary base** is an earlier 32-bit dependency result multiplied by 33,554,432.
- The **small field** is a retained result shifted left by eight, masked to 15 bits, then logically shifted right by six.
- The **small base** is twice that small field.

The primary base’s +0x14 word is zeroed before reading its +0x05 byte. The small base’s +0x14 word is then zeroed, the widened byte is written to primary +0x34, and primary +0x14 is read back and retained. Another zero store to small +0x14 follows.

The read-back is not unconditionally zero: intervening computed stores can alias it. Two successive byte writes at small +0x15 use the retained read-back value’s low byte.

### Fixed locations within the traversal region

The next fixed-location read is at **0x05**. Its result is widened to a 32-bit address-bearing value; its physical read width is **documented_only** because the symbol declaration is absent. This read precedes the byte-valued write at **0x01**, which receives the small base’s low byte.

Using the retained value from 0x05, execution writes a byte at its computed +0x15, mutates primary +0x11 and a destination derived from the retained read-back, then clears primary +0x11. It reads a 32-bit value at the retained fixed-read-derived base’s +0x54 before overwriting that field. Call the value read there the **relationship value**.

Subsequent mutations use the relationship value and freshly read +0x74 fields. They include self-reference stores at relationship +0x24, repeated byte writes at a reached base’s +0x09, and word writes at small +0x14.

Within those computed mutations:

- Fixed **0x54** receives the relationship value shifted left by one. Its physical store width is **documented_only**; the assigned expression is 32-bit.
- Further word stores at small +0x14 retain the unshifted relationship value.
- Two byte writes at small +0x15 use the low byte of four times the relationship value.
- Fixed **0x15** then receives the small base’s low byte.

These fixed writes are not isolated from surrounding computed accesses. They may affect subsequent traversals through aliasing.

### Traversal, constants, and repeated relationships

The traversal uses fourfold and eightfold derivatives of the relationship value, fresh reads from +0x74, a byte read at a derived +0x05, and further reads from +0x64 and +0x04. Mutations include self-reference words at +0x14, byte cross-references at +0x09, +0x0d, and +0x15, and word relationships at +0x44 and +0x54.

A repeated +0x05 read occurs after intervening mutations and must observe memory at that later point.

For the remaining description:

- The **retained node** is twice the address-bearing value obtained through the later byte-derived +0x74 traversal.
- The **traversal base** is four times that same value.
- The **adjusted value** is four times the relationship value minus 0x96.
- The **cross-reference base** is four times the adjusted value.

Execution writes 0x3a as a byte at the traversal base, as a 16-bit value at twice that base, and as a 32-bit value at traversal +0x20.

At cross-reference +0x20, four successive 32-bit stores contain 0x20, 0x2f, the widened byte freshly read from cross-reference +0x04, and 0x20. The byte read follows the first two stores.

Next, fixed **0x087fbbbc** is read into a retained 32-bit address-bearing value, called the **anchor**. The fixed symbol’s underlying read width is **documented_only**. This read precedes the fixed write of numerical **0x20 at 0x60**, whose physical store width is also **documented_only**.

A relationship store through eight times the original relationship value precedes byte cross-references at anchor +0x19 and cross-reference +0x11. Anchor +0x34 first receives eight times the original relationship value. The repeated region thereafter uses eight times the adjusted value for its relationship base, interleaving:

- A 32-bit traversal-base value stored at that relationship base’s +0x54.
- The cross-reference base’s low byte stored at anchor +0x19.
- The anchor’s low byte stored at cross-reference +0x11.
- The adjusted relationship base stored at anchor +0x34.

The bases are retained, not visibly advanced. This is not evidence of an array fill or append. Exact repetition multiplicity is not enumerated by this partial contract; complete trace multiplicity remains **documented_only**.

### Post-repetition snapshots and fixed-byte reloads

The post-repetition region first reads fixed **0xa9**, retaining a 32-bit value. The physical global read width is **documented_only**. This precedes a 32-bit read from traversal +0x74, then the byte-valued write at fixed **0x3a**. That byte is eight times the adjusted value plus 0x30, truncated to eight bits.

Only afterward are two further 32-bit snapshots taken: retained node +0x14, which supplies the terminal third argument, and cross-reference +0x04.

Before fixed 0x67 is written, computed mutations store:

- The cross-reference base’s low byte at retained node +0x0d.
- 0x6c at cross-reference +0x08.
- The anchor’s low byte at cross-reference +0x0d and +0x09.
- The retained node itself at its +0x54.

Fixed **0x67** then receives the low byte of the retained 0xa9 value. A 32-bit read from retained node +0x44 intervenes before fixed **0x67 is read again and copied to 0x21**. If that intervening read has observable memory behavior, its ordering remains significant.

The +0x44 value’s low byte is stored at cross-reference +0x10, and its full value at anchor +0x54. A byte at retained node +0x01 receives a right-shifted derivative using the cross-reference base’s low eight bits as shift count. The operand is represented as signed; exact machine shift-count behavior is **documented_only**. Anchor +0x54 is then overwritten with the cross-reference base.

Next, cross-reference +0x40 is read as a word. The earlier +0x04 snapshot is written at anchor +0x74; the +0x40 result is written at cross-reference +0x10; and the retained node is written at anchor +0x44.

At this point, **0x67 is freshly read** for a byte store at cross-reference +0x09. A byte store at retained node +0x09 and a word store of the retained 0xa9 value at cross-reference +0x44 intervene before reading cross-reference +0x54. Following that read, **0x67 is freshly read again** for another byte store at cross-reference +0x09.

Neither computed byte store is specified to reuse the originally retained low byte from 0xa9. Intervening computed stores may alias 0x67.

### Final mutations and terminal transfer

The retained traversal +0x74 snapshot’s low byte is written at the newly read cross-reference +0x54 base’s +0x0d. The earlier retained node +0x14 snapshot is restored to retained node +0x54. A fresh word read at cross-reference +0x14 follows.

Further byte and word mutations use these retained values before reading another address-bearing word at retained node +0x04. Call that newly read value the **final traversal base**. A 16-bit 0x65 is written at twice that base, before reading its +0x64 word. This +0x64 value is retained as the **terminal second argument**.

The low byte of the recent cross-reference +0x14 snapshot is written to the earlier linked base’s +0x11, followed by a 32-bit store of 100 at that linked base’s +0x54.

The final suffix has these ordering requirements:

1. Read the terminal fourth argument as a 32-bit value at final traversal +0x74.
2. Read another 32-bit address-bearing value at retained node +0x64.
3. Write the final traversal base’s low byte at retained node +0x09.
4. Write the retained terminal second argument as a word at the newly read base’s +0x54.
5. Write that newly read base as a 32-bit value at twice the terminal second argument.

Only then does the terminal indirect transfer receive, in order, 0x72, the retained terminal second argument, the earlier retained node +0x14 snapshot, and the retained terminal fourth argument.

The final stores may overwrite locations that supplied those arguments. They do not trigger argument reloads. A return follows in the representation; actual returning-call versus terminal-jump semantics remain **documented_only**.

## Side effects

Direct effects include computed-memory mutations, fixed-location writes, overlapping-width stores, and repeated writes. Indirect dependencies may have additional effects; none is established as preserving globals.

Retained read values and later memory reads are distinct. Reusing a previous read is not justified merely because the nominal address is unchanged. Conversely, retained terminal arguments must not be recomputed from potentially overwritten memory.

No transaction, rollback, or recovery boundary is visible. Earlier effects are not shown being undone if continuation fails.

No explicit allocation, release, retain, or ownership transfer is identifiable. Inputs and results are not established as owned or borrowed resources. Unknown dependencies may acquire, release, or mutate resources. Reentrancy, synchronization, thread safety, and atomicity are **documented_only**.

## Dependencies

F-e3bb8c9db174 is the only supplied opaque function ID. No opaque IDs are supplied for its indirect targets; their identities remain **documented_only**, without invented mappings.

The dependency surface includes targets selected from incoming return-link state and internally formed target values, ordinary 32-bit results, split 64-bit results, secondary machine results, and the terminal four-argument target.

Zero visible arguments do not establish absence of implicit inputs. No dependency is established as pure, deterministic, memory-preserving, non-blocking, or guaranteed to return.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Initial formation:** with truncating 32-bit shifts and third input 64, the second invocation receives zero, zero, and 1024. The initial invocation must still precede it.
- **Field boundary:** under that arithmetic assumption, third inputs 63 and 64 yield later extracted fields zero and one. Matching fields do not imply matching complete behavior.
- **Split result:** a 64-bit result with high portion 0x80000001 and low portion 1 supplies 256 where low-left-eight is required, while an unchanged-high argument remains 0x80000001.
- **Fixed ordering:** the 0x05 read precedes the 0x01 write; the 0x087fbbbc read precedes the 0x60 write. Aliasing does not permit reversing either pair.
- **Retained anchor:** if the 0x60 write aliases bytes of 0x087fbbbc, the repeated region still uses the anchor already read, not a replacement read.
- **0x67 aliasing:** suppose its initial byte store produces 0xab, but a subsequent computed store changes it to 0xcd before a specified reload. That reload supplies 0xcd. Later reloads likewise observe their own current memory state.
- **Overlapping writes:** a byte mutation overlapping a previously zeroed word affects a later word read according to byte order. An unconditional zero substitution is insufficient.
- **Store trace:** the cross-reference +0x20 writes remain 0x20, 0x2f, widened current byte, and 0x20 in order; final-value equality alone does not establish equivalence.
- **Final suffix:** if the retained terminal second argument is 0x1000 and the final +0x64 read yields 0x2000, the final word stores write 0x1000 at 0x2054, then 0x2000 at 0x2000, assuming valid addresses and truncating arithmetic. The second argument remains 0x1000.

## Uncertainties

The following remain **documented_only**:

- Application intent and fidelity to executable control flow.
- Indirect identities, calling conventions, and implicit-result production.
- Complete invocation trace and repeated-store multiplicity.
- Signed overflow, signed left-shift behavior, and variable shift-count semantics.
- Undeclared fixed-symbol access widths.
- Byte order, alignment, permissions, valid address ranges, and aliasing.
- Ordinary versus device or otherwise externally observable memory.
- Dependency ownership, global effects, and failure behavior.
- Terminal transfer and exception semantics.

No explicit error status or recovery path is visible. This does not establish that execution cannot fail.

## Evidence

The supplied static body supports the explicit widths, numerical transformations, computed offsets, ordered fixed accesses, post-repetition reload distinctions, final stores, and retained terminal arguments described above.

Application intent is inference. Missing declarations and unresolved dependencies are not replaced with assumptions. No implementation inspection beyond the supplied material, runtime execution, smoke run, or test result supports this contract.
