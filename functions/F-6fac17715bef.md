<!-- Trusted provenance: F-6fac17715bef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fac17715bef

## Purpose

F-6fac17715bef performs an ordered sequence of indirect dependency calls, 32-bit memory accesses, and byte accesses, then enters a represented unconditional loop. The loop continues making calls and clearing destinations selected through a fixed byte-selector location.

No application-level purpose is established. Object initialization, decoding, device control, and protocol processing are not established interpretations.

This contract describes the supplied static representation. Warnings about invalid instruction data, removed control-flow regions, and overlapping global symbols limit confidence that the representation accurately describes executable behavior. The visible call schedule and memory operations remain distinguishable from unknown dependency semantics.

## Interface

### Inputs and execution state

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits, presented as signed | Shifted left by 23 for the initial call. |
| Second argument | 32 bits | No explicit use in the visible body. |
| Third argument | 32 bits, unsigned | Logically shifted right by 7 for the initial call. |
| Fourth argument | 32-bit address-like value | Base of the initial word store at offset `0x74`. |
| Implicit incoming execution state | Address-sized in the representation | Determines the initial indirect target after addition of `0x1E8`. |

The explicit arguments alone do not determine execution. An unused displayed argument is not proof of irrelevance to the actual calling convention.

### Outputs and completion

The displayed interface returns no value. No normal return is visible: the terminal loop has no represented exit condition.

Observable outputs are dependency invocations, their explicit arguments and order, direct memory effects, and any effects of dependencies. Calls or accesses may fault, terminate execution, or fail to return. No error code, success indication, rollback, or recovery path is established.

### Widths and preconditions

The behavioral arithmetic below uses 32-bit words, unsigned 8-bit selector reads, and one-byte stores. Multiplication, addition, and left shifts retain the low 32 bits. Right shifts described below are logical. This is a bit-pattern interpretation of the representation, not a guarantee of portable signed-arithmetic semantics.

The initial result is represented as 64 bits. Its lower and upper 32-bit halves are captured separately. Other captured ordinary results used below are 32-bit values.

Each word store writes four bytes. Endianness, alignment restrictions, address-space mapping, and hardware-register semantics are unknown.

Progress requires executable call targets, compatible calling conventions, returning dependencies where continuation is expected, and accessible memory at every reached address. No visible checks enforce these conditions.

## Behavior

### Notation and dependency identification

The following descriptive names identify captured values, not source identifiers:

- **Initial word** and **initial upper word**: the lower and upper halves of the initial 64-bit result.
- **Descriptor**: the result of schedule group F-D02.
- **First byte source**: the result of F-D04.
- **Payload selector**: the result of F-D08.
- **Payload**: the word read immediately after F-D08.
- **Destination base**: eight times the payload logically shifted right by 8.
- **Secondary word**: the secondary-register-derived value multiplied by `0x01000000`.
- **Second byte source**: the result of F-D28.
- **Final byte source**: the result of F-D30.

A **four-scaled slot** for a value is address `4 × value + 0x14`. A **sixteen-scaled slot** is address `16 × value + 0x14`. These are address families, not established structures.

An **indexed byte write** at an index first reads one unsigned byte at `index + 1`, then writes at that byte’s value plus `0x11`. Its destination therefore lies between `0x11` and `0x110`. Each occurrence performs a fresh selector read, including repeated clears.

The schedule below gives every displayed call position. Within a group, calls have opaque site IDs formed by the group ID and their one-based position, such as F-D01.1. Expanding a repetition assigns consecutive positions.

Schedule terminology:

- **Bare call**: no explicit arguments are displayed; its ordinary result is not captured for subsequent direct use.
- **Result call**: no explicit arguments are displayed; its result is captured.
- **Relay pair**: a result call followed immediately by a call explicitly receiving that captured result as its sole argument.
- **Group result**: the result of the group’s final result call.

A bare call is not established to have no machine-level inputs or effects. Opaque site IDs distinguish documented positions, not necessarily distinct executable targets.

### Initial invocation

F-I001 is the initial indirect dependency, selected using the implicit incoming execution state plus `0x1E8`. Its explicit arguments, in order, are:

1. Third argument logically shifted right by 7.
2. First argument shifted left by 23.

Its lower result word is written to the fourth argument plus `0x74` before F-D01 begins.

### Ordered call and memory schedule

Every row follows the preceding row. Operations in the final column occur after that row’s calls and before the next row’s calls. Captures remain the earlier sampled values despite intervening dependency effects.

| Group | Dependency calls in order | Following direct effects and retained values |
|---|---|---|
| F-D01 | Four-input call receiving the initial upper word shifted left by 3 twice, then the initial word twice; relay pair; relay pair; three bare calls; result call. | Write the initial word at `4 × group result + 0x74`. |
| F-D02 | Call receiving four times F-D01’s result; three bare calls; result call. | Capture the descriptor. Write `(descriptor masked by 0x3FFFFFFF) >> 5` to its sixteen-scaled slot. Retain `(descriptor masked by 0x3FFFFFFF) >> 6` as the first byte index. |
| F-D03 | Call receiving four times the descriptor; three bare calls; result call. | Write the same descriptor-derived shifted-by-5 word to the group result’s four-scaled slot. |
| F-D04 | Call receiving four times F-D03’s result; three bare calls; result call. | Capture the first byte source. Perform an indexed byte write at the first byte index, storing the low byte of that source shifted left by 2. |
| F-D05 | Call receiving the first byte source shifted left by 2; three bare calls; result call. | Perform a fresh indexed byte write at the first byte index, storing the low byte of `(first byte source masked by 0x3FFFFFFF) >> 20`. |
| F-D06 | Call receiving F-D05’s result shifted left by 2; four bare calls. | Sample the global at numeric address `0x3C`, converting its represented value to 32 bits. Write that captured word at `4 × (initial word >> 7) + 0x14`. |
| F-D07 | Call receiving the captured global word; three bare calls; result call. | Write the same captured global word again to the same address. |
| F-D08 | Call receiving F-D07’s result shifted left by 2; three bare calls; result call. | Capture the payload selector. Write four times it to its sixteen-scaled slot, then read the payload from its four-scaled slot. Retain the selector shifted left by 26, with truncation. |
| F-D09 | Four-input call receiving sixteen times the payload selector, four times it, its retained left-26 value, and four times it again; relay pair; relay pair; three bare calls; result call. | Rewrite the payload selector’s sixteen-scaled slot with the earlier four-times value. |
| F-D10 | Call receiving four times F-D09’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot, then read a word from its four-scaled slot. Logically shift the retained payload-selector left-26 value right by 7. |
| F-D11 | Two-input call receiving four times F-D10’s result twice; relay pair; relay pair; three bare calls; result call. | Write the word sampled after F-D10, logically shifted right by 8, to the same four-scaled slot from which it was sampled. |
| F-D12 | Call receiving four times F-D11’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. Retain that four-times base. |
| F-D13 | Call receiving the payload shifted right by 16; three bare calls; result call. | Write the retained F-D12 four-times base to F-D12’s four-scaled slot. The stored value excludes the slot’s `0x14` offset. |
| F-D14 | Call receiving F-D13’s result shifted left by 2; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. |
| F-D15 | Call receiving sixteen times F-D14’s result; relay pair; result call; two-input call receiving that immediately preceding result and the payload shifted right by 16; three bare calls; result call. | Rewrite F-D14’s sixteen-scaled slot with the earlier four-times value. |
| F-D16 | Call receiving four times F-D15’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. Shift the retained payload-selector value right by another 7. Derive the destination base. |
| F-D17 | Two-input call receiving four times F-D16’s result and the destination base; relay pair; result call; two-input call receiving that immediately preceding result and the payload shifted right by 16; three bare calls; result call. | Write four times F-D16’s result at destination base plus `0x14`. |
| F-D18 | Call receiving four times F-D17’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. |
| F-D19 | Four-input call receiving the destination base twice, the payload shifted right by 16, and the retained payload-selector value shifted right by a third 7; three bare calls; result call. | Write the destination base itself at destination base plus `0x14`. |
| F-D20 | Call receiving F-D19’s result shifted left by 2; four bare calls. | Consume the represented secondary-register value associated with the final call. Form the secondary word and write it at `secondary-register value × 0x04000000 + 0x14`. |
| F-D21 | Call receiving the secondary word; three bare calls; result call. | Write the captured secondary word to the group result’s four-scaled slot. |
| F-D22 | Call receiving four times F-D21’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. |
| F-D23 | Call receiving four times F-D22’s result; relay pair; relay pair; three bare calls; result call. | Write four times F-D22’s result to the group result’s four-scaled slot. |
| F-D24 | Call receiving four times F-D23’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. Retain that four-times value for the later byte index. |
| F-D25 | Call receiving four times F-D24’s result; relay pair; relay pair; three bare calls; result call. | Write the payload shifted right by 24, as a 32-bit word, to the group result’s four-scaled slot. |
| F-D26 | Call receiving four times F-D25’s result; three bare calls; result call. | Write four times the group result to its sixteen-scaled slot. Form the second byte index as `(retained F-D24 four-times value masked by 0x7FFFFFFF) >> 7`. |
| F-D27 | Three-input call receiving sixteen times F-D26’s result twice, then the payload shifted right by 23; three bare calls; result call. | Rewrite F-D26’s sixteen-scaled slot with sixteen times F-D26’s result. |
| F-D28 | Call receiving F-D27’s result shifted left by 2; three bare calls; result call. | Capture the second byte source. Perform an indexed byte write at the second byte index, storing the low byte of that source shifted left by 2. |
| F-D29 | Three-input call receiving the second byte source shifted left by 2 in all three positions; relay pair; three bare calls; result call. | Perform a fresh indexed byte write at the second byte index, storing the low byte of `(second byte source masked by 0x3FFFFFFF) >> 20`. |
| F-D30 | Call receiving F-D29’s result shifted left by 2; three bare calls; result call. | Capture the final byte source and its left-2 value. Store that value’s low byte at `(payload >> 31) + 0x1C`, then perform an indexed clear at the second byte index. |
| F-D31 | Call receiving the payload shifted right by 31; bare call; call receiving `(final byte source masked by 0x3FFFFFFF) >> 6`; three bare calls; result call. | Perform another indexed clear at the second byte index, rereading its selector. |
| F-D32 | Call receiving F-D31’s result shifted left by 2; three bare calls; result call. | Capture `(group result masked by 0x3FFFFFFF) >> 25` as the initial loop shift value. Form the fixed clearing index as the retained final-byte-source left-2 value multiplied by `0x400`. Perform an indexed clear at that fixed index. |
| F-D33 | Call receiving twice the initial loop shift value; result call. | Use the group result as the first iteration’s opening first argument. Enter the loop with the entry-specific opening target. |

The first byte store in each byte pair has its lowest two bits clear. The second byte store retains source bits 20 through 27; the wider intermediate expression does not widen the store.

The payload capture’s store precedes its read without an intervening displayed dependency call. The addresses can alias. For a zero payload selector, both are `0x14`, and the store writes zero before the read.

The F-D11 write uses the earlier captured word, not a new read at write time. Likewise, both global publications use one captured global value.

The three right-7 operations applied to the retained payload-selector value equal a logical right shift by 21 of its already-truncated left-26 value. They do not recover discarded bits.

### Terminal loop and target transition

The opening indirect target differs between initial entry and the backedge:

- **F-L001** identifies the opening target used on the first iteration.
- **F-L002** identifies the different represented opening target used on every subsequent iteration.

The state selecting this target has its entry value after F-D33 and is overwritten during the loop body. The backedge therefore does not repeat the first iteration’s complete target schedule.

Each iteration has this order:

| Position | Dependency or direct effect |
|---|---|
| Opening | F-L001 on initial entry, otherwise F-L002, receives the carried first argument and the carried shift value logically shifted right by 8. |
| Next four calls | F-L003, F-L004, and F-L005 are bare calls. F-L006 is a result call. |
| Clear | Read the selector at the fixed clearing index plus one and clear the selected byte destination. |
| Following call | F-L007 receives F-L006’s result shifted left by 2. |
| Final four calls | F-L008, F-L009, and F-L010 are bare calls. F-L011 is a result call. |
| Backedge state | The next first argument becomes F-L011’s result shifted left by 2. The next carried shift value becomes that same result shifted left by 26. The next opening target is F-L002. |

The initial loop shift value is between 0 and 31, so the first opening call’s second argument is zero. On subsequent iterations, that argument equals the previous F-L011 result’s low six bits shifted left by 18.

The clearing index remains fixed. Its selector byte is reread each iteration and may have changed through earlier writes, dependencies, or external activity. There is no represented ordinary exit.

## Side effects

Direct effects comprise word stores, two word reads, byte-selector reads, byte stores, and one global sampling operation. Their ordering relative to calls is contractual within the supplied representation.

No allocation, release, ownership transfer, reference counting, or lifetime guarantee is established. Passing a numeric value that resembles an address does not establish ownership.

Dependencies must not be assumed to preserve globals, selector bytes, stored words, or other memory. Repeated stores remain separate observable operations even when their addresses and values match.

No synchronization, atomicity, transaction, or rollback guarantee is shown. Earlier effects remain performed if later execution fails.

## Dependencies

F-I001, the expanded F-D site IDs, and F-L001 through F-L011 are opaque dependency identifiers. They express the visible schedule and argument provenance without assigning application semantics.

Their complete signatures, target equivalences, hidden inputs, register effects, memory effects, and termination behavior are **documented_only** unknowns. Different site IDs do not prove different actual functions.

The final F-D20 call is followed by consumption of represented secondary-register state. Its provenance is visible at that boundary, but its validity as a defined secondary result is not established. It must not be replaced by the call’s ordinary result or derived from the explicit inputs alone.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial arguments and publication.** With first argument `1` and third argument `0x180`, F-I001 receives `3`, then `0x00800000`. If its lower result is `0x11223344`, that word is stored at fourth argument plus `0x74` before F-D01.1.

2. **Descriptor derivation and call input.** For descriptor `0xFFFFFFFF`, its first word store contains `0x01FFFFFF`, its first byte index is `0x00FFFFFF`, and F-D03.1 receives `0xFFFFFFFC`.

3. **First byte pair and separate provenance.** For first byte source `0x12345678`, the two stored bytes are `0xE0` and `0x23`. F-D05.1 receives `0x48D159E0`. F-D06.1 instead receives the separate F-D05 result shifted left by 2.

4. **Fresh selector reads.** Selector values `5` and later `9` select destinations `0x16` and `0x1A`. A dependency changing the selector between byte stores changes the second destination.

5. **Captured global.** If the global sample converts to `7`, both direct publications store `7`, even if intervening dependencies change the global to `9`.

6. **Store/read alias.** With payload selector zero, ordinary memory, and no intervening external mutation, the payload read at `0x14` observes the zero just stored there.

7. **Three explicit copies.** With second byte source `0x12345678`, F-D29.1 receives `0x48D159E0` in each of its three explicit argument positions after the first indexed byte store.

8. **High-bit destination and following call.** If the payload’s high bit is set, the later non-indexed byte store targets `0x1D`; otherwise it targets `0x1C`. After that store and the indexed clear, F-D31.1 receives respectively `1` or `0`.

9. **Loop target transition.** If calls return and accesses succeed, the first opening call uses F-L001 with second argument zero. The second iteration opens through F-L002, not F-L001.

10. **Loop arithmetic boundary.** If an iteration’s F-L011 result is `0xFFFFFFFF`, the next opening arguments are `0xFFFFFFFC` and `0x00FC0000`. The fixed selector location is reread before the next clear.

## Uncertainties

The following remain **documented_only**:

- Accuracy of reconstructed control flow, including the apparent unconditional loop.
- Actual dependency identities and complete calling conventions.
- Meaning of the incoming execution state used for F-I001.
- Definedness of the secondary-register value consumed after F-D20.
- Width of the underlying global read at `0x3C`, before conversion.
- Endianness, alignment, mappings, and hardware access semantics.
- Valid input ranges, environmental initialization, ownership, and lifetimes.
- Effects of concurrency, faults, exceptions, and non-returning dependencies.

A structured low-level operation is a possible interpretation of the repetitive pattern, but no protocol or application meaning is established.

## Evidence

Established static evidence includes the four displayed arguments, implicit initial target state, represented 64-bit initial result, ordered indirect calls, explicit argument forwarding, word and byte accesses, truncated arithmetic, secondary-register consumption, and different loop-opening targets on entry and backedge.

The supplied representation reports invalid instruction data, discarded control-flow regions, and overlapping global symbols. These warnings limit executable-level conclusions; they do not make displayed argument values or call positions unknown.

No runtime tests, dependency inspection, executable validation, or independent memory-layout verification have been performed. Acceptance examples are conditional consequences of the static representation, not observed execution results.
