<!-- Trusted provenance: F-b85cafe16d89; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b85cafe16d89

## Purpose

F-b85cafe16d89 performs a fixed, dependency-sensitive sequence of memory reads and writes, calls an entry-supplied callable with access to the fifth argument’s storage, invokes F-cdec09f5d432, and reaches a terminal mechanism represented as nonreturning.

No application-level purpose is established. Repeated offsets do not establish an object layout, protocol, allocation operation, or ownership transfer.

The contract distinguishes **established static evidence**, **inferred intent**, and **documented_only** unknowns. Effects are conditional on execution reaching them. No ordinary successful return, validation procedure, or rollback is established.

## Interface

### Explicit inputs

Seven positional arguments are represented:

| Argument | Represented width | Role |
|---|---:|---|
| First | 32 bits | Base for word reads at offsets 20, 100, and 116; base for a byte write at offset 1; operand of left shifts by 4 and 8. |
| Second | 32 bits | Combined with an initially loaded signed halfword to locate a byte. |
| Third | 32 bits | No explicit use represented. |
| Fourth | 8 bits | Value of the first memory write. |
| Fifth | 32 bits | Its storage address is passed to the entry-supplied callable. |
| Sixth | 32 bits | No explicit use represented. |
| Seventh | 32 bits | Its invocation-local storage is cleared after that callable returns. |

No return value is represented.

### Additional entry state

Execution also depends on:

- An **auxiliary base**, used initially for an offset-29 byte destination and an offset-68 word source.
- An **entry destination base**, used initially for offset-17 byte writes and offset-84 self-value stores.
- An incoming byte.
- An incoming callable target.

Their ABI origin is unresolved. Seven distinct global word sources supply successive mutation stages. A separate global datum supplies the first argument to F-cdec09f5d432.

These descriptions identify value roles, not object types.

### Widths and access requirements

Offsets are in bytes. Byte, halfword, and word accesses have represented widths of 8, 16, and 32 bits. Halfwords used as address operands are signed; for example, the pattern 0xFFFF supplies −1 when widened as represented.

Byte writes publish byte-width conversions. References below to a word’s “byte” mean its low-byte payload. Conversion of a loaded plain byte to a halfword or address operand depends on plain-byte signedness. Word-to-halfword stores use the represented halfword conversion.

Uninterrupted execution requires each reached address and callable target to support its represented operation. No null, bounds, alignment, allocation-membership, or callable-validity checks are represented.

Actual pointer width, endianness, signed overflow, out-of-range narrowing details, negative shifts, and address wrapping are **documented_only**. In particular, 32-bit operand representation does not establish modulo-\(2^{32}\) address arithmetic.

## Behavior

### Ordering and retained values

The represented body has one forward path without an explicit input-dependent branch.

Separate accesses remain separate. A write can alter a later read through full or partial overlap. A captured value remains the result of its particular read even if its source is later overwritten. Repeated reads are not interchangeable with a cached read, and repeated stores are not omitted merely because their addresses and payloads match.

This is the supplied static access order, not a claim about atomicity, volatile qualification, synchronization, or hardware ordering. Expressions containing multiple accesses do not independently establish every machine-level evaluation detail.

Descriptions such as “stage-five loaded base” name retained read results for this contract; they do not identify public fields or storage objects.

### Initial preparation

The fourth argument’s byte is written at offset 17 from the entry destination base.

A signed halfword is read at the sum of the entry destination and auxiliary bases. Its signed value plus the second argument locates an initial byte read. After capturing that byte, the incoming byte is written at auxiliary offset 29. The captured byte is converted to a halfword and stored at the auxiliary base plus the signed-halfword value.

The auxiliary offset-68 word is then captured. The entry destination base is stored as a word at its own offset 84, before either stage-one global evaluation.

### Stage transitions

Every stage transition has this represented order:

1. Read its global source to capture the stage’s **active base**.
2. Read that global source again and store this second word at offset 68 through the transition destination.
3. Store the incoming payload word at offset 20 from the captured active base.

The second global read supplies the offset-68 payload; it is not replaced by the first read. An overlapping offset-68 store does not retroactively change the captured active base.

| Stage | Transition destination base | Incoming payload source |
|---|---|---|
| One | Initial loaded byte, interpreted as an address operand | Auxiliary offset-68 snapshot |
| Two | Stage-one loaded byte | Stage-one active offset-68 snapshot |
| Three | Stage-two loaded byte | Stage-two active offset-68 snapshot |
| Four | Stage-three word loaded from entry destination offset 68 | Stage-three redirected offset-68 snapshot |
| Five | Stage-four loaded base | Stage-four redirected offset-68 snapshot |
| Six | Stage-five loaded base | Stage-five active offset-68 snapshot |
| Seven | Stage-six loaded base | Stage-six active offset-68 snapshot |

The pre-transition stores specified below occur **before both global evaluations**, not within or after the transition.

### Stage one

After its transition, the entry destination’s offset-17 byte is overwritten with the low byte of that base shifted left by one.

A signed halfword is read at the entry destination base plus the active base. Adding its signed value to the incoming auxiliary offset-68 snapshot locates the stage-one byte read.

The first argument’s offset-100 word is read and stored at offset 84 from the signed-halfword-derived base. The incoming auxiliary snapshot’s byte is then stored at offset 13 from the newly loaded byte-derived base.

The active base’s offset-100 word is captured as the **retained word**, used much later in additive-address stores and further reads. This is distinct from every first-argument offset-100 snapshot.

The stage-one byte is converted to a halfword and stored at the signed-halfword value plus the active base. The active offset-68 word is then captured.

Finally, the entry destination base is stored at its own offset 84. This precedes both stage-two global evaluations.

### Stage two

After its transition and offset-20 store, only the shifted-byte part of the earlier pattern occurs: the entry destination’s offset-17 byte receives the low byte of that base shifted left by one. The self-value store has already occurred before the transition.

The second first-argument offset-100 snapshot is captured. A signed halfword from the entry destination base plus the active base, combined with the incoming payload, locates another byte read.

The subsequent stores occur in this order:

- Active-base byte to first-argument offset 1.
- Incoming-payload byte to active offset 13.
- Retained word to active offset 68.
- Active-base byte to offset 13 from the newly loaded byte-derived base.
- Converted loaded byte as a halfword at the signed-halfword value plus the active base.

The active offset-68 word is then read afresh. Intervening overlap can make it differ from the retained word previously stored there.

The entry destination base is stored at its own offset 84 before both stage-three global evaluations.

### Stage three

After its transition, the second first-argument offset-100 snapshot supplies the byte written at entry destination offset 17. The third first-argument offset-100 snapshot is then captured.

In order, the routine captures:

- A word from entry destination offset 68: the stage-three loaded base.
- A signed halfword at the entry destination base plus the active base.
- A word from incoming-payload offset 116: the redirected base.
- A word from entry destination offset 20: the moving base used by stages four and five.

The redirected base’s byte is stored at stage-three loaded-base offset 13. A halfword conversion of the loaded base is stored at the signed-halfword value plus the redirected base. The redirected offset-68 word becomes the next incoming payload.

The moving base is stored at its own offset 84 before both stage-four global evaluations.

### Stage four

After its transition, the third first-argument offset-100 snapshot supplies a byte at moving-base offset 17.

A word is captured from moving-base offset 68 as the stage-four loaded base. A signed halfword is then read at the moving base plus the active base.

The active-base byte is written at offset 21 from the **incoming payload’s base**, not from the active base. The incoming payload is stored at active offset 68.

A word from stage-four loaded-base offset 116 supplies a redirected base. A halfword conversion of the loaded base is stored at the signed-halfword value plus that redirected base. Its offset-68 word becomes the next incoming payload.

The moving base is again stored at its own offset 84 before both stage-five global evaluations. This store is distinct from the later self-value store through the stage-five loaded base.

### Stage five

After its transition’s offset-20 store, the routine writes the low byte of the moving base shifted left by one at moving-base offset 17.

Only then does it capture the fourth first-argument offset-100 snapshot, followed by the moving offset-68 word as the stage-five loaded base, followed by a signed halfword at the moving base plus the active base. The shifted-byte write can therefore affect any of these reads through overlap.

A word at offset 20 from the signed-halfword-derived base supplies the next moving base.

The remaining operations have this order:

- Store the stage-five loaded base at its own offset 84.
- Store the retained word at the additive destination formed from the active base and incoming payload.
- Store the loaded base at active offset 20.
- Read a fresh word from retained-word offset 68 and write its byte at active offset 13.
- Write the loaded-base byte at next-moving-base offset 13.
- Store a halfword conversion of the loaded base at the signed-halfword value plus the active base.
- Capture the active offset-68 word for the next transition.

The next moving base is stored at its own offset 84 before both stage-six global evaluations.

### Stage six

After its transition’s offset-20 store, the fourth first-argument offset-100 snapshot supplies a byte at the current moving base’s offset 17.

The fifth first-argument offset-100 snapshot is then captured, followed by the current moving offset-68 word as the stage-six loaded base, followed by a signed halfword at the moving base plus the active base.

Before reading offset 20 through that signed-halfword-derived base, three stores intervene, in this order:

1. Store the current moving base at offset 84 from the incoming payload’s base.
2. Write the fifth snapshot’s byte at current-moving-base offset 17.
3. Store the captured active base at stage-six loaded-base offset 68.

The fifth snapshot’s byte is also retained for stage seven. These stores can alter the subsequent offset-20 read through overlap; that read must not be moved ahead of them.

The offset-20 read supplies the next moving base. Subsequently, the routine:

- Stores the stage-six loaded base at its own offset 84.
- Stores the retained word at the active-base-plus-incoming-payload destination.
- Stores the loaded base at active offset 20.
- Reads retained-word offset 68 afresh and writes its byte at active offset 13.
- Writes the loaded-base byte at next-moving-base offset 13.
- Stores a halfword conversion of the loaded base at the signed-halfword value plus the active base.
- Captures active offset 68 for the next transition.

The next moving base is stored at its own offset 84 before both stage-seven global evaluations.

### Stage seven and product-derived tail

After its transition, the retained fifth-snapshot byte is written at moving-base offset 17. A word is captured from moving offset 68, then a signed halfword is read at the moving base plus the stage-seven active base. Its signed value is the **narrow-derived base**.

Five word stores write the narrow-derived base at its own offset 84.

The first two each precede an identical companion byte store: the moving-base byte is written at active offset 17. Both complete pairs are represented separately.

A product of the narrow-derived base and the captured moving offset-68 word begins dependent word reads through offsets 68, 116, and 20. From the resulting base, another offset-20 word is captured before that location is overwritten with the moving base shifted left by one.

The captured pre-overwrite word remains an address operand. Its offset-20 word is multiplied by the narrow-derived base to obtain a new product. The third self-value store follows, then the captured pre-overwrite word’s byte is written at active offset 17.

The new product’s offset-116 word supplies the **tail value**. Multiplying that product by the narrow-derived base again produces the **final product-derived base**. This is a separate multiplication.

The fourth self-value store occurs, followed by the tail-value byte at active offset 17. Thereafter:

1. Capture final-product-derived offset 68.
2. Capture first-argument offset 116.
3. Store the tail value as a word at active offset 100.
4. Write the first captured final-product-derived byte at active offset 9.
5. Read final-product-derived offset 68 afresh and write its byte at that base’s offset 17.
6. Perform the fifth narrow-derived self-value store.
7. Write the tail-value byte at offset 17 from twice the final product-derived base.
8. Read a word at offset 68 from the product of the narrow-derived and final product-derived bases.
9. Read first-argument offset 20 to capture the last destination base.

Three byte writes then occur: the saved first-argument offset-116 byte at tail-value-plus-1; the additional product-derived byte at last-destination offset 17; and the byte of twice the final product-derived base at last-destination offset 9.

Both final source reads precede all three final byte writes.

### Local preparation and calls

Before the entry-supplied call, invocation-local storage contains two copies of the first argument shifted left by 4, four copies shifted left by 8, three copies of the tail value including its earlier captured location, and four zero-valued words.

Preparation writes one shift-by-4 result and one shift-by-8 result, then the four zeros, then the remaining copies. No public aggregate layout is established.

The entry-supplied callable receives the fifth argument’s storage address followed by two zeros. No returned value is used.

If it returns, local writes clear the seventh argument’s storage, write 0x08400E1C to another stack-like word location, and clear three further stack-like words, in that order.

F-cdec09f5d432 then receives the separate global datum followed by three zeros. The datum is evaluated after the entry-supplied call.

If F-cdec09f5d432 returns, a terminal mechanism receives zero and 0x08401154 and supplies an associated indirect invocation. The path is marked nonreturning; the constants’ platform meanings remain unresolved.

## Side effects

External memory receives byte, halfword, and word writes. Argument storage and other invocation-local locations are also modified. No transaction or rollback is represented; interruption can expose a completed prefix.

No allocation, release, reference-count change, or ownership transfer is established. Passing the fifth argument’s address does not establish a read-only restriction, a four-byte write limit, non-retention, or validity after the invocation ends.

Neither callable has an established preservation contract. No status result, explicit error branch, or recovery path is represented. Invalid accesses, arithmetic behavior, callable effects, or terminal execution may prevent later effects.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| Entry-supplied callable; opaque ID unavailable | Fifth-argument storage address and two zeros | ABI, writes, retention, ownership, preservation, return behavior |
| F-cdec09f5d432 | Later-evaluated global datum and three zeros | Datum interpretation, effects, ownership, preservation, return behavior |
| Terminal mechanism and associated callable; opaque IDs unavailable | Reached after both calls return; marked nonreturning | Platform action, target production, exception behavior |

The seven mutation globals and separate call datum are data dependencies. Their storage properties and initial values are unspecified.

## Acceptance examples

These are conditional criteria, not runtime observations.

1. **Stage-two pre-transition overlap.** If the entry-destination self-value store overlaps stage two’s global source, both global evaluations occur after that store. Moving it after the transition changes the contract.

2. **Stage-five shifted-byte overlap.** If moving offset 17 overlaps the first argument’s offset-100 word, the fourth snapshot is read after the shifted-byte write. The moving offset-68 and signed-halfword reads also follow it.

3. **Stage-five self-value overlap.** If moving offset 84 overlaps the stage-five global source, the self-value store precedes both evaluations, independently of the later loaded-base self-value store.

4. **Stage-six delayed read.** If any of the three intervening stores overlaps the signed-derived offset-20 source, the next moving-base read uses the resulting contents, subject to actual memory semantics.

5. **Stage-six snapshot replacement.** The moving offset-17 write using the fourth snapshot precedes the fifth snapshot read. Its later overwrite uses the fifth snapshot’s byte.

6. **Distinct global evaluations.** If a stage’s global yields different words on its two reads, the first fixes the active base and the second supplies the transition’s offset-68 payload.

7. **Signed and byte interpretation.** A signed halfword pattern 0xFFFF supplies −1. A loaded byte 0xFF does not independently establish whether its halfword conversion is 0xFFFF or 0x00FF.

8. **Tail snapshots.** An overlapping stage-seven offset-100 store cannot change the already captured first-argument offset-116 value. Conversely, the later final-product-derived offset-68 byte uses a fresh read.

9. **Repeated stores.** Where individual accesses are observable, all five narrow-derived self-value stores and both identical initial companion pairs remain present.

10. **Call mutation and nonreturn.** If the entry-supplied callable changes the separate global datum and returns, the subsequent call receives its later value. If it does not return, post-call local writes and later calls are not reached.

## Uncertainties

- **documented_only — Completeness:** Supplied annotations report omitted unreachable regions; they do not independently establish complete entry-state reconstruction or boundaries.
- **documented_only — Arithmetic and memory:** Overflow, narrowing, address width, endianness, alignment, allocation extent, concurrency, and special storage semantics lack a target contract.
- **documented_only — Local layout:** Exact ABI placement, caller-visible aliases, and significance of fixed local values are unknown.
- **documented_only — Dependencies and termination:** Bodies and platform documentation do not establish ownership, preservation, exceptions, or actual terminal behavior.
- **Inferred intent:** Repeated offsets may suggest structured state mutation, but no object model is required.
- **documented_only — Reachability:** No observed execution establishes uninterrupted completion of the represented path.

## Evidence

The supplied static body supports the represented widths, seven two-read global transitions, five first-argument offset-100 reads, distinct retained word, pre-transition self-value stores, stage-five shifted-byte placement, stage-six intervening stores, tail snapshots, local preparation, call arguments, and marked nonreturning path.

Prior prose, comments, and feedback are not independent evidence of runtime behavior. No dependency implementations, ABI specification, memory map, initialization fixture, or execution trace were supplied.

No runtime tests have been performed.
