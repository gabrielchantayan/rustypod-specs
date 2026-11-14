<!-- Trusted provenance: F-1a4eb64bdd3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1a4eb64bdd3a

## Purpose

F-1a4eb64bdd3a describes an ordered sequence of indirect calls, byte accesses, and 32-bit word accesses. Addresses depend on incoming state, loaded values, dependency results, and auxiliary register outputs. Several scalar snapshots remain available across intervening calls.

No application-level purpose is established. The visible operations do not establish allocation, serialization, collection management, or a device protocol.

This contract is conditional on the supplied static representation accurately describing execution. That representation reports invalid instruction data and ends at an undecodable boundary. Normal return and subsequent execution are not established.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use. |
| Second | 32 bits | Supplied three times to F-U001. |
| Third | 32 bits | No explicit use. |
| Fourth | Signed 32 bits | Base for a word read at offset `0x04`; also shifted left by 19 and supplied twice to F-U011. |

The absence of an explicit use does not establish irrelevance under the actual calling convention.

Additional incoming state comprises an unsigned 32-bit address-like value, an eight-bit value, and a control-flow value. Their relationship to the explicit arguments is unknown. Auxiliary register outputs later supply a signed index and an unsigned value.

The represented return type is void, but no ordinary return is visible.

### Widths, addressing, and preconditions

A **byte** is eight bits; a **word** is 32 bits. Byte stores retain the low eight bits. Selector-byte loads are unsigned.

Represented 64-bit results have separately consumed low and high 32-bit halves. Their decomposition does not establish a memory layout or ownership meaning.

Unsigned word arithmetic retains the low 32 bits, including multiplication and left shifts. Unsigned right shifts are logical. Signed shifts, signed multiplication, conversions into signed carriers, and overflowing signed address calculations require semantics not fully established by this representation. References below to a word’s bits do not silently resolve those signed-operation uncertainties.

Offsets describe access locations, not proven structure fields. Endianness, alignment, object extents, and address-space mappings are unknown. Reaching an operation requires its address or call target to be usable in the actual environment; no visible checks enforce this.

Descriptive labels below identify retained scalar values, not source identifiers or owned objects.

## Behavior

F-U001 through F-U141 identify successive call occurrences. A range denotes separate calls in increasing order. Unless otherwise specified, no explicit arguments or captured ordinary result are shown. This does not establish a zero-argument ABI or a void-returning dependency.

The sequence below specifies visible local accesses and call relationships. Dependencies may introduce additional effects.

### 1. Opening selector and calls

Read a byte at the incoming address plus `0x01`. Store the incoming eight-bit value at absolute address `0x11` plus that selector. The destination is therefore within `0x11` through `0x110`.

F-U001 receives three copies of the second explicit argument. Its represented target is the incoming control-flow value plus `0xffe`.

F-U002–F-U005 follow. F-U005 supplies a signed 32-bit result.

### 2. Small-base accesses and initial snapshots

Read a byte at the incoming address logically shifted right by two, plus `0x01`. Retain its unsigned value as the **small base**. Write zero at small base plus `0x11`.

F-U006 receives the F-U005 result shifted left by two. F-U007–F-U010 follow; F-U010 supplies a signed result.

Read a word at fourth argument plus `0x04`, then a word at small base plus `0x74`. Retain the latter as the **initial snapshot** and write its low byte at small base plus `0x01`.

F-U011 receives, in order:

- The initial snapshot.
- Two copies of the fourth argument shifted left by 19.
- The F-U010 result shifted left by two.

F-U012 supplies a word. Reduce the first loaded word by logical right shift of 22, producing a **ten-bit value**. F-U013 receives the F-U012 word and the initial snapshot logically shifted right by two.

### 3. Returned-base snapshot and indexed stores

F-U014–F-U017 execute; F-U017 supplies a signed result. Scale that result by four, read a word at the resulting base plus `0x74`, and write the loaded word’s low byte at base plus `0x01`.

F-U018 receives the scaled base. After F-U019–F-U022, an auxiliary signed register output supplies an index. Store the previously loaded full word at `0x14` plus four times that index. F-U023 receives the index itself.

F-U024 supplies a word. F-U025 receives that word followed by two copies of the ten-bit value masked with `0x1ff` and shifted left by 21.

F-U026–F-U029 follow; F-U029 supplies a signed destination index. Store the ten-bit value shifted left by 23 at `0x14` plus four times that index. This unsigned word shift retains only the original value’s low nine bits. F-U030 subsequently receives four times the same index.

### 4. Early masked state and eight-times base

F-U031–F-U034 execute; F-U034 supplies an unsigned word. Retain its four-times-scaled bit pattern as the **early scaled word**, and store that pattern at `0x14` plus sixteen times the original returned word.

Mask the returned word to 30 bits, producing the **early masked word**. Also retain its logical right shift by two.

F-U035 receives the early masked word. F-U036–F-U039 follow; F-U039 supplies a signed **retained index**. Store the early masked word at `0x14` plus four times this index. F-U040 receives four times the index.

F-U041–F-U044 follow; F-U044 supplies an unsigned **later word**. Form the **eight-times base** from eight times the retained index. Write its low byte at initial snapshot plus `0x01`, then write the later word shifted left by two at eight-times base plus `0x34`.

F-U045 receives that shifted word.

### 5. Forwarding and paired-result overwrite

F-U046 supplies a word. Mask the later word to 30 bits, retaining the result as the **retained base**. F-U047 receives the F-U046 word unchanged.

F-U048–F-U051 follow. F-U051 has no explicit arguments shown and supplies a represented 64-bit result. Its represented target is `0x0882ef05 + 0xffe`, equal to `0x0882ff03`.

Write the low byte of the eight-times base at retained base plus `0x01`. Then overwrite the word at eight-times base plus `0x34` with F-U051’s high half.

F-U052 receives F-U051’s low half interpreted as signed and shifted left by two.

Both writes to eight-times base plus `0x34` are observable effects in the contract. Calls separate them; the later overwrite does not make the earlier store dispensable.

### 6. Repeated byte writes

F-U053–F-U056 execute; F-U056 supplies a signed result. Form the **sixteen-times base** from sixteen times the retained index.

Write its low byte at retained base logically shifted right by 20, plus `0x11`. Next, write the early scaled word’s low byte at sixteen-times base plus `0x0d`.

F-U057 receives two copies of the F-U056 result shifted left by two. F-U058 supplies an unsigned word; F-U059 receives that word and its logical right shift by three.

F-U060–F-U063 follow; F-U063 supplies a signed result. Write the sixteen-times base’s low byte at absolute address `0x11`, then write the early scaled word’s low byte again at sixteen-times base plus `0x0d`.

F-U064 receives the F-U063 result shifted left by two. The repeated byte store remains distinct despite using an unchanged retained value.

### 7. Retained-base writes and new-base derivation

F-U065–F-U068 execute; F-U068 supplies an unsigned word. Before using it to derive another base, store the early scaled word at retained base plus `0x14`, then store the low byte of retained base shifted left by one at retained base plus `0x0d`.

From F-U068’s word, retain:

- A **new base** with the bit pattern of the word multiplied by 32.
- The word shifted left by seven.
- A **reduced selector** obtained by masking to 27 bits and logically shifting right by 17.

The reduced selector lies between zero and `0x3ff`.

F-U069–F-U071 follow; F-U071 supplies a signed result. Store the retained shifted-by-seven word at new base plus `0x14`, then store the low byte of the F-U071 result shifted left by two at new base plus `0x0d`.

F-U072 receives that signed result shifted left by two. F-U073–F-U076 follow without replacing the retained F-U071 scalar.

### 8. Byte-mediated selection and explicit forwarding

Write the low byte of twice the retained early-masked-word logical-right-shift-by-two value at reduced selector plus `0x07`.

Shift the retained F-U071 result left by 21. Read a byte at new base plus `0x01`, then write the reduced selector’s low byte at the loaded selector plus `0x11`.

F-U077 receives three copies of the shifted-by-21 result.

F-U078 supplies an unsigned **forwarded word**. F-U079 receives it unchanged. F-U080–F-U083 follow; F-U083 supplies a signed result.

Read new base plus `0x01` again. Retain this fresh value as the **selected byte**, and write zero at selected byte plus `0x11`. F-U084 then receives the F-U083 result shifted left by two.

The second selector read is not replaceable with the first: intervening dependencies may change the selected destination.

### 9. Absolute byte state and shared-target call

F-U085–F-U088 execute. Form the **small scaled base** as twice the selected byte, giving a range of zero through 510.

Retain the low byte of the forwarded word logically shifted right by three as the **saved byte**. Write it at new base plus `0x0d`, then write the selected byte at absolute address `0x0d`.

F-U089 receives the small scaled base, `0x98`, and `0x260`, in that order. F-U090 follows. F-U091 receives an auxiliary unsigned register output logically shifted right by three.

F-U092–F-U095 execute; F-U095 supplies a signed result. F-U096 receives that result shifted left by two and supplies another signed 32-bit result.

F-U096’s represented target is `0x0882f0fb + 0xe08`, also equal to `0x0882ff03`. Its target address therefore equals F-U051’s, despite the different represented arguments and result use.

After F-U096, write the saved byte at small scaled base plus `0x0d`, then write `0xc0` at absolute address `0x0d`.

F-U097 receives the F-U096 result shifted left by two. F-U098–F-U101 follow; F-U101 supplies a signed result.

### 10. Linked accesses and separate paired-result producers

Store the F-U101 result shifted left by two at small scaled base plus `0x24`. Then read a word at small scaled base plus `0x74`, retaining it as the **linked base**.

Retain the F-U101 result shifted left by five. F-U102 receives:

- The linked base.
- The linked base shifted left by one.
- The retained signed shifted-by-five value.
- The linked base masked with `0x7fffffff` and logically shifted right by two.

After F-U103–F-U106, store the retained shifted-by-five value at linked base plus `0x24`. Read a word at linked base plus `0x74` and supply that loaded word directly to F-U107.

F-U108–F-U111 follow; F-U111 supplies a represented 64-bit result. Retain its low half, interpreted as signed and shifted left by two, as the **first shifted half**. Use its unsigned high half as an index: store the full forwarded word logically shifted right by three at `0x14` plus four times that high half.

F-U112 receives three copies of the first shifted half, followed by the high half logically shifted right by three. No result is captured from F-U112.

The immediately following F-U113 has no explicit arguments shown and supplies a separate represented 64-bit result. F-U114 receives that result’s low half interpreted as signed, high half as unsigned, repeated signed low half, and high half logically shifted right by three.

F-U115–F-U118 follow; F-U118 supplies a signed index. Store the first shifted half at `0x14` plus four times this index. F-U119 receives four times the index.

F-U120–F-U123 follow; F-U123 supplies an unsigned word.

### 11. Final selector reads and clears

Retain the bit pattern of F-U123’s word shifted left by two as the **final shifted word**. It is carried as signed in later represented expressions.

Read a selector byte at linked base plus `0x01`, then write the final shifted word’s low byte at selector plus `0x11`.

F-U124–F-U126 execute; F-U126 supplies a signed result. Read the selector at linked base plus `0x01` afresh. At the newly selected destination plus `0x11`, write the low byte of F-U123’s word masked to 30 bits and logically shifted right by 20.

F-U127 receives the F-U126 result shifted left by two. F-U128–F-U131 follow; F-U131 supplies a signed result.

Read a selector at final shifted word multiplied by `0x400`, plus `0x01`. Write zero at selector plus `0x11`. The multiplication uses the represented signed carrier; overflowing arithmetic is not resolved here.

F-U132 receives two copies of final shifted word shifted left by three, the final shifted word itself, and the F-U131 result shifted left by two.

F-U133–F-U136 follow; F-U136 supplies a signed result. Read the selector again from the same calculated selector address and clear the newly selected destination plus `0x11`.

F-U137 receives the F-U136 result shifted left by two. F-U138–F-U141 follow. The representation then reaches the undecodable boundary.

## Side effects

The visible sequence contains byte and word reads, byte and word writes, and 141 indirect-call occurrences. No validation, rollback, synchronization, or recovery branch is visible.

Repeated reads and writes remain separate effects. Dependencies may observe stores, modify their destinations, or change selector bytes before later reads. Retained scalar snapshots continue to supply later expressions without implying that their original memory locations remain unchanged.

Different address expressions may alias. When the selected byte is zero, the Stage 9 store at small scaled base plus `0x0d` targets absolute address `0x0d`; the subsequent `0xc0` store then overwrites that same byte.

No visible operation establishes allocation, release, reference counting, ownership transfer, or borrowing rules. Dependency ownership effects remain unknown.

No null, bounds, alignment, address-validity, or call-target checks are visible. Low and zero-derived addresses are not specially rejected. Fault delivery is unspecified, and earlier effects need not be undone.

The undecodable ending does not establish an intentional halt, a software exception contract, or guaranteed nonreturning behavior.

## Dependencies

F-U001–F-U141 are opaque occurrence IDs, not independently recovered function identities. Different IDs preserve separate positions in the observable sequence.

One target-address equivalence is statically established:

| Occurrence | Represented target calculation | Represented interface use |
|---|---|---|
| F-U051 | `0x0882ef05 + 0xffe = 0x0882ff03` | No explicit arguments shown; captured 64-bit result. |
| F-U096 | `0x0882f0fb + 0xe08 = 0x0882ff03` | One shifted signed argument; captured signed 32-bit result. |

This equality is an address fact, not proof of a common valid ABI, function implementation, memory-preservation rule, or ownership behavior. The occurrences must not be merged, and their result representations must not be normalized into an invented common signature. Other dependency identity relationships are not established here.

The Behavior section specifies captured results, auxiliary outputs, argument order, and result-half selection. F-U112 and F-U113 remain distinct: the former consumes explicit arguments without a captured result; the latter produces the paired result forwarded to F-U114.

No dependency is assumed to preserve globals, referenced memory, environmental state, or ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume the described execution prefix is reached and relevant accesses and arithmetic are valid.

1. **Initial selection.** A first selector of `0xff` and incoming byte of `0xa6` require an initial store of `0xa6` at `0x110`.

2. **Snapshot truncation.** An initial snapshot of `0x123456ab` produces a byte store of `0xab`; later snapshot arguments still use the full word.

3. **Unsigned packing.** A first loaded word of `0xffc00000` yields ten-bit value `0x3ff`. F-U025’s duplicated packed arguments are `0x3fe00000`. The later shifted store supplies `0xff800000`; with destination index three it targets `0x20`, after which F-U030 receives 12.

4. **Duplicate signed arguments.** If F-U056 returns three, F-U057 receives 12 twice.

5. **Shared target, distinct uses.** Both F-U051 and F-U096 resolve to represented target `0x0882ff03`. If F-U051’s high half is `0x11223344` and low half is three, the overwrite supplies `0x11223344` and F-U052 receives 12. Independently, if F-U095 returns five, F-U096 receives 20; if its captured signed result is seven, F-U097 receives 28.

6. **Fresh selector.** If the Stage 8 selector reads yield `0x20` and then `0x30`, the later clear targets `0x41`, not `0x31`. An F-U083 result of five makes the following F-U084 argument 20.

7. **Independent paired results.** If F-U111 supplies low half three and high half eight, F-U112 receives 12, 12, 12, and one. If F-U113 separately supplies low half seven and high half 16, F-U114 receives seven, 16, seven, and two.

8. **Byte versus full-word retention.** Forwarded word `0x12345678` yields saved byte `0xcf`. The later indexed word store instead supplies `0x02468acf`.

9. **Aliased absolute write.** With selected byte zero, Stage 9 writes the saved byte to `0x0d` and subsequently writes `0xc0` there. Both stores remain required.

10. **Completion boundary.** Reaching F-U141 does not establish successful return or any particular continuation.

## Uncertainties

The following remain **documented_only**:

- Accuracy and completeness of the represented execution sequence.
- Actual calling convention and provenance of incoming and auxiliary state.
- Dependency implementations, valid signatures, preservation guarantees, and ownership effects.
- Whether equal represented target addresses identify a valid common executable entry in the actual environment.
- Signed-overflow, conversion, signed-shift, and overflowing-address semantics.
- Endianness, alignment, object boundaries, address mappings, and lifetime requirements.
- Ordinary-memory versus device-memory behavior.
- Concurrency, reentrancy, fault delivery, and partial-effect visibility.
- Execution after the undecodable boundary.

Compact-reference encoding or linked-record maintenance would be inferred intent only. Neither interpretation adds an acceptance requirement.

## Evidence

Established static evidence comprises the supplied representation’s access widths, offsets, masks, shifts, explicit call arguments, captured results, auxiliary outputs, result-half selection, and ordering.

The two target calculations independently establish `0x0882ff03` for F-U051 and F-U096. Their differing represented arguments and result widths remain evidence rather than grounds for inventing a unified ABI.

The representation also distinguishes the uncaptured F-U112 occurrence from the immediately following paired-result-producing F-U113 occurrence.

Invalid-instruction warnings and the truncated ending limit machine-level conclusions. No runtime execution, smoke checks, or tests have been performed.
