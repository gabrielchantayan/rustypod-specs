<!-- Trusted provenance: F-ab2ea4f08e05; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab2ea4f08e05

## Purpose

F-ab2ea4f08e05 exposes a memory-mutating operation followed by an unresolved indirect control transfer. The supplied static representation establishes a sequence of byte, halfword, and word accesses, including repeated stores and accesses to low absolute addresses. It does not establish an application-level purpose.

The contract below describes the visible memory-access behavior without interpreting the constants as text, treating offsets as named structure fields, or assuming that the repeated operations can be collapsed.

**Evidence status:** static-only and conditional on the supplied representation accurately describing executable behavior. No runtime tests have been performed. Calling convention, arithmetic edge semantics, memory mapping, and the indirect target remain partly or wholly **documented_only**.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | Address base for byte writes at offset 9 |
| Second argument | 32 bits | No visible use before the indirect transfer |
| Third argument | 32 bits | Initial address base, stored value, and basis for subsequent scaled addresses |
| Fourth argument | 32 bits | Word payload and source of a truncated byte |

The visible declaration has no return value. This does not establish that execution necessarily returns: reaching the apparent return depends on the unresolved indirect transfer returning normally.

### Additional entry state

Four values carried in incoming machine state are consumed without initialization in the visible body. They are therefore effective inputs, even though they are absent from the explicit argument list:

- **Source base:** supplies a byte at offset 5 and contributes low-byte values to stores.
- **Auxiliary base:** receives a word at offset 0x44.
- **Peer base:** contributes a low byte and receives writes at offsets 0x15 and 0x44.
- **Link value:** is stored unchanged, doubled, and used in address formation.

No calling-convention guarantee for these values is established.

A separate **anchor value** is read once from an externally located word-sized slot after the initial memory mutations. Its value is then used as an address base and as a source of truncated bytes. The slot’s entry value cannot safely be substituted for its value at the actual read: earlier accesses might alias it.

### Contract notation and widths

The following terms describe derived values, not additional inputs:

- **Selected value:** the 32-bit value loaded from absolute address 0x74 plus the second byte read from source-base offset 5.
- **Doubled selection:** twice the selected value.
- **Quadrupled selection:** four times the selected value.
- **Adjusted third argument:** the original third argument minus 0x96.
- **Scaled base:** four times the adjusted third argument.
- **Octupled adjustment:** eight times the adjusted third argument.

A byte access is 8 bits, a halfword access is 16 bits, and a word access is 32 bits. A “low byte” means the least significant 8 bits of a value; it does not mean the byte at the lowest address in a word.

The representation mixes signed 32-bit arithmetic with unsigned 32-bit storage. Unsigned conversion and byte truncation are visible. Signed overflow, negative shifts, excessive shift counts, and address wraparound are not assigned portable semantics here.

### Preconditions for complete execution

Every executed read and write must address memory supporting that access width. This includes computed addresses, low absolute addresses, and potentially unaligned word accesses. No null checks, bounds checks, alignment checks, or memory-map checks are visible.

The indirect target must also be valid under an otherwise unknown transfer convention. No ownership or lifetime guarantee follows from an address being used.

## Behavior

### Initial publication and selection

The operation first reads one byte at source-base offset 5. Before reading that location again, it performs these observable publications in order:

- A byte at first-argument offset 9 receives the peer base’s low byte.
- A word at auxiliary-base offset 0x44 receives the first sampled byte, zero-extended to 32 bits.
- A word at original-third-argument offset 0x14 receives the fourth argument.
- A byte at peer-base offset 0x15 receives the source base’s low byte.
- A word at peer-base offset 0x44 receives the original third argument.

The second byte read at source-base offset 5 is independent of the first sample. Intervening writes may affect it through aliasing.

After that second sample, a byte at original-third-argument offset 0x15 receives the fourth argument’s low byte. The selected value is then loaded as a word from address 0x74 plus the second sample. The access therefore starts somewhere in the inclusive address range 0x74–0x173; it is not a word-indexed lookup.

After this read, the operation:

- Replaces the byte at original-third-argument offset 0x15 with the source base’s low byte.
- Stores the link value as a word at original-third-argument offset 0x54.
- Replaces the byte at first-argument offset 9 with the doubled selection’s low byte.
- Again stores the fourth argument as a word at original-third-argument offset 0x14.

The last word store overlaps the byte at offset 0x15. Its resulting byte value depends on byte order. The contract does not preserve the earlier byte there independently of the later word write.

### Constant stores and scaled-address preparation

The next phase writes numeric value 0x3A with three different widths:

| Destination | Width |
|---|---:|
| Quadrupled selection | Byte |
| Eight times the selected value | Halfword |
| Quadrupled selection plus 0x20 | Word |

The adjusted third argument and scaled base determine the subsequent destinations. At scaled-base offset 0x20, the operation makes four word stores in this order: numeric 0x20, numeric 0x2F, a zero-extended byte sampled from scaled-base offset 4, and numeric 0x20.

The byte sample occurs after the first two stores and before the third. The intermediate writes remain part of the visible access sequence even when the last word would overwrite them in ordinary non-overlapping memory.

The anchor value is read after this sequence. A word at absolute address 0x60 then receives numeric 0x20.

### Repeated association stores

A word at twice the link value plus 0x54 receives the quadrupled selection. The operation then establishes a repeated pattern connecting the anchor and scaled regions:

- Anchor offset 0x19 receives the scaled base’s low byte.
- Scaled-base offset 0x11 receives the anchor’s low byte.
- Anchor offset 0x34 receives a word-sized scaled link.

The first scaled link is twice the incoming link value.

The subsequent repeated groups additionally write the quadrupled selection as a word at octupled-adjustment plus 0x54. In those groups, anchor offset 0x34 receives the octupled adjustment instead of twice the incoming link value. Each complete group orders the selection-word store before the two byte stores and the anchor link-word store.

The supplied representation contains many explicitly repeated groups. Their exact multiplicity is **documented_only** in this contract; no numeric count is asserted. This is a precision limitation, not permission to replace the sequence with one final-state update. Repeated writes can matter for device memory, aliasing, and fault position.

### Late sampling and state rewiring

After the repeated phase, three word samples are taken, with an intervening byte write:

1. A word is sampled at absolute address 0xA9.
2. A word is sampled at quadrupled-selection plus 0x74.
3. A byte at absolute address 0x3A receives the low byte of eight times the adjusted third argument plus 0x30.
4. A word is sampled at doubled-selection plus 0x14.
5. A word is sampled at scaled-base plus 4.

The sample at scaled-base plus 4 is a new word read, not a reuse of the earlier byte sample at that address.

The subsequent mutations have overlapping destinations and must retain their ordering:

- Doubled-selection offset 0x0D receives the scaled base’s low byte.
- Scaled-base offset 8 receives byte 0x6C; offsets 0x0D and 9 receive the anchor’s low byte, in that order.
- Doubled-selection offset 0x54 receives the doubled selection as a word.
- Absolute address 0x67 receives the low byte of the previously sampled word from 0xA9.

A word is then read from doubled-selection offset 0x44; call it the **first late reference**. The byte at 0x67 is read and copied to absolute address 0x21. This copy must not be replaced with an assumption that the earlier stored value remained unchanged.

The first late reference is published as a byte at scaled-base offset 0x10 and as a word at anchor offset 0x54. Its signed right-shifted result is also stored as a byte at doubled-selection offset 1. The shift count is the scaled base’s low 8 bits. Semantics outside the supported signed-shift domain remain unknown.

Anchor offset 0x54 is then overwritten with the scaled base. Next, a word is read from scaled-base offset 0x40. After that read:

- Anchor offset 0x74 receives the earlier word sample from scaled-base offset 4.
- Scaled-base offset 0x10 receives the newly sampled word from offset 0x40.
- Anchor offset 0x44 receives the doubled selection.

The word write at scaled-base offset 0x10 overwrites the earlier byte write there and overlaps offset 0x11.

Next, scaled-base offset 9 receives a fresh byte read from absolute address 0x67. Doubled-selection offset 9 receives the scaled base’s low byte. Scaled-base offset 0x44 receives the saved word originally sampled from absolute address 0xA9.

### Final reference chain

A word read from scaled-base offset 0x54 supplies the **second late reference**. After that read, scaled-base offset 9 again receives a fresh byte read from 0x67.

The low byte of the word sampled earlier from quadrupled-selection plus 0x74 is stored at second-late-reference offset 0x0D. Doubled-selection offset 0x54 is restored to the word sampled earlier from doubled-selection offset 0x14.

A new word is sampled at scaled-base offset 0x14. The low byte of the earlier quadrupled-selection sample is then stored at doubled-selection offset 0x11. Scaled-base offset 9 receives the second late reference’s low byte, and scaled-base offset 0x44 receives that reference as a word.

The remaining reference chain consists of:

- A word read from doubled-selection offset 4, producing the **third late reference**.
- A halfword store of numeric 0x65 at twice that reference.
- A word read from third-late-reference offset 0x64, producing the **terminal link**.
- A byte store at second-late-reference offset 0x11 using the low byte of the recently sampled scaled-base-offset-0x14 word.
- A word store of numeric 100 at second-late-reference offset 0x54.
- A word sample from third-late-reference offset 0x74, retained for the indirect transfer.
- A word read from doubled-selection offset 0x64, producing the **terminal reference**.

Finally, doubled-selection offset 9 receives the third late reference’s low byte. Terminal-reference offset 0x54 receives the terminal link as a word. A word at twice the terminal link receives the terminal reference.

### Indirect transfer

The visible transfer supplies four 32-bit argument values, in order:

1. Numeric 0x72.
2. The terminal link.
3. The saved word sampled from doubled-selection offset 0x14.
4. The saved word sampled from third-late-reference offset 0x74.

The target is unresolved. Its return value is not visibly consumed. Its memory effects, register effects, ownership behavior, and return behavior are unknown.

## Side effects

The established output is the ordered memory-access trace and any resulting memory state, not a returned result.

No allocation, release, ownership transfer, lock, rollback, or explicit error-reporting operation is visible before the indirect transfer. Storing a value that resembles an address does not establish ownership.

Aliasing is unrestricted by the supplied evidence. In particular:

- Explicit and implicit bases may overlap.
- Computed destinations may overlap absolute-address storage.
- Byte writes may change later word reads.
- Repeated or wider stores may overwrite earlier byte publications.
- The anchor slot may be changed before it is sampled.

There is no visible recovery path for an invalid access. If execution faults, prior completed writes may remain observable; no transactional guarantee is established.

The ordered trace describes the supplied representation. Hardware-level atomicity, memory barriers, inter-thread synchronization, and device-bus transaction behavior are not established.

## Dependencies

| Dependency | Identification | Established contract |
|---|---|---|
| This operation | F-ab2ea4f08e05 | Memory behavior described above |
| Terminal indirect target | Opaque function ID unavailable; **documented_only** | Receives the four listed values if reached |

No other explicit function call is visible. The unavailable target ID must not be replaced with an invented dependency identity.

No assumption is made that the indirect target preserves globals, returns normally, or owns or releases memory.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Separate initial samples

If the first source byte is 0x12 and intervening writes cause the second read to produce 0x34, auxiliary-base offset 0x44 receives word 0x00000012. The selection read begins at address 0xA8, because its address uses the second sample.

A result based exclusively on the first sample would violate the visible read distinction.

### Width-sensitive constant writes

If the selected value is 0x100 and arithmetic is representable, the initial constant stores target address 0x400 as a byte, address 0x800 as a halfword, and address 0x420 as a word.

All three values are numeric 0x3A. The contract does not reinterpret the halfword or word as repeated character bytes.

### Overlapping payload field

If the original-third-argument region is writable ordinary memory and has no other aliases, its final word at offset 0x14 after the initial phase equals the fourth argument.

The byte at offset 0x15 consequently comes from that word’s representation, rather than necessarily retaining the source base’s low byte. Which payload byte appears there depends on endianness.

### Scaled destination

If the original third argument is 0x196, the adjusted value is 0x100 and the scaled base is 0x400. The four word stores at address 0x420 carry 0x20, 0x2F, the zero-extended byte sampled from 0x404, and 0x20, in that order.

This example assumes earlier accesses do not fault or invalidate the relevant memory.

### Unsupported shift domain

If the scaled base’s low byte is 40, the visible expression requests a signed 32-bit right shift by 40. This contract supplies no expected resulting byte and does not assume a masked count.

### Transfer arguments remain sampled values

If execution reaches the indirect transfer, its third argument is the saved doubled-selection-offset-0x14 sample, even if later stores change that memory. Its fourth argument likewise comes from the earlier third-late-reference-offset-0x74 read, not a new read at transfer time.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and data-structure identities.
- Accuracy and completeness of the supplied static representation.
- Calling convention and provenance of implicit entry values.
- Exact repeated-group multiplicity.
- Endianness, alignment support, and mapped-memory permissions.
- Signed overflow, address wraparound, and exceptional shift semantics.
- Volatile, concurrent, or device-memory behavior.
- Indirect target identity, transfer convention, effects, and return behavior.
- Fault handling outside the visible operation.

A plausible interpretation is that some stores connect address-bearing records. That is inferred intent only; the representation does not justify a list, tree, ownership, or serialization contract.

## Evidence

The supplied static body supports the access widths, visible ordering, constants, repeated-store pattern, saved read values, and terminal argument sequence described above.

It also shows that one explicit argument is unused before the unresolved transfer and that several effective inputs arrive through unspecified machine state.

These are source-facing observations, not verified runtime facts. No executable, surrounding callers, dependency implementation, authoritative type layout, or runtime trace was available. No runtime tests have been performed.
