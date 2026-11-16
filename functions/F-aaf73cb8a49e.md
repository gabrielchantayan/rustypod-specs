<!-- Trusted provenance: F-aaf73cb8a49e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaf73cb8a49e

## Purpose

F-aaf73cb8a49e accepts two 32-bit inputs, performs an extensive ordered sequence of indirect invocations, updates externally addressed storage, and ends with an unresolved indirect transfer.

The supplied static representation establishes argument relationships, memory-access widths, retained values, and significant ordering constraints. It does not establish an application-level purpose. Descriptions such as parser, formatter, allocator, object initializer, or collection operation would require evidence not supplied here.

The observable behavior comprises dependency activity, direct memory reads and writes, and the terminal transfer. No explicit result value or success indication is established.

**Evidence status:** static review only. No runtime tests have been performed. Unresolved execution semantics are identified as **documented_only**, not replaced with conventional assumptions.

## Interface

### Explicit inputs and output

| Element | Established representation | Behavioral boundary |
|---|---|---|
| First input | Uninterpreted 32-bit value | Passed unchanged as the first represented argument of the initial invocation. |
| Second input | Signed 32-bit integer representation | Passed unchanged and also supplied after a six-bit left shift. No range validation is visible. |
| Result | No explicit return value | No success status, error code, or returned allocation is established. |
| Completion | Terminal indirect invocation followed by a represented return | Normal return depends on the unresolved target and the accuracy of the represented control flow. |

Neither input has an established pointer, handle, size, or domain-specific interpretation. There is no explicit buffer capacity, length parameter, or error-output location.

### Implicit inputs

Execution also depends on:

- Incoming execution-context state used to form the first indirect target.
- Dependency results represented as 32-bit words or 64-bit quantities.
- Additional machine-result channels whose calling-convention meaning is unresolved.
- Fixed-location storage, including a 32-bit read at 0x087FBBBC.
- Memory reached through dependency results, retained bases, loaded words, and unsigned bytes.

Identical explicit inputs therefore do not establish identical behavior. In particular, the extensive repeated-store region depends on a retained global input, not solely on dependency results.

### Preconditions

Reaching the represented completion requires executable indirect targets and accessible storage for every actual read and write. Required access widths are one, two, or four bytes. Computed addresses must meet the actual platform’s mapping and alignment requirements.

These are environmental preconditions, not validations performed by the function. No sufficient valid-input domain is established.

## Behavior

### Dependency sequence

The initial invocation, F-opaque-01, receives the first input unchanged, the second input unchanged, and the second input shifted left by six bits. No validation precedes it.

An extensive straight-line sequence of indirect invocations follows, documented collectively as F-opaque-02. Many have no represented arguments; others consume transformed results from preceding invocations. Some values are retained across intervening calls.

Established argument relationships include:

- A returned word shifted left by eight bits.
- The low 32 bits of a represented 64-bit result shifted left by eight, alongside its unchanged high 32 bits.
- High-result portions supplied unchanged, shifted left, or logically shifted right.
- Explicit zero arguments.
- Identical transformed values supplied in multiple argument positions.
- Mask-selected fields shifted or scaled before forwarding.

Calls without represented arguments remain behaviorally significant. They may consume implicit state or produce effects even when no explicit result is retained. Similar argument tuples do not establish identical targets or interchangeable invocations.

No direct data-memory mutation is represented in this invocation prefix. That does not imply unchanged memory: unresolved dependencies may read or modify globals, externally addressed storage, devices, or execution state.

### Widths and arithmetic

Most arithmetic and address-bearing quantities are represented as 32 bits. Represented 64-bit results supply separate low and high 32-bit portions; extracting a high portion must not be confused with shifting the low portion.

Under a 32-bit machine-bitvector interpretation:

- Left shifts and multiplication retain only the low 32 bits.
- A word shifted left by eight and then logically right by eight retains its original low 24 bits.
- Selecting the low 20 bits and shifting right by ten yields a ten-bit field.
- Selecting the low 24 bits and shifting right by 20 yields a four-bit field.
- Logically shifting a 64-bit result right by 36 yields its high word logically shifted right by four.
- Byte, halfword, and word stores retain eight, 16, and 32 bits respectively.

The representation includes masks of several widths, including 11 through 24 bits. Such masks restrict data; they do not validate addresses.

Some consecutive shifts discard every original bit under the bitvector interpretation. Treating those expressions as unlimited-width multiplication would change their result. Where represented signed shifts, signed overflow, or address arithmetic require platform semantics, the result remains **documented_only**.

A late right shift uses the low eight bits of an address-derived value as its count. Counts outside the ordinary 32-bit range, and right shifts of negative signed values, cannot be resolved from this material alone.

### First direct mutations

The direct-memory suffix begins with a word store at offset 0x14 from a high-result-derived base. A byte store at offset 9 of another retained base follows. A scaled low-result-derived destination then receives a word at offset 0x54, after which the offset-9 byte destination is written again.

The two byte stores have different represented source values. Their equality must not be assumed.

Another computed base receives a zero word at offset 0x14. Its unsigned byte at offset 5 is subsequently read and published, zero-extended, as a word at offset 0x34. A separately doubled masked value supplies a base that receives repeated zero-word stores at offset 0x14.

The initially zeroed field is read again after intervening accesses. Its loaded value participates in later addressing and byte writes. Aliasing may make that read nonzero; the earlier zero store does not justify constant substitution.

The following mutation groups include:

- Byte-sized publication of address-derived values.
- Word stores containing the destination’s own base.
- Reads through words loaded at offset 0x74.
- Address formation using an unsigned byte loaded at offset 5.
- Repeated stores and overwrites at offsets 0x14, 0x44, and 0x54.

These are address and access relationships, not an established object schema.

### Layout boundary

Offsets are byte offsets from computed bases.

| Width | Established examples |
|---|---|
| 8 bits | Offsets 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, and 0x19 |
| 16 bits | Stores at doubled or otherwise scaled addresses |
| 32 bits | Offsets 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74 |

No common allocation size, field meaning, object boundary, or non-aliasing relationship is established. An access at offset 0x74 does not prove that all bases denote the same record type.

### Overwrites and the retained global base

A computed word destination at offset 0x20 receives four successive publications: 0x20, then 0x2F, then a zero-extended byte read from offset 4 of that computed region, then 0x20 again. The byte read occurs after the first two writes. Matching only the final word would omit established intermediate accesses.

After that overwrite sequence, the function reads **32 bits at 0x087FBBBC** and retains the result as a base. The represented word store of 0x20 at fixed address 0x60 follows this read.

The source ordering is therefore significant: **the global read follows the offset-0x20 overwrite sequence but precedes the fixed word store at 0x60**. Placing the read after both operations is not supported by the supplied representation.

For the repeated region, the following descriptive terms identify retained values, not source identifiers:

- **Anchor:** an earlier word read through a base obtained from fixed-location storage.
- **Displaced base:** four times the anchor, reduced by decimal 150.
- **Publication region:** four times the displaced base.
- **Global base:** the retained 32-bit value read at 0x087FBBBC.
- **Linked region:** twice a word obtained through a byte-derived address at offset 0x74.
- **Publication payload:** four times that same word.

These relationships are conditional on the arithmetic interpretation described above.

The first publication group stores the payload as a word at offset 0x54 of eight times the anchor. It also writes the publication region’s low byte at global-base offset 0x19, the global base’s low byte at publication-region offset 0x11, and eight times the anchor as a word at global-base offset 0x34.

Subsequent repeated groups instead use eight times the displaced base for the scaled word destination and for the word published at global-base offset 0x34. Within each complete group, the represented access order is the payload word store, the offset-0x19 byte store, the offset-0x11 byte store, and the offset-0x34 word store.

The global base and the two byte values are retained values, not fresh reads from their original storage on every repetition. A write that aliases 0x087FBBBC does not, by itself, change the already retained base.

No advancing index or traversal condition is represented. Repetition must not be reinterpreted as processing distinct elements or collapsed into final-state assignments. Exact executable repetition counts remain **documented_only**.

### Late fixed-location ordering and cached values

After the repeated publications, the function reads a **32-bit word at 0xA9**. That entire word is cached. Its low byte is later used separately from the retained full word.

This read precedes both:

- A word read at offset 0x74 from the base associated with the publication payload.
- A byte write at fixed address 0x3A.

The offset-0x74 result is also cached for later low-byte publications. The byte written at 0x3A is derived from eight times the displaced base plus 0x30, with byte truncation; it is not the cached byte from 0xA9.

Before publishing the cached low byte to 0x67, the function also caches words from linked-region offset 0x14 and publication-region offset 4, and performs intervening byte and word mutations. These include writing 0x6C at publication-region offset 8, publishing retained address bytes, and storing the linked-region base at its own offset 0x54.

The cached low byte of the earlier 0xA9 word is then written to fixed address **0x67**. An indirect word read from linked-region offset 0x44 intervenes before a **fresh byte read of 0x67** supplies the byte written to fixed address **0x21**.

The indirect word read is not interchangeable with the earlier cache operation. Whether that access has device effects, aliases relevant storage through the actual mapping, or observes external mutation is unresolved.

Two later byte stores to publication-region offset 9 each obtain their value from another **fresh read of 0x67**:

- The first follows several mutations involving the global base and publication region, including a word publication at publication-region offset 0x10.
- Between the first and second reads, the function writes a retained byte at linked-region offset 9, writes the cached **full 32-bit 0xA9 word** at publication-region offset 0x44, and reads a word from publication-region offset 0x54.

Those intervening operations may overlap or otherwise affect fixed-location storage. The later 0x67 reads must not be replaced with the original cached low byte without an additional equivalence premise.

Thus the contract distinguishes three sources: the cached full word from 0xA9, its cached low byte used to populate 0x67, and the separate fresh observations of 0x67.

### Final mutations and transfer

The late suffix continues to overwrite byte and word fields. Notable retained-value relationships include:

- The cached linked-region offset-0x14 word is eventually restored at linked-region offset 0x54 and is also retained for the terminal transfer.
- The cached offset-0x74 result supplies later low-byte stores at offsets 0x0D and 0x11.
- A fresh word read from publication-region offset 0x54 becomes a base for further mutations.
- A word read through linked-region offset 4 supplies a later base. A halfword value 0x65 is written at twice that base.
- Decimal 100 is written as a word at offset 0x54 of another loaded base.
- Late reads at offsets 0x64 and 0x74 supply further stores and terminal arguments.

Constants also include 0x3A in byte, halfword, and word stores at different scaled destinations. Their correspondence to character encodings does not establish text output, strings, or termination conventions.

The terminal transfer, F-opaque-03, has four represented arguments:

1. 0x72.
2. A retained word read from offset 0x64 of the later base obtained through linked-region offset 4.
3. The cached earlier word from linked-region offset 0x14.
4. A later word read from offset 0x74 of that same later base.

Intervening stores do not turn these cached arguments into fresh reads. Target identity, normal return, and ordinary-call versus other indirect-transfer semantics remain unresolved.

## Side effects

### Memory and shared state

Direct accesses use eight-, 16-, and 32-bit widths. Fixed-location effects include byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67, and word writes at 0x54 and 0x60.

Meaningful fixed-location inputs include the 32-bit words read at 0x087FBBBC and 0xA9, fresh byte reads of 0x67, and storage at 0x05 used in address formation. The supplied material does not independently establish the width or interpretation of every symbolically represented fixed-location access.

The representation identifies overlapping global storage views. Distinct symbolic views must not be assumed independent. Computed accesses may also overlap fixed locations or each other.

The listed order describes the represented access sequence, not a guarantee that concurrent observers see every access in that order. No atomicity, locking, memory barrier, or reentrancy guarantee is established.

### Ownership and lifetime

No allocation, release, reference-count operation, or ownership transfer is established.

Passing address-like values to dependencies does not prove borrowing, retention, or transfer of responsibility. Storing an address-like word does not establish ownership of the referenced storage.

Storage must remain usable through each actual access. Dependency retention beyond an invocation is **documented_only**.

### Errors

No explicit null check, bounds check, error return, rollback, or recovery path is represented. A fault or dependency failure may leave earlier effects observable. No particular fault delivery mechanism or error code is established.

## Dependencies

These opaque IDs identify documentation roles, not verified distinct functions.

| ID | Role | Unresolved properties |
|---|---|---|
| F-opaque-01 | Initial execution-context-derived invocation | Identity, calling convention, implicit inputs, effects, result channels |
| F-opaque-02 | Remaining indirect invocation sites before direct mutation | Number of distinct targets, signatures, memory effects, ownership, return behavior |
| F-opaque-03 | Terminal indirect transfer | Identity, argument interpretation, completion and transfer semantics |

F-opaque-02 denotes a family of invocation sites, not one established shared helper.

No dependency is assumed to preserve globals, preserve externally addressed memory, release storage, retain arguments, or return normally.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Initial forwarding and shift width

If the second input is 3, the initial invocation receives 3 unchanged and 192 as its shifted argument. The first input remains unchanged.

Under 32-bit bitvector semantics, an input bit pattern of 0x04000000 produces zero after the six-bit left shift. A wider result of 0x100000000 would violate that interpretation.

### Unsigned byte publication

If the relevant offset-5 byte read yields 0xFE, its zero-extended word publication is 254, not −2.

### Intermediate overwrite visibility

For observable storage, the offset-0x20 trace includes 0x20, 0x2F, the byte-derived word, and 0x20 again. Final-state equality alone does not establish trace equivalence.

### Global input and retained addressing

If the 32-bit read at 0x087FBBBC yields an accessible base, repeated byte publications at offsets 0x19 and word publications at offset 0x34 use that retained base.

The read follows the overwrite sequence and precedes the fixed word write at 0x60. Hoisting it to entry or delaying it until after that write requires additional evidence.

If subsequent writes change storage at 0x087FBBBC, the cached base remains the previously read value.

### Cached word versus fresh fixed-byte reads

If the 0xA9 word is 0x123456AB, the later byte publication to 0x67 uses 0xAB, while the full-word publication at publication-region offset 0x44 uses 0x123456AB.

If the fresh 0x67 read before the 0x21 write observes 0xCD, then 0x21 receives 0xCD, not necessarily 0xAB.

If the two still-later reads of 0x67 observe 0x11 and 0x22, the corresponding publication-region offset-9 stores use those respective bytes. Such observations are conditional on aliasing, device behavior, or external mutation permitting them.

### Aliasing and partial failure

An intervening overlapping byte write can change a previously zeroed word before its later read. Acceptance must account for that write rather than require zero.

If a later destination is inaccessible, earlier completed mutations need not be restored.

### Halfword and terminal boundaries

The 0x65 halfword store affects two bytes; a byte-only store is not equivalent. Byte order remains platform-dependent.

Reaching F-opaque-03 establishes its first argument as 0x72 and the retained-value relationships above, not successful completion.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended input domain.
- Dependency identities, complete signatures, and implicit result channels.
- Calling convention, byte order, alignment, and actual address arithmetic.
- Signed overflow and shift semantics, especially the late variable-count shift.
- Validity and device or ordinary-memory meaning of fixed addresses.
- Object boundaries, aliasing, and field interpretation.
- Executable fidelity and exact dynamic counts of the extensive represented invocation and repeated-store sequences.
- Allocation, ownership, release, and argument-retention behavior.
- Terminal-transfer semantics and normal-return reachability.

These unknowns prevent certification of a complete executable replacement. They do not justify suppressing accesses, substituting cached values for fresh reads, or inventing safe defaults.

## Evidence

Review was limited to the supplied static representation of implementation material. No independent binary inspection, dependency resolution, or runtime execution was performed.

Established static evidence includes input widths, initial forwarding, indirect invocation relationships, memory-access widths, repeated overwrites, the ordered 0x087FBBBC read, retained global-base use, the cached 0xA9 word, fresh 0x67 reads, and terminal argument dependencies.

**[INFERENCE]** The suffix publishes relationships among computed and loaded address-like values. No stronger domain-specific intent is established.

No runtime tests have been performed. No snapshot hash is asserted.
