<!-- Trusted provenance: F-d0ed4ed9d7d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0ed4ed9d7d5

## Purpose

F-d0ed4ed9d7d5 performs an ordered sequence of 24 indirect dependency calls, interleaved with one four-byte store, two byte reads, and two byte stores. It derives call arguments and memory addresses from implicit incoming state and dependency results.

No application-level purpose is established. The supplied evidence does not identify these operations as allocation, initialization, serialization, device access, or resource cleanup.

The visible representation ends at an invalid-instruction boundary and reports damaged control flow. This contract describes the represented sequence, conditional on execution reaching each operation. It does not establish normal return or behavior beyond that boundary.

## Interface

### Declared and implicit interface

The represented interface declares no parameters and no return value. Nevertheless, the body consumes three implicit incoming values:

| Descriptive input | Representation | Visible use |
|---|---|---|
| Context word | Unsigned 32-bit quantity | Supplies the first call argument and the address expression for both byte reads |
| Payload word | Four uninterpreted bytes | Supplies the value of the direct four-byte store |
| Incoming control value | Integer participating in call-target formation | Supplies the base for the first indirect-call target |

These descriptions identify contract inputs, not source identifiers. Their machine-level locations and calling-convention obligations are **documented_only**.

The first dependency receives the context word logically shifted right by 12 bits. That argument is between 0 and 1,048,575 inclusive.

Both direct byte reads use the address obtained by adding one to the context word. This is unsigned 32-bit arithmetic in the representation; at the maximum context value, the numerical sum wraps to zero. Whether the resulting address is accessible is a separate requirement.

### Dependency results consumed locally

**Five dependency results** affect later visible operations:

| Producing dependency | Result | Subsequent use |
|---|---|---|
| F-D05 | 32-bit index result, represented as signed | Scaled by four for the word-store address and the argument to F-D06 |
| F-D10 | Unsigned 32-bit bitfield result | Supplies both byte-store values and all three arguments to F-D11 |
| F-D12 | Uninterpreted four-byte transfer word | Forwarded to F-D13 |
| F-D17 | 32-bit later index result, represented as signed | Shifted left by two for F-D18 |
| F-D22 | A separate 32-bit later index result, represented as signed | Shifted left by two for F-D23 |

Other call results have no visible local use. This does not establish that those calls are effect-free.

### Outputs and preconditions

No returned value is established. Observable behavior consists of the ordered calls, direct memory accesses, dependency effects, and arrival at the represented endpoint if all earlier operations complete.

Reaching that endpoint requires:

- Every executed indirect target to be callable under the applicable machine conventions.
- Every preceding dependency call to return.
- Each direct memory access to be valid in the execution environment.
- The represented path to correspond to executable behavior despite the reported control-flow damage.

No visible validation enforces these conditions.

## Behavior

### Ordered sequence

Opaque dependency IDs distinguish call sites. Different IDs do not establish different runtime implementations.

| Stage | Required order and data flow |
|---|---|
| Initial calls | Invoke F-D01 with the context word logically shifted right by 12. Then invoke F-D02, F-D03, F-D04, and F-D05, in that order. Retain F-D05’s index result. |
| Word store | Store the payload word as four bytes at the numerical address obtained by multiplying the index result by four and adding 20. Then invoke F-D06 with the index result multiplied by four. |
| Bitfield acquisition | Invoke F-D07, F-D08, F-D09, and F-D10, in that order. Retain F-D10’s unsigned 32-bit bitfield result. |
| First byte store | Read an unsigned byte from the context-relative address. Add 17 to that byte’s numerical value to select the destination. Store the low eight bits of the bitfield result shifted left by two at that destination. |
| Scaled call | Invoke F-D11 with three arguments: the 32-bit bitfield result shifted left by two, that same shifted value again, and the bitfield result shifted left by four. |
| Transfer | Invoke F-D12. Pass its four-byte result to F-D13. |
| Later acquisition | Invoke F-D14, F-D15, F-D16, and F-D17, in that order. Retain F-D17’s result. |
| Second byte store | Read the context-relative byte again and retain this newly read value. Select the destination by adding 17 to its numerical value. Clear the highest two bits of the retained bitfield result, logically shift right by 20, and store the low eight bits at that destination. |
| Final calls | Invoke F-D18 with F-D17’s result shifted left by two. Invoke F-D19, F-D20, F-D21, and F-D22, in that order. Invoke F-D23 with F-D22’s result shifted left by two and the retained second byte, zero-extended to a 32-bit unsigned argument. Finally invoke F-D24. |
| Endpoint | Reach the represented invalid-instruction boundary. No continuation is established. |

A call listed without arguments has no explicit arguments in the supplied representation. Its actual machine-level inputs remain unknown.

Dependency effects occur at their respective positions. In particular, intervening calls may affect what a later memory read observes.

### Word-store arithmetic and layout

The word store has a width of exactly four bytes. Its numerical destination is four times the first index result plus 20.

For executions using 32-bit wrapping arithmetic, the destination is:

**(4 × index result + 20) modulo 2³².**

The corresponding argument to F-D06 is the 32-bit scaled index without the added 20.

The representation uses a signed 32-bit quantity for this result and its scaling. It does not establish portable signed-overflow behavior. Consequently, the modulo expression describes the wrapping-machine interpretation, not an unconditional language-level overflow guarantee.

Under that interpretation, the numerical word-store address is divisible by four. This does not establish a valid allocation, array layout, accessible address, or permission to write there.

The evidence establishes a four-byte store but not the order of its constituent bytes in memory. Endianness is **documented_only**.

### First byte value and wider call arguments

The first stored byte is:

**(4 × bitfield result) modulo 256**

or, equivalently:

**4 × (bitfield result modulo 64).**

The grouping is significant. This is not the product reduced modulo 64. For a bitfield result of 16, the stored byte is 64.

Only original bits 0 through 5 affect this byte. Its lowest two bits are always zero, and its possible values are the multiples of four from 0 through 252.

The first two arguments to F-D11 are not narrowed to a byte. They contain the 32-bit left-shift-by-two result. The third argument independently contains the 32-bit left-shift-by-four result. For these unsigned shifts, bits exceeding the 32-bit width are discarded.

### Second byte value

The second byte store uses the same retained bitfield result supplied by F-D10. There is no visible reacquisition of that result.

The mask retains original bits 0 through 29. A subsequent logical right shift by 20 leaves a value that can occupy ten bits. Narrowing that value to a byte retains exactly original bits 20 through 27.

Therefore:

- Original bits 20 through 27 determine the stored byte.
- Original bits 28 and 29 survive the mask and shift but not the final narrowing.
- Original bits 30 and 31 are removed by the mask.
- Original bits below 20 do not affect this byte.

### Destination selection and retained byte

Each byte-store destination is formed from a freshly loaded unsigned byte plus 17. Each destination therefore lies in the inclusive numerical range 17 through 272.

The destination is not the context address plus a field offset. The context-relative access supplies a byte whose numerical value is then used to select a separate low address.

The two reads use the same address expression but occur at different times. The first loaded value is not reused for the second store. Memory changes between the reads can therefore change the second destination.

The second loaded byte has two roles: selecting the second store’s destination and supplying F-D23’s second argument. F-D23 receives this retained byte, not a third read and not the byte written by the second store.

## Side effects

### Direct memory accesses

Excluding dependency internals, instruction fetches, and call mechanics, the visible direct data-memory sequence is:

1. One four-byte store of the payload word.
2. One unsigned byte read from the context-relative address.
3. One byte store to the address selected by that read.
4. A second unsigned byte read from the context-relative address.
5. A second byte store to the address selected by the second read.

The first read follows F-D10 and precedes F-D11. The second follows F-D17 and precedes F-D18. Both reads precede their associated stores.

### Aliasing and state changes

No non-aliasing guarantee is established. The word-store region, context-relative source byte, and either selected byte-store destination may overlap.

Consequences include:

- The word store can change the byte observed by the first or second read.
- The first byte store can change the source of the second read.
- Both byte stores can target the same address.
- A byte store can overwrite part of the earlier four-byte store.
- Dependency effects can modify memory before any subsequent read or store.

If both destination reads produce the same byte, the second direct byte store targets the same address as the first. Immediately after the second store, its written byte is the latest direct write there, subject to external or memory-mapped effects.

No atomicity, synchronization, volatile-access semantics, or concurrency guarantees are established.

### Ownership and lifetime

The visible body establishes no direct allocation, release, ownership transfer, or lifetime extension.

The four-byte value passed from F-D12 to F-D13 has no established interpretation as a pointer, handle, owned object, or borrowed reference. Passing it onward does not establish resource transfer.

Dependency behavior is unresolved. No callee is assumed to preserve globals, leave memory unchanged, retain an object, or release memory.

### Errors and termination

There are no visible validation branches, explicit error results, retries, or rollback actions.

If a dependency fails to return or a memory access faults, later operations are not established to occur. Earlier effects have no visible undo mechanism.

The invalid-instruction boundary is a limitation of the supplied representation. It does not establish successful completion, intentional termination, a particular exception, or a recoverable failure protocol.

## Dependencies

All 24 represented calls are indirect.

| Opaque ID | Explicit arguments | Locally consumed result |
|---|---|---|
| F-D01 | Context word logically shifted right by 12 | None |
| F-D02–F-D04 | None shown | None |
| F-D05 | None shown | First index result |
| F-D06 | First index result multiplied by four | None |
| F-D07–F-D09 | None shown | None |
| F-D10 | None shown | Unsigned 32-bit bitfield result |
| F-D11 | Bitfield shifted left by two; the same value; bitfield shifted left by four | None |
| F-D12 | None shown | Four-byte transfer word |
| F-D13 | Transfer word | None |
| F-D14–F-D16 | None shown | None |
| F-D17 | None shown | First later index result |
| F-D18 | First later index result shifted left by two | None |
| F-D19–F-D21 | None shown | None |
| F-D22 | None shown | Second later index result |
| F-D23 | Second later index result shifted left by two; retained second byte zero-extended to 32 bits | None |
| F-D24 | None shown | None |

F-D01’s target is formed from the incoming control value plus 4,094. The remaining targets use embedded address-like anchors with fixed offsets: 4,094 except for F-D11, whose offset is 4,076.

These target expressions do not resolve dependency identities, calling conventions, executable validity, or instruction-set interpretation. The opaque IDs identify positions in the visible sequence only.

The later index results are represented as signed 32-bit quantities before left shifts. Their overflow-boundary and negative-input behavior is not established as a portable signed-arithmetic contract. Under a 32-bit machine bit-shift interpretation, the arguments retain the low 32 bits of the corresponding fourfold scaling.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes that the preceding calls return and relevant accesses are valid. Any additional arithmetic or memory assumptions are stated explicitly.

### Word-store order and offset

If F-D05 returns 3, the four-byte payload store targets numerical address 32. F-D06 subsequently receives 12.

Calling F-D06 before that store, or storing at address 12 rather than 32, would not match the represented behavior.

### First-byte arithmetic boundary

If F-D10 returns 16, the first byte store writes 64. F-D11 receives 64, 64, and 256.

A first stored value of zero would incorrectly apply modulo 64 to the product rather than to the bitfield before multiplication.

### Byte narrowing versus call arguments

If F-D10 returns 65 and the first context-relative byte read yields 7:

- The first byte store targets numerical address 24.
- It writes 4.
- F-D11 receives 260, 260, and 1,040.

Passing the stored byte value 4 as either of F-D11’s first two arguments would not match the contract.

### Second-byte extraction

A bitfield result of 0x0AB00000 produces a second stored byte of 0xAB.

A bitfield result of 0x3AB00000 also produces 0xAB. The additional original bits 28 and 29 are discarded during byte narrowing.

### Different destinations and retained state

If the first source-byte read yields 7 and the second yields 9, the first store targets address 24 and the second targets address 26.

F-D23 receives 9 as its second argument. If a dependency after the second read changes the source byte to 12, that argument remains 9 because no third source-byte read supplies it.

### Repeated destination

If both source-byte reads yield zero and the bitfield result is 0x0AB00041, both stores target address 17.

The first writes 4. The second writes 0xAB. Immediately after the second direct store, that address contains 0xAB under ordinary byte-addressable memory semantics without concurrent interference.

### Self-affecting source byte

Suppose the context word is 16, so the source-byte address is 17. Suppose its first read yields zero, the bitfield result is 65, and intervening dependencies do not alter that source byte.

The first byte store targets address 17 and writes 4, changing the source itself. The second read consequently yields 4, so the second store targets address 21 and F-D23 receives 4.

This behavior requires no dependency-induced mutation.

### Interrupted execution

If F-D13 does not return, the earlier word store and first byte store have already occurred. The second byte read, second byte store, and calls F-D14 through F-D24 are not established to occur.

### Conditional address wrapping

Under 32-bit wrapping arithmetic, an index result of 0x3FFFFFFB produces a word-store address of zero after scaling and adding 20.

This establishes neither permission to write address zero nor continued execution. No visible guard rejects that address.

## Uncertainties

The following remain **documented_only**:

- **Calling convention:** Locations of implicit inputs, actual machine-level call arguments, and preservation obligations are unresolved.
- **Dependency behavior:** Callees may affect memory, globals, resources, or execution state. Their ownership and preservation behavior is unknown.
- **Target identity:** The indirect targets have not been resolved to independently identified functions.
- **Signed arithmetic boundaries:** The representation shows signed index scaling and shifts but does not establish portable behavior for overflowing or negative operands.
- **Memory environment:** Address validity, endianness, access atomicity, concurrency, and memory-mapped behavior are unknown.
- **Data model:** No allocation, array, structure, register block, or application-level layout is established.
- **Endpoint:** No complete return, failure, exception, or continuation contract is available.
- **Representation reliability:** Reported instruction and control-flow damage limits confidence that the entire represented sequence is one valid executable path.

Describing the operations as publishing or transforming structured state would be inferred intent only. No such interpretation is required by this contract.

## Evidence

Established static evidence supports:

- A represented parameterless, return-valueless interface with implicit incoming state.
- Twenty-four ordered indirect call sites.
- Five dependency results consumed by later visible operations.
- One four-byte store, two separate unsigned byte reads, and two byte stores.
- First-byte arithmetic equivalent to **(4 × bitfield result) modulo 256**, or **4 × (bitfield result modulo 64)**.
- Wider, non-byte-narrowed arguments to F-D11.
- Extraction of original bitfield bits 20 through 27 for the second stored byte.
- Reuse of the bitfield result across both stores.
- Retention of the second loaded byte for F-D23.
- A terminal invalid-instruction boundary and a warning concerning damaged control flow.

No runtime tests have been performed. Acceptance examples are static consequences of the supplied representation, conditional on the execution, arithmetic, and memory assumptions stated above.
