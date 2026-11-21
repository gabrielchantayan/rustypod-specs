<!-- Trusted provenance: F-40f1cbfcbc0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-40f1cbfcbc0b

## Purpose

F-40f1cbfcbc0b performs an ordered sequence of dependency invocations, byte-selector operations, computed memory accesses, and shared-storage updates. Its terminal sequence clears a dependency-exposed byte, constructs a shared 32-bit value, clears a halfword, and invokes address zero.

No application-level purpose is established. In particular, the supplied representation does not establish initialization, allocation, decoding, cleanup, or a service interface.

This contract distinguishes:

- **Established static evidence:** operations, widths, retained-value relationships, and ordering visible in the supplied representation.
- **Inferred intent:** possible interpretations, not behavioral requirements.
- **documented_only:** unresolved properties that must not be supplied by assumption.

All requirements are conditional on the supplied representation matching execution and on preceding calls and memory accesses completing.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, interpretation unspecified | No explicit use appears. |
| Second argument | Unsigned 32 bits | Passed to the first dependency; later supplies selector-source addresses after logical right shift by 9. |

The first argument’s lack of explicit use does not establish irrelevance to the actual calling convention. Dependencies may observe machine state not captured by the displayed argument lists.

### Implicit inputs and preconditions

Execution depends on:

- An incoming continuation-related value determining the first call target.
- An incoming base-like value determining the first selector-source address.
- Dependency results, including secondary machine results and split 64-bit results.
- Contents of computed memory locations, absolute low addresses, and shared storage.
- A stack-resident address supplied repeatedly to one dependency.

Completion requires every reached memory access and invocation to be valid in the execution environment. Valid input ranges, memory mappings, alignment requirements, stack extent, and actual calling conventions are not established. No validation enforces these preconditions.

### Outputs and completion

The represented interface has no explicit return value. Outputs are memory effects and dependency interactions.

A return is represented only after an invocation of address zero with three zero arguments. That invocation is not established to be valid or to return; normal completion is therefore not guaranteed.

### Widths and arithmetic

Direct accesses include 8-bit bytes, 16-bit halfwords, and 32-bit words. Some dependency results are represented as 64 bits and divided into low and high 32-bit halves.

Unsigned right shifts are logical. Byte stores retain the low eight bits. Unsigned byte reads widened to 32 bits produce values from 0 through 255. Widening of unspecified byte and halfword scalar types at dependency boundaries remains **documented_only**.

Calculations explicitly carried through unsigned 32-bit quantities use 32-bit arithmetic, including wraparound. Signed multiplication and left-shift expressions do not independently establish portable overflow semantics. Any machine-width interpretation of those expressions is conditional, not an unconditional guarantee.

Endianness, alignment behavior, and executable access behavior for overlapping shared symbols remain unresolved.

## Behavior

### 1. Execution and observation boundaries

The visible path is linear: no explicit loop, conditional control-flow branch, validation branch, or recovery path appears.

Dependencies intervene between many direct accesses. Each invocation is an observation and mutation boundary. Dependencies are not assumed to preserve globals, memory, ownership, or allocation lifetime.

Consequently, repeated loads remain separate observations, repeated stores remain separate effects, and a previously stored value cannot be propagated through calls into a later shared-memory load. Conversely, a value explicitly captured before calls remains the operand of subsequent calculations that use that capture; it must not be replaced by a fresh global read.

The layout descriptions below identify access relationships, not proven objects or arrays.

### 2. Initial dispatch and indexed word updates

F-000000000001 receives the second explicit argument. Several F-000000000002 interactions occur before the first direct write.

The first direct operation reads an unsigned byte one byte beyond the incoming base-like value, then stores zero at the absolute address equal to that byte plus `0x11`. The destination is between `0x11` and `0x110`, inclusive. The read byte selects an address; it is not a pointer-width load.

Subsequent calls receive dependency-derived quantities scaled by four. One call receives four zero arguments.

The early word operations include:

- A 32-bit store of a secondary result at `0x14 + 4 × that result`.
- A 32-bit load at `0x14 + 0x400 × (captured unsigned result logically shifted right by 18)`. The loaded word is logically shifted right by 19 before forwarding.
- Stores at `0x14 + 4 × dependency-selected index`.
- A store at `0x14 + 16 × index`, containing `4 × index`.
- A subsequent load at `0x14 + 4 × that index`, shifted right by 19 for a dependency argument.
- A later store carrying the retained four-times-scaled value to another selected word slot.

The factor-of-four and factor-of-sixteen address families are distinct. Their accesses are not established as disjoint.

### 3. Second-argument selectors and packed fields

A captured unsigned result supplies two byte stores. Each separately reads its selector at `(second argument logically shifted right by 9) + 1`, then writes at selector plus `0x11`.

The first store contains the low byte of the captured result shifted left by 2. After intervening calls, the second contains the low byte of the result masked with `0x3fffffff` and logically shifted right by 20. The selector is reread; the destinations may differ.

The same captured result supplies fields obtained by masking to 30 bits and shifting right by 17 and by 7. A later invocation has seven represented arguments, including four repetitions of one result, both fields, and a call-site-related quantity. The actual ABI remains **documented_only**.

A following 64-bit result is split. Its low half is retained for later calculations. Before the high half is forwarded, the function clears a selector-selected byte using a selector read one byte beyond the earlier 30-bit field shifted right by 7.

### 4. Repeated selector clears and payload access

Further calls alternate with fresh selector reads and zero-byte stores at selector plus `0x11`.

Selector-source addresses in this span include:

- A retained field plus 1.
- A retained value shifted right by 9, plus 1.
- A retained value plus 1.
- Absolute address 1.
- A retained value multiplied by `0x1000`, plus 1.
- A retained value shifted right by 19, plus 1.
- An address equal to 1 or `0x80000001`, selected through a retained shifted value under unsigned 32-bit arithmetic.

Distinct payload accesses include:

- An unsigned byte at twice a retained unsigned value.
- A word at a retained base plus either zero or `0x80000000`, according to bit 19 of that base.
- A byte at offset `0x0e` from a secondary-result-derived base.
- A byte at a retained base plus a secondary-result-derived displacement.
- A halfword at the sum of a retained value shifted right by 19 and a previously captured unsigned byte.

Several payloads are supplied in four repeated argument positions.

One invocation receives the same stack-resident address four times. The representation does not establish initialized contents, writable extent, permission to retain the address, or ownership transfer.

### 5. Offset-byte changes and further indexed stores

After the halfword path, two selected word slots receive a retained right-shifted field.

A separate retained base receives two byte stores at offset `0x0d`: first the low byte of twice that base, then the constant 4 after intervening calls. The first store is independently observable.

Other operations include:

- A word load at `0x44 + 4 × captured value`.
- A later word load at `0x44 + ((captured value masked to 30 bits) logically shifted right by 20)`.
- A word store at `0x14 + 0x100 × selected index`, containing four times an earlier captured value.
- A selected word store containing a value masked with `0x03ffffff` and shifted right by 13.
- Repeated factor-of-sixteen stores containing four times their index, followed by stores of retained scaled values into separately selected slots.

The two `0x44` loads have different scaling. Neither alignment nor an array layout follows from their common offset.

### 6. State at offsets `0x24`, `0x74`, and `0x15`

A factor-of-sixteen-derived base receives a zero word at offset `0x24`; its word at offset `0x74` is then loaded and retained as another base.

At that retained base:

- A dependency result logically shifted right by 20 is stored as a word at offset `0x24`.
- A word at offset `0x74` is loaded for a dependency argument.
- A selector at offset 1 chooses a packed-field byte-store destination.
- Following calls, the selector is reread for a zero-byte store.

The value shifted right by 20 is at most `0xfff`. Its further right shift by 21 is necessarily zero. Calls receiving that derived field therefore receive zero.

Three unsigned bytes are later read at offset `0x15` from three dependency-derived bases. Calls and selector clears separate these observations. The third captured byte is subsequently stored, zero-extended, into a selected 32-bit slot.

### 7. Bit-derived values and shared-region reset

A secondary result masked with `0x1fffffff` and shifted right by 18 produces an 11-bit field. That field shifted left by 13 feeds calls and a selected word store.

Another captured result supplies a low-byte left-shifted store, a masked right-shifted byte store, and a field formed by masking its already-left-shifted value with `0x1fffffff`, then shifting right by 6. A selector source in this span reduces to absolute address 1 because its retained address operand was explicitly set to zero.

A separate secondary result is shifted left by 3, then by 2, and later by 2 again. Under 32-bit bitvector semantics, the final stored quantity is the original result shifted left by 7. Earlier stages remain relevant where separately consumed.

The function clears a shared 32-bit location, **region A**. Numerous calls and accesses follow. Later region-A loads are fresh observations, not guaranteed zeros.

### 8. Zero-derived addressing and byte indirection

A later retained unsigned quantity is explicitly zero. Its subsequent masked and shifted forms also remain zero until replaced. This constrains several selector-source addresses to absolute address 1 and several stored or forwarded fields to zero.

A payload byte captured at offset 1 from a dependency-derived base is later used numerically: another byte is read at that captured byte plus 1. This is byte-sized address indirection, not reconstruction of a pointer from neighboring bytes.

A later zero-derived payload address is absolute address 1. Its byte is captured, passed to a dependency, and subsequently stored zero-extended into a selected word slot. Intervening mutations do not retroactively change the captured byte.

Some intermediate reductions would produce at most 11 bits even without the stronger known-zero fact. Shifting such a quantity right by 19 or 22 necessarily produces zero.

### 9. Late linked accesses and publication

A late zero-byte store targets `9 + 0x2000 × a retained four-times-scaled result`. Another retained base receives its own low byte at offset 9, followed by a word load at offset `0x34` for a dependency argument.

A secondary result selects a word store at `0x14 + 4 × that result`. The next word load instead uses `0x14 + that result`. These accesses have different scaling and may overlap.

A captured word from this chain determines a word load at `0x14 + (captured word logically shifted right by 22)`. Call the resulting retained unsigned word **W**. Following dependency calls, the consequential sequence is:

- Store a dependency-derived value shifted left by 2 as a 32-bit word at `0x14 + 4 × (W logically shifted right by 1)`.
- Read a 32-bit word at `W + 0x14`, using the retained W.
- Only then invoke the next dependency, passing W shifted right by 1.

The store and load are not established as disjoint. The load must observe the state after the store; using a value read earlier or moving the next call before the load changes the represented ordering. The loaded word is retained for subsequent right-shifted stores.

A fresh region-A value is captured and shifted right by 23. A further right shift by 23 necessarily yields zero, since the first reduction leaves at most nine bits.

Later, the function explicitly captures the unsigned byte at absolute address 1. Call this snapshot **S**. It immediately clears the byte at `S + 0x11`, then crosses multiple dependency calls. A dependency result is published as a 32-bit value to **region B**, passed onward, and followed by another invocation.

Only afterward does the function load a 32-bit word at `S + 0x74` and store its low byte at `S + 1`. Both addresses use the retained snapshot S, not a fresh read of address 1. Dependencies may change address 1 without changing these retained address operands.

Subsequent word-to-byte updates use other dependency-derived, four-times-scaled bases: load a word at offset `0x74`, then store its low byte at offset 1. One update is repeated at the same retained base after intervening calls and requires a fresh word load. A further captured word is retained for later indexed stores.

### 10. Terminal sequence

The final span captures region A again. Calls and selected word stores intervene before a word is loaded at offset `0x14` from that captured region-A value.

The loaded word supplies a low-byte selector-target store and, later, a selector-target store of its value shifted right by 22. The latter selector-source calculation uses a separate captured result masked to 30 bits and shifted right by 21.

After further calls, a final byte-sized dependency result is obtained. The terminal order is:

1. Store zero through a byte pointer exposed as a secondary machine result.
2. Evaluate the represented 24-bit quantity associated with absolute address 9 while constructing the final region-B assignment.
3. Store the combined 32-bit value to region B: that quantity occupies bits 31–8; the final dependency byte occupies bits 7–0.
4. Store a zero halfword at `(retained unsigned value masked with 0x3fffffff) logically shifted right by 7`.
5. Invoke F-000000000004 at address zero with three zero arguments.

The represented evaluation of the 24-bit quantity follows the pointer-directed byte clear. That clear may affect overlapping storage; moving the evaluation before it is not justified. The underlying storage interpretation and actual executable access width or mechanism remain **documented_only**.

## Side effects

Direct effects include low-address byte writes, computed word stores, shared-region updates, a terminal halfword clear, and exposure of a stack address.

No allocation, release, ownership transfer, synchronization, atomicity, or rollback guarantee is established. Dependency effects remain unknown.

The supplied material flags overlapping shared symbols. Regions A and B, absolute locations, computed accesses, and dependency-exposed pointers must not be presumed disjoint.

No explicit error result or recovery path appears. Invalid memory, alignment, targets, or dependency results may prevent later effects. No particular trap or exception behavior is established, and no visible mechanism rolls back completed effects.

## Dependencies

| Opaque ID | Documentary role |
|---|---|
| F-000000000001 | First unresolved indirect invocation, receiving the second argument. |
| F-000000000002 | Remaining calculated-target interactions; an aggregate handle, not one proven callee. |
| F-000000000003 | Separately represented fixed-target invocation receiving two equal arguments. |
| F-000000000004 | Final address-zero invocation receiving three zeros. |

These handles do not establish identities, shared implementations, or ABI correctness. Return guarantees, secondary-result provenance, global preservation, ownership effects, retention permissions, and failure behavior are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Selector bounds:** A selector of `0xff` targets `0x110`. Neighboring bytes do not participate.

2. **Fresh selectors:** Selector reads returning 3 and 7 across intervening calls target `0x14` and `0x18`, respectively.

3. **Byte truncation:** A captured unsigned value `0x41`, shifted left by 2, supplies byte `0x04`.

4. **Known-zero field:** Any unsigned 32-bit value shifted right by 20 and then by 21 produces zero.

5. **Fresh region A:** After the explicit clear, a dependency may replace region A. A later load must not be forced to zero.

6. **Retained W and aliasing:** If W is zero and the scaled dependency value is `0x11223344`, the late store and following word load both address `0x14`. With ordinary writable memory and no external interference, the load obtains the just-stored value before the next invocation.

7. **Retained absolute-byte snapshot:** If S is 5, the clear targets `0x16`. Even if intervening dependencies change address 1 to 9, the post-publication word load remains at `0x79`, and its low byte is stored at `0x06`.

8. **Full-width observation:** A word-to-byte update requires the represented 32-bit load followed by truncation. A byte load is not an established substitute.

9. **Terminal overlap:** If the final pointer clear changes storage contributing to the 24-bit quantity, constructing region B uses the quantity evaluated afterward. Its actual executable access remains unresolved.

10. **Terminal failure:** If address zero cannot be invoked, normal return is not established. Prior effects have no visible rollback.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid input domain.
- Accuracy of represented signatures, argument counts, and secondary machine results.
- Signed-overflow and signed-left-shift interpretation.
- Endianness, alignment, memory mapping, and executable access behavior.
- Exact shared-storage layouts and aliases.
- Stack-address extent and retention permission.
- Dependency ownership, global mutation, and failure semantics.
- Actual access mechanism for the terminal 24-bit quantity.
- Validity and return behavior of the address-zero invocation.

Repeated scaling may suggest a structured representation, but that inferred intent does not establish an object model.

## Evidence

This contract derives from static inspection of the supplied representation, treated as evidence rather than independently validated executable behavior.

The representation establishes the retained absolute-address-1 snapshot and its reuse after region-B publication; the retained-W store-before-load sequence preceding the next call; and evaluation of the terminal 24-bit quantity after the pointer-directed clear.

No runtime tests, traces, memory observations, or dependency implementations were available or used. Acceptance examples express conditional arithmetic and ordering consequences, not execution results.
