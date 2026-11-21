<!-- Trusted provenance: F-8c8dbe2fbaa7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c8dbe2fbaa7

## Purpose

F-8c8dbe2fbaa7 performs an ordered sequence of indirect calls, computed-address memory accesses, and shared-storage mutations. It accepts two explicit 32-bit inputs, initially publishes the first at an address selected by the second, and subsequently propagates dependency-produced values through word slots, byte-selected destinations, and shared storage. Its last represented operation is a call to address zero with three zero arguments.

No application-level purpose is established. Allocation, initialization, dispatch, serialization, and cleanup are not established interpretations.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, arithmetic, argument relationships, and ordering.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved execution requirements or behavior.

All requirements for later operations are conditional on preceding accesses completing and dependencies returning normally. Earlier effects are not conditional on eventual completion.

## Interface

### Explicit inputs and output

| Component | Width | Treatment |
|---|---:|---|
| Payload | 32 bits | Initially stored unchanged and supplied in two positions of the first call. |
| Initial selector | 32 bits, represented as signed | Selects the initial destination through multiplication by four and addition of `0x14`. |
| Result | No specified value | A return is represented only after the terminal address-zero call. |

The initial destination is the absolute address four times the initial selector plus `0x14`. No separate array base or selector validation is established.

### Implicit inputs

The invocation also depends on incoming continuation-related machine state, two incoming memory bases, ordinary and secondary call results, and memory contents.

Two shared 32-bit storage views are called **shared slot A** and **shared slot B**. A separate three-byte storage view contributes to the final publication to B. These names do not establish disjoint storage: overlapping views are explicitly possible.

Some calls produce represented 64-bit results whose low and high 32-bit halves are consumed separately. Whether these are actual 64-bit ABI results or paired machine-register outputs is documented_only.

### Widths, addresses, and layout

Direct accesses are 8, 16, or 32 bits. Byte-selector reads are unsigned. Byte publications retain only the low eight bits of the represented value.

A **word slot** denotes a 32-bit access at four times a selector plus `0x14`. Other recurring addresses include:

| Address relationship | Use |
|---|---|
| Sixteen times a selector plus `0x14` | 32-bit publication, frequently of four times that selector |
| Sixteen times a selector plus `0x24` or `0x74` | Separate field write and field read |
| Base plus `0x74` | 32-bit field read |
| Base plus one | Byte read or write |
| Base plus `0x9`, `0xD`, `0xE`, or `0x15` | Byte field |
| Base plus `0x24` or `0x34` | 32-bit field |

These relationships do not establish object types, allocated extents, or ownership.

A **byte-selected destination** is the numeric value of a freshly read unsigned byte plus `0x11`. Its range is `0x11` through `0x110`, inclusive. It is not relative to the selector’s source address and does not involve a pointer-width selector load.

Logical right shifts of represented unsigned values use 32-bit widths. Signed-looking scaling, address overflow, and signed left-shift semantics remain documented_only; this contract does not impose portable-language or modular-overflow behavior without target evidence.

### Preconditions

Reaching the end requires valid executed memory accesses, compatible call targets and argument conventions, usable dependency results, and suitable implicit machine state. Alignment, permissions, and special-memory behavior must permit the operations.

No visible checks enforce these requirements.

## Behavior

### 1. Initial publication

The first memory operation stores the 32-bit payload in the initial selector’s word slot.

After that store, a 32-bit value is read at the first incoming base plus `0x14`, logically shifted right by 19, and supplied to F-unresolved-001. The four represented arguments are the payload, that extract, zero, and the payload again.

Further indirect calls follow. A later ordinary result selects another word slot receiving the unchanged payload. The following call receives four times that selector. These are distinct payload stores, separated by calls.

### 2. Early byte publications and split results

A later 32-bit result supplies a left-by-two value and three extracts: its low 30 bits shifted right by 17, 20, and seven.

The selector byte at the second incoming base plus one is read, and the low byte of the left-by-two value is written to its selected destination. After intervening calls, the selector byte is read again for publication of the low byte of the right-by-20 extract.

The second selector read is mandatory. Calls or overlapping writes can change its value.

Later calls consume scaled results, repeated arguments, the other masked extracts, and a continuation-like numeric argument. A split result supplies a new low-half working value and a high half passed onward.

Two zero-byte publications use separate reads of the selector at the earlier right-by-seven extract plus one, with calls between them. Another zero-byte publication uses a selector at the working value shifted right by nine, plus one.

### 3. Byte, word, halfword, and exposed-storage reads

The next region includes these data accesses, in this order, separated by call sequences and further selector-based clears:

1. An unsigned byte at twice a current working value; its numeric value is supplied four times.
2. A 32-bit word at a returned base plus an offset formed by shifting that base right by 19 and then left by 31.
3. An 8-bit field at a secondary output plus `0xE`; that field is supplied four times.
4. An ordinary result supplied four times to another dependency.
5. A stack-storage address supplied in four argument positions.
6. An unsigned byte at a returned base plus a secondary output.
7. A 16-bit value at that byte’s numeric value plus a different secondary output shifted right by 19; the halfword is supplied four times.

The high-bit offset in item 2 is a represented 32-bit shift, not an unbounded mathematical multiplication.

Intervening byte clears use fresh selectors derived from working values, secondary outputs, scaled values, and the byte at absolute address one.

The exposed stack address has no established referent size, initialization guarantee, or retention permission. The halfword’s wider ABI representation is documented_only; no explicit byte swap or signed extension is established.

### 4. Word-slot and byte-field publications

A subsequent split result supplies a retained base and a high-half selector. An earlier right-by-19 extract is stored in the high-half-selected word slot. After calls, the same extract is stored in another dependency-selected word slot.

At the retained base plus `0xD`, the function first writes the low byte of twice that base. After intervening calls, it writes four to the same byte field.

Between those stores, a 32-bit read at four times another result plus `0x44` supplies a call argument. Following the second byte store, a different word read uses that result’s low-30-bit, right-by-20 extract plus `0x44`. The second read is not four-byte indexed.

Further publications include:

- Four times an earlier result at 256 times a returned selector plus `0x14`.
- A low-26-bit, right-by-13 extract in a word slot.
- Repeated sixteen-scaled publications containing four times their selectors.
- Intervening word-slot stores carrying retained scaled values.

These establish individual mutations, not complete-object copies.

### 5. Linked fields and repeated byte consumption

A sixteen-scaled location has its `0x24` field cleared before its separate `0x74` field is read as a new base.

After calls, another result shifted right by 20 is stored at the new base plus `0x24`. A separate 32-bit read at that base plus `0x74` supplies a dependency argument.

A masked extract is published through the selector at the new base plus one. After calls, the selector is read again and its selected destination cleared.

Two unsigned-byte reads at distinct secondary outputs plus `0x15` feed subsequent calls. Additional clears repeatedly read selectors at 1,024 times the earlier right-by-20 extract plus one.

The right-by-20 extract is at most `0xFFF`; shifting it right by 21 therefore produces zero. Calls receiving that further extract receive zero under the represented widths.

A third byte read at another secondary output plus `0x15` is zero-extended into a word-slot publication. Later stores publish its selector and other scaled values.

### 6. Masked extracts and shared-slot clearing

A secondary result is masked to 29 bits and shifted right by 18. Its left-by-13 value is passed onward and later published in a word slot.

A later result shifted left by two supplies a byte publication. Its masked right-by-20 extract is subsequently published through another selector. Further selector-based clears follow.

This region also scales a secondary result by eight and then by additional factors of four; publishes a dependency-produced value and its selector in separate word slots; performs sixteen-scaled publications; and reads a word slot before passing its right-by-21 extract to a call.

Shared slot A is explicitly cleared as a 32-bit storage view. Further calls, word-slot clears, and scaled publications follow.

The clear is a historical event, not a lasting invariant. Dependencies and overlapping stores may change A before either later read.

### 7. Retained zeros and low-address reads

Retained working values explicitly set to zero simplify later addresses and arguments. They establish selector reads at absolute address one and zero-valued masked call arguments without requiring any assumption that memory remains unchanged.

One ordinary dependency result supplies the base for an unsigned-byte read at offset one. After calls, that byte’s numeric value becomes the base for another byte read at offset one. The latter address is between one and `0x100`; this is not a pointer-width dereference.

A publication derived by shifting the first unsigned-byte value right by 12 necessarily writes zero. Other publications in this region derive from wider dependency results and must not be assumed zero merely because they also involve right shifts.

An unsigned byte subsequently read at absolute address one is zero-extended into a word slot. Other retained small extracts become zero after additional shifts, including a later word-slot publication.

These simplifications do not remove selector reads, calls, or stores.

### 8. Tail fields, chained reads, and the first snapshot of A

Following further publications, the function writes a call-produced byte shifted left by two through a selector, clears another selected destination, and clears a byte at a scaled call result times `0x2000`, plus nine.

It later writes a retained base’s low byte at that base plus nine, then reads the 32-bit field at base plus `0x34` for a call.

A secondary output selects a word-slot store of the retained base. After that store, a separate 32-bit read at the secondary output plus `0x14` begins a chain involving right-by-22 extraction and additional `0x14` reads. These are distinct from the four-scaled store destination.

After a later word-slot publication of a retained scaled call result, **the function reads shared slot A once and retains its 32-bit value**. This read precedes the following sequence of five dependency calls. Only after those calls does the function shift a previously captured chained-word value right by 22 and publish that extract through a secondary-output-selected word slot.

The value captured from A is shifted right by 23 for a subsequent dependency argument. A further right-by-23 shift consumes that retained extract, not a fresh read of A. The second shift necessarily yields zero because the first extract is at most `0x1FF`.

Changes to A after this snapshot do not affect those retained extracts. They may still affect other memory or subsequent operations.

### 9. Shared-slot B and field-byte refreshes

A later ordinary result is stored as a 32-bit value in shared slot B before being passed onward.

The function reads a 32-bit field at a retained byte-derived base plus `0x74`, then writes its low byte at that base plus one.

Similar refreshes occur at dependency-selected four-scaled bases. One base’s `0x74` field is reread after intervening calls before another byte write; the second refresh cannot reuse the earlier field value.

A later `0x74` field value is captured, truncated into its base-plus-one byte, and subsequently stored unchanged at a sixteen-scaled destination and a word slot.

Further stores publish a scaled result at an aligned address expression plus `0x14`, and zero in another word slot.

### 10. Second snapshot of A and final publication

A split result supplies a low half scaled by four and a high-half selector. The ordering is:

1. Store the scaled low half in the high-half-selected word slot.
2. Read shared slot A once, retaining its 32-bit value as an address component.
3. Invoke the intervening sequence of five dependencies; the first receives the scaled value and high-half selector among its arguments, and the last returns another selector.
4. Store the retained scaled value in that second selector’s word slot.
5. Read a 32-bit word at the **captured A value plus `0x14`**.

There is no reread of A between steps 2 and 5. The word read itself occurs after the second store; the word’s contents are not snapshotted when A is captured.

After further calls, the captured word supplies two selector-based byte publications: first its low byte, and later the low byte of its right-by-22 extract. Those publications reuse the captured word value rather than rereading either A or the addressed word.

The final represented state changes occur in this order:

1. Write a zero byte through a secondary address result associated with the last call.
2. Publish 32 bits to B, combining the final dependency-produced byte in the least significant eight bits with a separate three-byte storage view in the upper 24 bits.
3. Write a 16-bit zero at the retained split-result low half masked to 30 bits and shifted right by seven.
4. Invoke F-unresolved-004 at absolute address zero with three zero arguments.

Normal return requires the terminal call to return.

## Side effects

Direct effects include byte-selected low-address writes, word-slot mutations, byte and word field updates, shared-storage publications, a dependency-addressed byte clear, and the final halfword clear.

Call effects are not bounded by those direct accesses. Dependencies may alter selectors, globals, addressed words, overlapping views, or other memory.

The two reads of A are separate snapshots taken at the points specified in Behavior. Neither uses the earlier explicit clear as a substitute for reading storage. Conversely, later uses of either snapshot must not reread A. Snapshot retention is a data-flow requirement, not a concurrency or atomicity guarantee.

No explicit allocation, release, ownership transfer, error result, rollback, retry, bounds check, or exception translation is established. Earlier writes may remain when a subsequent operation fails.

## Dependencies

Only the subject function has a resolved opaque identity. The following references distinguish unresolved call-site categories, not proven callee identities:

| Reference | Role |
|---|---|
| F-unresolved-001 | Initial target calculated from incoming continuation-related state |
| F-unresolved-002 | Remaining continuation-relative and alternate-offset indirect calls |
| F-unresolved-003 | Separately referenced fixed-target call receiving a repeated ordinary result |
| F-unresolved-004 | Terminal address-zero call |

F-unresolved-002 includes calls with no explicit arguments, repeated arguments, secondary outputs, split results, and exposed stack storage.

Grouping does not permit merging, reordering, or omitting calls. Callee preservation, ownership, retention, failure behavior, and ABI contracts are documented_only.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial store.** Selector three and payload `0x11223344` require a 32-bit store at `0x20` before the first call. Later failure does not erase that event.

2. **Fresh selectors and truncation.** Selector bytes five and seven, read around intervening calls, select `0x16` and `0x18`. Publishing the left-by-two result of `0x12345678` as a byte writes `0xE0`.

3. **Absolute selection.** Selector byte `0xFF` selects `0x110`, independent of the selector’s source address.

4. **First A snapshot.** Suppose A is `0x00800000` at the section 8 read. If the intervening calls change A to zero, the subsequent argument derived from the captured value shifted right by 23 is still one. Its further right-by-23 extract is zero. The read must precede those calls and the subsequent right-by-22 publication.

5. **Second A snapshot.** Suppose A is `0x1000` immediately after the first split-result-selected store. If an intervening dependency changes A to `0x2000`, the later word read still targets `0x1014`, after the second selected-slot store—not `0x2014`.

6. **Delayed pointed-word read.** In example 5, if a dependency changes the word at `0x1014` before the later read, that read consumes the changed word. Capturing A early does not capture its addressed word early.

7. **Overlapping store.** If the first split-result-selected store overlaps A, the immediately following A read observes storage after that completed store. A later overlapping second store does not replace the retained address component.

8. **Terminal order.** The final byte clear precedes B’s 32-bit publication, then the halfword clear, then the zero-address call. A fault or non-return at that call does not establish successful return.

## Uncertainties

- **documented_only — arithmetic:** Exact signed overflow, signed shifts, pointer conversion, and address wrapping require target evidence.
- **documented_only — endianness:** Access widths and final concatenation significance are established; physical byte order is not.
- **documented_only — ABI:** Implicit incoming state, secondary outputs, split results, and narrow argument extension are unresolved.
- **documented_only — storage overlap:** Named shared views and computed destinations are not proven disjoint.
- **documented_only — execution:** Reachability, low-address mappings, alignment tolerance, and fault behavior are unknown.
- **documented_only — dependencies:** No global preservation, ownership, release, or retention guarantees are established.
- **documented_only — terminal call:** Address zero may be executable, trapping, invalid, or non-returning.
- **Inferred intent:** Repeated scaled slots and masked extracts suggest structured state processing, but no domain interpretation is required.

## Evidence

The contract rests on the supplied static representation of access widths, masks, shifts, address relationships, call arguments, retained results, and operation order.

Both A snapshots are established by a shared-storage read before intervening calls and later consumption of the retained value. The second snapshot is specifically between two selected-slot stores; the addressed word is read only after the second store.

No runtime tests have been performed. No execution trace, successful invocation, concrete fault behavior, dependency implementation, or ownership behavior has been observed.
