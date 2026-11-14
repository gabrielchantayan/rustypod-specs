<!-- Trusted provenance: F-d1ceca64c67c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d1ceca64c67c

## Purpose

F-d1ceca64c67c is a void-returning routine with globally controlled dispatch. Its represented operations include two acquisition call sites, a processing call site, byte transformations and comparisons, and publication of two 32-bit words into caller-supplied storage.

“Acquisition,” “processing,” and “publication” describe inferred roles, not established algorithms or success conditions. Missing global contents prevent establishing a normal execution sequence, mandatory call ordering, a fixed comparison length, or termination.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, assignments, dependency arguments, and arithmetic relationships, conditional on reaching the relevant operation.
- **Inferred intent:** descriptive interpretations of those operations.
- **documented_only:** behavior unresolved without global contents, dependency contracts, or target execution semantics.

No runtime execution or tests have been performed.

## Interface

### Arguments and return

| Argument | Represented form | Established use |
|---|---|---|
| Destination address | Signed 32-bit integer | Base for two conditional 32-bit stores |
| Descriptor address | Pointer to 32-bit words | Source of four descriptor fields |

The incoming arguments are retained at entry. Separate active destination and descriptor holders initially contain zero and null. State `0x1b` installs the retained arguments into those active holders.

Consequently, valid incoming arguments do not alone establish valid accesses: operations reached before installation can use the initial zero or null active holders.

The routine has no explicit return value. State `0x18` returns without an additional output write or status publication.

### Descriptor layout

| Byte offset | Read width | Use |
|---|---:|---|
| 0 | 32 bits | Transformed sixth argument to F-adc12a38f159 |
| 4 | 32 bits | Transformed second argument to F-adc12a38f159 |
| 16 | 32 bits | Third argument at either F-d701d8557b58 call site |
| 20 | 32 bits | Transformed second argument at either F-d701d8557b58 call site |

No direct reads at offsets 8 or 12 are represented. Descriptor fields are read when their operations execute, not captured as an entry snapshot.

### Destination layout

Publication performs separate stores in this order:

| Byte offset | Write width | Value |
|---|---:|---|
| 40 | 32 bits | Low word of an unsigned 64-bit product |
| 44 | 32 bits | Zero, after reducing the represented unsigned expression |

There is no established atomic 64-bit publication or unconditional initialization of these fields.

### Local storage and ownership

The routine supplies dependencies with addresses of three local arrays: 256 bytes, 2,048 bytes, and 20 bytes. None has explicit initialization.

Two distinct local signed 32-bit in/out words initially contain zero. Each acquisition call site overwrites its corresponding word from a global immediately before calling its dependency.

These objects have automatic lifetime. Their addresses do not establish ownership transfer, permission to retain pointers after return, or dependency compliance with the actual capacities. The routine represents no explicit allocation or release.

### Arithmetic terminology

The following are descriptive contract labels, not implementation identifiers:

- **A:** signed 32-bit value captured from a global at entry.
- **B:** unsigned 32-bit value captured from another global immediately before A.
- **Scratch:** unsigned 32-bit shared arithmetic holder, initially zero.
- **Carrier:** another unsigned 32-bit shared holder, initially zero.
- **Index:** signed 32-bit shared holder, initially zero.
- **Retained processing value:** unsigned 32-bit holder, initially zero and assigned a processing-derived value only in state `0xe`.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). The representation also includes signed 32-bit operations, byte reads, signed and unsigned 16-bit conversions, and explicit unsigned 64-bit multiplication.

Signed overflow, signed right shifts, and out-of-range conversions to signed values remain **documented_only** where target semantics are required. Algebraic descriptions below do not override those qualifications.

## Behavior

### Dispatch and initialization

An initial dispatch value is read from a global. Each dispatch selection adds a separately read global adjustment. States either select a new global-derived dispatch value directly or calculate a choice between global-derived alternatives.

Unknown global values leave state reachability, ordering, repetition, and termination unresolved. Dependencies are not assumed to preserve those globals.

State `0x1b` installs the active arguments and establishes byte positions:

- Positions 0, 1, 2, and 3 for the first four-byte operation.
- Positions 4, 5, 6, and 7 for publication.

All eight position holders initially contain zero. An operation reached before installation can therefore read the same position repeatedly rather than four consecutive bytes.

A separate active byte-source pointer initially contains null. States `1` and `0xb` assign it the 256-byte array address. Installing the arguments does not assign this pointer.

An unrecognized dispatch selection performs the first acquisition operation, just as state `3` does. It does not inherently reject the selection or return an error.

### First acquisition

State `3`, including the default selection, initializes the first in/out word from a global. It then reads descriptor offset 20, transforms it using A and B, and places that transformed value in Carrier before the call.

F-d701d8557b58 receives:

1. Address of the 256-byte array.
2. The A-and-B-scaled word from descriptor offset 20.
3. The word from descriptor offset 16.
4. Address of the first in/out word.
5. A global-derived argument.
6. The value 1.

The offset-20 preparation precedes the call. A total order among the remaining call-argument evaluations is not established.

The return is multiplied by a newly read global factor. An additionally A-and-B-scaled form is retained in acquisition-related arithmetic storage. Dispatch uses the high bit of a complemented bitwise combination of two scaled forms of the intermediate return.

No return value is established as success or failure. The in/out word’s meaning and the number of initialized array bytes are unknown.

### First four-byte operation

State `7` reads four bytes from the active byte source. With installed positions, their order is 0, 1, 2, then 3.

Each byte first undergoes multiplication by 249, addition of 116, and reduction to eight bits. Further global-dependent arithmetic includes signed 16-bit conversions, complements, bitwise combinations, and contributions scaled by \(2^{24}\), \(2^{16}\), \(2^8\), and 1.

The complete transformation is not established as an ordinary endian decode. Its B-scaled result is compared with 1:

- A nonmatching result selects a global-derived continuation.
- A matching result enables a further predicate involving the first in/out word, globals, and 256.

State `7` overwrites Scratch, Carrier, and Index with intermediate values. Carrier is not reserved for comparison-index updates.

State `0x14` computes another predicate from the first in/out word and globals, overwriting Scratch and Carrier. Neither operation is established as a conventional bounds check.

### Second acquisition

State `0xa` initializes the second in/out word from a separate global. It assigns Scratch an offset-like expression derived from the first in/out word and globals.

It then rereads descriptor offset 20, assigns its A-and-B-scaled value to Carrier, and calls F-d701d8557b58 with:

1. Address of the 2,048-byte array.
2. The newly transformed descriptor word from offset 20.
3. A newly read descriptor word from offset 16.
4. Address of the second in/out word.
5. A separate global-derived argument.
6. The value 1.

The return undergoes global scaling, retention, and a dispatch calculation analogous in structure to the first acquisition.

Scratch is not a persistent acquisition offset. States `7`, `0x14`, `0x16`, `0x17`, and `0x19` also overwrite it. A later consumer receives the acquisition-derived value only if the intervening path preserves it.

### Pointer establishment and persistence

State `5` assigns the 20-byte array address to a source-pointer holder. It also computes and stores a region pointer from the 256-byte array base plus the unsigned 32-bit product of B and the Scratch value present at that execution. The same product is copied into separate arithmetic storage.

The stored region pointer is a snapshot of that computation. **Subsequent Scratch assignments do not change it.** A replacement Scratch value affects region construction only if state `5` executes again.

Safe use requires the constructed pointer and later indexed addresses to satisfy the target’s pointer-arithmetic and memory-access requirements. No represented check establishes this.

The source-pointer holder initially contains null. Its only represented non-null assignments occur in states `5` and `0xe`.

### Processing call and retained value

State `0xe` assigns the source-pointer holder the 20-byte array address. It reads descriptor offset 0 and places its A-and-B-scaled value in Carrier before calling F-adc12a38f159.

The seven arguments are:

| Position | Value |
|---|---|
| 1 | Second in/out word multiplied by a global factor |
| 2 | Descriptor word at offset 4 multiplied by A and B |
| 3 | Address of the 20-byte array |
| 4 | Address of the 2,048-byte array |
| 5 | Address of the 256-byte array |
| 6 | Transformed descriptor word from offset 0 |
| 7 | Current Scratch multiplied by another global factor |

Argument seven independently consumes Scratch at this execution. It is not tied to the Scratch value previously used to establish the region pointer. The remaining argument expressions have no established total evaluation order.

For dependency return R, the routine first forms an A-scaled intermediate \(T=A\times R\). The retained processing value is then assigned the represented-width form of \(A\times B\times T\), containing two factors of A overall.

Dispatch instead uses T in a complemented bitwise combination of \(B\times T\) and another globally scaled form of T. It does not use the fully scaled retained value for that predicate.

Only state `0xe` assigns the retained processing value. It remains zero if that state has not executed, and otherwise holds the value produced by its latest execution that reaches the assignment.

### Comparison-source selection

State `0x17` computes an Index-related predicate, overwrites Scratch, and copies the current source-pointer holder into a separate comparison pointer.

This is a pointer snapshot:

- After state `5` or `0xe`, it selects the 20-byte array.
- Before either assignment, it copies null.
- A subsequent source-holder assignment does not update an already copied comparison pointer.

The comparison pointer initially contains null. Constants related to 19 and 21 appear in the predicate, but no established invariant makes this a 20-iteration loop.

### Comparison reads and index updates

State `0x16` reads a byte through the comparison pointer at displacement \(B\times\text{Index}\), evaluated with unsigned 32-bit multiplication. After transforming that byte, it assigns the represented sum \(\text{Index}+A\) to Carrier.

It then reads a byte through the stored region pointer at the same scaled displacement. Thus the comparison-source read precedes the Carrier assignment, which precedes the region-source read.

Both bytes undergo arithmetic transformations. Their transformed difference contributes a high-bit dispatch predicate. Scratch is also overwritten.

State `9` assigns Index from the current Carrier multiplied by A and B. This product uses unsigned 32-bit arithmetic before conversion to signed Index.

If Carrier still contains the sum from state `0x16`, the update is:

\[
\text{Index}_{\mathrm{new}}
=
A\times B\times(\text{Index}_{\mathrm{old}}+A),
\]

subject to the signed sum’s semantics, unsigned wrapping, and final signed conversion.

Carrier can instead contain an acquisition argument, processing argument, byte-transformation result, or predicate intermediate. State `9` consumes whichever value reaches it.

State `0xd` resets Index to zero. Other operations reuse Index for non-comparison intermediates. No established invariant guarantees unit increments, monotonic progression, fixed comparison count, equality semantics, mismatch termination, or constant-time execution.

### Publication

State `0x19` reads four bytes from the active byte source. With installed positions, the order is 4, 5, 6, then 7.

For the first byte, multiplication by 249 and addition of 116 are not immediately reduced to eight bits. One subsequent contribution uses the unreduced value; another uses a signed 16-bit conversion. The other three bytes have explicit eight-bit reduction before further transformations.

Global-dependent arithmetic produces an unsigned 32-bit expression. Its multiplication by B is explicitly unsigned 64-bit.

The product’s low word is stored at destination offset 40. The following store at offset 44 evaluates to zero, not the actual high product word.

For unsigned high and low words \(h\) and \(l\):

- Twice \(h\), plus the complemented-and-shifted contribution, equals \(2^{32}-1\) modulo \(2^{32}\).
- The indicators for \(l>2^{32}-3\) and for the wrapping value \(l+2>1\) sum to exactly one.
- The resulting stored word therefore wraps to zero, including at both upper low-word boundary values.

Publication overwrites Scratch, Carrier, and Index during its transformations. After both destination stores, it restores the **latest retained processing value, or initial zero if state `0xe` has not executed**, into dispatch-related arithmetic storage.

Publication then continues dispatch. It is not itself a return.

### Other arithmetic states

The remaining represented states load, clear, scale, combine, or copy arithmetic holders before continuing dispatch. Some rescale acquisition-related results or replace the dispatch-related arithmetic value.

These operations do not establish a conventional success path. They can affect later predicates, and subsequent arithmetic states can replace the value restored after publication. They do not assign the retained processing value or recompute the stored region pointer.

## Side effects

Direct effects include global reads, descriptor reads, local-memory accesses, dependency calls, and the two conditional destination stores.

A and B are entry captures. Other globals are read at their represented uses, including after dependency calls. Dependency preservation of global values is not assumed.

No direct logging, file access, allocation, release, explicit exception mechanism, or caller-memory cleanup is represented. Dependency effects remain unknown.

The routine does not explicitly validate nullness, alignment, accessible lengths, initialized contents, or pointer ranges. Defined execution requires appropriate active pointers, accessible descriptor fields and destination words, initialized bytes at every read, valid scaled addresses, and dependency writes within actual capacities.

Descriptor/destination aliasing is not rejected. Earlier destination stores or dependency effects can influence later reads when storage aliases or dependencies modify reachable memory. No transaction, rollback, all-or-nothing publication, or guaranteed error result is established.

## Dependencies

### F-d701d8557b58

Two six-argument call sites supply different local arrays and distinct initialized 32-bit in/out-word addresses. Both supply descriptor-derived arguments and final argument 1.

The routine consumes each return and can later consume modified in/out-word contents.

**documented_only:** argument units, buffer directions, required capacities, write extents, status conventions, ownership, pointer retention, global mutations, and failure behavior.

### F-adc12a38f159

One seven-argument call site supplies all three local arrays, transformed descriptor fields, a transformed second in/out word, and a transformed current Scratch value.

The routine distinguishes the A-scaled return used for dispatch from the additionally A-and-B-scaled retained processing value.

**documented_only:** algorithm, buffer directions, initialized output extent, required lengths, return convention, ownership, pointer retention, global mutations, and other side effects.

## Acceptance examples

These are conditional static obligations, not executed tests or assertions that the missing globals permit the stated paths.

1. **Ordered publication.** Given valid active storage and initialized source bytes, publication stores the product’s low word at offset 40, then zero at offset 44.

2. **Publication boundaries.** Low product words \(2^{32}-2\) and \(2^{32}-1\) still produce zero at offset 44.

3. **Descriptor reread.** Changes to descriptor offsets 16 or 20 before the second acquisition affect that call’s newly read arguments.

4. **Unestablished comparison source.** State `0x17` before either source-holder assignment copies null. A later comparison read has no established validity or guaranteed error return.

5. **Stored region pointer.** Suppose B is 3 and Scratch is 4 when state `5` executes. The stored region pointer uses displacement 12. Changing Scratch to 9 afterward leaves that pointer unchanged. Another execution of state `5` would use displacement 27, assuming valid arithmetic and addresses.

6. **Independent processing input.** In the preceding example, state `0xe` after the Scratch change uses 9, multiplied by its argument-seven global factor, even if state `5` has not executed again.

7. **Preserved comparison Carrier.** With A = 2, B = 3, and Index = 4, state `0x16` places 6 in Carrier. If preserved until state `9`, Index becomes 36, not 6.

8. **Replaced comparison Carrier.** With A = 2 and B = 3, processing preparation for descriptor offset 0 containing 7 places 42 in Carrier. If preserved until state `9`, Index becomes 252.

9. **Distinct return scales.** With A = 2, B = 3, and processing return 5, absent overflow, the predicate intermediate is 10 and the retained processing value is 60.

10. **Publication before processing.** If publication occurs without a prior state `0xe`, its post-store restoration supplies initial zero to dispatch-related arithmetic storage. This does not imply either destination word was derived from processing.

11. **Latest processing value.** If multiple executions of state `0xe` assign retained processing values before publication, publication restores the most recent one, not an earlier value.

12. **Return without publication.** Reaching state `0x18` without prior publication does not establish destination output or a failure marker.

## Uncertainties

The following remain **documented_only**:

- Global contents, invariants, reachable paths, repetition, and termination.
- Whether argument installation and pointer initialization precede every use.
- Which Scratch and Carrier assignments reach each consumer.
- Whether state `5` reexecutes between region-pointer uses.
- Whether publication follows any processing call.
- Dependency status meanings, memory effects, ownership, retention, and global mutations.
- Descriptor-field meanings and the semantic interpretation of published output.
- Whether transformations implement decoding, equality, or extent validation.
- Valid aliasing arrangements, accessible lengths, and required initialization.
- Invalid-access behavior and consequences of oversized dependency writes.
- Exact ABI, pointer width, alignment, byte order, signed shifts, signed overflow, and out-of-range signed conversions.

No protocol, digest algorithm, validation guarantee, or ordinary success path is established.

## Evidence

This contract rests on static inspection of the supplied routine representation and opaque dependency mapping.

Established evidence includes represented widths and offsets, local capacities, initialization, state-specific assignments, dependency argument relationships, shared-holder reuse, ordered byte reads, destination-store order, and explicit return behavior.

State `5` assigns the stored region pointer from the Scratch value present at that execution; later Scratch assignments do not modify that pointer. State `0xe` separately consumes current Scratch for its seventh argument.

The retained processing holder begins at zero, is assigned only in state `0xe`, and is copied into dispatch-related arithmetic storage after publication’s stores. This establishes both initial-zero restoration and latest-processing-value restoration, depending on prior execution.

Unsigned arithmetic establishes the zero destination word at offset 44. Missing globals prevent establishing the end-to-end execution graph. Dependency internals were not supplied. No runtime observations or tests support additional claims.
