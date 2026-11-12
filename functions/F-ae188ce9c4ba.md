<!-- Trusted provenance: F-ae188ce9c4ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae188ce9c4ba

## Purpose

The visible routine interleaves indirect calls with selector-dependent byte writes, 32-bit word accesses, and two assignments to global storage whose declared width is unavailable. Dependency results supply later arguments, stored values, and address calculations.

No application-level purpose is established. The accessed locations are not demonstrated to be allocated objects, valid buffers, device registers, or mapped memory.

**Evidence status: documented_only.** The supplied static representation reports invalid instruction data and terminates at a truncated control-flow boundary. The contract below describes its visible ordering and data relationships, conditional on execution reaching each operation. It does not establish that the whole path is executable.

## Interface

### Explicit inputs and termination

Two explicit 32-bit inputs are represented: **first input** and **second input**. Their signedness and domain meanings are unspecified.

The first dependency receives three explicit arguments, ordered as:

1. Second input.
2. Second input again.
3. First input.

No ordinary return or meaningful return value is demonstrated. The terminal invalid-instruction boundary is not evidence of an intentional application-level termination operation.

### Additional incoming state

Three additional incoming machine-state values affect behavior:

| Descriptive name | Established use |
|---|---|
| Entry base | Signed 32-bit value used in entry-relative address calculations |
| Entry payload | Unsigned 32-bit value supplying two byte-write values |
| Entry call anchor | Value used to calculate the first indirect target |

Their provenance and entry calling convention are unknown. The two explicit inputs are therefore insufficient to determine the represented behavior.

### Captured values

These labels describe values, not recovered object types:

| Label | Meaning |
|---|---|
| Initial word | Unsigned 32-bit result captured from F-U02 |
| Selector | Unsigned byte reread at entry base plus 1 after F-U07 |
| Small base | Twice selector; range 0–510 |
| Narrow fragment | Low eight bits of initial word logically shifted right by 3 |
| Transform result | Signed 32-bit result captured from F-U20 |
| Scale seed | Signed 32-bit result captured from F-U25 |
| Linked base | Unsigned 32-bit word read at small base plus 116 |
| First pair | Low and high 32-bit portions of F-U35’s represented 64-bit result |
| Saved scale | First pair’s low portion, interpreted as signed 32-bit, shifted left by 2 |
| Second pair | Low and high 32-bit portions of F-U37’s represented 64-bit result |
| Index result | Signed 32-bit result captured from F-U42 |
| Late word | Unsigned 32-bit result captured from F-U47 |
| Late base | Late word shifted left by 2 as unsigned 32-bit, then assigned to signed 32-bit storage |

### Width and arithmetic rules

Explicit byte reads are unsigned, producing values from 0 through 255. Explicit byte-pointer writes retain the low eight bits. Explicit word-pointer accesses are 32 bits; their byte order is unknown.

Unsigned 32-bit arithmetic, including left shifts, follows modulo-$2^{32}$ value arithmetic in the supplied representation. This does not establish the physical address width or whether the resulting address is accessible.

Signed 32-bit shifts, products, and additions require separate care. Negative left-shift operands and results outside the signed range do not have a portable interpretation established here. Converting an out-of-range unsigned value to signed 32-bit likewise requires a target-specific rule not supplied by the evidence. Such cases remain **documented_only**, rather than being silently assigned wrapping semantics.

Two assignments address global storage identified at address 13. Its declaration is absent. Neither its width nor its complete written byte range is established by the byte-valued first assignment or by the second assignment’s value, 192.

## Behavior

### Ordering convention

F-U01 through F-U65 identify **call occurrences in visible order**, not independently recovered function identities. Multiple occurrences may target the same implementation.

“Empty” means no explicit arguments are displayed. It does not exclude access to registers, globals, memory, or other implicit state.

Each phase is conditional on all preceding calls returning and all preceding accesses succeeding. Calls with unused results remain observable dependencies and cannot be omitted.

### Entry write and initial calls

The first observable data access reads an unsigned byte at entry base plus 1. Its value plus 17 is the destination address for a byte write containing the low eight bits of entry payload.

This destination lies between addresses 17 and 272. It is not entry base plus that offset.

The following calls then occur:

| Occurrence | Explicit arguments | Captured result |
|---|---|---|
| F-U01 | Second input, second input, first input | None |
| F-U02 | Empty | Initial word |
| F-U03 | Initial word | None |
| F-U04–F-U06 | Empty, in increasing order | None |
| F-U07 | Empty | Signed 32-bit scaling input |

### Selector capture and first global assignment

After F-U07, entry base plus 1 is read again. This fresh byte becomes selector. A byte write to selector plus 17 contains the low eight bits of entry payload logically shifted right by 22.

F-U08 receives F-U07’s result shifted left by 2. F-U09 through F-U12 follow with empty explicit argument lists.

Before F-U13:

- Small base is formed from the captured selector.
- Narrow fragment is written as one byte at entry base plus 13.
- Selector is assigned to the global storage at address 13, using that storage’s unknown destination width.

F-U13 then receives small base, 152, and 608, in that order.

The initial selector read and the post-F-U07 read are distinct. Dependencies may change the location between them. After capture, small base and later selector-based operations use the captured selector rather than another entry-relative read.

### Secondary result state and transformation

The next call group is:

| Occurrence | Explicit arguments | Captured result or state |
|---|---|---|
| F-U14 | Empty | A secondary result-register value is observed afterward |
| F-U15 | That unsigned 32-bit observation logically shifted right by 3 | None |
| F-U16–F-U18 | Empty, in increasing order | None |
| F-U19 | Empty | Signed 32-bit result |
| F-U20 | F-U19’s result shifted left by 2 | Transform result |

The secondary register observation is not established as a second language-level return value of F-U14.

After F-U20 returns, narrow fragment is written as one byte at small base plus 13. The value 192 is then assigned to the global storage at address 13. This assignment’s width remains unknown.

F-U21 receives transform result shifted left by 2. F-U22 through F-U24 follow with empty argument lists. F-U25 then returns scale seed.

The two writes before F-U21 are separate from the earlier entry-relative write and global assignment. Intervening calls may observe or alter their destinations.

### Small-base and linked-base word accesses

The next direct write stores scale seed shifted left by 2 as a 32-bit word at small base plus 36. A subsequent 32-bit read at small base plus 116 captures linked base.

A second scale value is formed by shifting scale seed left by 5. F-U26 receives:

1. Linked base.
2. Linked base shifted left by 1 using unsigned 32-bit arithmetic.
3. The second scale value.
4. Linked base masked with `0x7fffffff`, then logically shifted right by 2.

F-U27 through F-U30 follow with empty argument lists. The second scale value is then written as a 32-bit word at linked base plus 36.

A fresh 32-bit read at linked base plus 116 supplies the sole explicit argument to F-U31. F-U32 through F-U34 follow with empty argument lists.

Linked base remains a captured numeric value across these calls. Its capture does not establish memory validity or prevent dependencies from modifying, freeing, or invalidating any associated storage.

Small base plus 36 and small base plus 116 are four-byte aligned exactly when selector is even. No alignment handling is represented.

### Paired results and indexed stores

F-U35 is invoked with an empty argument list and produces a represented 64-bit result. Its high portion is treated as unsigned 32-bit. Its low portion is interpreted as signed 32-bit when forming saved scale.

Before F-U36, initial word logically shifted right by 3 is written as a full 32-bit word at the address formed by four times the first pair’s high portion, plus 20. That address expression uses unsigned 32-bit arithmetic.

F-U36 receives saved scale three times, followed by the first pair’s high portion logically shifted right by 3.

The remaining group is:

| Occurrence | Explicit arguments | Captured result |
|---|---|---|
| F-U37 | Empty | Second pair |
| F-U38 | Second pair’s low portion as signed 32-bit; high portion as unsigned 32-bit; low portion again; high portion logically shifted right by 3 | None |
| F-U39–F-U41 | Empty, in increasing order | None |
| F-U42 | Empty | Index result |

After F-U42, saved scale is written as a 32-bit word at four times index result plus 20. F-U43 then receives four times index result. These calculations use signed 32-bit operands; problematic products or additions require an independently established execution rule.

F-U44 through F-U46 follow with empty argument lists.

The store derived from initial word preserves the full logical-shift result, potentially containing 29 significant bits. It is not another narrow-fragment byte store.

Neither represented 64-bit result proves that its dependency returns a single language-level 64-bit value rather than multiple ABI-level result components.

### Late selector-dependent writes

F-U47 returns late word. Its unsigned left shift by 2 supplies the value assigned to late base.

An unsigned byte is read at linked base plus 1. That byte plus 17 identifies the destination of a byte write containing the low eight bits of late base.

F-U48 and F-U49 are invoked with empty argument lists, followed by F-U50, whose signed 32-bit result is captured.

Linked base plus 1 is then read again. This fresh byte plus 17 identifies a second byte-write destination. The written value is the low eight bits of late word masked with `0x3fffffff` and logically shifted right by 20.

F-U51 receives F-U50’s result shifted left by 2. F-U52 through F-U54 follow with empty argument lists. F-U55 then returns a signed 32-bit result.

The two linked-base-relative reads cannot be merged: F-U48 through F-U50 intervene, and the first write may also alias later-read storage.

### Repeated indirect clearing and terminal boundary

A byte is read at the address formed by late base multiplied by 1,024, plus 1. A zero byte is written at that byte’s value plus 17.

F-U56 receives, in order:

1. Late base shifted left by 3.
2. The same shifted value again.
3. Late base.
4. F-U55’s result shifted left by 2.

F-U57 through F-U59 follow with empty argument lists. F-U60 then returns a signed 32-bit result.

The byte at late base multiplied by 1,024, plus 1 is read again. Another zero byte is written at this fresh byte’s value plus 17.

F-U61 receives F-U60’s result shifted left by 2. F-U62 through F-U65 follow with empty argument lists.

The two clearing writes are distinct events. Their destinations may differ because calls intervene between fresh selector reads. The signed arithmetic used for these read addresses and shifted arguments remains subject to the stated arithmetic limitations.

The visible path ends after F-U65 at the invalid-instruction boundary. No subsequent behavior is established.

## Side effects

### Direct access inventory

The visible path contains **fourteen store assignments**:

| Store category | Count | Established destination width |
|---|---:|---|
| Explicit byte-pointer writes | 8 | Eight bits each |
| Explicit word-pointer writes | 4 | 32 bits each |
| Assignments to global storage at address 13 | 2 | Unknown |

The eight explicit byte writes comprise two early selector-derived writes, two narrow-fragment writes, two late linked-selector-derived writes, and two clearing writes.

If an independently supplied declaration establishes that the global destination is byte-sized, the total becomes **ten byte stores and four 32-bit stores**. That declaration is not supplied here.

There are **eight explicit memory reads**:

- Six unsigned byte reads: two entry-relative, two linked-base-relative, and two late clearing-selector reads.
- Two 32-bit word reads: one small-base-relative and one linked-base-relative.

These totals exclude dependency internals, instruction fetches, and implicit calling-convention activity.

### Aliasing and visibility

The six byte writes whose destinations are obtained from selector bytes target addresses 17–272. The other two explicit byte writes use entry base plus 13 and small base plus 13.

Both global assignments begin at the storage identified at address 13, but their complete footprints are unknown. In particular, without its declaration, overlap analysis cannot treat this storage as a single byte. A wider destination could overlap nearby addresses that a byte-sized destination would not.

Other accesses may alias each other, the global storage, or dependency-visible memory. No distinct-object guarantee, volatility rule, atomicity guarantee, or concurrency model is established.

### Ownership and errors

No explicit allocation, release, ownership transfer, reference-count operation, or lifetime extension is visible. Passing an address-like integer does not establish ownership or retention rights. Dependencies may nevertheless allocate, release, retain, or invalidate resources.

No explicit bounds checks, null checks, alignment checks, recoverable error branches, rollback, or error result are represented.

A faulting access or nonreturning dependency prevents the subsequent visible sequence from being established. Earlier writes may already be observable. No transactional behavior is supported.

## Dependencies

F-U01 through F-U65 are opaque dependency-occurrence IDs. Their identities, prototypes, memory effects, ownership rules, and failure conventions are unknown.

The first target depends on incoming call-anchor state. Later targets are represented by embedded address calculations. Their executable validity and distinctness are unproved.

The contract does not assume that dependencies:

- Preserve globals or addressed memory.
- Return successfully.
- Interpret integer arguments as pointers.
- Preserve resource lifetimes.
- Implement prototypes matching the displayed explicit argument lists.
- Are removable because no result is captured.

Captured scalar values and later memory reads are intentionally distinguished. Unknown dependency effects require preserving fresh reads; they do not justify inventing particular mutations.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Argument order.** For distinct explicit inputs, F-U01 receives the second input twice and then the first. Reversing that order violates the represented contract.

2. **Initial truncation.** If the first entry-relative byte is 5 and entry payload is `0x12345678`, the first write stores byte `0x78` at address 22, assuming permitted addressing.

3. **Selector recapture.** If that initial byte is 5 but its post-F-U07 value is 9, the second selector-derived write targets address 26 and small base is 18. The original value does not determine these later operations.

4. **Byte versus word value.** For initial word `0x12345abc`, the logical right shift by 3 is `0x02468b57`. The two narrow-fragment writes contain byte `0x57`; the later first-pair-addressed word write contains `0x02468b57`.

5. **Global width remains unresolved.** With selector 9, the first global assignment supplies value 9 and the later assignment supplies 192. An eight-bit declaration would make these two additional byte stores. Without that declaration, neither their widths nor their complete byte footprints are accepted as established.

6. **Maximum selector and alignment.** Selector 255 gives small base 510. Its fragment write uses address 523; its word write and word read use addresses 546 and 626. Those word addresses are not four-byte aligned. Successful access depends on the environment.

7. **Unsigned linked-base operations.** For linked base `0x80000004`, F-U26’s second argument is `0x00000008` under unsigned 32-bit shifting, and its fourth argument is 1. This establishes argument values, not pointer validity.

8. **Fresh late selectors.** If linked base plus 1 reads as 2 before F-U48 and as 7 after F-U50, the late writes target addresses 19 and 24.

9. **Distinct clearing destinations.** If the two clearing-selector reads yield 4 and 8, the zero-byte writes target addresses 21 and 25. Successful signed address arithmetic and permitted accesses are prerequisites.

10. **Partial execution.** If F-U20 does not return, the following small-base fragment write and global assignment of 192 are not reached. Earlier effects are not rolled back.

## Uncertainties

The following remain **documented_only**:

- Whether the visible path corresponds to executable instructions.
- Complete entry ABI and provenance of additional incoming state.
- Identities and contracts of all dependency targets.
- Meaning and producer of the secondary register observation after F-U14.
- ABI interpretation of the two represented 64-bit results.
- Declaration, width, representation, and full footprint of the global storage at address 13.
- Target-specific signed overflow, shifting, and unsigned-to-signed conversion behavior.
- Physical address width, pointer conversion rules, and address validity.
- Byte order, alignment requirements, and device-memory semantics.
- Resource lifetimes and dependency-induced memory changes.
- Intended object layouts and application purpose.
- Actual behavior at or beyond the terminal invalid-instruction boundary.

## Evidence

Established static evidence supports two explicit 32-bit inputs, three additional incoming-state dependencies, and 65 ordered indirect call occurrences.

The direct access inventory is **fourteen store assignments**—eight explicitly byte-sized pointer writes, four explicitly 32-bit pointer writes, and two global assignments of unknown width—and **eight explicit reads**—six unsigned byte reads and two 32-bit word reads.

The representation also supports the stated truncations, unsigned shifts, masking, captured-value relationships, fresh-read boundaries, and invalid-instruction warnings.

Terms such as “selector,” “linked base,” and “scale seed” are explanatory labels. They do not establish domain intent, object identity, or ownership.

No runtime tests have been performed. No successful execution, dependency behavior, memory validity, or ordinary return is claimed.
