<!-- Trusted provenance: F-c3c29167daee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3c29167daee

## Purpose

F-c3c29167daee traverses an indexed primary-record collection and issues reporting calls and software interrupts. Each primary record’s status controls additional association scans, linked-list membership reports, and optional reporting of two byte-sized values.

A diagnostic listing with periodic cancellation is the inferred intent. The established behavior is the sequence of memory reads, dependency calls, interrupts, and traversal decisions described below. Output destination, application meanings, and interrupt semantics are not established.

This contract describes static evidence only. Dependencies are not assumed to preserve globals, leave supplied memory unchanged, or manage returned values in any particular way.

## Interface

**Explicit inputs:** None.

**Explicit output:** None. Normal completion returns without a value.

**Implicit inputs:**

- A pointer to a signed 32-bit inclusive upper bound for primary indices.
- Global bases for the primary table, association table, three object collections, and name storage.
- Pointers to signed 32-bit association and collection counts.
- Record fields, linked-list heads, and linked-list nodes.
- Dependency results and any state changes caused by dependencies or interrupts.

The supplied representation uses four-byte integers and pointer-bearing values. Status tests operate on a 32-bit word. Header values and forwarded helper results occupy four bytes, without an established semantic type. Trailing record values occupy one byte each and are promoted to unsigned 32-bit values.

### Observed layouts

Offsets are in bytes from the selected record or node address.

| Structure | Record stride | Relevant fields |
|---|---:|---|
| Primary table | 52 | Status at +28; four-byte header values at +36 and +48; bytes at +40 and +41 |
| Association table | 4 | Signed 32-bit primary index |
| Collection A | 52 | Four-byte discriminator at +16; list head at +20; association fields at +28, +32, +36, +40 |
| Collection B | 20 | List head at +4; association field at +12 |
| Collection C | 32 | List head at +8 |
| List node | Not established | Four-byte next pointer at +0; four-byte primary index at +32 |

Names are addressed using a name-storage base plus 17 times the corresponding collection index. This establishes address spacing, not string encoding, termination, readable length, or allocation boundaries.

### Preconditions and arithmetic

Every location actually accessed must be valid at the time of access. Dependency arguments must satisfy the dependencies’ unknown external requirements.

Primary indices start at zero. Association and collection indices start at one. Their bounds and comparisons are signed 32-bit operations in the supplied representation. Address calculations use the listed strides and four-byte address-sized values.

No range, overflow, alignment, list-cycle, or memory-validity checks are visible. The contract does not assume either safe wrapping or rejection of overflowing arithmetic. Endianness, target arithmetic behavior outside the representable range, and invalid-access consequences remain documented_only.

## Behavior

### Entry, bounds, and periodic return

First, the Collection A name-storage base is captured. That value remains in use throughout this invocation.

Two software interrupts follow. Every visible software interrupt in this function uses selector `0x123456`; other operands and effects are unknown.

The primary index and periodic-check counter are initialized to zero. At the start of each iteration, the current primary-bound pointer is obtained and its signed value read. If the bound is less than the current primary index, the function returns before any further work for that iteration.

Thus, under stable state:

- A negative bound allows only the two entry interrupts.
- A bound of zero permits primary record zero.
- A bound of \(n \geq 0\) permits indices zero through \(n\), inclusive.

After the bound check, a periodic checkpoint occurs if its counter is greater than 21. The counter is reset to one before calling F-c7198d86625a. Results `0x58` and `0x78` cause immediate return; other results continue.

With normally returning dependencies and an unchanged primary index, the first checkpoint occurs after 22 completed records, before record 22’s header. Further checkpoints occur after each additional 21 completed records. If the bound check already requires return, no checkpoint occurs.

### Primary header and initial status reads

Before preparing the header, the global primary-table base is read **twice, in order**:

1. The first value is retained for the first two status tests.
2. The second value supplies the selected record address for copying the header fields.

These are separate reads. Their results need not agree if state changes between them.

Using the second base, the function reads four bytes at +36, then four bytes at +48, into local storage. It calls F-a3fdaba172fb, then calls F-88996a971ccf with a header-format reference and:

- The address of the primary-index local.
- The four-byte result of F-a3fdaba172fb.
- Addresses of the two copied header values, in copy order.

The formatter receives the index by address. Its mutation or retention of that pointer is not excluded. Subsequent record accesses, comparisons, helper arguments, and the final increment use the index current at their respective evaluation points. There is no new bounds check immediately after such a mutation.

Writes through the two header-copy addresses do not, through those addresses alone, modify the source record.

### Status-test order and primary-base selection

The categories are evaluated in this order:

1. Bit `0x01`.
2. Bit `0x100`.
3. Bit `0x40`.
4. Bit `0x80`.
5. Bit `0x20`.
6. Bit `0x04`.
7. Bit `0x08`.
8. Whole-word equality to `0x10`.
9. Whole-word equality to zero.

Each test reads status separately. These are not mutually exclusive alternatives or a single status snapshot.

The primary-base rules are:

- The `0x01` and `0x100` tests use the first pre-header base capture, independently calculating the selected status address with the then-current primary index.
- The `0x40` test obtains a fresh global primary base.
- After `0x40` handling, the Collection A count pointer is captured, then a primary base is captured for later use.
- The `0x80` test obtains **another** global primary base, distinct from that retained capture.
- The retained capture preceding the `0x80` test supplies all subsequent status tests and trailing-byte handling. Its selected record address can nevertheless change if the primary index changes.

Bits `0x01` and `0x100` each cause one software interrupt when set.

### Bit `0x40`: indexed associations

If `0x40` is set, one software interrupt occurs before scanning.

Association indices run from one through the current signed association count. On **each loop-condition evaluation**, the association-count pointer is obtained anew and its pointed-to value read. For each eligible entry, the association-table base is refreshed before reading that entry.

An entry matches when its 32-bit value equals the current primary index. For each match, in ascending association-index order:

1. F-e34704c65325 receives the association index.
2. Its four-byte result is passed to F-88996a971ccf with a short name-oriented format reference.
3. F-bfc162dd4e46 receives selector 6, the primary index current at this call site, and the association index.
4. F-bfc162dd4e46 receives selector 1, the primary index current at this later call site, and the same association index.
5. Collection A is scanned.
6. Collection B is scanned.

The two helper calls do not necessarily receive equal primary indices. In particular, selector 6 may cause a change through an index pointer previously retained by the header formatter. The caller reads the index again for selector 1.

#### Collection A association scan

After selector 1 returns, the Collection A count pointer is captured for this scan. Its pointer value remains fixed; its pointed-to signed count is reread on every condition check.

Before **every** Collection A loop-condition comparison, including the terminating comparison, these Collection B globals are captured in order:

1. Count pointer.
2. Name-storage base.
3. Table base.

These captures precede reading the Collection A count for that condition.

For each eligible Collection A index, the Collection A table base is refreshed and the selected 52-byte record address computed. That selected address is retained across all four field comparisons and any intervening formatter calls.

Fields are compared with the association index in this order:

1. +36.
2. +28.
3. +32.
4. +40.

Each successful comparison independently calls F-88996a971ccf with a category-specific format reference and the Collection A name address. All names use the function-entry name-base capture.

There is no first-match exit or deduplication. A formatter can modify later fields at the retained record address, affecting later comparisons. Replacing the global Collection A table base does not redirect the remaining comparisons for that record; it can affect a later record.

#### Collection B association scan

The Collection B count pointer, name base, and table base captured during the **terminating Collection A condition check** supply the entire following Collection B scan. This applies even when Collection A executes no record body.

Collection B indices run from one through the signed count read through that retained count pointer. Only the pointed-to count is reread on subsequent conditions; the three captured pointer/base values are not refreshed during this scan.

Each record’s +12 field is compared with the association index. Every match invokes F-88996a971ccf with the corresponding name address and a distinct category format.

### Bit `0x80`: Collection A list membership

As described above, the Collection A count pointer is captured before the retained primary base and before the separately based `0x80` test. This count pointer remains fixed throughout the scan; its pointed-to count is reread for every outer-loop condition.

For each eligible Collection A index:

- Refresh the Collection A table base and retain the selected record address.
- Compute the name address using the entry-captured Collection A name base.
- Read the list head at +20.
- Traverse until a null node pointer.

Each node’s +32 value is compared with the current primary index. For each match:

1. Read the containing record’s discriminator at +16 using the retained record address.
2. Issue one software interrupt.
3. Call F-88996a971ccf with the computed name address.
4. Call F-bfc162dd4e46 with selector 3, the then-current primary index, and the Collection A index.

Both discriminator branches visibly issue the same interrupt selector. Their complete machine-visible effects cannot be established as identical.

Every matching node is reported, including duplicates. The next pointer at +0 is read after the match-related calls. Changes to that link during those calls therefore affect continuation. The head and selected containing-record address are not refreshed for every node.

### Bit `0x20`: Collection B list membership

Before testing `0x20`, and before its possible interrupt, the function captures:

1. The Collection B count pointer.
2. The Collection B name-storage base.

These captures occur regardless of whether the bit is set. The status test uses the retained primary base.

If set, one software interrupt precedes the scan. The captured count pointer and name base remain fixed throughout it, while the pointed-to count is reread on every outer condition. The Collection B table base is refreshed per eligible record.

For each record, compute its name address, read its list head at +4, and traverse until null. Each node matching the current primary index at +32 causes:

1. A name-oriented F-88996a971ccf call.
2. An F-bfc162dd4e46 call with selector 5, the primary index current after that formatting call, and the Collection B index.

Every match is reported. The next link is read after these calls.

### Bit `0x04`: direct report

If `0x04` is set, issue one software interrupt, then call F-bfc162dd4e46 with selector 0 and the current primary index.

Only those two arguments are explicitly supplied. No third argument or meaningful third-argument value is established.

### Bit `0x08`: Collection C list membership

After `0x04` handling, but before testing `0x08` or issuing its interrupt, capture:

1. The Collection C count pointer.
2. The Collection C name-storage base.

These captures occur even when `0x08` is clear. The test uses the retained primary base.

If set, issue one software interrupt. Scan Collection C from index one through the signed count reread through the retained count pointer. Keep the captured name base fixed and refresh the Collection C table base per record.

For each record, compute its name address and read its list head at +8. Each node whose +32 value matches the current primary index causes a name-oriented F-88996a971ccf call followed by F-bfc162dd4e46 with selector 4, the then-current primary index, and the Collection C index.

Duplicate matches are reported independently. Next links are read after match-related calls.

### Exact status tests and trailing bytes

A whole status word equal to `0x10` causes one software interrupt. A separate subsequent read equal to zero causes another.

For stable status and index, these conditions are mutually exclusive. An intervening interrupt may change state, so the function does not enforce that exclusivity across the two reads.

Next, the function calculates a primary-record address using the retained primary base and current index, then reads byte +41 as a gate. If nonzero:

1. Read byte +40 using that calculated record address and zero-extend it to 32 bits.
2. Recalculate the +41 address using the retained primary base and current primary index, read that byte, and zero-extend it to 32 bits.
3. Pass addresses of those two local values to F-88996a971ccf with a two-number, tick-oriented format reference.

The gate read and value read at +41 are distinct. Each promoted value is between zero and 255; a later +41 value of zero is not excluded merely because the gate read was nonzero.

Finally, issue one software interrupt, increment the periodic counter, increment the then-current primary index, and return to the initial bound check.

## Side effects

The visible body directly writes only local counters and temporary storage. It does not directly update global tables, association entries, or list nodes. Nevertheless, dependencies and interrupts may modify memory or external state.

Observable effects include ordered dependency calls, interrupts, and the memory accesses described above. Capture boundaries are part of the contract:

- Replacing a captured count pointer does not redirect an active scan, but changing the count at that pointer can affect its next condition.
- Replacing an association-count pointer can affect the next association condition because that pointer is refreshed.
- Replacing a per-record-refreshed table base can affect a later record, not an already retained record address.
- Replacing Collection B globals during its association scan does not replace that scan’s terminating-check captures.
- Changing captured name-storage contents can affect a later consumer even when replacing the global name base would not.

No allocation, deallocation, ownership transfer, or explicit cleanup appears in the visible body. Four-byte helper results are forwarded without a visible release. This establishes neither ownership nor an absence of allocation inside dependencies.

Local addresses supplied to F-88996a971ccf are not proven read-only or non-retainable. Their validity after function return is not established.

There is no explicit error result, failure translation, or local recovery path. Cycles, inaccessible memory, dependency failures, or nonreturning interrupts can prevent normal completion.

## Dependencies

| Opaque ID or facility | Established call-site behavior |
|---|---|
| F-c7198d86625a | No explicit arguments; checkpoint results `0x58` and `0x78` cause return. |
| F-a3fdaba172fb | No explicit arguments; four-byte result forwarded to the primary-header formatter. |
| F-88996a971ccf | Receives format references and varying arguments: local addresses, helper results, and computed name addresses. Output, mutation, retention, and failure behavior are unknown. |
| F-e34704c65325 | Receives a matched association index; four-byte result forwarded to a formatter. |
| F-bfc162dd4e46 | Receives selectors 6 and 1 for associations, 3 for A membership, 5 for B membership, 0 for direct reporting, and 4 for C membership. Primary indices are evaluated at each call site. All but selector 0 explicitly include a related-object index. |
| Software-interrupt facility | Every visible invocation uses selector `0x123456`; complete operands, results, and effects are unknown. |

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, dependencies return normally without mutation, arithmetic stays representable, and referenced memory is valid.

1. **Negative bound:** The function captures the A name base, issues two entry interrupts, then returns without a header or checkpoint call.

2. **Single zero-status record:** With bound zero and byte +41 zero, record zero receives its header, the exact-zero interrupt, and the final iteration interrupt. No membership helper is called.

3. **Distinct initial bases:** If the first pre-header primary-base read yields one table and the second yields another, header copies come from the second table. The first two status tests use the first table.

4. **Combined categories:** Stable status `0x05` causes the `0x01` interrupt and later the `0x04` interrupt followed by selector 0 reporting. It satisfies neither exact status test.

5. **Four association matches:** An A record matching one association at all four fields produces four formatter calls, ordered +36, +28, +32, +40.

6. **Mutation between association helpers:** If selector 6 changes the retained primary-index local from 2 to 9 through a previously retained pointer, selector 1 receives primary index 9. The association index remains unchanged.

7. **Empty A association scan:** Even when A’s first condition fails, the B count pointer, name base, and table base are captured in that order before the failed comparison. Those captures drive the subsequent B scan.

8. **A record retained across callbacks:** If the first association formatter replaces A’s global table base, the remaining three field comparisons still access the selected old record. A later A record uses a refreshed table base.

9. **Pre-interrupt B captures:** If the `0x20` interrupt replaces B’s count pointer and name base, the current scan retains their pre-interrupt values. A replacement table base can affect the first record because that base is refreshed afterward.

10. **Duplicate list members:** Two matching nodes in one A list cause two name reports and two selector 3 calls. No deduplication occurs.

11. **Unsigned trailing values:** Bytes +40 and +41 equal to 255 and 128 produce local 32-bit values 255 and 128 for formatting, not negative values.

12. **Checkpoint cadence:** With enough eligible records, the first checkpoint follows 22 completed records. A stopping result prevents record 22’s header; a non-stopping result permits it, with the next checkpoint after 21 further records.

13. **Equality versus bit inclusion:** Stable status `0x10` causes the exact-`0x10` interrupt. Stable `0x11` causes the `0x01` category but not the exact-`0x10` category.

## Uncertainties

**Inferred intent:** Diagnostic listing, character-based cancellation, named relationships, and tick-related trailing values are plausible interpretations, not established application semantics.

**documented_only unknowns:**

- Full interrupt interface and effects.
- Exact rendered text, format interpretation, output destination, and output errors.
- Application meanings of status bits, fields, and selectors.
- Semantic types and ownership of four-byte helper results.
- Dependency allocation, release, global mutation, and local-pointer retention.
- Synchronization, concurrent mutation, and memory-visibility guarantees.
- Target-specific overflow, address wrapping, alignment, and fault behavior.

The recorded read order does not itself guarantee coherent snapshots or safe concurrent access. Cyclic lists, changing counts, altered indices, and unchecked arithmetic can change or prevent termination.

## Evidence

The supplied static body establishes the layouts, widths, loop bounds, comparisons, capture and refresh boundaries, explicit arguments, local-address passing, and ordering described here.

No dependency implementations, runtime observations, external format contract, or execution-environment specification were supplied. No runtime tests have been performed. Acceptance examples are conditional consequences of the static control flow, not observed results.
