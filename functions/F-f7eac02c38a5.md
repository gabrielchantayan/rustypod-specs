<!-- Trusted provenance: F-f7eac02c38a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f7eac02c38a5

## Purpose

F-f7eac02c38a5 performs an ordered prefix of indirect invocations, retains and transforms execution-dependent values, mutates a graph of computed and fixed memory locations, and makes a final indirect transfer with four visible arguments.

No domain-level purpose is established. The numeric constants, repeated offsets, and mixed-width writes do not establish formatting, parsing, initialization, object construction, or device configuration.

This contract distinguishes three evidence classes:

- **Established static evidence:** represented accesses, widths, ordering, and value dependencies.
- **[INFERENCE]:** consequences requiring an additional execution-model assumption.
- **documented_only:** unresolved semantics or explicitly unenumerated documentary coverage. This does not make visible static operations optional.

The operation is context-dependent and effectful, not a pure function with no inputs.

## Interface

### Inputs and result

The represented interface has no explicit parameters and no declared result. No status code, output-buffer argument, or returned ownership token is established.

Its effective inputs include:

- Entry execution state determining the first indirect destination.
- An inherited register-sized value consumed by early shifts.
- Primary, paired, and incompletely represented secondary dependency outputs.
- Memory used as data, address bases, and sources of subsequent addresses.
- Fixed-address storage.
- An unresolved terminal destination.

Some dependency outputs occupy a represented 64-bit carrier whose lower and upper 32-bit portions are consumed separately. This does not establish an ordinary 64-bit return-value ABI.

### Widths and arithmetic

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Address-forming scalars are represented as 32 bits. Byte writes retain the source’s low eight bits; explicitly unsigned byte reads are zero-extended when promoted.

Offsets below are byte displacements. Neither object boundaries nor independence between bases is established.

Unsigned shifts have the represented unsigned-width behavior. Signed overflowing shifts, signed multiplication overflow, and signed right shifts require target interpretation.

**[INFERENCE] Under 32-bit truncating machine arithmetic**, additions, multiplications, and left shifts retain their low 32 bits. Successive shifts must then truncate at each intermediate width; they are not arbitrary-precision arithmetic. Multibyte byte order, alignment requirements, and permitted unaligned accesses remain **documented_only**.

### Preconditions

Normal completion requires every reached destination to be executable, every reached memory access to be supported, and the ambient execution state to satisfy unrecovered dependency interfaces.

These are prerequisites, not checks performed by this function. No null, bounds, alignment, or executable-address validation is represented. Conventional valid heap pointers alone are insufficient because the operation explicitly accesses low fixed addresses.

## Behavior

### Indirect invocation prefix

The first invocation destination depends on entry execution state plus 4094 bytes. Later destinations generally use embedded code-related bases plus the same displacement. Their identities and complete interfaces are not established.

The represented prefix is sequential, without an explicit governing loop or conditional branch. Each continuation requires the preceding invocation to return normally. Dependencies may modify memory or external state; no preservation rule follows from the absence of direct writes in this prefix.

Visible argument lists contain zero through four values. Recurrent transformations include left shifts, logical right shifts, explicit zeros, repeated arguments, extraction of paired-result halves, and forwarding of retained values across intervening invocations. Secondary outputs are also consumed without complete recovered interfaces.

The shifts reach 31 bits. Masks select low-bit fields of several widths, including 24, 22, 21, 20, 19, 18, 17, 14, 13, 12, and 11 bits. A ten-bit field is obtained by retaining 20 low bits and shifting right ten. Other expressions consume an upper result half shifted right by two, three, four, or twelve. These are transformations, not validations.

The exact invocation-by-invocation argument ledger is **documented_only documentary coverage** here. Similar argument shapes do not authorize merging or reordering invocations.

### Retained values entering direct mutation

The following descriptive handles identify value relationships, not structure fields or source identifiers:

| Handle | Meaning |
|---|---|
| **Shifted base** | Lower portion of a late paired output shifted left 25 bits |
| **Scaled seed** | A subsequent primary output multiplied by \(2^{25}\) |
| **Seed** | The unscaled output underlying the scaled seed |
| **Aligned scalar** | Another primary output shifted left eight bits |
| **Small index** | A later primary output shifted left eight, masked to 15 bits, then logically shifted right six |
| **Index base** | Twice the small index |
| **Final halves** | Lower and upper 32-bit portions of the last paired output before direct mutation |

Under the stated 32-bit truncating interpretation, the small index is a multiple of four from 0 through 508, and the index base ranges from 0 through 1016.

These values are retained scalars. Later reads from addresses derived from them are separate observations.

### Initial overwrites and alias-sensitive reload

The first direct effects publish the shifted base as a word at **final upper half + 20**, then publish the upper half’s low byte at **shifted base + 9**. A word containing the scaled seed is written at **256 times the final lower half + 84**. The byte at **shifted base + 9** is then overwritten from the shifted base itself.

The next access group is order-sensitive:

- The word at **scaled seed + 20** is cleared before the unsigned byte at **scaled seed + 5** is read.
- The word at **index base + 20** is cleared before that byte’s promoted value is written as a word at **scaled seed + 52**.
- The word at **scaled seed + 20** is then reloaded and retained as the **cleared-word observation**.
- Another clear at **index base + 20** precedes two byte stores at **index base + 21**, both sourced from the cleared-word observation.

The reload is not unconditionally replaceable with zero. Intervening writes may overlap it, and the memory model is unresolved.

The unsigned byte at **fixed address 5** is read next and retained as the **lookup byte**. Only afterward does **fixed address 1** receive the index base’s low byte. A byte at **lookup byte + 21** receives the small index’s low byte.

The aligned scalar’s low byte is written at **scaled seed + 17**, and its full word is written at **cleared-word observation + 84**. The byte at **scaled seed + 17** is subsequently cleared.

A word read at **lookup byte + 84** supplies the retained **routing value**; that location is then overwritten with the cleared-word observation. The small index is published as a word at **index base + 52**.

### Routing-value propagation

A word read at **scaled seed + 116** supplies the **first route**. At **first route + 84**, a word derived from the seed shifted left 26 bits is written before a later overwrite from the aligned scalar.

Between those writes, two represented word reads at **index base + 116** participate in writing a word at offset 84 from a read-derived base. One supplies the destination base and one supplies the stored value. Their mutual evaluation order is not established; they must not silently be treated as one read.

The routing value is written twice into **routing value + 36**. Each publication is followed by a byte publication at **first route + 9**; the aligned-scalar overwrite at **first route + 84** separates the two pairs.

The next word at **index base + 20** contains the seed shifted left 26 bits. Then **fixed address 84** receives a word containing the routing value shifted left one bit. Two word stores at **index base + 20** subsequently publish the routing value.

For the remaining routing phase:

- **Fourfold base** means four times the routing value.
- **Eightfold value** means the routing value shifted left three bits.

Two byte stores at **index base + 21** publish the fourfold base’s low byte. **Fixed address 21** then receives the index base’s low byte.

The aligned scalar is published at **fourfold base + 84** before the fourfold base is written at **index base + 52**. A fresh word read at **index base + 116**, doubled, becomes the **doubled route**. The word at **fourfold base + 84** is then overwritten with the fourfold base.

The byte at **fourfold base + 21** first receives the eightfold value’s low byte. The doubled route is written at **routing value + 68**.

An unsigned byte read at **fourfold base + 5** selects a word address by adding 116. The word obtained there is the **selected route**. Its offset-68 word receives the doubled route. Subsequent publications place:

- The fourfold base’s low byte at **fourfold base + 21**.
- The selected route’s low byte at **fourfold base + 13**.
- The fourfold base at its own offsets 20 and 84.

A word read at **fourfold base + 100** supplies another base, whose offset-13 byte receives the selected route’s low byte and whose offset-20 word receives that base itself.

A word read at **fourfold base + 4** supplies the **byte-source base**. The doubled route’s offset-9 byte receives the selected route’s low byte, and its offset-68 word receives the eightfold value.

An unsigned byte read at **byte-source base + 5** is promoted into the word at **selected route + 68**. Before that word publication, **fourfold base + 9** receives the doubled route’s low byte.

Further writes publish the eightfold value at **fourfold base + 20**, the byte-source base’s low byte at **doubled route + 21**, and the fourfold base at **doubled route + 68**.

The byte at **byte-source base + 5** is read again. This is a new observation after intervening stores. Following a byte publication of the eightfold value at **fourfold base + 21**, a word read at **116 plus that newly read byte** supplies the **final route**. Twice this final route becomes the **terminal mutation base**.

Before constant publication, the fourfold base’s offset-21 byte receives the byte-source base’s low byte; its offset-84 word receives the fourfold base; its offset-9 byte receives the terminal mutation base’s low byte; and its offset-20 word receives the eightfold value.

### Constants, adjusted scaling, and repeated publication

The constant 58 is published as:

- A byte at **four times the final route**.
- A halfword at **eight times the final route**.
- A word at **four times the final route + 32**.

These remain distinct access widths.

The **adjusted scaling source** is specifically **the prior fourfold base minus 150**. It is not a newly read value or an unspecified adjustment. Four times it becomes the **adjusted base**; eight times it becomes the **adjusted eightfold base**.

At **adjusted base + 32**, the ordered word publications are 32, 47, a zero-extended byte read from **adjusted base + 4**, and 32 again. That byte read occurs between the second and third word stores.

The word at **fixed address 0x087fbbbc** is then read and retained as the **publication base**. This read precedes the word store of 32 at **fixed address 96**.

The initial repeated-publication group writes four times the final route at **eightfold value + 84**, then writes reciprocal low bytes at **publication base + 25** and **adjusted base + 17**, then writes the original eightfold value at **publication base + 52**.

Subsequent groups preserve the same access order and retained byte sources, but:

- Their offset-84 destination is **adjusted eightfold base + 84**.
- Their offset-52 value is the **adjusted eightfold base**.

The publication base is retained from the earlier fixed-address read, not reloaded for each group. Likewise, reciprocal byte values are retained truncations, not fresh reads through either base.

The repeated groups’ exact count is **documented_only documentary coverage** in this contract. The repetitions are explicit static effects, not unknown dependency behavior, a retry policy, or permission to collapse the sequence to final memory contents.

### Late fixed-address accesses and retained observations

After repeated publication, the late phase begins with a word read at **fixed address 169**. Its complete value is retained as the **fixed-word observation**.

A word read at **four times the final route + 116** supplies the **route-tail observation**. Next, **fixed address 58** receives the low byte of eight times the adjusted scaling source plus 48.

Two further word reads retain:

- The **retained terminal word** from **terminal mutation base + 20**.
- The **retained offset-four word** from **adjusted base + 4**.

The following publications precede the write to fixed address 103: the adjusted base’s low byte at **terminal mutation base + 13**; 108 at **adjusted base + 8**; the publication base’s low byte at adjusted-base offsets 13 and 9; and the terminal mutation base at its own offset 84.

**Fixed address 103** then receives the fixed-word observation’s low byte. A word read at **terminal mutation base + 68** supplies the **late route observation**. After this read, **fixed address 103 is read as a byte**, and that observation is written to **fixed address 33**.

The late route observation’s low byte is published at **adjusted base + 16**, and its word at **publication base + 84**. A byte at **terminal mutation base + 1** receives a signed-right-shift-derived value: the late route observation shifted by the adjusted base’s low eight bits. The word at **publication base + 84** is then overwritten with the adjusted base.

A word read at **adjusted base + 64** precedes publication of the retained offset-four word at **publication base + 116**. The newly read offset-64 word is written at **adjusted base + 16**, overlapping the earlier byte publication there. The terminal mutation base is written at **publication base + 68**.

The remaining address-103 uses are separate byte reads:

- One supplies a byte store at **adjusted base + 9**, before the adjusted base’s low byte is written at **terminal mutation base + 9**.
- The fixed-word observation is then published at **adjusted base + 68**, and a fresh word read at **adjusted base + 84** supplies the **late destination**.
- Another read of address 103 supplies another byte store at **adjusted base + 9**.

Thus the original retained low byte from address 169 is not a substitute for later reads at address 103. Intervening computed writes may alias that address.

### Final mutation and transfer

The route-tail observation’s low byte is published at **late destination + 13**, and the retained terminal word at **terminal mutation base + 84**.

A fresh word read at **adjusted base + 20** is retained for a later byte publication. The route-tail observation’s low byte is written at **terminal mutation base + 17**. The late destination’s low byte is then written at **adjusted base + 9**, followed by its full word at **adjusted base + 68**.

A word read at **terminal mutation base + 4** supplies the **final object base**. A halfword containing 101 is written at twice that base. A word read at **final object base + 100** supplies the **terminal second argument**.

The retained adjusted-base offset-20 observation supplies the byte at **late destination + 17**. A word containing 100 is written at **late destination + 84**. A subsequent word read at **final object base + 116** supplies the **terminal fourth argument**.

A word read at **terminal mutation base + 100** supplies the final publication destination. The final object base’s low byte is written at **terminal mutation base + 9**. The terminal second argument is written at offset 84 from that final publication destination, and the final publication destination is written as a word at twice the terminal second argument.

The final indirect transfer receives, in order:

1. 114.
2. The retained terminal second argument.
3. The retained terminal word from the earlier terminal-mutation-base offset-20 read.
4. The retained terminal fourth argument.

A return is represented afterward. Whether the transfer actually behaves as an ordinary returning call remains unresolved.

## Side effects

All described memory accesses are potentially alias-sensitive. Fixed addresses are not exempt: computed addresses may overlap them, and multibyte accesses may overlap smaller locations.

The fixed-address access subsequence is therefore significant: byte read at 5; byte write at 1; word write at 84; byte write at 21; word read at 0x087fbbbc; word write at 96; word read at 169; byte write at 58; byte write at 103; byte read at 103 supplying the write at 33; and two later byte reads at 103 supplying separate computed writes.

No atomicity, transaction, rollback, synchronization, barrier, or dependency preservation guarantee is established. Access order is represented order; actual hardware observability depends on the unresolved environment.

No known allocation, release, reference-counting operation, or ownership transfer is identified. Address-like publications do not establish ownership, borrowing, retention, or release.

No explicit error-reporting path is represented. A dependency failure, invalid access, or invalid transfer can interrupt execution after earlier effects. Recovery behavior is **documented_only**.

## Dependencies

The following opaque references are documentary placeholders, not additional resolved identities:

| Opaque reference | Role |
|---|---|
| **F-unresolved-entry** | First destination derived from entry execution state |
| **F-unresolved-prefix** | Unresolved collection of subsequent indirect destinations; not a claim of shared identity |
| **F-unresolved-terminal** | Final destination receiving four visible values |

Only F-f7eac02c38a5 has a supplied resolved opaque identity. Dependency calling conventions, hidden inputs, memory effects, register preservation, ownership rules, and return behavior remain **documented_only**.

## Acceptance examples

These are conditional criteria, not executed tests.

- **Initial reload:** If an intervening overlapping store changes the initially cleared word, the cleared-word observation must reflect the later read. Substitution of zero is not accepted without additional memory assumptions.
- **Early fixed addresses:** If accesses are observable, the byte read at address 5 must precede the byte write at address 1. The lookup byte remains retained even if address 5 changes afterward.
- **Adjusted scale:** If the retained routing value is 100 and arithmetic is nonoverflowing, the fourfold base is 400, the adjusted scaling source is 250, the adjusted base is 1000, and the adjusted eightfold base is 2000. Subsequent publication groups use 2000, not the original eightfold value 800.
- **Width-sensitive constants:** Valid destinations for 58 require distinct byte, halfword, and word accesses. Three byte stores are not equivalent.
- **Publication ordering:** The read at 0x087fbbbc must precede the fixed-address word write at 96. Later publication uses the retained read result, even if overlapping stores change its original location.
- **Address-103 rereads:** If a computed write changes address 103 between its uses, each subsequent byte read must observe the then-current value under the target memory model. Reusing the original address-169 truncation is insufficient.
- **Unsigned promotion:** An unsigned byte observation of 255 supplies a promoted word value of 255, not a sign-extended negative value.
- **Terminal retention:** Changes to the earlier offset-20 source after its read do not change the retained third terminal argument. Reloading that source at transfer time requires proof of equivalence.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, intended caller, and valid entry context.
- Dependency identities and complete interfaces.
- Architecture, byte order, alignment, address mappings, and device-backed storage.
- Signed overflow, signed right-shift behavior, and variable shifts exceeding 31.
- Object boundaries and actual alias relationships.
- Ownership, lifetime, and dependency preservation rules.
- Mutual evaluation order of the two reads in the compound read-derived publication.
- Exact prefix argument ledger and repeated-publication count, as documentary coverage omissions.
- Terminal transfer mechanism and actual return behavior.

The documentary omissions are distinct from unresolved execution semantics. Known fixed-address effects and the minus-150 scaling relationship are established static dependencies.

## Evidence

This contract is based only on the supplied static representation, independently of prior wording and reviewer interpretations.

Established evidence comprises the parameterless represented interface, ordered indirect invocations, retained result portions, mixed-width memory operations, fixed-address accesses, repeated publications, alias-sensitive reloads, explicit adjusted scaling, and final four-argument transfer.

Machine-truncation consequences are expressly **[INFERENCE]**. No runtime tests, implementation execution, or external dependency verification have been performed.
