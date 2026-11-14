<!-- Trusted provenance: F-fdc2d1f2fa62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdc2d1f2fa62

## Purpose

F-fdc2d1f2fa62 describes a straight-line sequence of indirect calls, calculated memory accesses, shared-state updates, and a final call through address zero. Visible operations include indexed word stores, byte lookups that determine byte-store destinations, extraction of low bytes from words, and propagation of values obtained from unresolved calls.

No application-level purpose is established. In particular, the evidence does not establish initialization, serialization, resource management, or a particular operating-system service.

This contract describes the supplied static representation. Reaching any later operation is conditional on all preceding accesses completing and all preceding calls returning. An apparent return exists after the zero-address call; successful termination is not established.

## Interface

### Explicit inputs and result

The displayed interface contains three 32-bit inputs and no explicit return value.

| Input | Directly established use |
|---|---|
| First input | No displayed expression consumes it. |
| Second input | No displayed expression consumes it. |
| Third input | Its unsigned value, logically shifted right by 23, is stored as a 32-bit word. |

The first two inputs are not proven irrelevant: unresolved calls have incomplete interfaces and may observe incoming machine state.

If execution reaches the final call and that call returns, the displayed function returns without a value. No status, exception conversion, or error result is shown.

### Ambient inputs and descriptive labels

The function also depends on an incoming link-like value, two ambient 32-bit values, secondary call results, shared storage, and memory at computed addresses.

The following labels describe dataflow, not recovered program identifiers:

- **Ambient A:** a 32-bit unsigned value used in shifts and byte-address calculations.
- **Ambient B:** a 32-bit value written during the first indexed store.
- **Secondary S:** an early secondary 32-bit call result reused for an indexed store, a dependency argument, and a masked-address word read.
- **Secondary C:** a later secondary word retained for repeated and shifted call arguments.
- **Control:** a shared word-sized location assigned twice.
- **Shared base:** a shared word-sized location read once explicitly and then retained as an address.
- **Retained index:** a later call result whose four-times-scaled form is kept across subsequent calls.
- **Replicated word:** a complete word retained from a later read at offset 0x74.
- **Saved word:** a word subsequently read through the captured shared base.
- **Final low** and **final high:** the respective 32-bit halves of the last displayed 64-bit call result.

The representation reuses these captured values across calls. This establishes displayed dataflow, not an independently verified calling-convention guarantee about physical registers.

### Widths, layouts, and preconditions

Visible quantities include 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit call results. Byte lookup results are unsigned. Word-to-byte conversions retain the least significant eight bits of the numeric word.

For address descriptions:

- **Word slot at index x:** address 0x14 + 4 × x.
- **Wide-stride slot at index x:** address 0x14 + 16 × x.
- **Byte-derived destination:** address 0x11 plus an unsigned byte just read.

These expressions do not establish arrays, object boundaries, valid pointers, or allocation ownership.

Logical shifts operate on unsigned values. The sign selector instead uses a signed interpretation of Ambient A and an arithmetic right shift by 31.

For execution to complete, every reached access must be valid for its displayed width and every reached indirect target must be callable and return. No corresponding checks are shown. Alignment, mapped low addresses, and accessibility are therefore environmental preconditions rather than guarantees.

Examples using truncating arithmetic explicitly assume 32-bit bit-vector behavior. Signed multiplication, signed left shifts, address overflow, and their relationship to executable machine semantics remain separately unresolved.

## Behavior

### Execution and call ordering

The visible body contains no branch or loop. Its explicit memory effects follow the sequence below. Indirect calls between accesses are observable dependency interactions: they may modify memory, shared state, or execution state, or fail to return.

The dependency labels in this contract are opaque references:

- **F-dep-001:** the initial link-derived call occurrence.
- **F-dep-002:** the family of subsequent unresolved indirect call occurrences.
- **F-dep-003:** the terminal zero-address call occurrence.

F-dep-002 is a collective label, not a claim that those occurrences resolve to one function. Where intervening calls are mentioned below, they belong to this unresolved family. Calls with no displayed arguments must not be interpreted as proven zero-argument interfaces.

### Initial indexed stores and first byte transfer

Four calls precede the first explicit memory write. F-dep-001 uses a target derived from the incoming link-like value; the next three use fixed-value target expressions. The fourth call provides a primary index.

Ambient B is written as a word to the word slot selected by that index. The next dependency receives four times the index.

After four further calls, Secondary S supplies the index for a word store of Ambient A logically shifted right by 22. A dependency then receives **that same Secondary S**, unscaled.

Subsequent calls include a primary word result immediately forwarded to another dependency. A later primary index selects the word slot receiving the third explicit input logically shifted right by 23.

Immediately after that store, a word is read at:

**0x14 + ((Secondary S AND 0x1fffffff) logically shifted right by 19).**

This is the same retained Secondary S used for the earlier Ambient-A-derived word store and dependency argument. No fresh secondary result is shown for this address calculation. The variable address component ranges from 0 through 1023; it is not multiplied by four and need not be word-aligned.

The newly loaded word is captured before another dependency receives four times the latest primary index. Four more calls precede the first byte transfer.

The captured word is logically shifted right by 22, retaining a value from 0 through 1023. A byte is read at Ambient A + 1. That byte plus 0x11 selects the destination for the low byte of the shifted value.

Consequently, this store writes original word bits 22–29. Original bits 30–31 remain present in the retained 10-bit shifted value but are discarded by the byte store.

### Byte-derived zero stores and retained secondary data

A separate unsigned quantity is explicitly zero. A following dependency receives another secondary word result, distinct from Secondary S. After an intervening call, another dependency receives the zero quantity logically shifted right by 23, hence zero.

Further calls produce a primary index. Before forwarding its four-times-scaled value, the function reads a byte at:

**4 × the retained 10-bit shifted value + 1.**

It stores zero at that byte plus 0x11. The lookup uses the full retained 10-bit value, not the previously stored eight-bit truncation.

The next call sequence includes a primary result passed twice in four-times-scaled form. A later primary word is retained and forwarded. Following another call, a sign-selected byte lookup determines another zero-byte destination.

The selector is obtained by negating the arithmetic right shift of signed Ambient A by 31, then adding one:

- Signed Ambient A nonnegative: selector address 1.
- Signed Ambient A negative: selector address 2.

The selected byte plus 0x11 receives zero.

Secondary C is then captured and supplied three times to a dependency. A subsequently obtained primary word is passed to another dependency together with the earlier retained primary word logically shifted right by 23.

After further calls, the same sign-selector calculation causes a **new byte read**. Its unsigned result is retained as a small base, and zero is written at that base plus 0x11.

The two sign-selected reads are distinct accesses separated by calls and writes. They are not required to return equal bytes.

### First control assignment and word-to-byte transfers

A further call receives a scaled index, and a later call receives a duplicated scaled pair. A new primary 32-bit result is then assigned to Control.

That result is forwarded to a dependency. After one additional call, a word is read at the retained small base plus 0x74, and its low byte is written at the same base plus one.

A dependency receives Secondary C shifted left by 11. Another supplies a 64-bit result. Its low 32-bit half, high 32-bit half, and the recent Control-producing primary result logically shifted right by 23 are supplied as three arguments to the next dependency.

A later call result is multiplied by four to form another base. A word read at that base plus 0x74 supplies a low-byte store at that base plus one. The base is then passed to a dependency.

Further calls produce the Retained index. Its four-times-scaled value is captured separately. Before either retained value is consumed by the next argument-bearing call, the function repeats the preceding base-plus-0x74 word read and base-plus-one byte write.

This repetition is a fresh word read, not reuse of the earlier loaded value.

The next four-argument dependency receives, in order:

1. Secondary C shifted left by 12.
2. The retained scaled index.
3. The same retained scaled index again.
4. The Retained index shifted left by 18.

Later calls obtain and forward additional primary words.

### Retained-word replication and scaled links

Another primary index is multiplied by four to form a new base. A word read at that base plus 0x74 becomes the Replicated word. Its low byte is immediately written at the base plus one, and the base is passed to a dependency.

After further calls, the complete Replicated word is written to the wide-stride slot selected by a newly returned index. This destination has stride sixteen, unlike the ordinary word-slot stride of four.

The next argument-bearing dependency receives the retained scaled index **logically shifted right by two, then shifted left by 16**.

Under 32-bit truncating arithmetic, this result equals the Retained index shifted left by 16. Scaling by four loses only original bits 30–31; neither can contribute to the final left-shift-by-16 result. The displayed shift sequence remains part of the dataflow. This equivalence does not resolve signed-language overflow or left-shift semantics elsewhere.

Further calls forward primary words. A later index selects an ordinary word slot receiving the same complete Replicated word. The following dependency receives four times that index.

Another call chain produces a primary word that is shifted left by two. This shifted value is stored at:

**(retained scaled index AND 0xfffffffc) + 0x14.**

The next dependency receives that shifted value twice.

After additional result forwarding, a newly returned index selects an ordinary word slot receiving zero. Four times that index is then passed to a dependency.

### Split-result links and captured shared base

A subsequent call supplies a 64-bit result. Its low 32-bit half is shifted left by two, and that shifted value is stored in the word slot indexed by its high 32-bit half.

Only after this store does the function read Shared base and capture its value. The next dependency receives four arguments: the shifted low-half value, the high half, the shifted value again, and the shifted value again.

Following further calls, the same shifted value is written to a word slot selected by a newly returned primary index. A word is then read at the captured Shared-base value plus 0x14. This becomes Saved word. The next dependency receives four times the latest index.

The later word read does not reread Shared base. A dependency can change Shared base without changing the captured address. Dependencies or preceding stores can nevertheless change memory at that captured address before the word read occurs.

Another call sequence supplies the final 64-bit result, retained as Final low and Final high.

### Final byte transfers and termination

The Replicated word is logically shifted right by 23. A byte read at that result plus one determines a byte-derived destination. The low byte of Saved word is written there.

A dependency then receives Final high three times, followed by Final low shifted left by two. Further calls obtain and forward another primary word.

Next, Final low is masked with 0x3fffffff and logically shifted right by 21. A byte read at that result plus one determines another byte-derived destination. The stored byte is the low byte of Saved word logically shifted right by 22: Saved word bits 22–29.

The remaining call sequence includes a scaled argument, a duplicated scaled pair, a call using a different target offset, and forwarding of its primary result.

The last F-dep-002 occurrence supplies an 8-bit primary result and a pointer-like secondary result. The terminal explicit effects occur in this order:

1. A single zero byte is written through the secondary pointer.
2. Control receives a composed word: the final 8-bit result occupies numeric bits 0–7, and a displayed three-byte shared value anchored at address 0x09 supplies bits 8–31.
3. A zero halfword is written at address **(Final low AND 0x3fffffff) logically shifted right by 7**.
4. F-dep-003 is called through address zero with three displayed zero arguments.
5. The function returns only if that call returns.

The Control composition is not established as “preserve the old Control upper bits.” Its upper portion comes from the separately displayed three-byte source. That source is consumed after the pointer-based zero write, which matters if they overlap.

The final halfword address is not explicitly aligned or checked.

## Side effects

Explicit writes have their displayed widths: byte stores affect one byte, the terminal halfword store affects two bytes, and word stores affect four bytes. Numeric low-byte extraction does not establish the byte order of surrounding memory.

Ordering and retention requirements include:

- Each offset-0x74 word read precedes its associated low-byte store.
- The repeated transfer performs a fresh read after intervening calls.
- Replication uses the captured complete Replicated word without rereading its original source.
- Shared base is captured after the first split-result link store.
- Saved word is read after the second corresponding link store.
- The final pointer-based byte write precedes the three-byte-source composition, halfword store, and zero-address call.

Computed addresses can overlap other destinations, lookup bytes, Control, Shared base, or dependency-visible memory. No non-aliasing assumption is justified.

No rollback, bounds check, null check, ownership transfer, allocation, release, or explicit error-reporting mechanism is shown. A fault or non-returning dependency can leave every earlier completed effect in place. Unknown dependencies may manage resources, but this function’s visible calls do not establish which resources they acquire, preserve, or release.

## Dependencies

F-dep-001 derives its target from the incoming link-like value plus 0xffe.

F-dep-002 occurrences derive targets from fixed numeric or code-related values and fixed offsets. Most use offset 0xffe; other displayed offsets include 0xf38, 0xfec, 0xf88, 0xfb0, 0xf94, 0xf9c, 0xf90, and 0xb0. These expressions establish neither executable validity nor a resolved callee identity.

F-dep-003 targets zero. Its displayed three zero arguments do not establish the callee’s complete calling convention.

Dependency results include primary byte, word, and 64-bit values, secondary words, and a secondary pointer-like value. Secondary-result attribution and the full register or stack interface are unresolved. No dependency is assumed pure, memory-preserving, global-preserving, terminating, or responsible for releasing memory.

## Acceptance examples

These are conditional arithmetic and ordering requirements, not runtime observations.

1. **Input extraction.** If the third-input store is reached, input 0xffffffff yields word 0x000001ff; input 0x007fffff yields zero.

2. **Secondary S reuse.** If Secondary S is 0x00080003, its Ambient-A-derived word slot is 0x00200020. The subsequent unscaled dependency argument is 0x00080003. The later masked word read uses address 0x15, derived from the same retained value.

3. **Retained ten-bit value.** If the early captured word is 0xffffffff, its right-shift-by-22 result is 1023. The first byte store writes 255, but the following lookup reads address 4093, not an address calculated from 255.

4. **Sign selection.** Ambient A values 0x7fffffff and 0x80000000 select addresses 1 and 2 respectively. Intervening effects may cause the two reads of the selected address to differ.

5. **Fresh transfer.** A source word 0x123456ab produces byte 0xab. If an intervening dependency changes that same source word to 0x987654cd before the repeated transfer, the repeated store writes 0xcd.

6. **Different strides.** Index 3 selects ordinary word address 0x20 and wide-stride address 0x44.

7. **Shift equivalence.** Under 32-bit truncating arithmetic, Retained index 0xc0012345 scales to 0x00048d14. Shifting that value right by two and then left by 16 yields 0x23450000, exactly as shifting the original index left by 16 does.

8. **Split-result link.** Low half 5 and high half 7 produce a word value of 20 at address 0x30, provided the store is valid.

9. **Captured base.** If Shared base is captured as 0x1000 and subsequently changed to 0x2000, the Saved-word read still targets 0x1014.

10. **Final composition and failure.** If the three-byte source has numeric value 0x123456 and the final primary byte is 0xab, Control receives 0x123456ab. If the later zero-address call cannot execute, no displayed mechanism undoes this or preceding writes.

## Uncertainties

### Established static evidence

The supplied representation establishes access widths, displayed ordering, masks, shifts, value reuse, duplicated arguments, split-result dataflow, two Control assignments, and the absence of visible branches or loops.

### Inferred intent

Scaled slots may represent structured storage, and byte-derived destinations may implement a compact lookup scheme. These are interpretations only; they establish no object types, capacities, ownership rules, or application purpose.

### Documented_only unknowns

- Complete calling conventions and reliability of secondary-result recovery.
- Physical preservation of ambient or captured state across unresolved calls.
- Executability, instruction-state interpretation, and return behavior of targets.
- Exact memory map, alignment rules, endianness, and memory-mapped effects.
- Signed arithmetic semantics and address overflow behavior.
- Concurrency, atomicity, and memory-ordering guarantees.
- Exact overlapping storage layout involving Control and the source anchored at 0x09.
- Machine access granularity used to obtain that three-byte value.
- Whether the displayed boundary and apparent return faithfully describe executable control flow.

These unknowns are not replaced by ordinary library, ABI, or platform assumptions.

## Evidence

This contract is based solely on the supplied static representation. Comments, prior prose, and reviewer feedback are not independent behavioral evidence.

The retained Secondary S relationship follows from its repeated use without a displayed replacement. The corrected shift equivalence follows from 32-bit bit positions, separately from unresolved signed-arithmetic semantics.

No runtime tests, executable checks, memory traces, or target-platform observations have been performed. Acceptance examples are conditional consequences of the static operations.
