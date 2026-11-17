<!-- Trusted provenance: F-ca91eb5311ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca91eb5311ea

## Purpose

F-ca91eb5311ea performs an ordered sequence of indirect dependency invocations, then mutates byte-addressed memory through computed and fixed addresses, and finally invokes an unresolved target with four arguments.

**Established static evidence:** the supplied representation exposes three 32-bit inputs, dependency-result transformations, mixed-width memory accesses, repeated stores, accesses near address zero, and a terminal indirect invocation. The visible body contains no validation branch, recovery path, or input-controlled loop.

**Inferred intent:** recurring offsets, loaded links, and self-referential stores suggest structured-state manipulation. They do not establish a particular object model, subsystem, data format, or ownership protocol. Constants that coincide with character values are not sufficient evidence of text construction.

This contract describes the supplied representation without claiming that its recovered interface or machine semantics are independently verified. Unresolved matters are marked **documented_only**. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body |
| Second argument | 32 bits | No explicit use in the visible body |
| Third argument | Signed 32 bits | Supplies initial shifted arguments and an early intermediate |

The first two arguments have no established direct effect. Their absence from visible expressions does not prove that incoming register contents are irrelevant to every dependency; the complete calling convention is unavailable.

There is no explicit buffer, length, capacity, allocator, or error-output parameter.

### Implicit inputs

Execution depends on an incoming return-link value, indirectly selected executable targets, primary and secondary dependency-result channels, existing memory contents, and the address-space mapping.

One separately stored 32-bit value is captured during the repeated-write phase and subsequently used as an address base. Its storage is not established to be disjoint from computed writes. It must not be treated as immutable configuration or as an entry-state value.

### Outputs

The represented interface declares no return value. Observable effects consist of dependency invocations, memory reads and writes, and terminal dispatch.

The terminal invocation receives four represented 32-bit arguments:

1. `0x72`.
2. A late-loaded word used as an address base.
3. A word captured earlier from a derived base plus `0x14`.
4. A late-loaded word from another base plus `0x74`.

The representation returns if the terminal invocation returns. Termination, register preservation, and conventional call-versus-jump behavior remain unresolved.

### Widths and memory interpretation

“Byte,” “halfword,” and “word” mean 8, 16, and 32 bits respectively. Some dependency results are represented as 64-bit quantities. Narrow stores retain the low bits of their values; unsigned byte loads used as words are zero-extended.

Address offsets are measured in bytes. A word access covers four consecutive byte addresses, subject to the unknown byte order and memory model.

Conditional arithmetic examples use 32-bit bit-vector semantics: arithmetic and left shifts retain the low 32 bits at each represented 32-bit intermediate. This is not an assertion that signed overflow or signed shifts have been resolved at the instruction level. Address wrapping, alignment requirements, and invalid-access behavior remain **documented_only**.

## Behavior

### Dependency-driven preparation

Preparation is a long, straight-line sequence of indirect invocations. The first invocation, F-U001, receives the third input shifted left by 11 bits in both visible argument positions. Its target is derived from the incoming return-link value.

Later invocations, collectively designated F-U002, consume transformed dependency results, explicit zeros, repeated arguments, and secondary result-channel values. Some sites expose no explicit arguments; that does not establish the absence of implicit inputs.

Visible transformations include:

- Left shifts of dependency-produced words, frequently by eight bits.
- Separation of represented 64-bit results into low and high words.
- Unsigned 64-bit right shifts by 34, 36, and 44 bits.
- Extraction of bits 10–19 using a 20-bit mask followed by a right shift by 10.
- Masks retaining low fields of several widths, including 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, and 24 bits.
- Further shifts and scaling of those fields.

Width boundaries matter. A represented unsigned 64-bit result shifted right by 36 contributes its high word shifted right by four, not its low word. Under the conditional bit-vector interpretation, a word shifted left by 25 and then left by seven becomes zero because the first intermediate is already 32 bits wide.

Several explicitly zeroed intermediates remain zero through later transformations. Invocations receiving those zeros still occur. Unused visible results do not justify omitting calls, since dependency effects are unknown.

No direct data-memory dereference is visible in this preparation region. Dependencies may nevertheless read or modify memory, including globals used later. Their individual identities and a dependency-complete call trace remain **documented_only**.

### Opening memory effects

The following descriptive labels identify captured values rather than validated pointers:

| Label | Meaning |
|---|---|
| Final result | Last represented 64-bit dependency result before direct mutation |
| High base | High word of the final result |
| Shifted base | Earlier dependency-result low word shifted left by 25 |
| Scaled base | Another earlier dependency-produced word multiplied by `0x02000000` |
| Small field | A dependency-produced word shifted left by eight, masked with `0x7fff`, then shifted right by six |
| Small base | Twice the small field |
| Saved word | Another earlier dependency-produced word shifted left by eight |

The opening effects preserve this ordering:

- The shifted base is stored as a word at high base + `0x14`. Its byte at offset `9` is then written with the low byte of the high base.
- A word store places the scaled base at 256 times the final result’s low word + `0x54`. The byte at shifted base + `9` is then overwritten with the shifted base’s own low byte.
- A zero word is stored at scaled base + `0x14` before the byte read at scaled base + `5`.
- A zero word is stored at small base + `0x14`; the captured unsigned byte is then stored as a word at scaled base + `0x34`.
- The word at scaled base + `0x14` is read. Another zero word is subsequently stored at small base + `0x14`, followed by two separate byte stores of the captured word’s low byte at small base + `0x15`.

Thus, the word read is not an entry-state read. Prior writes may alias it. The two equal byte stores remain distinct represented accesses.

### Link-derived mutation

The function next reads the byte at absolute address `0x05`, writes the low byte of the small base at absolute address `0x01`, and uses the captured unsigned byte as an address base.

The word at that byte-derived base + `0x54` is captured before being replaced. Call this captured word the **anchor**. It is later used both as a stored value and as an address component.

Subsequent effects include:

- Byte writes at offsets `9`, `0x0d`, `0x11`, and `0x15`.
- Word accesses at offsets `4`, `0x14`, `0x24`, `0x34`, `0x44`, `0x54`, `0x64`, and `0x74`.
- Link loads whose values become later address bases.
- Self-referential word stores.
- Scaling of captured values by two, four, or eight.
- A word store at absolute address `0x54` and a byte store at absolute address `0x15`.

Repeated loads are not generally interchangeable with cached values. In particular, the representation reloads a byte through a link-derived address before following a word at the resulting unsigned-byte base + `0x74`. Intervening writes may affect that later load.

For the remaining contract:

- **Anchor-scaled base** means four times the captured anchor.
- **Followed word** means the word obtained by the later byte-derived access at offset `0x74`.
- **Double-link base** means twice the followed word.
- **Quadruple-link base** means four times the followed word.
- **Adjusted value** means anchor-scaled base minus `0x96`.
- **Adjusted base** means four times the adjusted value.

These names do not imply disjoint storage or a public structure layout.

Before the repeated-write region, the followed word controls three stores of `0x3a`: a byte at four times that word, a halfword at eight times it, and a word at four times it + `0x20`. The represented byte store precedes the halfword and word stores.

### Repeated-write phase and captured base

At adjusted base + `0x20`, the represented word-write sequence is:

- `0x20`;
- `0x2f`;
- the zero-extended byte freshly read from adjusted base + `4`;
- `0x20` again.

After this sequence, the separately stored 32-bit base is read and captured. Call its captured value the **external base**. **Only after that capture** does the function store word `0x20` at absolute address `0x60`.

This boundary has two consequences:

1. Earlier computed writes can affect the external-base capture if their storage overlaps its source.
2. The absolute store at `0x60` cannot retroactively change the captured external base, even if it overlaps that source.

Later repeated writes use the captured value, not a newly read version of its storage.

The first write group places the quadruple-link base at eight times the anchor + `0x54`. It then writes, in order, the low byte of the adjusted base at external base + `0x19`, the low byte of the external base at adjusted base + `0x11`, and eight times the anchor as a word at external base + `0x34`.

Following groups retain those two byte destinations and their captured byte values. Their first word destination instead uses eight times the adjusted value + `0x54`, and their last word value is eight times the adjusted value. The first word value remains the quadruple-link base.

This is an explicitly repeated straight-line region, not an input-controlled loop. Repetitions must not be collapsed into final-state assignments when access order or multiplicity is observable. The exact repetition count is **documented_only** in this contract; no exact-count acceptance claim is made.

### Final capture boundary at absolute address `0x3a`

After the repeated groups, two word captures occur before the absolute byte write:

- The word at absolute address `0xa9`, called the **absolute captured word**.
- The word at quadruple-link base + `0x74`, called the **captured link word**.

The byte at absolute address `0x3a` is then written with the low eight bits of eight times the adjusted value plus `0x30`.

Two further word captures occur **after** that byte write:

- Double-link base + `0x14`, called the **restoration word**.
- Adjusted base + `4`, called the **captured payload**.

The earlier captures retain their pre-write values. The later reads observe memory after the absolute byte write wherever aliasing and the actual memory model make that write relevant. Moving all four reads before the store would change the contract.

The function subsequently writes the low byte of the adjusted base at double-link base + `0x0d`, byte `0x6c` at adjusted base + `8`, and the low byte of the external base at adjusted base + `0x0d` and + `9`. It also establishes a word self-reference at double-link base + `0x54`.

### Absolute address `0x67`: three distinct reads

The low byte of the absolute captured word is written to absolute address `0x67`. That written byte is not a permanently cached value for the remaining phase.

The relevant read/write boundaries are:

| Boundary | Established ordering |
|---|---|
| First read of `0x67` | Follows the write to `0x67` and a word read at double-link base + `0x44`; supplies the byte assignment to absolute address `0x21` |
| Second read of `0x67` | Follows the intervening mutation region described below; supplies a byte store at adjusted base + `9` |
| Third read of `0x67` | Follows further computed writes and a word read; supplies another byte store at adjusted base + `9` |

Between the first and second reads, the word captured from double-link base + `0x44` supplies a byte at adjusted base + `0x10` and a word at external base + `0x54`. It also supplies a shifted byte at double-link base + `1`. The shift count is the low eight bits of the adjusted base.

The external base + `0x54` word is subsequently overwritten with the adjusted base. A word is then read at adjusted base + `0x40`. After that read, the captured payload is stored at external base + `0x74`, the newly read word is stored at adjusted base + `0x10`, and the double-link base is stored at external base + `0x44`. The second read of `0x67` follows these effects.

After the second read supplies its byte store, the low byte of the adjusted base is written at double-link base + `9`, and the absolute captured word is stored at adjusted base + `0x44`. A word is then read at adjusted base + `0x54`, establishing the **late base**. The third read of `0x67` follows that word read and supplies the second byte store at adjusted base + `9`.

Computed stores in these intervals may overlap `0x67`. Consequently, the three reads need not return the initially written byte or equal one another. These are distinct represented memory reads, not three uses of a preserved byte.

### Remaining mutation and dispatch

After the third `0x67` read, the low byte of the captured link word is stored at late base + `0x0d`. The restoration word is then stored at double-link base + `0x54`, restoring a previously captured value rather than reloading its original source.

A subsequent word read at adjusted base + `0x14` supplies a later byte at late base + `0x11`. Before that byte store, the function also writes the captured link word’s low byte at double-link base + `0x11`, writes the late base’s low byte at adjusted base + `9`, and stores the late base as a word at adjusted base + `0x44`.

The word read at double-link base + `4` establishes the **terminal parent**. A halfword `0x65` is stored at twice that parent before a word is read at terminal parent + `0x64`. This latter capture is the **terminal second argument**.

After the byte store at late base + `0x11`, word decimal `100` is stored at late base + `0x54`. It is a memory effect, not a return value.

The function then captures a word from terminal parent + `0x74` for the terminal fourth argument and another word from double-link base + `0x64` for a final address base. Remaining writes place the terminal parent’s low byte at double-link base + `9`, the terminal second argument at that final base + `0x54`, and the final base at twice the terminal second argument.

F-U003 is invoked afterward with `0x72`, the terminal second argument, the restoration word, and the terminal fourth argument. There is no visible rollback boundary before dispatch.

## Side effects

### Aliasing and layout

No pair of derived bases is established to be disjoint. Fixed-address storage can also overlap computed accesses.

Overlapping-width cases include bytes at offset `0x15` within words at offset `0x14`, bytes at offset `0x11` within words at offset `0x10`, and later word writes covering earlier byte writes.

Accordingly:

- Reads refer to memory at their represented positions in the effect order.
- Captured values remain distinct from later contents of their source locations.
- Equal-value duplicate stores are not automatically removable.
- A final-memory-image comparison alone does not establish preservation of access-sensitive behavior.

Offsets establish access locations, not record sizes, field meanings, packing, or lifetimes. Atomicity, volatility, device-memory behavior, and concurrent-observer guarantees are unknown.

### Ownership

No direct allocation, release, retain, or ownership-transfer operation is identifiable. Dependencies may perform such operations, but their contracts are unavailable.

No dependency is assumed to preserve globals, leave reachable memory unchanged, or own or release memory passed implicitly or explicitly. Exclusive ownership of any accessed region is not established.

### Preconditions and errors

Completion requires valid indirect targets and valid memory mappings for every reached access, including low absolute addresses and dynamically loaded address bases.

The visible body supplies no null, range, alignment, or capacity checks and no explicit error return. Failures can occur after partial mutation. Graceful recovery, rollback, exception behavior, and continuation to terminal dispatch are not promised.

## Dependencies

Only the subject function has a resolved opaque identity. The following contract-local IDs do not assert resolved target identities:

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-U001 | Initial return-link-derived invocation | Identity, ABI, results, side effects |
| F-U002 | Remaining preparation invocations | Individual identities, arguments beyond the visible representation, result-channel semantics, effects |
| F-U003 | Terminal four-argument invocation | Identity, control-transfer semantics, return behavior, effects |

F-U002 denotes unresolved invocation sites, not a claim that they share one implementation. Secondary result channels are not assumed to contain pointers, status codes, or preserved values.

## Acceptance examples

These are conditional static examples, not executed tests.

### Input transformation and truncation

With third input `1` and the stated bit-vector interpretation, F-U001 receives `0x800` twice. With third input zero, it receives zero twice; preparation is not skipped.

With third input `0x00200000`, the same interpretation yields zero in both positions after truncation. Retaining an unbounded intermediate would not match that conditional requirement.

### Capturing the external base

If the separately stored base contains value X when read, the repeated groups use X even if the subsequent absolute word store at `0x60` changes its storage. Conversely, if an earlier computed store changed that storage before the read, X is the post-change value, not necessarily the entry value.

### The `0x3a` capture boundary

If the byte write at `0x3a` overlaps the word at double-link base + `0x14`, the restoration-word capture includes its effect according to the target byte order. The already captured absolute word and captured link word are not recaptured.

An implementation that reads the restoration word before the byte write fails this conditional ordering requirement.

### Re-reading `0x67`

If the three reads of `0x67` obtain bytes A, B, and C at their respective boundaries, the assignment to `0x21` uses A, and the two subsequent byte stores at adjusted base + `9` use B and C respectively.

No requirement makes A, B, and C equal. Reusing the initially written byte fails whenever an intervening aliased store changes a later read.

### Narrow stores and dispatch

A captured word `0x12345678` used in a byte store contributes `0x78`, not the full word.

If terminal dispatch is reached, its first visible argument is `0x72`. The separate word store of decimal `100` does not define a function result.

If a reached access is invalid, successful completion and rollback are not acceptance requirements.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and subsystem identity.
- Accuracy of recovered boundaries, signatures, and implicit calling-convention details.
- Individual preparation dependency identities and the complete invocation trace.
- Meaning and validity of secondary result channels.
- Exact repeated-write count as an acceptance constraint.
- Instruction-level signed overflow, signed left-shift, and signed right-shift semantics.
- Variable right-shift behavior for counts of 32 or greater; no count masking to five bits is established.
- Endianness, alignment, address wrapping, and mapping validity.
- Atomicity, volatility, device-memory, concurrency, and fault behavior.
- Dependency allocation, retention, release, and global-state effects.
- Whether terminal dispatch behaves as a conventional returning call.

These limitations do not erase the established ordering of the external-base capture, the `0x3a` capture boundary, or the three distinct reads of `0x67`.

## Evidence

Evidence is limited to static inspection of the supplied representation. The prior draft and feedback are not independent proof.

The body supports the represented widths, constants, invocation patterns, memory-access relationships, capture boundaries, and final dispatch described here. Structured-state interpretations remain inference only.

No external implementation or dependency body was inspected. No runtime tests or observed execution results are claimed. No snapshot hash or trusted provenance value is supplied by this contract.
