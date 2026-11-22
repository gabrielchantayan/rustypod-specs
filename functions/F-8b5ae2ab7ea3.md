<!-- Trusted provenance: F-8b5ae2ab7ea3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8b5ae2ab7ea3

## Purpose

F-8b5ae2ab7ea3 issues an ordered sequence of dependency calls and directly reads and modifies memory. Observable effects include selector-directed byte writes into low addresses, indexed 32-bit stores, mixed-width reads, two updates to one shared location, and an attempted call through address zero.

No application-level purpose is established. Encoded-state manipulation is an inferred interpretation, not a contractual requirement. Allocation, parsing, initialization, and device control are not established.

This contract distinguishes established static behavior from arithmetic deductions and unresolved execution details. Normal completion is not established.

## Interface

### Explicit and implicit inputs

The represented interface accepts two 32-bit values. Their signedness, pointer meaning, and ownership meaning are unspecified. The first dependency receives three represented arguments: the first input, the second input, and the second input again.

Additional execution inputs are required:

- An incoming control-context value determining the first indirect target.
- An incoming 32-bit payload used in early byte writes.
- An incoming address used for early selector reads.
- Primary and secondary results of dependency calls.
- Existing memory contents, including absolute low addresses and shared storage.
- A stack-resident address passed to a dependency.

These are not established additional API parameters. Their initialization and calling-convention provenance remain **documented_only**.

No return value, status code, output-buffer parameter, or explicit length is represented. The represented return follows a call through address zero; reaching it requires that call to return.

### Widths and arithmetic

Direct memory operations distinguish bytes, 16-bit halfwords, and 32-bit words. Some dependency results are represented as 64-bit quantities whose lower and upper 32-bit portions are consumed separately. This does not establish the dependencies’ actual return ABI.

Byte stores retain the low eight bits. Explicit unsigned byte reads widen to values from 0 through 255. Extension rules for other narrow dependency arguments remain unresolved.

Unsigned right shifts fill with zero. Address arithmetic is represented using 32-bit-sized operands. Unsigned arithmetic supports low-32-bit interpretation; signed multiplication and signed left shifts do not independently establish portable overflow behavior. Overflow-sensitive conclusions are **documented_only** without instruction-level evidence.

Byte order, alignment policy, and atomicity are unknown. A represented three-byte read establishes a 24-bit value dependency, not necessarily a particular physical load instruction.

Two shared locations are called **shared location A** and **shared location B** below. These are editorial labels, not source identifiers or claims of disjoint storage.

## Behavior

### Ordering and coverage

The represented body has no conditional branches, loops, explicit validation, or recovery path. Every subsequent event depends on earlier calls returning and earlier accesses completing.

A dependency boundary is observable: a dependency may inspect or change memory, including shared storage and selector bytes. Repeated reads therefore cannot be replaced with cached values unless the representation explicitly retains a snapshot.

This document specifies the access families and ordered observation boundaries described below. It is not an exhaustive call-by-call trace. Individual unresolved targets, every argument-free invocation, and every intervening indexed-store occurrence are outside that trace coverage unless explicitly described. Their omission does not assert that they are absent, interchangeable, or effect-free. An implementation-equivalence determination for the entire body requires those additional events.

### Entry and early selector operations

After the initial forwarding call and one further call, a byte is read at the incoming selector address plus one. Its unsigned numeric value, plus 0x11, determines the destination of a byte store containing the incoming payload’s low byte.

The destination is not the incoming selector address plus 0x11. It lies between absolute addresses 0x11 and 0x110.

Five dependency invocations separate this first write from a fresh read of the same selector location. The second selected write stores the low byte of the payload masked to 30 bits and shifted right by 20. The resulting stored byte contains original payload bits 20–27.

Subsequent calls consume four-scaled dependency results, sometimes repeated in multiple arguments. These argument patterns establish no allocation or sizing semantics.

An early byte read at a secondary-result-derived address plus 0x0E supplies four represented argument positions of one dependency. A following paired-result dependency precedes a selector clear using a fresh byte read at a secondary-result-derived address shifted right by nine, plus one.

Another selected clear follows a separate five-call sequence. It uses another fresh selector read, not the earlier byte snapshot.

Further early clear pairs use:

- A retained unsigned result shifted right by 18, multiplied by 0x400, then incremented to locate each selector byte.
- A retained secondary result shifted right by nine, then incremented.

Within each pair, separate call sequences intervene and each clear rereads the selector. The retained address calculation does not imply retained selector contents.

A 32-bit value at absolute address 0x30 is also used in four argument positions. It is mutable memory, not a constant. The representation does not establish whether the repeated argument expressions require one physical load or several.

### Indexed word stores and mixed-width observations

A recurring store family writes a 32-bit value at four times an index plus 0x14. Other stores use sixteen times an index plus 0x14, or 256 times an index plus 0x14. These layouts are distinct.

Early indexed behavior includes a secondary-result index being stored at its own four-scaled destination. A following dependency receives that index and a word read from a different address: a retained unsigned result shifted right by 18, multiplied by 0x400, plus 0x14, with the loaded word shifted right by 19. Later calls precede another indexed store of the retained index.

Several sequences store four times a dependency result at sixteen times that result plus 0x14, then preserve the scaled value across calls and store it through another four-scaled index. Neither the first destination nor the saved value may be substituted for the later dependency-produced index.

Other established observations include:

- A byte read at twice a retained result, widened unsigned and supplied four times to a dependency.
- A word read at a retained base plus a field shifted left by 31.
- Another byte read at a secondary-result-derived address plus 0x0E.
- A byte read at a retained base plus a secondary-result-derived displacement.
- A halfword read at the sum of a retained extracted field and a previously widened byte.
- A stack-resident address supplied in four argument positions.

For a 32-bit bit-vector interpretation, shifting a field left by 31 preserves only its low bit, producing either zero or 0x80000000. It is not unrestricted widening multiplication. A separate operand already scaled by four becomes zero when subsequently shifted left by 31 under that interpretation; overflow-sensitive machine semantics remain unresolved.

The halfword-read value is supplied four times to a dependency. The paired result subsequently determines an indexed store: the upper portion supplies the index, while a retained extracted field supplies the stored value. Calls precede a further store of that field through another index.

A retained address then receives two byte writes at offset 0x0D. The first contains the low byte of twice that address; the second contains 4. Five dependency calls intervene.

The two word reads at offset 0x44 have different address constructions:

- The first uses four times a retained unsigned result, plus 0x44. Its value supplies a dependency argument.
- After the intervening calls and the second offset-0x0D write, the second uses that same retained result masked to 30 bits, shifted right by 20, plus 0x44. There is no fourfold scaling in this second address. Its value is another dependency argument, alongside a scaled result from the intervening calls.

Later stores include four times the retained result at a 256-scaled index plus 0x14, followed by an extracted field stored through another four-scaled index. Repeated sixteen-scaled stores and later four-scaled stores remain separately observable across dependency boundaries.

### Offset fields and the clear of shared location A

A dependency-derived index determines a sixteen-scaled base. The function clears a 32-bit field at base plus 0x24, then reads a 32-bit value at base plus 0x74. That loaded value becomes a retained address.

After dependency calls, another unsigned result shifted right by 20 is stored at the retained address plus 0x24. A subsequent word read at that address plus 0x74 supplies a dependency argument.

Selected writes then use fresh bytes from the retained address plus one: first to store an extracted byte from an earlier result, and later to clear a selected destination. Calls separate those operations.

Three subsequent byte observations use secondary-result-derived addresses plus 0x15. Their values feed dependencies. Selected clears between these observations use a retained extracted field multiplied by 0x400, plus one, as the selector-read address. Each occurrence performs a fresh selector read.

Later indexed stores preserve a widened byte, dependency-derived indices, scaled values, and extracted fields across separate calls. One retained secondary result undergoes successive left shifts whose represented total scaling is 128 before being stored through an indexed destination; overflow interpretation remains subject to the width qualification above.

Shared location A is then directly cleared as a represented 32-bit value. Calls and many memory operations occur before either of its two later explicit reads. The clear does not constrain those later snapshots to zero.

### Absolute-address-one reads

The following ordered observations explicitly read the byte at absolute address 1. They are separate events, including where address arithmetic reduces to 1.

**First mid-body occurrence.** After the byte read at twice a retained result, its dependency use, and further selected clearing and calls, a fresh read at address 1 selects a byte destination to clear. Scaling-related calls follow.

**Before the clear of shared location A.** A locally retained zero makes a selector expression reduce to address 1. A fresh byte there selects the destination for an extracted-byte write. Subsequent calls and indexed stores precede the direct clear of shared location A.

**After that shared clear, before the two-level byte read.** The ordered address-one selector observations are:

1. A fresh read selects a write containing the low byte of a four-scaled dependency result.
2. After calls, another fresh read selects a write containing an extracted byte of that result.
3. After further calls, including F-000000000003, another fresh read selects a clear.
4. After a separate call sequence, another fresh read selects another clear.
5. After a byte read at a retained dependency result plus one, its dependency use, other selected clearing, and calls, a fresh address-one read selects a write containing bits 12–19 of that retained result.
6. After the second-level byte read described below and further calls, another fresh address-one read selects a zero byte write. Its stored expression is zero because its source is an unsigned byte shifted right by 12.
7. Following additional calls, address 1 is read directly as byte data, not merely as a selector. That byte is supplied to a dependency and subsequently widened into an indexed 32-bit store.

The two-level byte read first obtains a byte at a dependency-derived address plus one. A later read uses that byte’s numeric value plus one as its address. The second address lies between 1 and 256; it is not another field of the original dependency-derived object.

**After the first later snapshot of shared location A.** A value obtained by shifting that 32-bit snapshot right by 23 and then right by 23 is zero. Consequently, a later selector read using that value plus one reads absolute address 1. It selects a write containing the low byte of a different loaded word shifted right by 22.

Further selected clearing, calls, and a dependency-result use precede another fresh address-one selector read and clear.

Finally, after another call sequence, address 1 is read once more and its unsigned byte value is retained. This **late selector snapshot** immediately selects a byte clear at snapshot plus 0x11. It is then kept across calls and reused as the base of the later offset-0x74 read and offset-one write. That reuse does not reread address 1.

Other computed selector addresses may equal 1 for particular runtime values. Those conditional coincidences are not additional statically constant-address observations.

### Two distinct later reads of shared location A

The first later read occurs after an indexed store of a retained four-scaled result. Five dependency calls follow the snapshot before another indexed store of a different retained word shifted right by 22.

The snapshot is shifted right by 23. That shifted snapshot is supplied as an argument to a subsequent dependency, alongside a repeated secondary-result index and the other shifted word. A further right shift by 23 yields zero because the first shift leaves at most nine bits. This zero later determines the absolute-address-one selector access described above.

The second later read occurs much later, after the late selector snapshot, the first update of shared location B, repeated offset-0x74 transfers, and additional indexed operations.

At this second observation:

- A paired dependency result has already supplied a four-scaled lower portion stored through an index from its upper portion.
- Shared location A is freshly read and retained as an address-like snapshot.
- A dependency receives the scaled lower portion in three argument positions and the upper portion in another.
- Four further calls precede another indexed store of the retained scaled lower portion.
- Only then is a 32-bit word read at the second snapshot plus 0x14.

This later word read uses the second snapshot, not the first snapshot, not the earlier clear value, and not a fresh read of shared location A. Its low byte and later extracted bits feed subsequent selected writes.

### Shared location B, repeated transfers, and termination

After retaining the late address-one selector snapshot and clearing at snapshot plus 0x11, additional calls produce a 32-bit value. The function immediately stores that value into shared location B.

Two calls follow: one receives the just-stored value; the next has no represented explicit arguments. Only afterward does the function read a word at the retained selector snapshot plus 0x74 and store its low byte at snapshot plus one.

This is the first update of shared location B. It is distinct from the terminal packed update. The two intervening dependencies may observe or modify B, so later state cannot be inferred solely from the first store.

Further offset-0x74 transfers use four-scaled dependency-derived bases. One base is reused for two transfers with calls between them; each transfer freshly reads its word before storing the low byte at base plus one. Another transfer retains the entire loaded word for later indexed stores, including a sixteen-scaled destination and a four-scaled destination.

Other late operations include a byte clear at a retained scaled address multiplied by 0x2000, plus 9; a low-address-byte write at another retained address plus 9; a word read at that address plus 0x34 used as a dependency argument; and additional word reads at offset 0x14 with differently transformed bases.

After the second snapshot of A and its offset-0x14 read, further selected writes, calls, and result transformations precede the terminal sequence:

1. A dependency supplies a byte-sized result and a secondary-result address.
2. A zero byte is written through that address.
3. A 24-bit value is read from absolute address 9 and combined with the dependency’s byte result as the low eight bits. The composed 32-bit value is stored in shared location B.
4. A zero halfword is written at a retained unsigned result masked to 30 bits and shifted right by seven.
5. F-000000000004 is called at address zero with three zero arguments.

The halfword address ranges from zero through 0x007FFFFF and is not necessarily aligned. The terminal byte clear precedes the address-9 observation; aliasing between those locations can therefore affect the packed update.

## Side effects

Direct writes can overlap selectors, indexed fields, shared locations, stack-accessible memory, or memory accessible to dependencies. No disjointness, rollback, synchronization, atomic transaction, or reentrancy protection is established.

No allocation, release, or ownership transfer is established. Argument repetition and scaling do not justify assigning such semantics to dependencies. The stack address’s object extent, initialization, retention permission, and post-invocation lifetime remain **documented_only**.

No visible null, bounds, alignment, or target-validity checks precede accesses. Invalid memory or calls can interrupt execution after partial mutation. No portable exception, diagnostic, error code, or cleanup behavior is established.

Normal return requires all preceding accesses to complete, all executed dependencies to return, and the address-zero call to be callable and return.

## Dependencies

The IDs below identify editorial relationships, not verified identities of individual unresolved targets.

| Opaque ID | Established relationship |
|---|---|
| F-000000000001 | Entry target derived from incoming control context; receives the first input followed by two copies of the second. |
| F-000000000002 | Remaining unresolved indirect-call family. Different target constructions and displacements occur; one semantic operation must not be assumed. |
| F-000000000003 | Separately represented fixed-target dependency; receives two copies of one retained 32-bit result. |
| F-000000000004 | Terminal target at address zero; receives three zero arguments. |

Primary, secondary, narrow, and paired results are consumed according to the represented dataflow. Calls without displayed explicit arguments may still have implicit execution inputs.

No dependency contract establishes preservation of shared storage, pointed-to memory, ownership, or normal return.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Initial forwarding:** Distinct input values must reach the first three represented argument positions in first-input, second-input, second-input order.
- **Selected destination:** With selector 0x7F and payload 0x12345678, the initial write targets 0x90 and stores 0x78. The later extracted write stores 0x23, but its destination follows its fresh selector read.
- **Narrowing:** Payload 0x3FF00000 produces an extracted value of 0x3FF; the byte store retains 0xFF.
- **Different layouts:** Index 3 gives destination 0x20 under fourfold scaling and 0x44 under sixteenfold scaling.
- **Distinct offset-0x44 reads:** For retained value 0x00100000, the first address is 0x00400044; the second is 0x45. Neither may replace the other.
- **Mutable A:** If A is cleared, later becomes 0x80000000 before its first snapshot, and becomes 0x1000 before its second snapshot, the first snapshot shifted right by 23 is 0x100. The eventual second-snapshot-based word read targets 0x1014, provided intervening execution completes.
- **Retained selector:** If the late address-one snapshot is 0x20, its immediate clear targets 0x31. Changing address 1 afterward does not redirect its later word read at 0x94 or byte write at 0x21.
- **First B update:** The dependency-produced word must be stored before the two calls preceding that retained-selector transfer. Acceptance cannot require B still to contain it afterward.
- **Terminal packing:** A final byte 0xAB and address-9 value 0x123456 compose numeric value 0x123456AB. This does not specify memory byte order.
- **Terminal failure:** If address zero is uncallable and all earlier events complete, the terminal byte clear, packed B update, and halfword clear precede the failed call attempt. No rollback is established.

## Uncertainties

The following remain **documented_only**:

- Higher-level purpose and validity of the represented function boundaries.
- Complete calling convention, implicit inputs, and dependency identities.
- Dependency memory, ownership, retention, and control-flow effects.
- Overflow-sensitive instruction semantics and narrow-argument extension.
- Memory mapping, byte order, alignment, atomicity, and alias relationships.
- Exact shared-symbol extents and overlap with smaller storage descriptions.
- Whether low addresses and address zero are valid in the execution environment.
- The exhaustive event trace outside the coverage explicitly declared above.

## Evidence

Established evidence consists solely of the supplied static representation: explicit widths, argument dataflow, ordered accesses, retained snapshots, shared updates, target constructions, and represented termination.

Address bounds, byte truncations, and provably zero shifted values are arithmetic deductions under the stated width qualifications. Application-level interpretations are inference only.

No runtime tests have been performed. Successful execution, dependency behavior, memory mappings, and instruction-level semantics have not been independently verified.
