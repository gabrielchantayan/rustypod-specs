<!-- Trusted provenance: F-12bc938fcaac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-12bc938fcaac

## Purpose

F-12bc938fcaac performs an ordered sequence of indirect invocations, derives values and addresses from implicit execution state and dependency results, mutates memory at mixed widths, and finishes with a four-argument indirect invocation.

No application-level purpose is established. Constants resembling characters, repeated offsets, and address-valued stores do not establish formatting, serialization, allocation, or collection management.

This contract distinguishes:

- **Established static evidence:** explicit operations, access widths, and ordering in the supplied representation.
- **[INFERENCE]:** consequences conditional on a 32-bit bit-vector execution model or ordinary memory semantics.
- **documented_only:** unresolved behavior that cannot safely become an implementation requirement.

Every reached operation depends on preceding invocations returning and preceding accesses succeeding. No successful-input domain or guarantee of completion is established.

## Interface

### Explicit inputs and output

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | Forwarded unchanged to the initial indirect invocation. |
| Second argument | 32 bits, represented as signed | Forwarded unchanged to that invocation and separately supplied after a six-position left shift. |

Neither argument has an established pointer, size, flag, or ownership interpretation. The visible body does not directly dereference the first argument.

No explicit caller-visible result is represented. Dependency results are consumed internally; the final invocation’s result is not explicitly returned.

### Implicit inputs

Behavior also depends on incoming link-related state, an incoming 32-bit register value without visible initialization, auxiliary register outputs from dependencies, 32-bit and 64-bit dependency results, fixed-location memory, and dependent-address memory.

The origin of the final indirect target is unresolved. Equivalence cannot be established from the two explicit arguments alone.

### Widths, layout, and access requirements

Visible memory operations include 8-bit reads and writes, 16-bit writes, and 32-bit reads and writes. Some dependency results are represented as 64-bit quantities split into low and high 32-bit portions.

Offsets are byte offsets, not established structure fields. No record size, alignment, allocation boundary, byte order, or non-aliasing guarantee is supplied. Word and byte accesses can overlap. Fixed locations and calculated addresses must be treated as potentially sharing the same address space.

No null checks, bounds checks, target validation, recoverable error return, or rollback protocol is represented. Reached targets must be executable, and reached memory accesses must have suitable mappings and permissions. Low-address mapping and support for unaligned words are **documented_only**.

Invalid targets or addresses cannot be reinterpreted as successful no-ops.

## Behavior

### Indirect invocation prefix

The first indirect target is derived from incoming link-related state with displacement 0xFFE. Its three visible arguments are the first input, the second input, and the second input shifted left by six.

A long, straight-line sequence of additional indirect invocations precedes direct memory mutation. Targets are predominantly location-derived addresses with the same displacement. Visible invocations have zero through four explicit arguments.

Arguments include retained results, shifted results, masked fields, explicit zeros, and auxiliary register outputs. Several invocations receive the low half of a 64-bit result shifted left by eight alongside its unshifted high half. Other combinations reuse earlier retained values across intervening invocations.

An invocation with no explicit arguments is not established to be inert. An unused ordinary return value does not justify removing an invocation. Nothing establishes preservation of globals, memory, auxiliary registers, or ownership by these dependencies.

No explicit loop or conditional branch is represented in the prefix. Dependencies may nevertheless branch, fail, or transfer control without returning.

### Arithmetic interpretation

**[INFERENCE]:** Under a 32-bit bit-vector model, shifts, multiplication, addition, and subtraction retain the low 32 result bits; byte and halfword stores retain the low eight and sixteen bits respectively. This is not a claim that the represented signed operations have defined semantics in a particular implementation language.

Consequences include:

- A left shift by eight followed by a logical right shift by eight retains only the original low 24 bits.
- Two successive 28-position left shifts yield zero.
- A low-24-bit mask followed by a logical right shift by 20 yields zero through 15.
- A low-22-bit mask followed by a left shift by eight occupies at most 30 bits.
- A 64-bit logical right shift by 36, retained in 32 bits, extracts the high half shifted right by four.

Near the end of the prefix, a returned word is shifted left by eight, masked to 15 bits, and logically shifted right by six. Call the resulting quantity **Small quantity**. Under this model it is four times the returned word’s low seven bits: a multiple of four from zero through 508. **Small base** is twice that quantity, hence a multiple of eight from zero through 1,016.

These low addresses are not evidence of allocated objects.

### Initial mutation and constrained zero readback

The following descriptive terms identify retained values, not source identifiers:

- **Pair:** the last 64-bit dependency result before direct mutation.
- **Retained bits:** an earlier result shifted left by 25.
- **Shifted base:** a different retained result multiplied by 2²⁵.
- **Shifted word:** another retained result shifted left by eight.
- **Scale source:** the retained result used to form Shifted base.

The initial updates have material ordering. A word at Pair’s high half plus 0x14 receives Retained bits. A byte at Retained bits plus 0x09 receives the low byte of Pair’s high half. A word at 256 times Pair’s low half plus 0x54 receives Shifted base. The byte at Retained bits plus 0x09 is then overwritten with the low byte of Retained bits.

Next, a word at Shifted base plus 0x14 is cleared, followed by a byte read at Shifted base plus 0x05. A word at Small base plus 0x14 is cleared before the zero-extended byte is stored as a word at Shifted base plus 0x34. The first word read at Shifted base plus 0x14 follows these stores.

**[INFERENCE]: Under ordinary memory semantics and the stated 32-bit model, this first read returns zero.** Shifted base is aligned to 2²⁵. The store at its offset 0x34 cannot overlap its word at offset 0x14. Small base is at most 1,016 and aligned to eight; its offset-0x14 word can overlap the read only when both bases are zero, and that store also writes zero. There is no intervening aliasing store capable of making this read nonzero.

Device-specific readback and concurrent modification remain unresolved; neither establishes an ordinary-memory aliasing exception.

Small base plus 0x14 is cleared again, followed by two byte stores at Small base plus 0x15 using the low byte of that readback.

### Early fixed locations and dependent updates

A byte is loaded from fixed address 0x05 and zero-extended as a base. The byte at fixed address 0x01 then receives the low byte of Small base. These accesses precede the stores through the loaded byte-valued base.

The loaded base’s offset 0x15 receives the low byte of Small quantity. Shifted base plus 0x11 receives the low byte of Shifted word. A word at the initial readback plus 0x54 receives Shifted word, after which Shifted base plus 0x11 is cleared.

Under ordinary memory and the 32-bit model, the readback is zero, so this word store is specifically at fixed address 0x54. It is distinct from the later explicit fixed-location word update there.

The word at the loaded byte-valued base plus 0x54 is retained as **Old word**, then replaced with the initial readback. Small base plus 0x34 receives Small quantity.

A word loaded from Shifted base plus 0x74 becomes another base. Its offset 0x54 first receives Scale source shifted left by 26. A separate update uses two word reads at Small base plus 0x74: one supplies a destination base and the other a stored value. Two accesses are represented; their relative operand-evaluation order is not established.

Old word supplies both addresses and values in subsequent repeated updates. Its offset 0x24 receives itself twice, and the other loaded base’s offset 0x09 receives Old word’s low byte twice. A store of Shifted word at that base’s offset 0x54 occurs between the repeated pairs.

The fixed-location updates then have this order:

- Small base plus 0x14 receives Scale source shifted left by 26.
- The **32-bit word at fixed address 0x54** receives Old word shifted left by one.
- Small base plus 0x14 receives Old word twice.
- Two bytes at Small base plus 0x15 receive the low byte of four times Old word.
- The **byte at fixed address 0x15** receives the low byte of Small base.

These fixed accesses are not merely names for independent fields; calculated stores may alias them.

### Middle dependent-address region

Let **Fourfold base** mean four times Old word. Subsequent updates publish scaled values through Fourfold base, Small base, Old word, and further loaded bases.

A word at Fourfold base plus 0x54 receives Shifted word before later receiving Fourfold base itself. Small base plus 0x34 receives Fourfold base. A word read at Small base plus 0x74, doubled, supplies another retained base.

Byte reads at offset 0x05 are zero-extended and used to locate word reads at offset 0x74. The results feed offset-0x44 word stores, offset-0x0D byte stores, and offset-0x14 self-valued word stores. A word loaded at Fourfold base plus 0x64 supplies another base whose offset 0x0D receives a retained byte and whose offset 0x14 receives its own address.

Later, a word loaded from Fourfold base plus 0x04 supplies the base for two distinct byte reads at offset 0x05. Dependent-address stores occur between these reads. The later byte is therefore not established to equal the earlier one.

The word reached through the later byte at offset 0x74 is **Selected word**. Its doubled value is **Final linked base**, and its fourfold value is **Publication value**.

Repeated stores remain part of the trace. Self-valued words and byte-valued links do not establish a list, tree, allocation, or ownership invariant.

### Constants and publication ordering

Numeric 0x3A is stored as a byte at four times Selected word, as a halfword at eight times Selected word, and as a word at four times Selected word plus 0x20.

Let **Adjusted quantity** be Fourfold base minus decimal 150, and **Arithmetic base** be four times Adjusted quantity. At Arithmetic base plus 0x20, four word stores occur in order: 0x20, 0x2F, a zero-extended byte read from Arithmetic base plus 0x04, and 0x20 again. The byte read follows the first two stores and can reflect aliasing effects.

**The publication-base word is loaded from fixed address 0x087FBBBC before the word at fixed address 0x60 is assigned 0x20.** The loaded publication base is retained; the load does not observe state after that assignment.

Publication then consists of an initial group and many repeated groups:

- The initial group writes Publication value at eight times Old word plus 0x54.
- Each later group writes Publication value at eight times Adjusted quantity plus 0x54.
- Within every group, that word store precedes a byte store at publication base plus 0x19, a byte store at Arithmetic base plus 0x11, and a word store at publication base plus 0x34.

The byte values are respectively the low byte of Arithmetic base and the low byte of publication base. The initial offset-0x34 word receives eight times Old word; later ones receive eight times Adjusted quantity.

No cursor advancement or allocation is represented. Exact repetition-count conformance remains **documented_only** here; repeated writes are not licensed to collapse into one.

### Final fixed-location accesses and fresh byte reads

After publication, a **32-bit word is read at fixed address 0xA9** and retained as **Fixed word**. A word is then read at Publication value plus 0x74. Next, the **byte at fixed address 0x3A** receives the low eight bits of eight times Adjusted quantity plus 0x30.

Words are subsequently retained from Final linked base plus 0x14 and Arithmetic base plus 0x04. Several dependent-address stores follow: bytes at Final linked base plus 0x0D and Arithmetic base plus 0x08, 0x0D, and 0x09, then a self-valued word at Final linked base plus 0x54. The byte at Arithmetic base plus 0x08 receives 0x6C.

Only after those stores does the **byte at fixed address 0x67 receive the low byte of Fixed word**. A word is then loaded from Final linked base plus 0x44. After that load, the **byte at fixed address 0x67 is read and written to fixed address 0x21**. This is an explicit byte read after the offset-0x44 word read, not an assignment from an immutable retained byte.

The newly loaded word supplies a byte at Arithmetic base plus 0x10 and a word at publication base plus 0x54. A shift-derived byte is stored at Final linked base plus 0x01, then publication base plus 0x54 is overwritten with Arithmetic base.

The shift operates on a represented signed word; its count is Arithmetic base’s low eight bits. Signed-right-shift rules and counts outside the word width are **documented_only**. Clamping or reducing the count modulo 32 is not established.

A word loaded from Arithmetic base plus 0x40 replaces the preceding byte at Arithmetic base plus 0x10 through a word store. The publication base also receives words at offsets 0x74 and 0x44.

**Fixed address 0x67 is then read again** for a byte store at Arithmetic base plus 0x09. After this read, dependent stores update Final linked base plus 0x09 and Arithmetic base plus 0x44; a word is loaded from Arithmetic base plus 0x54. **Another fresh read of fixed byte 0x67** supplies a second byte store at Arithmetic base plus 0x09.

Those intervening writes may alias 0x67. Neither later read is specified as reuse of the byte originally written there.

### Completion and final invocation

The final region uses the newly loaded Arithmetic-base offset-0x54 word as another base. It updates bytes at offsets 0x0D and 0x11, restores the earlier retained word to Final linked base plus 0x54, and performs further reads and stores through Arithmetic base and Final linked base.

A word read at Final linked base plus 0x04 supplies a base for:

- A halfword store of 0x65 at twice that base.
- Word reads at offsets 0x64 and 0x74.

The word read at offset 0x64 is retained for final dispatch. Numeric 100 is stored as a word through the earlier Arithmetic-base offset-0x54 result. Another word loaded at Final linked base plus 0x64 receives an offset-0x54 update, then is stored at twice the retained dispatch word.

The final indirect invocation receives, in order:

1. Numeric 0x72.
2. The word retained from the offset-0x64 read described above.
3. The earlier word retained from Final linked base plus 0x14.
4. The later word read at offset 0x74.

A return is represented afterward. Whether the indirect target returns normally is unresolved.

## Side effects

Effects are nontransactional: dependency effects and direct mutations occur before completion, with no represented rollback.

Direct effects include fixed-location writes, dependent-address updates, overlapping widths, repeated stores, and fresh reads after mutations. Final memory requires accounting for aliasing and ordering, not merely collecting the last apparent assignment to each named role.

No allocation, release, reference-count adjustment, or ownership transfer is established. Unknown dependencies may perform such actions; their absence from the visible body proves neither caller ownership nor a leak.

Atomicity, synchronization, volatile behavior, device-memory semantics, reentrancy, and concurrency safety are **documented_only**.

## Dependencies

F-12bc938fcaac is the only supplied opaque function ID. No opaque IDs or behavioral contracts are available for its indirect dependencies.

| Dependency role | Status |
|---|---|
| Incoming-state-derived initial target | documented_only |
| Ordered location-derived invocation sequence | documented_only |
| Auxiliary register-result channels | documented_only |
| Final unresolved indirect target | documented_only |

These are unresolved roles, not fabricated function identities. No dependency is assumed to preserve globals or to acquire, transfer, or release memory.

## Acceptance examples

These are conditional criteria, not executed tests.

### Argument forwarding

For second input one, the initial invocation receives one unchanged and 64 as its third argument.

For second input 0x04000000, the third argument is zero under 32-bit truncation; the unchanged second argument remains 0x04000000.

### Small-address bounds

If the applicable returned word has low seven bits equal to 127, Small quantity is 508 and Small base is 1,016 under the bit-vector model. This establishes arithmetic, not access validity.

### Initial zero readback

With ordinary memory, no concurrent modification, and successful accesses, the first read at Shifted base plus 0x14 returns zero for every permitted Small base. This includes both bases being zero: the overlapping intervening store also writes zero.

### Publication-base ordering

An access observer must see the publication-base word read before the fixed-address-0x60 word store. Reversing them violates the represented order, even if a particular ordinary-memory execution produces the same retained base.

### Fixed-byte freshness

If intervening dependent stores change byte 0x67 before either later read, the corresponding Arithmetic-base offset-0x09 store uses that read’s current byte. Substituting the low byte of the earlier Fixed word is not equivalent.

### Overlapping widths

After a zero word store at Small base plus 0x14 followed by a byte store of 0x7F at offset 0x15, that byte is 0x7F. The resulting numeric word depends on byte order.

### Observable repeated writes

Where writes are externally observable, retaining only the final 0x20 in the four-store constant sequence is insufficient. Intermediate values and the intervening byte read remain observable.

### Final dispatch

If reached, the final invocation’s first visible argument is numeric 0x72. No string interpretation or caller-visible return value is established.

## Uncertainties

The following remain **documented_only**:

- Application intent, callable boundaries, and recovered invocation signatures.
- Dependency identities, effects, and return behavior.
- Provenance of implicit execution-state inputs.
- Byte order, alignment requirements, and low-address mappings.
- Signed overflow and out-of-range shift semantics.
- Exact publication repetition count.
- Ownership, allocation, release, and lifetime rules.
- Fault handling, termination, concurrency, and device-memory behavior.

These unknowns do not negate the conditional ordinary-memory zero-readback result or justify caching the later fixed-byte reads.

## Evidence

This contract is based solely on the supplied static representation, independently interpreting its accesses, arithmetic, and ordering. Prior prose and feedback are not treated as proof.

Static evidence establishes the explicit widths, invocation dataflow, fixed-location accesses, mixed-width mutations, publication-base load before the 0x60 store, repeated reads of byte 0x67, and final four-argument invocation.

Alignment, range, and zero-readback conclusions are conditional deductions under the stated 32-bit and ordinary-memory assumptions.

No implementation access, dependency inspection, runtime execution, smoke run, or acceptance test has been performed. No snapshot hash or independently verified provenance is asserted.
