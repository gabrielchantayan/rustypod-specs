<!-- Trusted provenance: F-bb492cbc84fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bb492cbc84fb

## Purpose

F-bb492cbc84fb selects a resource-processing layout using a signed 32-bit selector. Recognized layouts cause calls on payload fields, traversal of records or linked nodes, and, for non-chain layouts, direct allocation-accounting updates followed by allocator-related calls.

**Established behavior:** selector membership, accessed field widths and offsets, dependency-call order, pointer capture versus rereading, and direct accounting writes.

**Inferred intent:** destruction of heterogeneous objects and associated resources. Neither the function body nor the available dependency identities establish ownership transfer, successful deallocation, or preservation of state by callees.

“Cleanup” and “outer release” below name processing sequences, not guaranteed memory-release effects.

## Interface

### Arguments and result

- **Selector:** signed 32-bit integer.
- **Descriptor:** pointer to storage containing the selected payload pointer or chain head. The target representation uses 32-bit pointers.
- **Result:** no return value and no explicit success or failure status.

Most recognized selectors read the pointer at descriptor byte offset `0`. Selector `0xE9` instead reads the pointer at offset `4`. Unrecognized selectors return without accessing the descriptor.

A recognized selector requires a valid descriptor for its selected read; the descriptor itself is not null-checked. A zero pointer obtained from that read causes an immediate return without dependency calls or accounting writes.

### Storage and arithmetic requirements

Offsets in this contract are byte offsets. Fields are 32 bits unless another width is specified.

Nonzero payloads, arrays, and nodes must remain accessible for every reached read, including reads after dependency calls. Outer release additionally accesses allocation metadata before the payload and accounting-context storage.

There are no visible general checks for bounds, ownership, allocation compatibility, aliasing, chain cycles, or dependency-induced invalidation.

Counter results are described modulo \(2^{32}\). Address arithmetic follows the supplied 32-bit representation; this does not establish portable behavior for invalid pointers, signed address overflow, or overflowing traversal expressions. Valid ranges and environmental guarantees are documented_only.

## Behavior

### Dispatch

| Selectors | Selected behavior |
|---|---|
| `0x9B`, `0xBD`, `0xCE` | Preliminary payload call, then outer release |
| `0xBC`, `0xC1`, `0xC9`, `0xCA` | Counted 48-byte inline records |
| `0xAE`, `0xB6`, `0xC0`, `0xC3`, `0xC5`, `0xC7`, `0xD1`, `0xD3`, `0xD4`, `0xD7`, `0xD8`, `0xDE` | Counted 12-byte external records |
| `0xCD`, `0xD0`, `0xD6` | Counted 8-byte external entries, descriptor offset `0` |
| `0xE9` | Counted 8-byte external entries, descriptor offset `4` |
| `0xA9`, `0xAA`, `0xC2`, `0xC4`, `0xCC`, `0xD2`, `0xDA`, `0xDD`, `0xDF`, `0xEB`, `0xEE` | Fixed resource aggregate |
| `0xE7`, `0xEC` | Linked chain |
| Every other signed 32-bit value | Return without descriptor access or dependency calls |

Each non-chain branch captures its selected payload pointer. Changes to the descriptor after capture do not redirect subsequent payload processing. If the payload is nonzero and preceding operations complete normally, the branch reaches outer release.

### Preliminary payload call

For `0x9B`, `0xBD`, and `0xCE`, pass the nonzero payload from descriptor offset `0` to F-8d9399c13232. Outer release then uses that same captured pointer, without rereading the descriptor.

### Counted 48-byte inline records

The first payload word supplies a count by interpretation of its low 16 bits as a signed 16-bit integer. Records begin at payload offset `8` and have a 48-byte stride.

Iteration starts at index zero. Every loop condition rereads the count. A nonpositive count skips record processing.

Each visited record produces these calls in order:

| Record offset | Dependency |
|---|---|
| `0` | F-94cb53877c15 |
| `4` | F-94cb53877c15 |
| `8` | F-94cb53877c15 |
| `12` | F-da1a289a3277 |
| `16` | F-6c7ac9769d41 |
| `28` | F-841dc33ee157 |
| `32` | F-11bd3a21366b |

Each argument is loaded when its call is reached, not captured with the whole record. Earlier calls can therefore affect later argument reads and the next count check.

After the seven calls, the cursor advances by 48 bytes. Outer release follows the loop. No separate call targets the inline record region, and other record fields are not directly read by this traversal.

### Counted 12-byte external records

The payload contains:

| Payload offset | Field |
|---|---|
| `0` | Signed 32-bit count |
| `12` | External record-array pointer |

The array pointer is captured before the first count check, even if the count is nonpositive. Iteration starts at zero and rereads the signed count on every condition.

For each visited record:

- Its offset-`0` word is passed to F-841dc33ee157.
- After that call returns, its offset-`4` word is read and passed to F-94cb53877c15.
- The captured cursor advances by 12 bytes.

Record offset `8` is not directly read.

After iteration, payload offset `12` is reread and its current value is passed to F-94cb53877c15. Outer release follows.

Changing the array-pointer field during a dependency does not redirect the captured traversal cursor, but can change the post-loop argument. A nonpositive count skips record calls, not the initial array-pointer read, final array call, or outer release.

### Counted 8-byte external entries

The payload contains:

| Payload offset | Field |
|---|---|
| `0` | External entry-array pointer |
| `4` | Signed 32-bit count |

The descriptor slot is offset `0` for `0xCD`, `0xD0`, and `0xD6`, but offset `4` for `0xE9`. The latter does not read descriptor offset `0`.

Iteration starts at zero. Every loop condition rereads the count. For each visited index, the array pointer is reread, the entry is located using an 8-byte stride, and its first word is passed to F-94cb53877c15. The second word is not directly read.

After iteration, the array pointer is reread again and passed to F-94cb53877c15. Outer release follows.

Unlike the 12-byte layout, an array-pointer change can redirect subsequent entry accesses. A nonpositive count still reaches the final array call and outer release.

### Fixed resource aggregate

The nonzero payload is processed in this order:

| Order | Access and call |
|---|---|
| 1 | Read flags at offset `32`; if bit zero is set, read offset `28` and pass it to F-94cb53877c15 |
| 2 | Read flags at offset `24`; if bit zero is set, read offset `20` and pass it to F-94cb53877c15 |
| 3 | Read offset `8` and pass it to F-841dc33ee157 |
| 4 | Read offset `12` and pass it to F-841dc33ee157 |
| 5 | Read offset `16` and pass it to F-e5380ae57ea1 |
| 6 | Read offset `56` and pass it to F-6c7ac9769d41 |
| 7 | Perform outer release |

Only bit zero controls each conditional call. When that bit is clear, the associated argument field is not read.

Later flags and arguments are loaded after earlier calls. They are not an initial aggregate snapshot.

### Linked chain

For `0xE7` and `0xEC`, read the head from descriptor offset `0`. A zero head returns immediately.

For each nonzero node, capture its successor from offset `36` before invoking any node dependency. Then process the node in this order:

| Order | Access and call |
|---|---|
| 1 | Read flags at offset `20`; if bit zero is set, read offset `16` and pass it to F-94cb53877c15 |
| 2 | Read offset `24` and pass it to F-841dc33ee157 |
| 3 | Read offset `28` and pass it to F-e5380ae57ea1 |
| 4 | Read offset `12` and pass it to F-6c7ac9769d41 |
| 5 | Read offset `32` and pass it to F-11bd3a21366b |
| 6 | Pass the node pointer itself to F-94cb53877c15 |
| 7 | Continue using the captured successor |

Other than the successor, arguments are read as their calls are reached. A later change to the current node’s link does not replace the captured successor.

This branch performs no inline outer-release sequence and does not directly clear the descriptor’s head. Repeated nodes and cycles are not detected. Capturing a successor does not establish that its storage survives intervening calls.

### Outer release: initial accounting

Outer release uses the captured payload. Its accounting-context accesses are distinct global reads, not a demonstrated stable-context snapshot.

The visible order is:

1. Capture the current global accounting-context pointer for the eventual high-word destination.
2. Check the payload for zero and return if zero. Ordinary dispatch paths already exclude this case.
3. Read the 32-bit displacement at payload offset `−4`; subtract it from the payload address to obtain the allocation reference.
4. Read the 32-bit accounting amount at allocation-reference offset `−8`.
5. Reread the global accounting-context pointer and load its old low word at offset `56`.
6. Reread that global pointer and form the high-word source address at offset `60`, without yet loading the high word.
7. Reread the global pointer for the low-word destination and write the low result at offset `56`.
8. Load the high word through the previously formed source address.
9. Write the high result at offset `60` of the context captured in step 1.
10. Invoke F-2fd8826557a4 with no explicit arguments.

The low result subtracts the amount from the loaded old low word, modulo \(2^{32}\).

The high result subtracts both:

- The amount’s arithmetic sign-extension word: zero if its top bit is clear, otherwise `0xFFFFFFFF`.
- A borrow of one exactly when the loaded old low word is less than the amount under unsigned comparison.

The borrow uses the previously loaded low value, not a reread of the destination. The four accounting-context roles may name different storage; equality requires assumptions not established here.

The high-word load occurs after the low-word write. If their addresses alias, that write can affect the high value subsequently loaded.

### Outer release: allocator processing

After F-2fd8826557a4 returns, obtain and capture an allocator-context pointer through a separate global pointer location.

If the allocation reference equals numeric address `8`, return without further allocator processing. This comparison occurs after initial accounting, F-2fd8826557a4, and allocator-context lookup. It does not bypass the earlier accounting-amount read, which in this case addresses numeric zero.

Otherwise, metadata relative to the allocation reference is accessed as follows:

| Relative offset | Field |
|---|---|
| `−16` | 32-bit header |
| `−12` | Unsigned 16-bit accounting index |
| `−10` | Unsigned 16-bit accounting index |

A zero header or a header with bit zero set reaches F-a817d8d58c10, annotated as non-returning. The zero test reads the header first. If nonzero, the bit test reads it again; validation is not necessarily against one snapshot.

On a path passing validation, the order is:

1. Pass the captured allocator context to F-c0fc6d033caa.
2. Read the unsigned index at offset `−12`.
3. Reread the header and derive a size with mask `0x3FFFFFFC`.
4. Subtract that size from context offset `548 + 4 × index`, then offset `780`.
5. Read the unsigned index at offset `−10`.
6. Subtract the same captured size from context offset `224 + 4 × index`, then offset `544`.
7. Invoke F-09dcb55e8e0a without explicit arguments.
8. Interpret its result as a signed 32-bit index and decrement the word at context offset `784 + 4 × result`.
9. Decrement the word at context offset `912`.
10. Reread and mask the header, then subtract this newly obtained size from context offset `8`.
11. Pass the captured context and the header’s address to F-cc6c9b4a0f68.
12. Pass the captured context to F-e586bff64fb7.

These are individual 32-bit counter updates without visible underflow or index-range checks. The size loaded after F-c0fc6d033caa can differ from the validated header’s size. The final size can differ again after F-09dcb55e8e0a.

The allocator-context pointer is not refreshed after these dependencies. Its storage and the header are nevertheless not assumed to be preserved.

## Side effects

Direct writes target the accounting and allocator-context fields described above. The body does not directly clear descriptors, payload fields, records, nodes, or links.

The updates are ordered individual operations, not a demonstrated atomic transaction. Initial accounting writes precede header validation. **No explicit rollback appears in the visible body before the call to F-a817d8d58c10.** Whether that dependency reverses those writes or otherwise changes accounting state is unknown. Its non-returning annotation does not establish preservation of prior writes.

Dependencies may modify globals or referenced storage, release memory, fail, or exit nonlocally. No ownership, preservation, or release contract is available for them. Nested zero values are passed without local null filtering whenever their calls are selected.

Aliasing can cause a write in one described role to affect a later read in another role. The body does not enforce disjointness.

## Dependencies

| Opaque ID | Observable role |
|---|---|
| F-8d9399c13232 | Preliminary call on the payload |
| F-94cb53877c15 | Calls on selected fields, external arrays, and chain nodes |
| F-841dc33ee157 | Calls on selected record, aggregate, and node fields |
| F-da1a289a3277 | Call on one inline-record field |
| F-6c7ac9769d41 | Calls on selected record, aggregate, and node fields |
| F-11bd3a21366b | Calls on selected record and node fields |
| F-e5380ae57ea1 | Calls on aggregate and node fields |
| F-2fd8826557a4 | No-argument call after initial accounting |
| F-a817d8d58c10 | No-argument invalid-header call, annotated non-returning |
| F-c0fc6d033caa | Allocator-context call before allocator counter updates |
| F-09dcb55e8e0a | No-argument call returning a signed 32-bit accounting index |
| F-cc6c9b4a0f68 | Receives allocator context and header address |
| F-e586bff64fb7 | Final allocator-context call |

No synchronization semantics or memory-ownership guarantees are assigned to these dependencies.

## Acceptance examples

These examples are conditional static deductions, not executed tests. Unless stated otherwise, reached storage is valid and dependencies return normally.

- **Ignored selector:** `0xC6` returns without reading a null descriptor.
- **Zero payload:** `0xC2` with zero at descriptor offset `0` causes no dependency calls or accounting writes.
- **Alternate descriptor slot:** `0xE9` selects its payload solely from descriptor offset `4`.
- **Signed narrow count:** A 48-byte-layout count word with low half `0xFFFF` skips record calls but reaches outer release.
- **One inline record:** A stable count of one produces the seven specified record calls, in order, before outer release.
- **Empty external records:** A zero 12-byte-layout count still causes the initial array-pointer read and the post-loop call using the then-current array pointer.
- **Mutable array pointer:** In the 12-byte layout, replacing the array field does not redirect the captured cursor but changes the final array argument. In the 8-byte layout, replacement can redirect later entry accesses.
- **Conditional fields:** Flags `2` skip the associated argument read and call; flags `3` select both.
- **Captured successor:** Changing a node’s link after capture does not change the successor used for continuation.
- **Stable-context borrow:** With coincident, nonaliasing accounting contexts, amount one and old low zero produce low `0xFFFFFFFF` and subtract one from the late-loaded high word.
- **Signed amount:** With the same context assumptions, amount `0xFFFFFFFF` and old low zero produce low one and leave the high value unchanged because sign-extension and borrow cancel modulo \(2^{32}\).
- **Distinct contexts:** Let the high destination, low source, high source, and low destination name disjoint contexts A, B, C, and D respectively. Amount one, B’s low value zero, and C’s high value seven cause writes of `0xFFFFFFFF` to D’s low field and six to A’s high field.
- **Late high load:** If the low destination aliases the high-source address, the high calculation uses the value present after the low write.
- **Invalid header:** A zero header reaches F-a817d8d58c10 after initial accounting and F-2fd8826557a4, before F-c0fc6d033caa. The body performs no explicit intervening rollback. Accounting state after entry into F-a817d8d58c10 is not specified because that dependency’s effects are unavailable.

## Uncertainties

**Inferred intent:** selector-specific destruction, nested-resource cleanup, and allocator bookkeeping.

**documented_only:**

- Dependency ownership, null tolerance, deallocation behavior, state preservation, synchronization, and failure mechanisms.
- Whether F-a817d8d58c10 reverses prior accounting writes, changes other state, or has additional non-returning effects.
- Allocation provenance and the purpose of the numeric-address-eight case.
- Concurrency and reentrancy rules, including whether repeated global-context reads agree.
- Whether a particular execution environment permits mutation between the described accesses.
- Valid count and index ranges, especially the result of F-09dcb55e8e0a.
- Behavior for inaccessible memory, overflowing address calculations, cyclic chains, or an unexpected return from the dependency annotated non-returning.
- Application invariants preventing dependencies or accounting writes from invalidating later accesses.

## Evidence

The supplied function representation establishes the dispatch groups, access widths and offsets, dependency-call sequence, loop conditions, captured versus reread pointers, direct writes, and the invalid-header call’s non-returning annotation.

Initial accounting separately obtains its destination and source contexts. It writes the low result before loading the high source; it does not establish a shared-context two-word snapshot.

The visible body places initial accounting writes before header validation and contains no explicit rollback before F-a817d8d58c10. The dependency identity and non-returning annotation provide no evidence about that dependency’s accounting effects.

Dependency implementations are unavailable. No runtime tests have been performed. Acceptance examples describe conditional consequences of static evidence, not observed executions.
