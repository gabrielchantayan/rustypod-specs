<!-- Trusted provenance: F-22ee9fb2a6fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-22ee9fb2a6fb

## Purpose

F-22ee9fb2a6fb performs an ordered series of mixed-width memory accesses, mutates storage reached through those accesses, and transfers control to an indirectly obtained target with four arguments.

**Established static evidence:** byte, halfword, and word accesses; signed narrow values used as addresses; repeated reads of fixed cells; additive and scaled addressing; and a final indirect invocation.

**Inferred intent:** some reciprocal stores resemble record links. This resemblance does not establish a data structure, valid object layout, or application-level operation.

The ordering below describes the supplied static representation. Whether individual accesses are externally observable depends on execution semantics, aliasing, and memory attributes. No runtime tests have been performed.

## Interface

### Entry values

| Input | Represented width | Use |
|---|---:|---|
| First explicit argument | 32 bits | No explicit use in the visible body. |
| Second explicit argument | 32 bits | Additive address component for an initial byte store and a subsequent signed-byte read. |
| Third explicit argument | 32 bits | Additive address component for an initial halfword store. |
| Fourth explicit argument | 32 bits | Word payload, low-byte payload, and address base for a word read at offset `0x44`. |
| Hidden base | 32-bit address-bearing integer | Initial memory base and stored address-like value. |
| Hidden byte | 8 bits | Initial byte-store payload. |

The hidden values are used without visible initialization. Their provenance and calling-convention locations are **documented_only**.

### Fixed anchor cells

Each anchor is a four-byte cell. Its label identifies storage, not a permanently captured value.

| Anchor | Address | Role |
|---|---|---|
| A01 | `0x08768540` | Initial anchor |
| A02 | `0x087685bc` | Anchor after two narrow-address traversals |
| A03 | `0x0876850c` | First twice-read anchor |
| A04 | `0x08768520` | Second twice-read anchor |
| A05 | `0x08768724` | Second shift-lookup input |
| A06 | `0x08768710` | Doubled signed-halfword lookup base |
| A07 | `0x0876871c` | Subsequent mutation anchor |
| A08 | `0x087687a0` | Additive-link anchor |
| A09 | `0x08768674` | First twice-read anchor in the 512-scaled section |
| A10 | `0x08768690` | Second twice-read anchor in that section |
| A11 | `0x0876887c` | Late mutation anchor |
| A12 | `0x0876870c` | Final signed-halfword lookup anchor |

Other absolute accesses are identified directly below.

### Outputs and preconditions

There is no explicit result value or status code. Outputs consist of memory mutations and the indirect target’s effects. The representation ends with a void return if the target returns normally.

Every reached address must support its specified access; the indirect target must support execution under the actual ABI. There are no visible accessibility, alignment, bounds, permission, or target-validity checks.

No disjointness is established between arguments, anchors, derived storage, low-address storage, or target-relative storage.

## Behavior

### Widths and ordering conventions

A byte occupies one byte, a halfword two bytes, and a word four bytes. Narrow stores retain the low 8 or 16 bits. Signed-byte and signed-halfword reads widen with sign extension; unsigned-byte reads yield values from 0 through 255.

Address-bearing arithmetic is represented with 32-bit components. Portable signed-overflow, shift, wrapping, and pointer-conversion semantics are not established. Scaling by 32 or 512 and doubling an address must not be interpreted as unlimited-precision arithmetic.

Offsets are byte displacements. A byte at offset `0x15` overlaps a word beginning at `0x14`; endianness remains unknown.

Descriptive names below denote captured values or traversal roles, not source identifiers or object types. Unless a read is explicitly repeated, subsequent references mean the captured value.

### Initial mutation and two narrow-address traversals

A01 is read first. The hidden byte is stored at hidden-base-plus-second-argument, then A01’s low halfword at hidden-base-plus-third-argument. The hidden base’s `0x74` word is then captured as the **initial branch**.

The hidden byte is stored at A01 offset `0x0d`, and A01’s low byte at hidden-base offset `0x11`. A01’s `0x74` word supplies the **initial record**. Reading hidden-base offset `0x44` locates an offset-`9` byte destination receiving the hidden base’s low byte; hidden-base offset `1` receives the same byte.

A signed byte at initial-record-plus-second-argument becomes an address. Initial-record offset `4` is captured before the fourth argument’s low byte is stored at hidden-base offset `0x1d`. The captured word’s low half is stored at A01-plus-hidden-base.

The signed-byte-derived address supplies a `0x44` word. That word is copied to the derived address’s `0x34` field; A01 offset `0x54` receives the derived address; and the supplied word’s `0x14` field receives A01.

A fresh A01-relative `0x14` read supplies the **initial parent**. Initial-record offset `0x14` receives the full fourth argument. The parent’s `0x74` word is captured before initial-record offset `0x15` receives the initial branch’s low byte. The parent’s `4` word is then read and copied to initial-record offset `0x34`; parent offset `0x54` receives the initial record.

The initial branch’s `0x74` word is read twice. The first result supplies a byte payload and later a `0x54` destination; the second supplies a later traversal base. Reading the captured parent-relative record’s `0x64` word locates an offset-`0x11` byte destination receiving the first result’s low byte. That `0x64` field is reread, and its new value plus the first result addresses a signed-halfword read. The widened result becomes the **first narrow base**. Initial-branch offset `0x34` receives the second branch result, then the first result’s `0x54` field receives the initial branch.

The initial branch’s `0x74` field is read twice again. The first narrow base likewise supplies two `0x64` reads separated by an offset-`0x11` byte store using the latest first branch result. The second `0x64` value plus that first result addresses another signed-halfword read, producing the **second narrow base**. The earlier second branch result receives the latest second result at `0x34`; the latest first result receives the earlier second result at `0x54`.

### A02 and the first 32-scaled section

A02 is captured, then second-narrow-base offset `0x64` supplies the **working base**. In order:

- Working-base-plus-initial-branch receives the fourth argument’s low byte.
- Working-base-plus-earlier-second-branch-result receives A02’s low halfword.
- A02 offset `0x0d` receives the fourth argument’s low byte.

Second-narrow-base offset `0x74` supplies the **saved traversal**. Working-base offset `0x11` receives A02’s low byte, and A02 offset `0x74` supplies the **middle record**. A working-base `0x44` read locates an offset-`9` destination receiving the working base’s low byte; working-base offset `1` receives the same byte.

A02 offset `5` supplies an **unsigned-byte base**. Middle-record offset `0x14` supplies the **scale source**. The unsigned-byte base receives itself as a word at `0x54`; working-base-plus-initial-branch now receives the full fourth argument; working-base offset `0x14` receives the unsigned-byte base.

Reading fourth-argument offset `0x44` supplies the byte stored at working-base offset `0x0d`. A fresh working-base `0x44` read supplies the byte stored at working-base offset `0x11`. The following stores occur in this exact order:

1. Unsigned-byte-base offset `0x44` receives the working base as a word.
2. Unsigned-byte-base offset `0x19` receives the scale source’s low byte.
3. Working-base offset `0x74` receives the scale source as a word.
4. Unsigned-byte-base offset `0x11` receives the scale source’s low byte.
5. Working-base offset `9` receives the fourth argument’s low byte.
6. Working-base-plus-unsigned-byte-base receives the scale source’s low halfword.

Scale-source offset `0x14` is captured before its `0x34` field receives the unsigned-byte base and working-base offset `0x15` receives the captured word’s low byte.

The scale source multiplied by 32 forms the **first scaled base**. Its `0x64` word, added to the unscaled source, addresses a signed-halfword read. The widened result is stored at offset `0x14` through the first-scaled-base `0x14` pointer. The result’s `0x64` word supplies a further traversal. Saved-traversal-plus-result receives the earlier captured word’s low halfword; first-scaled-base offset `9` receives the saved traversal’s low byte; first-scaled-base offset `0x54` receives the result.

### First low-address group and first shift lookup

The further traversal’s `0x74` word supplies the **destination record**. Middle-record offset `0x44` supplies a **self-reference record**. First-scaled-base offset `5` supplies an unsigned byte, and the signed-halfword result’s `0x14` word supplies a **link value**.

Destination-record offset `0x11` receives the saved traversal’s low byte; self-reference-record offset `0x30` receives that record itself; destination-record offset `9` receives the link value’s low byte.

The word at absolute `0xb7` is captured, then self-reference-record offset `0x54` is read. Absolute `0xc7` receives the link value as a word. Destination-record offset `0x11` receives the self-reference record’s low byte. A signed byte at captured-`0xb7`-plus-captured-unsigned-byte is widened and stored at offset `0x44` through the captured `0x54` destination.

Link-value offset `0x74` supplies the **new record**. A fresh middle-record `0x44` read supplies the first shift input; new-record offset `0x14` supplies the **peer**. Before the lookup, destination-record offset `0x11` receives the saved traversal’s low byte, middle-record offset `0x54` receives the saved traversal, new-record offset `0x54` receives the peer, and peer offsets `0x11` and `0x44` receive respectively the saved traversal’s low byte and the new record.

The lookup reads a word at `0x60` plus the shift input shifted left four and then right sixteen. Its input is the fresh middle-record read, not an anchor.

Afterward, peer offset `0x19`, new-record offset `0x74`, and peer offset `0x11` receive respectively the destination record’s low byte, full value, and low byte. New-record offset `9` receives the captured `0xb7` word’s low byte. New-record-plus-lookup-result receives the destination record as a word; destination-record offset `0x0d` receives the new record’s low byte; twice-new-record receives the lookup result’s low halfword. Finally, peer offset `9` receives the saved traversal’s low byte and peer offset `0x54` receives the new record.

### A03 and A04 reload groups

For A03, the order is: capture the anchor; read new-record offset `0x64`; reread the anchor; store the peer at the second anchor result’s `0x54` field; store the first anchor result’s low byte at new-record offset `0x11`; store the captured `0x64` word’s low half at saved-traversal-plus-new-record; update peer offsets `9` and `0x54` with the saved traversal’s low byte and new record.

A04 undergoes the same sequence. Neither anchor’s two results are established as equal.

### Second shift lookup, A06, and A07

Destination-record offset `0x74` supplies the **advanced destination**. Middle-record offset `0x44` is reread; peer offset `5` supplies an unsigned byte; new-record offset `0x14` supplies the **child**.

Advanced-destination offset `0x11` receives the saved traversal’s low byte. The freshly reached self-reference record receives itself at `0x30`; advanced-destination offset `9` receives the child’s low byte. New-record offset `0x44` supplies a byte-lookup base, and new-record offset `0x54` receives the child. The self-reference record’s `0x54` destination is captured before advanced-destination offset `0x11` receives that record’s low byte. A signed byte at byte-lookup-base-plus-unsigned-byte is widened and published at the captured destination’s `0x44` field.

A05 is then captured. Child offset `0x74` supplies the **child record**. Its own `0x44` word supplies the byte written at its `0x11` field; child offset `0x44` receives the child record. The second lookup reads a word at `0x60` plus A05 shifted left four and then right twenty-four, producing the **lookup record**.

Child offset `0x19`, child-record offset `0x74`, and child offset `0x11` receive the advanced destination’s low byte, full value, and low byte. Child-record offset `9` receives the byte-lookup base’s low byte. Child-record-plus-child receives the advanced destination’s low halfword; advanced-destination offset `0x34` receives the child. Reading advanced-destination offset `0x14` supplies the byte written at child-record offset `0x15`.

The advanced destination multiplied by 32 forms the **second scaled base**. Its `0x64` word supplies the byte written at advanced-destination offset `0x0d`; its `0x14` word is captured as the **scaled payload**. A06 is read, then a signed halfword at twice A06. Middle-record offset `0x34` receives advanced-destination-plus-five, and the widened halfword’s `0x54` field receives the middle record.

A07 is captured next. A fresh second-scaled-base `0x64` read locates an offset-`0x44` word destination receiving the scaled payload. Second-scaled-base offset `0x74` supplies the **reached record**. Lookup-record offset `0x30` receives itself; A07 offset `9` receives the second scaled base’s low byte.

The reached record’s `0x44` word is captured as the **publication base**. Its `0x54` field then receives the second scaled base. Only afterward is lookup-record offset `0x54` read for the next publication destination. A07 offset `0x11` receives the lookup record’s low byte, and a signed byte at publication-base-plus-scaled-payload is widened into that destination’s `0x44` field.

A fresh second-scaled-base `0x74` read contributes to a halfword-store address with A07. The payload is the low 16 bits of A07 minus one; no checked underflow handling appears.

### Mutation interval before A08

Second-scaled-base offset `0x44` is captured and copied to its `0x34` field. A07 offset `0x54` receives the scaled base, and the captured record’s `0x14` field receives A07.

A07 offset `0x14` supplies a **parent**. Middle-record offset `0x14` receives the publication base. Parent offset `0x74` supplies a **traversal record**. Middle-record offset `0x15` receives the lookup record’s low byte; parent offset `4` is copied to middle-record offset `0x34`; parent offset `0x54` receives the middle record.

Traversal-record offset `0x64` supplies a **reached value**, installed at lookup-record offset `0x14`. Reading reached-value offset `0x64` locates a `0x54` field receiving the reached value. Next, reached-value offset `0x15` receives the parent’s low byte. Only then is a signed halfword read at parent-plus-reached-value.

Lookup-record offset `0x14` is reread as the **reloaded base**. Traversal-record offset `0x54` first receives itself. Reached-value offset `0x0d` receives the lookup record’s low byte; signed-halfword-result-plus-reached-value receives the lookup record’s low halfword; traversal-record offset `9` receives the halfword’s low byte. Traversal-record offset `0x54` then receives the reached value. Both stores appear in this order; external observability is not established.

Traversal-record offsets `0x14` and `0x64` supply a payload and destination. Reloaded-base offset `0x0d` receives the destination’s low byte, and destination offset `0x44` receives the payload. A fresh traversal-record `0x74` read contributes to a signed-halfword lookup with the reloaded base; its widened result is published at lookup-record offset `0x34`. Reloaded-base offset `0x54` receives the lookup record.

### A08 and the absolute `0x60` read

A08 is captured. A fresh traversal-record `0x64` read locates another `0x44` destination receiving the preceding payload. Traversal-record offset `0x74` supplies the **scaling record**.

A08 offset `9` receives the traversal record’s low byte. Scaling-record offset `0x44` is captured as the **retained payload**. Scaling-record offset `0x54` receives the traversal record; its offset `9` receives A08’s low byte; traversal-record offset `0x0d` receives the scaling record’s low byte.

Lookup-record offset `0x54` receives A08-plus-scaling-record. Scaling-record offset `0x54` again receives the traversal record. Traversal-record offset `0x11` receives the sum’s low byte, and traversal-record offset `0x44` receives the scaling record.

The word at absolute `0x60` is read now. Traversal-record offset `0x19` receives A08’s low byte; scaling-record offset `0x74` receives A08; traversal-record offset `0x11` receives A08’s low byte; scaling-record offset `9` receives the retained payload’s low byte.

Scaling-record-plus-captured-`0x60` receives A08 as a word. A08 offset `0x0d` receives the scaling record’s low byte; twice-scaling-record receives the captured `0x60` word’s low halfword. Traversal-record offset `9` receives the earlier sum’s low byte, and its `0x54` field receives the scaling record.

### A09, A10, and absolute `0xc2`

A09 is captured. The scaling record is multiplied by 512, forming the **512-scaled address**. A09 is reread for the value stored at traversal-record offset `0x44`. The first A09 result receives A08 at `0x14`; A08 offset `0x14` supplies the **A08-derived record**.

The 512-scaled address’s `0x11` byte receives the earlier sum’s low byte. A signed halfword is read at 512-scaled-address-plus-first-A09. Twice-captured-`0x60`-plus-`0x54` receives the retained payload as a word. First-A09 offset `9` is cleared; A08-derived-record offset `0x0d` receives first-A09’s low byte. Halfword-result-plus-first-A09 receives the retained payload’s low halfword. A08-derived-record offsets `9` and `0x54` receive the halfword’s low byte and first A09 result.

A10 is captured and reread. The second result is stored at A08-derived-record offset `0x44`; the first receives A08 at `0x14`. A fresh A08-relative `0x14` read is written as a word to absolute `0xc2`.

The 512-scaled address’s `0x11` byte receives the preceding halfword’s low byte. A signed halfword is read at 512-scaled-address-plus-first-A10, then first-A10 offset `9` is cleared.

Four separate `0xc2` reloads follow, interleaved with mutations:

1. The first supplies the offset-`0x0d` byte destination receiving first-A10’s low byte.
2. Halfword-result-plus-first-A10 receives the retained payload’s low halfword.
3. The second reload supplies the offset-`9` byte destination receiving the halfword’s low byte.
4. The third supplies the offset-`0x54` word destination receiving first A10.
5. A signed halfword is read at twice first A10.
6. The fourth reload supplies the offset-`0x14` word destination receiving that widened halfword.

The reloads need not yield identical bases.

### Final fixed-location mutations

The last widened halfword becomes the **narrow-derived base**; its `0x44` word supplies a **replacement record**. Absolute `0x9e` receives word `0x6e`. Narrow-derived-base offset `9` receives the low byte of narrow-derived-base-minus-A08, and replacement-record offset `0x54` receives the full difference.

Absolute `0xe2` is captured. A08 offset `0x74` supplies a record whose `0x44` field receives A08; lookup-record offset `9` receives that record’s low byte. Replacement-record offsets `0x54` and `0x14` receive the narrow-derived base.

Narrow-derived-base offset `0x14` supplies a **late parent**. Lookup-record offset `0x14` receives the captured `0xe2` word. Late-parent offset `0x74` is captured before lookup-record offset `0x15` receives byte `0x6e`. Late-parent offset `4` is copied to lookup-record offset `0x34`; late-parent offset `0x54` receives the lookup record.

Absolute `0x82` is captured. The captured late-parent-relative record’s `0x64` word supplies the **late record**, whose `0x54` field receives word `0x6e`. Absolute `0xb2` is captured for dispatch. The late record’s low byte is stored at captured-`0x82` offset `0x0d`.

A11 is captured. Late-record offset `0x14` receives the captured `0xe2` word; A11-plus-late-record receives A11’s low halfword; A11 offset `9` receives byte `0x6e`; A11 offset `0x54` receives the late record.

A12 is captured. Late-record offset `0x34` supplies the **final traversal base**. A11-plus-final-traversal-base receives the late record as a word. Late-record offset `0x15` receives A11’s low byte. A signed halfword is captured at A12-plus-late-record.

Final-traversal-base offset `0x74` supplies the **dispatch record**. Its `0x44` field receives the final traversal base. Absolute `0xa2` receives the sign-extended halfword as a word; A12 offset `0x54` receives word `0x6e`.

### Indirect invocation

Dispatch-record offset `0x64` supplies the target, which is written back to that same field. Dispatch-record offset `0x15` receives the final traversal base’s low byte.

Target-relative offset `0x11` receives the dispatch record’s low byte. Dispatch-record offset `0x74` is captured. Target-relative offset `0x11` then receives A12’s low byte. A12 offset `0x74` is captured. A target-relative `0x44` word read locates an offset-`9` byte destination receiving the target’s low byte.

F-unknown-01 is invoked with these captured arguments, in order:

1. The absolute-`0xb2` word.
2. The final sign-extended halfword, also published at `0xa2`.
3. The late A12-relative `0x74` word.
4. The late dispatch-record-relative `0x74` word.

## Side effects

Mutations occur in place without visible rollback. A fault or failed transfer can leave earlier effects intact.

No explicit allocation, release, reference counting, synchronization, or ownership transfer appears before invocation. Address-like stores do not establish ownership.

Target-relative stores precede invocation. Whether they affect instructions, descriptors, ordinary data, or another mapping is **documented_only**.

No explicit error branch, fallback, null check, or error result appears. Fault delivery and recovery depend on the environment.

## Dependencies

- **F-unknown-01:** indirectly obtained execution target. Identity, ABI requirements, return behavior, global effects, and resource behavior are unknown.

No other explicit invocation appears. Hidden entry state, memory mappings, anchor contents, integer semantics, and the execution ABI are additional prerequisites. F-unknown-01 is not assumed to preserve globals or reachable memory, or to acquire or release ownership.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial aliasing:** if the initial halfword destination overlaps hidden-base offset `0x74`, the subsequent word read must account for that store according to the actual byte order.

2. **Signed versus unsigned addressing:** signed-byte pattern `0xff` widens to −1. The unsigned A02-relative byte pattern `0xff` instead yields 255, including for its offset-`0x54` word store.

3. **Ordered A02 aliasing:** if unsigned-byte-base offset `0x19` overlaps working-base offset `0x74`, the byte store precedes the word store. The later unsigned-byte-base offset `0x11` store follows both.

4. **A07 reached-record mutation:** if reached-record offset `0x54` aliases lookup-record offset `0x54`, the subsequent destination read must account for the intervening word store of the second scaled base.

5. **Pre-A08 halfword aliasing:** if the reached value’s offset-`0x15` byte lies within the parent-plus-reached-value halfword, the halfword read follows the parent-low-byte store.

6. **Distinct anchor reads:** if A03 or A04 changes between its two reads, the second result determines the `0x54` destination; the first supplies the later byte payload. For A09 and A10, the second result is installed in the reached record’s `0x44` field, while subsequent anchor-relative operations use the first.

7. **Changing `0xc2`:** if an intervening derived store changes `0xc2`, the next destination uses the reloaded value, not a cached earlier base.

8. **Signed dispatch payload:** final halfword pattern `0x8001` produces the sign-extended 32-bit pattern `0xffff8001` both at `0xa2` and in the second dispatched argument.

9. **Repeated target writes:** both target-relative `0x11` stores appear, separated by the dispatch-record `0x74` read. If that read overlaps the first store, the ordering can affect the captured fourth argument.

10. **Normal return:** if F-unknown-01 returns normally, the visible path reaches a void return without explicitly exposing the target’s return value.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid layouts, and field meanings.
- Hidden-input provenance and complete calling convention.
- Initial memory contents and permitted alias relationships.
- Endianness, alignment support, overflow, wrapping, shifts, and pointer conversions.
- The nature and accessibility of low-address storage.
- Concurrency, volatile behavior, and external observability of repeated accesses.
- Whether the final transfer is precisely a conventional call.
- Whether target-relative mutations affect executable instructions.
- F-unknown-01’s effects, ownership rules, and return guarantees.

These unknowns do not justify merging represented reads, deleting represented stores, or substituting a conventional linked-record algorithm.

## Evidence

The supplied static representation supports the stated access widths, explicit narrow conversions, fixed addresses, arithmetic forms, access ordering, and indirect invocation. It also identifies overlapping global storage and unresolved indirect control flow; neither establishes safe memory behavior.

Descriptive record roles are explanatory, not recovered types. Application intent is inferred; environmental semantics and dependency behavior remain unresolved. No runtime execution, memory observation, or acceptance testing has been performed.
