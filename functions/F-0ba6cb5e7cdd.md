<!-- Trusted provenance: F-0ba6cb5e7cdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ba6cb5e7cdd

## Purpose

F-0ba6cb5e7cdd exposes a three-word entry interface, performs a long sequence of unresolved indirect calls and bit-field transformations, then executes an extensive sequence of memory mutations before a final unresolved indirect transfer.

**Established static evidence:** the supplied representation describes ordered calls, 32-bit arithmetic, byte and word accesses, several absolute low-address accesses, and repeated writes to dynamically derived addresses.

**Inferred intent:** the repeated offsets and cross-linked writes suggest manipulation of structured state. Neither the structures’ identities nor an application-level purpose can be established. In particular, character-like constants do not establish text formatting, and reciprocal-looking assignments do not establish list or ownership operations.

This contract therefore specifies the recoverable behavioral boundaries and observable access requirements. It does not promise a conventional initialization, serialization, rendering, or container operation.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit reference after entry is shown. |
| Second argument | 32 bits | No explicit reference after entry is shown. |
| Third argument | 32 bits, represented as signed | Its value shifted left by six bits is explicitly supplied to an early indirect call. |

Lack of an explicit reference to the first two arguments does **not** prove that changing them cannot change behavior. The initial indirect calls have unresolved calling conventions and may observe incoming machine state.

No argument is explicitly identified as a buffer, capacity, string, object pointer, or ownership token.

### Implicit inputs

Execution also depends on:

- Incoming return-link state, which participates in the first indirect target calculation.
- Executable destinations derived from continuation-relative values.
- Primary and secondary results of unresolved calls.
- Memory contents read through calculated addresses.
- Several absolute storage locations.
- An unresolved final control-transfer destination.

Consequently, the explicit arguments alone are insufficient to determine the resulting access trace or final memory state.

### Outputs and completion

The represented interface has no explicit return value. Its recoverable outputs are memory writes, dependency effects, and the final indirect transfer.

The final transfer carries four explicit values: the constant 0x72, a recently loaded 32-bit value, and two retained 32-bit values. Whether that transfer returns normally, transfers control permanently, or has a different calling convention is **documented_only**.

### Preconditions

For any execution claimed to complete according to this representation:

- Every reached indirect destination must be executable under the actual calling convention.
- Every reached memory access must be permitted at its effective address and width.
- Required secondary call results and incoming machine state must be defined.
- The environment must accommodate absolute low-address accesses.
- The environment must supply the actual behavior of unresolved dependencies.

There is no established restricted range for the third argument that makes these conditions sufficient. No visible validation establishes them.

## Behavior

### Execution ordering

The represented body has two broad regions:

1. An extended indirect-call region interleaved with shifts, masks, and result retention.
2. A direct-memory mutation region followed by the final unresolved transfer.

No explicit conditional branch or loop is shown in the body. This does not establish branch-free execution inside dependencies, nor prove that the supplied representation has recovered all control flow.

Calls shown without explicit arguments remain distinct ordered interactions. They must not be treated as pure, parameterless helpers. Unresolved dependencies may read or modify global state, affect memory later accessed directly, allocate or release storage, or fail to return.

### Call-result processing

The call region repeatedly transforms 32-bit results and secondary result words. Some results are represented as 64-bit quantities whose low and high 32-bit halves are used separately.

Recoverable arithmetic includes:

- Left shifts of 32-bit values by fixed counts ranging from small bit positions through bit 31.
- Logical right shifts of unsigned quantities.
- Masks selecting low fields of 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, or 24 bits.
- Combinations of masked extraction and subsequent left alignment.
- Reuse of values retained across intervening calls.
- Explicit zero arguments, including zeros derived by shifting values already represented as zero.

These transformations establish bit patterns, not units or semantic types. A shifted result cannot be assumed to represent a size, index, pointer, fixed-point number, or encoded command.

The early use of the third input is a six-bit left shift. No range check, saturation, or overflow report precedes that use.

### Width and arithmetic requirements

The representation uses 8-bit bytes, 16-bit halfwords, 32-bit words and address-bearing quantities, and 64-bit paired results.

Where a machine-bit interpretation is required, a 32-bit left shift retains the low 32 bits and discards bits shifted beyond bit 31. This is an interpretation of the represented machine-width operations, not a claim that signed high-level arithmetic has portable overflow semantics.

For example:

- A value aligned left by 25 bits has its low 25 bits clear.
- A value shifted left by eight bits and later logically right by eight bits retains only its original low 24 bits.
- Two consecutive 28-bit left shifts of a retained 32-bit value produce zero under this interpretation.
- Masking before shifting can materially change the result and must not be omitted.

Multiplication by powers of two and address additions likewise require the target’s 32-bit arithmetic semantics. Checked arithmetic or widening would introduce behavior not established here.

Near the end, one right shift uses a count obtained from the low eight bits of a calculated address-bearing value. The behavior for large counts, and whether the shift is arithmetic or otherwise target-specific, is **documented_only**. It must not be replaced by a guessed portable rule.

### Transition to direct memory operations

The last call result before the direct-memory region is represented as two 32-bit halves. Earlier retained values supply additional addresses and payloads.

The initial direct-access trace establishes these ordering constraints:

- A 32-bit retained value is stored at offset 0x14 from the final result’s high half.
- A byte derived from that high half is stored at offset 9 from a retained address base.
- A 32-bit retained value is stored at offset 0x54 from a base formed by multiplying the final result’s low half by 256.
- The byte at offset 9 is written again using a different retained payload.
- A 32-bit zero is stored at offset 0x14 from another derived base.
- A byte is read at offset 5 from that base.
- A 32-bit zero is written through a separate, small derived base.
- The previously read byte is zero-extended and stored as a 32-bit word at offset 0x34.
- A word is read from offset 0x14 of the earlier base.

The stores and readbacks are not interchangeable with a single final-state assignment. Aliasing, access permissions, mapped devices, or observers may distinguish them.

### Address and layout observations

A small address-producing quantity is formed by shifting a 32-bit result left by eight bits, retaining its low 15 bits, and shifting right by six bits.

Under 32-bit bit-vector arithmetic, that quantity is a multiple of four in the range 0 through 508. Doubling it yields a multiple of eight in the range 0 through 1016. The representation uses this doubled quantity directly as an address base; it does not show addition of a conventional allocation base.

Other address-bearing values are formed by multiplication by 2, 4, 8, 256, or 2²⁵, or by large left shifts. Nonzero ordinary object pointers are not guaranteed.

The recurring offsets are:

| Access width | Observed offsets or forms |
|---|---|
| 8 bits | 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Addresses formed by multiplying retained values by 2 or 8 |
| 32 bits | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

This table describes access locations, not a validated record layout. In particular, byte writes at offset 0x15 overlap a word beginning at offset 0x14 when their bases coincide. A word beginning at offset 0x10 overlaps byte offsets 0x10 and 0x11.

Byte order, required alignment, and actual record boundaries are not established.

### Mutation and readback behavior

The direct-memory region includes:

- Repeated word clears followed by reads.
- Byte writes containing only the low eight bits of address-bearing values.
- Zero-extension of byte reads into word stores.
- Self-valued word stores at offsets including 0x14, 0x24, and 0x54.
- Reads through offsets 4, 0x44, 0x54, 0x64, and 0x74 whose results become subsequent address bases.
- A byte read at offset 5 that is subsequently used as a small address base for a word read at offset 0x74.
- Repeated writes to the same locations without intervening visible reads.

A value loaded before intervening writes remains a retained value unless explicitly reloaded. Conversely, a later read must observe the memory state at that later point; it cannot automatically reuse an earlier read of the same apparent field.

### Constant stores and repeated write region

One portion writes 0x3a at three locations using three different widths: one byte, one halfword, and one word. These are separate accesses; the halfword and word stores are not repeated-byte fills.

Another location, at offset 0x20 from a calculated base, receives these ordered word writes:

1. 0x20.
2. 0x2f.
3. The zero-extended byte read from offset 4 of that same base.
4. 0x20.

An absolute word location is then assigned 0x20.

A subsequent region repeatedly performs a four-access publication pattern: a word store at offset 0x54 of a scaled base, a byte store at offset 0x19 of a globally obtained base, a byte store at offset 0x11 of another base, and a word store at offset 0x34 of the globally obtained base.

The first such group uses a different scaled source from the ensuing repeated groups. The repetition is explicit in the supplied representation. Its semantic purpose and validated execution count are **documented_only**; neither suppression of duplicate writes nor replacement by an invented loop count is justified.

### Final mutations

After the repeated region, the representation performs further dependent loads and writes, including:

- Absolute reads and byte writes.
- A byte store of 0x6c at offset 8.
- Word transfers between offsets 0x40, 0x10, 0x74, and 0x44.
- An earlier byte at offset 0x10 being overwritten by a later word store beginning at that offset.
- A word store of decimal 100 at offset 0x54.
- A halfword store of 0x65 at a doubled address.
- A final word store through a doubled loaded value.

These operations precede the unresolved final transfer. There is no visible transaction boundary or rollback.

## Side effects

### Memory and globals

The function directly reads and writes both calculated addresses and absolute low addresses, including byte locations at 0x01, 0x15, 0x21, 0x3a, and 0x67, and word locations at 0x54 and 0x60. It also reads absolute state at 0x05 and 0xa9 and a separate globally stored address-bearing value.

These locations must not be modeled as independent abstract globals without checking overlap with computed accesses. The supplied warning about overlapping storage is relevant to interpretation but does not establish a complete alias map.

### Ownership and lifetime

No direct allocation or release operation is identifiable in the visible memory region. Ownership transfer, retention, reference counting, and destruction are **documented_only**.

Nothing establishes that unresolved callees preserve globals or preserve the lifetime of memory later accessed. Pointer-shaped writes do not by themselves transfer ownership.

### Errors, concurrency, and partial effects

No explicit error status, bounds check, null check, retry, or recovery path is shown.

Invalid targets or inaccessible addresses may prevent completion; the resulting target-specific failure behavior is unknown. Prior effects are not visibly undone.

Thread safety, reentrancy, atomicity, synchronization, and access to device-mapped memory are not established. Repeated stores must remain distinguishable in any acceptance model that observes individual accesses.

## Dependencies

Only F-0ba6cb5e7cdd has a supplied opaque function ID. The indirect destinations have no resolved opaque dependency IDs, so this contract does not invent identities or equate distinct call sites.

| Dependency class | Required information | Status |
|---|---|---|
| Initial indirect destination | Incoming return-link interpretation and calling convention | documented_only |
| Continuation-relative destinations | Executable targets, argument interpretation, primary and secondary results | documented_only |
| Final indirect destination | Destination, four-value calling convention, completion behavior | documented_only |

For every unresolved dependency, global effects, memory ownership effects, exceptional behavior, and return guarantees remain unknown. Identical-looking call forms do not establish identical callees or purity.

## Acceptance examples

These are conditional specification examples, not executed tests.

### Third-input transformation

If execution reaches the early call receiving the shifted third input, and that input is 1, its explicitly represented transformed argument is 64.

If that input has bit pattern 0x04000000, the transformed 32-bit argument is zero under the machine-bit interpretation. This does not imply identical whole-function behavior to an input of zero.

### Small derived base

If the result feeding the small-base extraction is 0x7f, the extracted quantity is 508 and its doubled base is 1016.

If that result is 0x80, the extracted quantity is zero. Acceptance must not insert an undocumented nonzero allocation base.

### Byte widening and narrowing

If an observed byte read yields 0xfe, its later zero-extended word payload is 254, not a negative signed-byte value.

If a byte store consumes a retained value 0x12345678, its payload is 0x78. No full-word store is implied.

### Ordered overwrites

If the offset-0x20 sequence executes with all accesses permitted, an access observer must see the ordered stores of 0x20, 0x2f, the captured byte value widened to a word, and 0x20. Matching only the final word does not establish trace equivalence.

### Aliasing

If a byte write at offset 0x15 aliases a previously written word at offset 0x14, a later word read must account for that byte mutation according to target byte order. It must not assume independent fields.

### Dependency failure

If an indirect dependency does not return, later represented memory operations are not required to occur. No cleanup or error return may be assumed.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and meaningful input domains.
- Actual indirect destinations and calling conventions.
- Reliability of the represented control-flow structure.
- Signed overflow and target-specific shift semantics.
- Byte order, alignment, and memory mapping.
- Complete layouts and alias relationships.
- Validated multiplicity of the long repeated-write region.
- Allocation, release, ownership, and lifetime rules.
- Failure signaling and normal-return behavior.
- Concurrency and external-observer guarantees.

These gaps prevent a complete application-level replacement contract. They do not justify inventing validation, deduplication, pointer normalization, or recovery behavior.

## Evidence

This document uses only the supplied static representation. Established observations are limited to its explicit interface, value transformations, ordered calls, access widths and offsets, retained values, constant stores, and final transfer.

Structural purpose is labeled as inference. Unresolved behavior is labeled **documented_only**. No implementation access, executable inspection, or runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
