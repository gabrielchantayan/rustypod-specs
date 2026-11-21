<!-- Trusted provenance: F-8c5434fcf9ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c5434fcf9ae

## Purpose

F-8c5434fcf9ae performs a state-dependent sequence of indirect calls, followed by ordered memory reads and mixed-width writes, and ends with an unresolved indirect transfer.

No application-level purpose is established. Address scaling, self-referential words, and character-sized literals do not establish text formatting, object construction, list management, or ownership transfer.

This contract distinguishes:

- **Established static evidence:** relationships explicitly present in the supplied representation.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** behavior that cannot be resolved without declarations, dependency contracts, instruction semantics, or execution evidence.

The requirements below describe the represented behavior conditionally; they do not independently validate the representation. No runtime tests have been performed.

## Interface

### Explicit inputs and output

The displayed interface accepts two 32-bit inputs and declares no return value.

The first indirect dependency receives both inputs unchanged and in their original order. The visible body does not subsequently inspect or dereference those inputs directly. Their meaning remains unknown: addresses, handles, encoded quantities, and ordinary scalar values are all unresolved.

There is no established success indicator, failure result, output-buffer parameter, or exception contract.

### Implicit inputs

Execution additionally depends on:

- Two incoming 32-bit machine values, logically shifted right by 15 and passed as the third and fourth arguments of the first call, respectively.
- An incoming control-flow value used to construct the first target by adding `0xffe`.
- Further continuation-like values used to construct later indirect targets with the same displacement.
- Secondary machine results not explained by the displayed dependency interfaces.
- Global and computed-address memory.
- A global word retained as a publication base.
- An unresolved final indirect target.

A callable interface consisting solely of the two explicit arguments is therefore not established.

### Widths, layouts, and arithmetic

The representation contains explicit 8-, 16-, and 32-bit memory accesses. Some dependency results are represented as 64-bit quantities and separated into numerical low and high 32-bit halves. This separation does not establish byte order or the actual return-register convention.

Common computed offsets include `+4`, `+5`, `+8`, `+9`, `+0x0d`, `+0x10`, `+0x11`, `+0x14`, `+0x15`, `+0x19`, `+0x20`, `+0x24`, `+0x34`, `+0x40`, `+0x44`, `+0x54`, `+0x64`, and `+0x74`. These identify access locations, not a recovered structure layout.

**The read of the global at `0x05` has unresolved width.** Its value is converted to an unsigned 32-bit quantity, but its declaration is absent. That conversion does not prove an 8-bit read, a 32-bit read, or zero-extension from a byte.

Byte stores narrow to the low eight bits. Explicit unsigned byte loads used in word stores are zero-extended. The two explicit 16-bit literal stores contain `0x3a` and `0x65`.

Arithmetic examples use low-32-bit truncation at each 32-bit operation. Signed overflowing shifts, signed multiplication, pointer arithmetic, and architecture-specific address formation remain **documented_only** at the executable-semantics level. Unlimited-precision arithmetic is not an acceptable substitute for the displayed widths.

### Preconditions

Completion requires valid reached call targets, compatible machine-level calling conventions, and accessible memory for every reached access at its required width. Values obtained from dependencies and mutable memory must remain usable when accessed.

No visible checks enforce these conditions. Zero-valued inputs or intermediate quantities are not rejected before use. Alignment, endianness, object boundaries, disjointness, and storage lifetime are not established.

## Behavior

### Ordered dependency phase

An extended straight-line sequence of indirect calls precedes the direct memory updates. No input-dependent branch or explicit loop is displayed in this phase.

The first call forwards the explicit inputs and shifted implicit inputs. Subsequent calls include boundaries with:

- No displayed explicit arguments.
- One shifted result.
- One or two explicit zeros.
- Low and high halves of wide results.
- Retained earlier results combined with newly obtained results.
- Secondary machine results absent from the displayed interfaces.
- Three or four arguments containing repeated or differently shifted quantities.

Calls without displayed arguments may still consume implicit machine state. Every represented call remains part of the ordered behavior; a result that appears unused does not establish that its producer has no effects.

Important numerical relationships include:

- Masking with `0x00ffffff`, then logically shifting right by 8, selects original bits 8–23 and yields at most `0xffff`.
- Masking with `0x0003ffff`, then shifting right by 11, yields at most `0x7f`.
- Masking with `0x0001ffff`, then shifting right by 4, yields at most `0x1fff`.
- Masking with `0x00ffffff`, then shifting right by 12, yields at most `0xfff`.
- A left shift by 8 before masking is distinct from masking the unshifted input.
- Successive 32-bit shifts retain intermediate truncation rather than accumulating unlimited-width bits.

Some retained quantities are explicitly set to zero before later shifts and argument construction. Those particular arguments remain zero in the represented dataflow. This does not imply preservation of globals or memory by intervening dependencies.

### Initial direct updates

The direct-update phase retains several dependency-derived quantities: two large scaled address quantities, another result shifted left by 8, and an index formed by shifting a result left by 8, masking with `0x7fff`, and shifting right by 6.

That index is a multiple of four between 0 and 508 inclusive. Its doubled value is a multiple of eight between 0 and 1016 inclusive.

Using the latest wide result, the first updates occur in this order:

- A word containing the first large address quantity is written at the high-half address plus `0x14`.
- Its `+9` byte location receives the narrowed high half.
- A word containing the second large address quantity is written at 256 times the low half, plus `0x54`.
- The same `+9` byte location receives the narrowed first address quantity.

The first byte write is not dispensable merely because a later write overwrites it.

The second large base then receives a zero word at `+0x14`. Its `+5` byte is read, a zero word is written at the doubled-index address plus `0x14`, and the zero-extended byte is stored at the second large base plus `0x34`.

The word at the second large base plus `0x14` is subsequently reread. The doubled-index word is cleared again, and two byte stores place the narrowed reread word at the doubled-index base plus `0x15`. Aliasing or special memory behavior prevents replacing the reread with an assumed literal zero.

### Fixed globals and interlinked updates

The fixed-global accesses participate in the same address space as the computed accesses unless additional evidence establishes otherwise.

The value associated with `0x05` is read and converted to unsigned 32-bit form **after** the duplicate doubled-index `+0x15` byte stores. Next, the byte at fixed address `0x01` receives the narrowed doubled index. A byte at the converted global value plus `0x15` then receives the narrowed undoubled index.

The second large base receives a byte derived from the retained shifted result at `+0x11`. A word at the earlier reread word plus `0x54` receives that shifted result, and the `+0x11` byte is then cleared.

A word at the converted global value plus `0x54` is saved before that location is overwritten. This saved word is called the **saved base** below; it is a numerical role, not an object or ownership classification.

Subsequent updates link addresses obtained through `+0x74`, write the saved base to its own `+0x24`, and repeat byte writes through another computed base. One expression contains two word reads from the doubled-index base plus `0x74`, using one value as stored data and another to form the destination. Their individual evaluation order is not established by the supplied expression.

The meaningful fixed-global relationships are:

- The word at fixed address `0x54` receives **twice the saved base**, narrowed to 32 bits.
- That fixed word store follows a computed word store of another derived value at doubled-index `+0x14`.
- It precedes two computed word stores of the saved base at doubled-index `+0x14`.
- Two subsequent byte writes at doubled-index `+0x15` contain the narrowed four-times-saved-base quantity.
- The byte at fixed address `0x15` then receives the narrowed doubled index.

Thus `0x01` and `0x15` receive the same numerical byte under ordinary narrowing, but at distinct points separated by computed writes. The fixed word at `0x54` is not merely an unspecified example of a low-address access.

Further updates use two-, four-, and eight-times-scaled saved values, byte-derived addresses, and words read at `+0x64` and `+0x74`. They include self-referential word stores, duplicate stores, and overwrites. Explicit byte reads at computed `+5` locations supply zero-extended address components for later `+0x74` word reads. These reads are distinct from the declaration-unresolved global read at `0x05`.

### Literal stores and repeated publication

For the remaining phases, the following descriptive quantities identify retained numerical values:

- **Quadrupled base:** four times the saved base.
- **Adjusted base:** quadrupled base minus `0x96`.
- **Destination base:** four times the adjusted base.
- **Loaded base:** a word obtained through the later byte-derived `+0x74` lookup.
- **Doubled loaded base:** twice the loaded base.
- **Quadrupled loaded base:** four times the loaded base.
- **Publication base:** the retained value read from the publication-base global.

The literal `0x3a` is stored as a byte at the quadrupled loaded base, as a 16-bit value at eight times the loaded base, and as a word at the quadrupled loaded base plus `0x20`.

At destination base plus `0x20`, word writes occur as `0x20`, `0x2f`, the zero-extended byte read from destination base plus 4, and `0x20` again.

**The publication-base global is read before the word store of `0x20` at fixed address `0x60`.** Publication uses the retained global value. If the fixed store overlaps that global, it does not retroactively replace the retained base.

The first publication group writes, in order:

- The quadrupled loaded base as a word at eight times the saved base plus `0x54`.
- The narrowed destination base at publication base plus `0x19`.
- The narrowed publication base at destination base plus `0x11`.
- Eight times the saved base as a word at publication base plus `0x34`.

Later groups repeat this four-write pattern, replacing the saved-base scaling with adjusted-base scaling in the first and fourth writes. Their other destinations and values remain retained quantities. There is no intervening dependency call or displayed progression through successive objects.

Repeated stores remain part of the represented access trace, even when their values match. Exact executable multiplicity remains **documented_only** without independent control-flow validation; reducing the region to final-state assignments would discard established repeated-write behavior.

### Final reads and fixed-address byte

After publication, the following ordering is explicit:

1. A 32-bit word is read at fixed address `0xa9`.
2. A 32-bit word is read at quadrupled loaded base plus `0x74`.
3. A byte is written at fixed address `0x3a`.

The byte written at `0x3a` is the low eight bits of **eight times the adjusted base plus `0x30`**. It is not necessarily the literal `0x3a` and is not derived from the word just read at `+0x74`.

These relationships apply even when the fixed and computed locations overlap.

The operation next saves a word from doubled loaded base plus `0x14`, followed by a word from destination base plus 4. It writes narrowed destination-base data at doubled loaded base plus `0x0d`, literal byte `0x6c` at destination base plus 8, and narrowed publication-base data at destination base plus `0x0d` and `+9`. A self-valued word is also stored at doubled loaded base plus `0x54`.

### Three distinct reads of `0x67`

The byte at fixed address `0x67` is assigned the low byte of the saved `0xa9` word. Its later uses require **three distinct reads**, not reuse of that saved byte:

- First, after reading the word at doubled loaded base plus `0x44`, the current byte at `0x67` is read for the store at fixed address `0x21`.
- Second, after intervening computed writes, `0x67` is read again for a byte store at destination base plus 9.
- Third, after further computed accesses, `0x67` is read again for another byte store at destination base plus 9.

Between the first and second reads, the operation writes a byte at destination base plus `0x10`, overwrites publication base plus `0x54` first with the loaded `+0x44` word and then with the destination base, and writes a shift-derived byte at doubled loaded base plus 1. It also reads destination base plus `0x40`, publishes the saved destination `+4` word at publication base plus `0x74`, replaces destination base plus `0x10` with the `+0x40` word, and writes doubled loaded base at publication base plus `0x44`.

Between the second and third reads, it writes the narrowed destination base at doubled loaded base plus 9, writes the saved `0xa9` word at destination base plus `0x44`, and reads a word from destination base plus `0x54`.

Any intervening computed write may overlap `0x67`. Consequently, the three read values may differ, and neither computed `+9` store is specified as unconditional propagation of the original low byte from `0xa9`.

The variable shift uses the word read at doubled loaded base plus `0x44` and a count restricted to the low eight bits of the destination base. The represented operand is signed; right-shift semantics and counts of 32 or greater remain **documented_only**.

### Remaining updates and final transfer

Following the third `0x67` read, the saved quadrupled-loaded-base `+0x74` word supplies a narrowed byte for writes through the latest destination `+0x54` word and at doubled loaded base plus `0x11`. The earlier doubled-loaded-base `+0x14` word is restored at its `+0x54`.

A word is read at destination base plus `0x14`. Destination base plus 9 and `+0x44` are then overwritten with narrowed and full-width forms of the latest destination `+0x54` word.

A word loaded from doubled loaded base plus 4 becomes another address base. A 16-bit `0x65` is written at twice that base, followed by a word read at that base plus `0x64`. The earlier destination `+0x14` word supplies a byte written through the latest destination `+0x54` value, whose `+0x54` word then receives decimal 100.

A word is subsequently read at the new base plus `0x74`, followed by a word read at doubled loaded base plus `0x64`. Final writes use these retained quantities, including a word store at twice the saved new-base `+0x64` value.

The final indirect dependency receives:

1. `0x72`.
2. The word saved from the new base plus `0x64`.
3. The word saved earlier from doubled loaded base plus `0x14`.
4. The word read late from the new base plus `0x74`.

If that transfer returns, the displayed body returns without an explicit result.

## Side effects

There is no established transaction, rollback, cleanup, retry, or recovery boundary. Earlier effects precede later faults or non-returning transfers.

Effects include indirect calls, explicit mixed-width writes, repeated writes, fixed-global accesses, and reads whose results determine subsequent addresses. Access roles do not imply disjoint storage. Overlapping byte and word accesses depend on platform byte order.

No identified allocation or release contract appears. Unknown dependencies may allocate, release, retain, invalidate, or mutate storage and globals. Caller ownership, lifetime preservation, and normal dependency return must not be assumed.

Concurrency safety, reentrancy, synchronization, and hardware memory ordering beyond represented sequencing are **documented_only**.

## Dependencies

The supplied map identifies only F-8c5434fcf9ae. The following opaque IDs label unresolved boundaries within this document:

| ID | Boundary | Established contract |
|---|---|---|
| F-000000000001 | Entry indirect dependency | Receives two unchanged explicit inputs and two shifted implicit inputs |
| F-000000000002 | Intermediate indirect boundaries | Ordered calls with varying displayed arguments and result widths |
| F-000000000003 | Final indirect dependency | Receives `0x72` and the three identified saved words |

F-000000000002 is a grouping label, not evidence that the calls share a target. No dependency is established to preserve globals, validate addresses, manage ownership, or return normally.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Entry forwarding:** Explicit inputs `0x11223344` and `0xaabbccdd`, with respective implicit values `0x00008000` and `0xffffffff`, produce first-call arguments `0x11223344`, `0xaabbccdd`, 1, and `0x1ffff`.

- **Index boundary:** A result whose relevant low seven bits are all set produces index 508 and doubled index 1016, not index 511.

- **Unresolved global width:** Knowing that the converted value from `0x05` is `0x80` permits use of that numerical value in later addressing. It does not establish which bytes were read to obtain it.

- **Fixed-global relationships:** If the retained index is 132, fixed bytes `0x01` and `0x15` each receive `0x08`, at their separate specified positions. If the saved base is `0x1234`, the intervening fixed word store at `0x54` contains `0x2468`.

- **Publication capture:** If the publication-base global overlaps `0x60`, publication still uses the value read before the fixed `0x20` word store, subject to subsequent explicitly represented accesses.

- **Final arithmetic:** An adjusted base of `0x2a` produces byte `0x80` at fixed address `0x3a`. The `0xa9` word read and quadrupled-loaded-base `+0x74` word read both precede that store.

- **Global rereads:** If publication base plus `0x54` overlaps `0x67`, its intervening word overwrites can change the byte seen by a later `0x67` read. Acceptance must preserve all three reads rather than reuse the byte stored at `0x21`.

- **Invalid access:** An inaccessible reached address has no established fallback result or rollback. Earlier effects cannot be claimed undone.

## Uncertainties

The following remain **documented_only**:

- Application purpose and supported callers.
- Authentic entry, result, and return conventions.
- Indirect target identities and dependency effects.
- Executable control flow and repeated-write multiplicity.
- Declaration, read width, and conversion details for the global at `0x05`.
- Signed overflow, variable shifts, and address arithmetic.
- Memory map, alignment, endianness, faults, and special-memory behavior.
- Object boundaries, aliasing constraints, and storage lifetimes.
- Final-transfer return behavior, ownership, and concurrency guarantees.

Possible text-related, linking, or publication interpretations are inferred intent only.

## Evidence

The evidence is the supplied static representation and its single-entry dependency map. Explicit operations establish the stated widths where shown, arithmetic relationships, sequencing across statements, saved-value uses, global rereads, and final argument provenance.

Missing declarations do not establish the read width at `0x05`. Reported overlapping global storage prevents assuming symbolic globals are separate objects. The unresolved final control transfer prevents assigning it a known callee or guaranteed return behavior.

No dependency bodies, machine instructions, validated memory map, implementation access, or runtime observations were available. No runtime tests have been performed.
