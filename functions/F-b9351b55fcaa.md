<!-- Trusted provenance: F-b9351b55fcaa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9351b55fcaa

## Purpose

F-b9351b55fcaa performs an ordered sequence of indirect calls, byte-selector reads, byte stores, and 32-bit word accesses, followed by an unconditional repeating sequence.

No application-level purpose is established. The available evidence does not identify these operations as serialization, initialization, decoding, allocation, or cleanup. Small absolute addresses do not establish whether the accesses concern ordinary memory, specialized hardware, or an incorrectly reconstructed execution environment.

This contract describes effects conditional on execution reaching them. The supplied static representation reports invalid instruction data and discarded control-flow regions; it does not establish that the operation executes successfully as represented.

There is no displayed normal return.

## Interface

### Explicit arguments

| Argument | Width and interpretation | Displayed use |
|---|---|---|
| First | Unsigned 32-bit quantity | Supplies the first two byte-store values and the first indirect call’s explicit argument. |
| Second | 32-bit quantity; interpretation unspecified | No direct use is displayed. |
| Third | Unsigned 32-bit quantity | Selects the destination of two early word stores. |

The absence of a direct use of the second argument does not establish its invisibility to dependencies. Complete call signatures and the incoming calling convention are unresolved.

### Implicit inputs

Execution also depends on:

- An incoming register-derived base for the first two selector reads.
- An incoming return-address-like value involved in selecting the first indirect target.
- Memory at absolute address `0x3C`.
- Values returned by indirect dependencies.
- A secondary register result used in a later address calculation.
- The accessibility and behavior of all computed memory locations.

There is no demonstrated buffer-length, object-layout, or ownership contract.

### Results and completion

No return value is declared in the supplied interface. The displayed final control flow repeats without an exit condition.

A dependency may fault, fail to return, or transfer control elsewhere. Therefore, neither arrival at the repeating sequence nor uninterrupted repetition is guaranteed.

### Widths and address terminology

A **byte** is eight bits; a **word** is 32 bits. Displayed unsigned right shifts are logical. Byte stores retain the low eight bits of the supplied value.

The following address descriptions do not imply valid arrays or allocated objects:

- A **word-scaled address** for a quantity is `0x14` plus four times that quantity.
- A **record-scaled address** for a quantity is `0x14` plus sixteen times that quantity.
- A **byte-selected destination** is `0x11` plus an unsigned selector byte read from a specified address. Its range is `0x11` through `0x110`, inclusive.

Scaling applies to the named quantity exactly once. In particular, if a retained quantity already equals four times a dependency result, its value plus `0x14` is the word-scaled address of the **original result**, not the word-scaled address of the retained quantity.

Several represented calculations use signed integers, including products and left shifts of dependency results. Their overflow behavior, negative-operand shifts, and conversion into addresses are **documented_only**. Where examples use a 32-bit machine interpretation, arithmetic retains the low 32 bits; this is conditional, not an established language-level guarantee.

## Behavior

### Dependency-sequence terminology

To express ordering without assigning unsupported identities to callees:

- A **five-call group** consists of one call with the stated explicit arguments followed by four calls with no displayed explicit arguments. The fifth call supplies the result when a subsequent operation uses one.
- A **forwarding group** contains nine calls. The first receives the stated argument list. The second produces a word passed explicitly to the third; the fourth produces another word passed explicitly to the fifth. Four further calls follow, with the ninth supplying the next result. Where specified, the fifth call additionally receives a retained snapshot-derived argument.

These descriptions establish displayed argument flow and call order only. Calls without displayed arguments may still observe registers or other execution state.

### Initial byte stores and early snapshot

The first observable memory operation reads an unsigned byte at the incoming base plus one. The operation then stores the low byte of the first argument at the corresponding byte-selected destination.

A five-call group follows. Its first call receives the full first argument and has a target dependent on incoming execution state.

The selector at the incoming base plus one is then read again. The low byte of the first argument shifted right by 22 is stored at the newly selected destination. The two destinations need not match: intervening calls may change the selector.

Another five-call group begins with four times the preceding group’s result.

Next, the operation reads the value associated with absolute address `0x3C` and converts it to an unsigned 32-bit snapshot. The underlying read width is not established by the supplied declaration.

The destination for this snapshot is `0x14` plus four times the third argument shifted right by seven. A 32-bit store writes the snapshot there. A five-call group begins with the snapshot as its explicit argument.

The same retained snapshot is then written to the same retained destination. There is no second displayed read of `0x3C`. Another five-call group begins with four times the preceding result.

### First two word snapshots

The latest dependency result determines an initial record-scaled address and word-scaled address. The operation stores four times that result at the record-scaled address, then reads a word from the word-scaled address. The read becomes the **first word snapshot**.

The store precedes the read; possible address aliasing must not be excluded.

A forwarding group begins with four explicit arguments:

- Sixteen times the selector result.
- Four times that result.
- That result shifted left by 26.
- Four times that result again.

The shifted-left quantity is retained as a 32-bit unsigned value. Following the forwarding group, the same four-times quantity is written again at the same record-scaled address.

A five-call group begins with four times the forwarding group’s final result. Its result selects a new record-scaled address and word-scaled address. The operation stores four times this new result at its record-scaled address, then reads a word at its word-scaled address. This read becomes the **second word snapshot**.

The retained shifted-left quantity is shifted logically right by seven. A forwarding group begins with the new four-times quantity supplied twice. After that group, the second snapshot shifted right by eight is stored as a full word at its retained word-scaled source address.

The snapshots are captured values, not live views of memory. Subsequent writes or dependency effects do not refresh them.

### Intermediate stores and first-snapshot arguments

A five-call group begins with four times the preceding result. Its result selects another record-scaled store of four times that result.

A further five-call group begins with the first snapshot shifted right by 16. After it completes, the preceding retained four-times quantity is stored at **absolute address `0x14` plus that quantity**. This is the word-scaled address of the original selector result. The retained quantity is not multiplied by four again, and this destination does not use the latest group’s result.

Another five-call group begins with four times the latest result. Its result selects a record-scaled store of four times itself.

A forwarding group follows. Its first call receives the corresponding sixteen-times quantity; its fifth call receives its forwarded word and the first snapshot shifted right by 16. The record-scaled store is then repeated with its original value and destination.

The next five-call group begins with four times the forwarding group’s result. Its result selects another record-scaled store of four times itself. At this stage, the retained shifted-left quantity undergoes its second logical right shift by seven.

A forwarding group begins with:

- The latest four-times quantity.
- Eight times the first snapshot shifted right by eight.

Its fifth call receives a forwarded word and the first snapshot shifted right by 16. Afterward, the latest four-times quantity is stored at `0x14` plus the eight-times quantity.

A further five-call group begins with four times the latest result. Its result selects another record-scaled store of four times itself.

The next five-call group begins with four arguments:

- Eight times the first snapshot shifted right by eight.
- That same quantity again.
- The first snapshot shifted right by 16.
- The retained shifted-left quantity shifted right by another seven.

The last argument therefore reflects three successive right shifts by seven after the original 32-bit left shift by 26. It does not preserve discarded high bits.

After this group, the eight-times quantity is stored at its own value plus `0x14`. Right-shifting a word by eight does not isolate one byte.

### Secondary result and subsequent word stores

A five-call group begins with four times the latest result. A secondary register result associated with its final call supplies the next calculation. Its true result convention is unresolved.

Under the conditional 32-bit machine interpretation, the operation stores the secondary result multiplied by `0x01000000` at `0x14` plus that result multiplied by `0x04000000`.

A five-call group begins with the retained `0x01000000`-scaled value. The same value is then stored at the word-scaled address of this group’s result.

Another five-call group begins with four times that result. Its result selects a record-scaled store of four times itself. A forwarding group begins with the retained four-times value. Afterward, that value is written at the word-scaled address selected by the forwarding group’s final result.

The next five-call group begins with four times that final result. Its result supplies a new retained four-times quantity, which is stored at the corresponding record-scaled address.

A forwarding group begins with this retained quantity. The first snapshot shifted right by 24 is then written at the word-scaled address of the group’s final result. This snapshot-derived value is also retained for later byte addressing.

Another five-call group begins with four times that result. Its result selects a record-scaled store of four times itself.

The next five-call group begins with the corresponding sixteen-times quantity twice, followed by the first snapshot shifted right by 23. Afterward, that sixteen-times quantity is stored at its own value plus `0x14`.

### Late byte stores

The retained four-times quantity from before the most recent forwarding group is masked with `0x7FFFFFFF` and shifted right by seven. This supplies the **late selector base**.

A five-call group begins with four times the latest result. Its final result becomes the **first late byte source**.

The operation reads a selector byte at the late selector base plus one and stores the low byte of four times the first late byte source at the selected destination.

Seven calls follow. The first receives the four-times value three times. The second returns a word explicitly passed to the third; four more calls follow, with the seventh supplying a result.

The selector at the late selector base plus one is read again. The stored byte is the low byte of the first late byte source masked with `0x3FFFFFFF` and shifted right by 20.

A five-call group begins with four times the preceding result. Its result becomes the **second late byte source**. Four times this source is retained.

The next effects occur in this order:

1. Store the low byte of that four-times quantity at `0x1C` plus the first snapshot shifted right by 31.
2. Read the selector at the late selector base plus one and store zero at the selected destination.
3. Make seven calls: the first receives the first snapshot shifted right by 31; the third receives the second late byte source masked with `0x3FFFFFFF` and shifted right by six. The seventh supplies a result.
4. Reread the same selector location and store zero at its newly selected destination.

The direct destination in the first step is either `0x1C` or `0x1D`. That direct write may itself affect a following selector read if addresses alias.

### Loop entry and repetition

A five-call group begins with four times the latest result. Its final result is masked with `0x3FFFFFFF` and shifted right by 25, producing the initial **loop field**, ranging from zero through 31.

The operation reads a selector byte at absolute address one plus 1024 times the retained four-times second late byte source, then stores zero at the selected destination. This selector address remains fixed throughout the displayed loop, although its contents may change.

A call receives twice the initial loop field. The next call supplies the initial **carried result**.

Each iteration then:

- Makes a five-call group whose first call receives the carried result and the loop field shifted right by eight.
- Rereads the fixed selector address and stores zero at the selected destination.
- Makes another five-call group beginning with four times the preceding group’s result.
- Replaces the carried result with the final result shifted left by two.
- Replaces the loop field with that same final result shifted left by 26.

No displayed condition terminates repetition. The loop-entry first call and subsequent iteration-entry calls must not be assumed to share a target merely because they have the same argument shape.

The first iteration’s second explicit argument is zero. Under the conditional 32-bit interpretation, later loop fields retain only six bits of their originating result; their right-shift-by-eight values are multiples of `0x40000`.

## Side effects

All displayed stores directly affect addressed memory. No staging, rollback, transaction, or recovery mechanism is shown.

Every selector lookup is a fresh read. Equal selector addresses do not imply equal selector values. Repeated stores remain separate effects even when their retained values and destinations match.

Dependencies may modify globals, selector bytes, word sources, destinations, or other execution state. No preservation guarantee is established. Captured snapshots and explicitly retained values nevertheless remain the values used by the displayed later expressions.

No allocation, release, or ownership-transfer primitive is directly identified. This does not constrain what dependencies do internally. Passing a quantity that can also form an address does not establish transfer of memory ownership.

No bounds validation, null check, error return, or exception handler is displayed. Continuing execution requires accessible addresses, valid call targets, suitable execution state, and returning dependencies. Failure behavior is **documented_only**.

Endianness, atomicity, volatility, alignment requirements, and interprocessor ordering are not established.

## Dependencies

Only the identity F-b9351b55fcaa is supplied. The following opaque IDs classify dependency roles, not proven target identities:

| Opaque ID | Role |
|---|---|
| F-U001 | Initial indirect call, selected partly from incoming execution state. |
| F-U002 | Five-call groups producing selectors or intermediate results. |
| F-U003 | Nine-call forwarding groups and their compound argument lists. |
| F-U004 | Seven-call groups in the late byte-store region. |
| F-U005 | Dependency associated with the secondary register result. |
| F-U006 | Loop-entry and repeating-sequence dependencies. |

One target may occupy several roles; several targets may occupy one role. Signatures, side effects, ownership behavior, and preservation rules remain unresolved. No-argument displays are not proof of argument-free interfaces.

## Acceptance examples

These are conditional static examples, not runtime tests.

1. **Initial extraction.** With first argument `0x89ABCDEF`, the initial stored byte is `0xEF`; the second is `0x26`. Different observed selector bytes may produce different destinations.

2. **Third-argument boundaries.** Values zero through 127 select word address `0x14`; 128 through 255 select `0x18`. The maximum unsigned 32-bit argument selects `0x08000010`. This calculation does not overflow 32 bits.

3. **Snapshot retention.** If the read associated with `0x3C` converts to `0x12345678`, both early stores use that value even if a dependency changes the source between stores.

4. **Correct retained-offset destination.** Suppose the selector result in the first intermediate-store phase is three. Its four-times quantity is 12. The initial record-scaled store writes 12 at `0x44`. After the group beginning with the first snapshot shifted right by 16, the later store writes 12 at `0x20`, meaning `0x14` plus 12. Writing at `0x44` again would incorrectly rescale the retained quantity.

5. **Full-word shift.** A second snapshot of `0xFEDCBA98` produces a later full-word store of `0x00FEDCBA`, not `0xBA`.

6. **First-snapshot derivations.** A first snapshot of `0xA1B2C3D4` supplies `0xA1B2` when shifted right by 16, `0x050D9618` when shifted right by eight and multiplied by eight, and direct late byte destination `0x1D`.

7. **Fresh selection.** A selector byte of `0xFF` selects `0x110`. A subsequent selector read returning zero selects `0x11`, including when the changed value results from an aliased earlier write.

8. **Loop conditions.** Every possible initial loop field produces zero as the first iteration’s second argument. Under 32-bit truncation, an iteration-ending result of one produces next carried result four and next loop field `0x04000000`; the following second argument is `0x00040000`.

## Uncertainties

The following are **documented_only**:

- Executability and completeness of the represented control flow.
- The incoming register contract and dependency signatures.
- Signed overflow, signed left shifts, integer-to-address conversion, and effective address width.
- The underlying access width at `0x3C`.
- Memory mapping, access permissions, alignment, and device semantics.
- Dependency preservation, allocation, release, ownership, fault, and nonlocal-exit behavior.
- The secondary register result’s convention.
- Relevant behavior in discarded or otherwise unavailable regions.
- Application-level meaning.

## Evidence

**Established static evidence:** The supplied representation displays byte-selector reads, byte and 32-bit stores, two retained word snapshots, repeated writes, indirect-call sequences, unsigned masks and shifts, a secondary register result, and unconditional final repetition. The corrected intermediate destination is `0x14` plus the already-retained four-times quantity, without another scaling operation.

**Inferred intent:** None is adopted. “Selector,” “snapshot,” and scaled-address terminology describe data relationships, not application objects or purpose.

**Evidence limitations:** Warnings identify invalid instruction data, discarded unreachable regions, and overlapping global symbols. These limit confidence in recovered types, calling conventions, and control flow.

Only the supplied static material was examined. No implementation access, runtime observations, or runtime tests were used.
