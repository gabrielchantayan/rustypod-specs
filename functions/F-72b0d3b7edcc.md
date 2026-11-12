<!-- Trusted provenance: F-72b0d3b7edcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72b0d3b7edcc

## Purpose

The supplied static representation describes a fixed sequence of narrow memory writes, wider state writes, indirect calls, and a terminal instruction-decoding failure. It does not establish an application-level purpose.

The observable contract is therefore a conditional execution trace: if the represented control flow is executable, each required memory access succeeds, and each dependency returns, the operations below occur in the specified order.

The representation includes explicit warnings about invalid instruction data and overlapping global symbols. Consequently, its apparent function boundary, calling convention, and terminal behavior are not independently established. This contract distinguishes represented behavior from verified machine behavior; no runtime tests have been performed.

## Interface

### Explicit interface

The represented entry point has no declared parameters and no declared return value. It nevertheless consumes incoming machine state not described by that interface.

There is no established ordinary successful return.

### Implicit inputs

The contract uses these descriptive names:

| Input | Width | Meaning in the represented behavior |
|---|---:|---|
| **Initial address** | 32 bits | Address used for the first two halfword writes; its low 16 bits also supply their value. |
| **Context base** | 32 bits | Base used for three early word writes at byte offset `0x14`. |
| **State word** | 32 bits | Value written to state locations and used to derive later addresses and values. |
| **Entry link** | 32 bits | Incoming control-flow value used to derive the first indirect-call target. |

These are bit-pattern inputs, not established user-facing parameters. The supplied representation does not establish who initializes them or what objects they designate.

Each of eleven result-producing calls supplies a represented 64-bit value. For result number \(j\):

- \(L_j\) is its low 32 bits.
- \(H_j\) is its high 32 bits.

The actual ABI mechanism for returning those 64 bits is unknown.

### Width and arithmetic conventions

For purposes of describing the represented machine-width operations:

- A halfword write transfers 16 bits.
- A word write transfers 32 bits.
- Address arithmetic is described in a 32-bit address domain.
- Truncating to 16 bits retains the low 16 bits.
- Logical right shifts introduce zero bits.
- A left shift used as a 32-bit result discards bits beyond bit 31.

Where the representation mixes signed integer expressions and addresses, modulo-\(2^{32}\) arithmetic describes the apparent machine-level intent. Overflow behavior and actual machine instructions remain unverified; this is not a claim that every displayed expression has defined portable-language semantics.

### Memory preconditions

Continuation requires all reached write destinations to support the indicated access width. Destinations may be derived from incoming state or dependency results; no validation is shown.

Alignment requirements, mapping, privilege, byte order, and memory-mapped-device semantics are not established. In particular, a derived destination of zero is not rejected before use.

There is also a distinguished global storage location, called the **reset location** below. Its address, write width, and relationship to overlapping symbols are not established.

## Behavior

### Address and value transformations

Two transformations are applied to dependency results.

**Transformation A**

For a low result word \(L\):

- Destination address: \(L \mathbin{\text{logical-right-shift}} 8\).
- Stored halfword: the low 16 bits of that destination address.

The represented value extraction shifts the full 64-bit result before truncation, but its retained bits are still bits 8 through 23 of \(L\). The high result word does not contribute to this halfword.

**Transformation B**

For a low result word \(L\):

- Destination address: \((L \mathbin{\&} \texttt{0x07ffffff}) \mathbin{\text{logical-right-shift}} 3\).
- Stored halfword: the low 16 bits of \(L \mathbin{\text{logical-right-shift}} 3\).

The displayed value calculation first shifts a 32-bit value left by five and then right by eight. With 32-bit truncation between those operations, it retains bits 3 through 26 before the final halfword truncation. Thus the stored halfword is also the low 16 bits of the destination address.

Transformation A can produce addresses through `0x00ffffff`. Transformation B can produce addresses through `0x00ffffff` as well, despite selecting a different input bit range.

For both transformations, the function performs **two consecutive halfword writes of the same value to the same address**. These are two represented write events, not one write whose duplication may be discarded from this contract.

Additional derived quantities are:

- **Reduced address**: \((\text{State word} \mathbin{\&} \texttt{0x0fffffff}) \mathbin{\text{logical-right-shift}} 18\), ranging from zero through `0x3ff`.
- **Scaled base**: the low 32 bits of State word multiplied by 16.
- **Shifted state**: the low 32 bits of State word shifted left by eight.

### Entry writes

Before the first dependency call, the function performs:

1. Two consecutive halfword writes to Initial address. Both values are the low 16 bits of Initial address.
2. One halfword write of that same value to twice Initial address, using the represented 32-bit address calculation.
3. One word write of State word to Context base plus `0x14`.

The first indirect call then uses Entry link plus `0xffe` as its target. No explicit arguments are established for this call.

### Eleven result stages

The sequence contains eleven result-producing calls. Each is preceded by a separate preparatory call. Every result stage consumes the result produced immediately before that stage’s writes.

The following table specifies the ordered writes within each stage. “Reset” means an assignment of zero to the reset location, with width unresolved.

| Stage | First writes | Additional writes, in order |
|---:|---|---|
| 1 | Transformation A, twice | Reset; State word to Context base plus `0x14` |
| 2 | Transformation A, twice | Reduced-address writes described below; State word to Context base plus `0x14` |
| 3 | Transformation A, twice | Reset; State word to Scaled base plus `0x14` |
| 4 | Transformation B, twice | Reset; State word to Scaled base plus `0x14` |
| 5 | Transformation B, twice | Reset; State word to Scaled base plus `0x14` |
| 6 | Transformation B, twice | Reset; State word to Scaled base plus `0x14` |
| 7 | Transformation B, twice | Reset; State word to Scaled base plus `0x14` |
| 8 | Transformation A, twice | Reset; Shifted state to Scaled base plus `0x14` |
| 9 | Transformation B, twice | Reset; Shifted state to Scaled base plus `0x14` |
| 10 | Transformation B, twice | Reset; Shifted state to Scaled base plus `0x14` |
| 11 | Transformation A, twice | Reset |

All state-location writes in this table are 32-bit writes.

Stage 2’s additional reduced-address writes are:

- Two consecutive halfword writes to Reduced address, storing Reduced address as a 16-bit value.
- One halfword write of the same value to twice Reduced address.

The represented intermediate truncation in this computation does not change the result: the masked input contributes only 12 bits after shifting by 16, and a further shift by two produces exactly Reduced address.

Stage 2 does not contain a direct reset assignment. Stage 11 does not contain a direct state-location word write.

### Preparatory-call arguments

The call before stage 1 has no established explicit arguments.

Before each later result-producing call, a preparatory dependency receives arguments derived from the preceding stage. The following stage-specific tokens are literal 32-bit values used as data arguments:

| Completed stage | Token |
|---:|---:|
| 1 | `0x0890ffad` |
| 2 | `0x08910015` |
| 3 | `0x0891007d` |
| 4 | `0x089100e5` |
| 5 | `0x0891014d` |
| 6 | `0x089101b5` |
| 7 | `0x0891021d` |
| 8 | `0x08910285` |
| 9 | `0x089102ed` |
| 10 | `0x08910355` |
| 11 | `0x089103bd` |

After stage 1 and stages 3 through 10, the preparatory call receives three represented arguments:

- Zero.
- The preceding result’s high 32 bits.
- The completed stage’s token.

After stage 2, those arguments instead are:

- Reduced address.
- The stage 2 result’s high 32 bits.
- Reduced address plus the stage 2 token.

Each subsequent result-producing call has no explicit arguments established by the supplied representation. This does not establish that it ignores argument registers, stack contents, or global state.

### Final call sequence

After stage 11’s writes, the next dependency receives four represented arguments:

1. The 32-bit bit pattern `0xffffff1c`, equivalent to −228 under signed 32-bit interpretation.
2. \(H_{11}\) minus `0x264`, retaining a 32-bit result.
3. `0x4b`, decimal 75.
4. Zero.

Four more calls follow, in fixed order. No explicit arguments or consumed results are established for them.

The representation then terminates at invalid instruction data. This is not evidence of an intentional shutdown operation, a defined exception API, or a normal return.

## Side effects

### Direct writes

If the entire represented path is reached, it contains:

- **28 halfword writes**: three entry writes, 22 result-derived writes, and three additional stage 2 writes.
- **11 word writes**: three using Context base and eight using Scaled base.
- **10 reset-location assignments**.

These counts describe write operations in the supplied representation, not distinct memory locations. Aliasing may cause several operations to affect the same bytes.

The eight Scaled-base word writes consist of five writes of State word followed, at later stages, by three writes of Shifted state. Dependency calls intervene between those stages.

### Ordering and aliasing

Within each stage, the result-derived halfword writes precede the reset assignment or additional writes. Reset assignments precede the corresponding state-location word writes.

No direct memory reads are explicitly represented for preserving old destination contents. In particular, the reset assignment is not shown as a read-modify-write operation.

Destinations may overlap each other or the reset location. The contract does not assume disjointness. For overlapping writes, access width, order, and target byte order matter. No direct write may be treated as redundant merely because its value matches an earlier write.

Dependencies may read or change these locations between stages. A prior zero assignment does not establish that the reset location remains zero afterward.

### Ownership and failures

No direct allocation, deallocation, ownership transfer, or lifetime extension is established.

A result-derived address is used as a write destination, but that use does not establish that the dependency allocated it or transferred ownership. Similarly, unknown dependencies may manage memory internally; their behavior is not constrained by this representation.

There are no explicit checks for invalid addresses, failed dependencies, arithmetic overflow, or error status. No rollback or cleanup path is shown. If execution faults or a dependency does not return, later operations are not established to occur.

## Dependencies

The IDs below are local opaque labels for call sites. They do not establish identities, distinct implementations, or equivalence between callees.

| Stage | Preparatory dependency | Result-producing dependency |
|---:|---|---|
| 1 | F-000000000001 | F-000000000002 |
| 2 | F-000000000003 | F-000000000004 |
| 3 | F-000000000005 | F-000000000006 |
| 4 | F-000000000007 | F-000000000008 |
| 5 | F-000000000009 | F-00000000000a |
| 6 | F-00000000000b | F-00000000000c |
| 7 | F-00000000000d | F-00000000000e |
| 8 | F-00000000000f | F-000000000010 |
| 9 | F-000000000011 | F-000000000012 |
| 10 | F-000000000013 | F-000000000014 |
| 11 | F-000000000015 | F-000000000016 |

The final four-argument call is F-000000000017. The four following calls are F-000000000018 through F-00000000001b, in ascending order.

There are 27 represented calls in total, conditional on reaching the end.

The first 24 calls use computed targets. The first target depends on Entry link; the others are derived from literal control-flow values and offsets. The last three calls have fixed represented targets. Their semantics are unavailable.

No dependency is known to preserve globals, avoid destination aliasing, return normally, or preserve undocumented incoming machine state. Cross-call reliance on such preservation is an unresolved execution prerequisite, not an established dependency guarantee.

## Acceptance examples

These examples are conditional trace expectations, not executed tests. They assume the relevant calls return and all stated destinations permit the required accesses.

### Result transformation A

If a stage using Transformation A receives low result word `0x12345678`, its destination is `0x00123456`. It performs two consecutive halfword writes of `0x3456` there.

Changing only the high result word does not change those two writes. It can change the subsequent preparatory call’s second argument.

### Result transformation B

If a stage using Transformation B receives low result word `0xf1234567`, the masked value is `0x01234567`. Its destination is `0x002468ac`, and both halfword writes store `0x68ac`.

The upper five bits of the original low result word do not affect this destination.

### Stage 2 reduction

For State word `0xffffffff`, Reduced address is `0x3ff`. Stage 2 performs two halfword writes of `0x03ff` to `0x3ff`, then one halfword write of `0x03ff` to `0x7fe`.

Whether the odd-address halfword accesses are supported is target-dependent. The representation does not adjust their alignment.

### State destination and value transition

For State word `0x00123456`:

- Scaled base is `0x01234560`.
- Its state destination is `0x01234574`.
- Stages 3 through 7 write `0x00123456` to that destination.
- Stages 8 through 10 write `0x12345600` to that destination.

Each stage still performs its preceding halfword writes and reset assignment. Intervening dependencies can modify the destination between writes.

### Zero-derived destinations

If a result’s low word is zero, both transformations derive address zero and value zero. The represented behavior attempts two halfword writes there; it does not skip them.

If address zero is inaccessible, continuation through the remaining stage is not established.

### Final argument arithmetic

If the final high result word is `0x00000264`, F-000000000017 receives zero as its second argument.

Under the stated 32-bit arithmetic interpretation, a final high result word of zero produces second-argument bit pattern `0xfffffd9c`.

## Uncertainties

The following items are **documented_only**:

- Whether the apparent function boundary and full linear sequence correspond to executable machine code.
- The real entry calling convention and sources of the implicit inputs.
- The reset location’s address, assignment width, overlapping storage, and application meaning.
- Byte order, alignment rules, memory permissions, and device-write semantics.
- The ABI interpretation of the represented 64-bit results.
- Dependency identities, side effects, memory ownership, preservation guarantees, errors, and nonreturn behavior.
- Whether the final invalid instruction corresponds to a real executed instruction, an intentional trap, or a representation failure.
- Exact behavior where signed arithmetic or address conversion exceeds its represented range.

An initialization or reset-related purpose might be inferred from repeated zero assignments and patterned writes, but that interpretation is not established. No application-level success condition can be derived safely.

## Evidence

**Established static evidence:** The supplied representation contains the write widths, bit selections, call ordering, literal arguments, repeated stores, and terminal decoding failure described above. The write and call totals follow from that represented sequence.

**Inferred interpretation:** The 32-bit modular treatment of mixed signed/address calculations describes apparent machine-width behavior. Descriptive terms such as “state” and “reset” do not establish application semantics.

**Missing evidence:** There is no independent instruction validation, dependency implementation evidence, memory map, ABI documentation, or runtime observation. The acceptance examples are arithmetic consequences of the static representation, not reports of successful execution.
