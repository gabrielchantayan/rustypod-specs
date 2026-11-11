<!-- Trusted provenance: F-6f5f8ffdc146; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6f5f8ffdc146

## Purpose

The supplied static representation establishes an ordered memory-writing entry sequence, stack-relative state saves, conditional narrow stores, invalid-instruction boundaries, and two conditional 64-bit return constructions. It does not establish an application-level purpose or a reliably returning operation.

Under stable, non-null address assumptions, the ordinary continuation performs state saves, writes a 16-bit payload fragment, and reaches an invalid-instruction boundary. Other represented continuations include selector-dependent stores, a pointee transfer, three ordered global reads, and returns. Several require a previously tested address to produce the opposite result on a repeated test without an intervening assignment. Their reachability is unresolved.

This contract distinguishes established access sequences from uncertain control-flow reachability. It does not repair the represented operation or treat displayed return paths as necessarily callable outcomes.

**Inferred intent:** the saves and destination-relative updates may participate in a larger state-manipulation operation. That interpretation is not contractual.

## Interface

### Inputs

No explicit arguments are declared. Incoming machine state and memory nevertheless supply these inputs:

| Role | Use |
|---|---|
| Initial destination | Base for the first 32-bit write at byte offset `0x40` |
| Initial payload | Value of that write |
| Context | Supplies fields at offsets `0x08` and `0x30` |
| Incoming return state | A 32-bit value saved on continuation paths |
| Incoming stack state | Stack-relative addresses, a word, an alternate destination, and a pointer-valued payload |
| Conditional globals | Three replacement values read on one unresolved route |

These are behavioral input roles, not an established calling convention.

### Referenced storage

All offsets below are in bytes.

| Base | Offset | Width | Role |
|---|---:|---:|---|
| Initial destination | `0x40` | 32 bits | Initial write |
| Context | `0x08` | 32 bits | Destination-like value |
| Context | `0x30` | 32 bits | Descriptor address |
| Descriptor | `0x00` | 32 bits | Operand shifted left by two |
| Descriptor | `0x08` | 32 bits | Destination for the conditional pointee transfer |
| Descriptor | `0x10` | 32 bits | Pointer-valued payload |
| Alternate destination | `0x3E` | 16 bits | Selector on one route |

Destination-relative narrow writes use offsets `0x06`, `0x0E`, `0x1E`, `0x26`, and `0x2E`. These accesses do not establish complete object layouts or extents.

The following notation is introduced solely for this contract:

- **S:** frame reference used by a particular shared-continuation entry.
- **A:** alternate frame base selected before return processing.
- **N:** rebased frame reference, equal to A minus `0x18`.
- **Tail word:** the 32-bit value read at A plus `0x338`.

These names are descriptive notation, not recovered interface identifiers.

### Widths, arithmetic, and preconditions

The representation uses 32-bit words and address representations, 16-bit selectors and stores, and 64-bit return assembly. Word-indexed offsets have four-byte spacing.

A narrow store writes the payload’s low 16 bits. A pointer-valued payload supplies address bits for these stores; it is not dereferenced except on the explicitly identified pointee-transfer route.

The descriptor’s first word is shifted left by two in a represented 32-bit expression. Count or size semantics are not established. Signedness consequences, overflow behavior, and exact instruction semantics remain `documented_only`.

Completing any access requires suitable readable or writable storage. No protective null, bounds, or permission checks precede the entry accesses. Alignment, address wraparound, and target byte order are not established. Frame-offset identities below describe the represented address calculations, not a guarantee that all resulting addresses are valid.

### Outputs

The ordinary stable-address continuation has no established normal return.

Two conditional returns assemble 64 bits:

- Low half `0x00000031`; high half read from the current rebased frame address.
- Low half `0x000000B9`; high half read from alternate frame offset `-0x14`.

Both high-half reads occur after preceding stores. Aliasing can change the values ultimately returned. Neither constant has an established success or error meaning.

## Behavior

### Ordered entry accesses

Before branching, the operation:

1. Writes the initial 32-bit payload at initial destination plus `0x40`.
2. Reads the context field at `0x08`.
3. Reads the descriptor address from context offset `0x30`.
4. Reads the transfer-destination base from descriptor offset `0x08`.
5. Reads the pointer-valued payload from descriptor offset `0x10`.
6. Reads the descriptor’s first word and evaluates its left shift by two.

The initial write precedes every context and descriptor read. Overlap can therefore change the inputs subsequently observed.

### Entry-relative address checks

The first check concerns the entry-relative address at offset `0x338`.

If it is null, control enters the shared continuation. The frame reference on that route is not reliably established.

If it is non-null, four values are saved in order: payload, tested address, shifted descriptor word, and context-derived destination. The same address is then tested again.

A null result on this repeated test selects the forced-zero selector route described below. A stable non-null result instead:

1. Saves the entry-relative address at offset `0x320`.
2. Tests that address.
3. If null, writes the payload’s low 16 bits at the current destination plus `0x1E`, then reaches an invalid-instruction boundary.
4. If non-null, saves payload and shifted word, establishes another frame reference, saves the destination there, and tests the same address again.

A stable non-null result on the last test enters the shared continuation. A null result selects the selector-reading route. The opposite results required by these alternate routes are not explained by an intervening assignment.

### Shared continuation

Using frame reference S, this continuation forms the address S plus `0x338`.

If that address is null, an invalid-instruction boundary occurs before the following six saves. Otherwise, six 32-bit writes occur in this order:

| Offset from S | Value saved |
|---:|---|
| `-0x04` | Incoming return state |
| `-0x08` | Current pointer-valued payload |
| `-0x0C` | Address S plus `0x338` |
| `-0x10` | Current stack-supplied word |
| `-0x14` | Current shifted-word-role value |
| `-0x18` | Current destination-like value |

The location S minus `0x18` is retained as an alternate frame base. The shared frame reference itself remains S at this point.

The computed address is tested again. A non-null result writes the current payload’s low 16 bits to the current destination plus `0x1E`, then reaches an invalid-instruction boundary. This is the selected continuation when the tested value remains stable.

The displayed null-after-non-null suffix instead:

1. Reads an alternate destination from S plus `0x33C`.
2. Reads an alternate payload from S plus `0x340`.
3. Forms the address S plus `0x344`.
4. Enters the common narrow-store return sequence with alternate base A equal to S minus `0x18`.

This suffix bypasses selector acquisition and selector testing. Its complete return effects are specified below; its reachability is not established.

### Alternate selector setup

The forced-zero route, selected by the first repeated entry-address test becoming null:

- Writes the payload’s low 16 bits to the context-derived destination plus `0x0E`.
- Selects a zero 16-bit selector.
- Selects that destination and payload for subsequent operations.
- Uses an entry-local frame base as A.

The other selector route, selected by the second repeated address test becoming null, reads a 16-bit selector from the stack-supplied destination plus `0x3E`. It uses the newly established frame reference as A and retains the stack-supplied destination and payload.

Both routes then write the selected payload’s low 16 bits to the selected destination plus `0x0E`. The forced-zero route therefore contains two distinct writes to that offset.

These routes require inconsistent repeated address-test results under a stable-value interpretation.

### Common narrow-store return sequence

Two origins enter this sequence:

| Origin | A | Selected destination and payload | Third saved value |
|---|---|---|---|
| Zero selector | Base selected by selector setup | Values retained by that setup | Zero |
| Shared unresolved suffix | S minus `0x18` | Reads from S plus `0x33C` and S plus `0x340` | Address S plus `0x344` |

The third saved value is not universally zero. The shared suffix does not pass through selector setup and does not replace its formed address with zero.

For either origin, the sequence:

1. Reads the tail word at A plus `0x338`.
2. Performs six ordered 32-bit saves:

| Offset from A | Value saved |
|---:|---|
| `-0x04` | Incoming return state |
| `-0x08` | Selected payload |
| `-0x0C` | Origin-specific third saved value |
| `-0x10` | Tail word |
| `-0x14` | Current shifted-word-role value |
| `-0x18` | Selected destination |

3. Rebases the current frame reference to N, equal to A minus `0x18`.
4. Writes the selected payload’s low 16 bits at selected destination plus `0x06`.
5. Writes those same low 16 bits at selected destination plus `0x26`.
6. Writes the tail word at N plus `0x338`.
7. Reads the 32-bit word at N and uses it as the high half of the return with low half `0x31`.

The high-half read is fresh and follows both narrow stores and the final word store.

For the shared suffix specifically, these relationships resolve to:

- Tail-word read: S plus `0x320`.
- Six save locations, in order: S minus `0x1C`, `0x20`, `0x24`, `0x28`, `0x2C`, and `0x30`.
- Third save: address S plus `0x344`, written at S minus `0x24`.
- Final frame reference: S minus `0x30`.
- Final tail-word write: S plus `0x308`.
- Return high-half read: S minus `0x30`.

Thus this suffix performs a second group of six saves after the shared continuation’s first group. It performs both destination-relative narrow stores, despite bypassing the selector. These are conditional static effects, not a claim that the suffix is executable from entry.

### Nonzero selector and the `0xB9` return

For a nonzero selector, the operation reads the tail word at A plus `0x338`, forms that location’s address, and tests the address for null. A nonzero selector does not itself establish a non-null computed address.

If the address is null, the following operations occur in order:

| Order | Value obtained | Save location relative to A |
|---:|---|---:|
| 1 | Word read at `+0x38C` | `-0x04` |
| 2 | Address A plus `0x390` | `-0x08` |
| 3 | Word read at `+0x370` | `-0x0C` |
| 4 | Word read at `+0x36C` | `-0x10` |
| 5 | Word read at `+0x354` | `-0x14` |

The destination is then reread from A plus `0x354`, and the payload is reread from A plus `0x38C`. The payload’s low 16 bits are written to that destination plus `0x2E`. The relative evaluation order of these two operand reads is not independently established.

Finally, the word at A minus `0x14` is reread as the high half of the `0xB9` return. Earlier values cannot replace this final read without an aliasing guarantee.

This branch is locally selected by a null computed address, not excluded by the nonzero selector. Overall reachability remains unresolved.

If the computed address is non-null, six saves occur using the same layout as the common sequence, with that address as the third saved value and the already-read tail word as the fourth. The frame rebases to N.

The address is tested again. A stable non-null result proceeds directly to the tail-word write at N plus `0x338` and the fresh high-half read at N for the `0x31` return. It does **not** perform the narrow stores at destination offsets `0x06` and `0x26`.

### Conditional pointee transfer and global replacement

A null result on that last repeated test selects another unresolved route. It:

1. Reads a 32-bit word through the selected pointer-valued payload.
2. Writes that word at the descriptor-derived transfer destination plus `0x40`.
3. Reads a stack word at A plus `0x320`.
4. Reads the destination-replacement global.
5. Reads the shifted-word-role replacement global.
6. Reads the payload-replacement global.
7. Re-enters the shared continuation with the already rebased frame reference N.

The second global is assigned directly, without another left shift.

On re-entry, shared saves place the third global at frame offset `-0x08`, the newly read stack word at `-0x10`, the second global at `-0x14`, and the first global at `-0x18`.

If the shared computed address is non-null, the eventual narrow store uses the first global as its destination base, adds `0x1E`, and writes the third global’s low 16 bits. An invalid-instruction boundary follows.

## Side effects

The initial 32-bit write is unconditional within the represented entry sequence. Later writes depend on control flow and successful preceding accesses.

No disjointness guarantee separates destinations, context, descriptor, frame storage, globals, or the conditional pointee. Consequently:

- Earlier writes can change later reads.
- Repeated field reads are observably distinct.
- Destination-relative stores can overwrite saved return data.
- The final high-half reads need not recover the originally saved destination.
- Global reads observe memory after the preceding pointee transfer and stack-word read.

The representation supplies access ordering, not a complete hardware memory-order contract. Atomicity, synchronization, concurrency behavior, and rollback are unknown.

No allocation, release, reference-count update, or ownership transfer is established. Accessing storage does not establish ownership or lifetime.

A memory fault can interrupt execution after earlier writes. Invalid-instruction boundaries have no represented recovery path. Fault delivery and effects on an enclosing process remain `documented_only`.

## Dependencies

No independently identified callable dependency is established beyond the operation identified as F-6f5f8ffdc146.

The invalid-instruction boundary is not an established callable service with a defined interface, return behavior, or ownership effect.

Three conditional global dependency roles are established: destination replacement, shifted-word-role replacement, and payload replacement. They are read in that order. Their application meanings, initialization, overlap with other storage, and external preservation rules are unknown.

## Acceptance examples

These are conditional static obligations, not executed tests.

### Entry overlap

If the initial write overlaps context offset `0x08`, the context read observes memory after that write. An unconditional pre-write snapshot would not preserve the specified ordering.

### Narrow payload bits

If a narrow store receives payload bits `0x1234ABCD`, it stores the 16-bit value `0xABCD`. It does not read the payload’s pointee for that value. Byte ordering remains unspecified.

### Stable ordinary continuation

With valid storage, stable non-null entry-relative addresses, and a non-null shared computed address, the operation performs the entry accesses, applicable saves, and destination-relative `0x1E` store, then reaches an invalid-instruction boundary. Neither return construction is selected.

### Shared first-check failure

If S plus `0x338` is null on its first shared test, the invalid-instruction boundary occurs before the six shared saves. Earlier writes remain performed.

### Shared suffix considered in isolation

Conditionally entering the unresolved shared suffix with prior frame reference S must:

- Read destination and payload at S plus `0x33C` and `0x340`.
- Use A equal to S minus `0x18`.
- Read the tail word at S plus `0x320`.
- Save address S plus `0x344`, not zero, as the third value.
- Rebase to S minus `0x30`.
- Perform both destination-relative stores at `0x06` and `0x26`.
- Write the tail word at S plus `0x308`.
- Reread S minus `0x30` for the return high half.

No selector read or destination-relative `0x0E` store belongs to this suffix. This example does not establish entry-to-suffix reachability.

### Aliased return word

If either common-sequence narrow store overlaps N, the return high half comes from the post-store word at N. Returning an earlier saved destination instead would not preserve the represented behavior.

### Global replacement

If the global route supplies destination `0x2000` and payload `0x1234ABCD`, and the subsequent shared address is non-null, the narrow store targets `0x201E` with value `0xABCD`. The second global is saved without another shift.

### Nonzero-selector null address

Conditionally reaching the nonzero-selector branch with a null computed address selects the `0xB9` sequence. A nonzero selector alone must not suppress it. The final high-half read follows the destination-relative `0x2E` store.

## Uncertainties

The following remain `documented_only`:

- Actual function boundary, calling convention, and incoming machine locations.
- Fidelity of the represented frame model.
- Executable reachability of contradictory-test suffixes, selector routes, global replacement, and returns.
- Application meanings of fields, globals, and return constants.
- Exact shift, address-overflow, alignment, and byte-order behavior.
- Permitted aliasing, storage extents, access permissions, and concurrency.
- Runtime fault handling and behavior beyond invalid-instruction boundaries.

Warnings about invalid instruction data and removed unreachable regions limit control-flow confidence. They do not justify inventing missing transitions or a corrected operation.

## Evidence

**Established static evidence:** entry access order and widths; repeated address tests; frame and destination offsets; ordered saves; distinct zero-selector and shared-suffix inputs to the common return sequence; the suffix’s second rebase and both narrow stores; the local `0xB9` branch; ordered global replacements; two 64-bit return layouts; invalid-instruction boundaries.

**Inferred intent:** possible state manipulation within a larger operation. No application-level interpretation is adopted.

**Unknown:** actual executable reachability, accuracy of the frame representation, and runtime behavior outside the represented boundaries.

This specification is based only on the supplied static representation. No implementation access, runtime execution, or tests have been performed.
