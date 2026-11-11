<!-- Trusted provenance: F-c3cec9edb01d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3cec9edb01d

## Purpose

F-c3cec9edb01d repeatedly performs an ordered sequence of indirect calls, memory accesses, a software breakpoint, and changes to implicit execution state. There is no ordinary return or loop-exit condition in the supplied static representation.

Its application-level purpose is not established. The evidence does not justify identifying it as initialization, device control, record processing, or memory management. Terms such as “record base” below describe how a captured address is used, not a verified data structure.

This contract distinguishes explicit behavior from unresolved dependency behavior and machine semantics. All acceptance examples are conditional; no runtime tests have been performed.

## Interface

### Inputs and outputs

No explicit parameters or return value are declared. Execution nevertheless depends on implicit inputs:

- A 32-bit selector used to form the first indirect-call target.
- An unidentified 32-bit address context used for several offset accesses.
- An incoming auxiliary register value used in a late byte-store address.
- Two fixed global words supplying destination bases.
- Memory reached through those bases, absolute addresses, and dependency results.
- Auxiliary register-result channels associated with certain calls.

There is no normal output through a return value. Observable outputs are explicit stores, dependency invocations, the breakpoint, and changes to the address context and selector.

### Widths and representation

| Item | Established width or treatment |
|---|---|
| Byte access | 8 bits |
| Halfword access | 16 bits |
| Word access | 32 bits |
| Working address values | Represented as 32-bit values |
| Paired dependency result | 64 bits, split into numerical low and high 32-bit words |
| Unsigned halfword read | Zero-extended when used as a word |
| Halfword store | Retains the low 16 bits |
| Byte store | Retains the applicable 8-bit value |

Numerical low/high extraction does not establish memory byte order.

Unsigned 32-bit shifts have their usual bit-vector interpretation: right shifts insert zeros, and left shifts discard bits above bit 31. Several other expressions use reconstructed signed values, including additions, a left shift by two, a left shift by six, and multiplication by four. Their portable signed-overflow behavior is not established. Any interpretation as wrapping machine arithmetic remains conditional on the actual instruction semantics.

### Preconditions and failure model

Uninterrupted execution requires callable indirect targets, valid memory for each access width, any required alignment, returning dependencies, and breakpoint handling that permits continuation. The function performs no explicit checks establishing these conditions.

There is no structured error result, recovery branch, rollback, or cleanup path. A fault, non-returning dependency, or non-resuming breakpoint handler can prevent every subsequent operation.

## Behavior

### Ordering and dependency identifiers

F-000000000001 through F-000000000037 identify call occurrences in one iteration. These are opaque occurrence identifiers, not assertions that each occurrence has a distinct implementation.

“Captured” means obtained at the stated point and subsequently reused without an explicit reread. Dependencies may perform additional operations between the events described here. Their effects are not assumed absent.

### Opening calls and destination updates

At iteration entry, the function establishes a zero byte for a later store.

1. **F-000000000001** is invoked with two explicit zero arguments. Its target expression is the current selector plus `0x0ffe`.
2. A separate zero-valued byte used in a later argument expression is established.
3. **F-000000000002** and **F-000000000003** execute.
4. The function reads a destination base from the first fixed global word.
5. **F-000000000004** executes.
6. **F-000000000005** supplies a 64-bit paired result.
7. **F-000000000006** receives the first pair’s low word and the sum of its high word, low word, and the zero byte.

The zero term contributes no numerical change. The signed additions in the second argument do not independently establish portable overflow semantics.

After F-000000000006 returns, the function writes a byte from an auxiliary result-register channel to captured destination offset `0x11`.

**F-000000000007** then executes, followed by a zero-byte store to offset `0x15` of that same captured destination.

**F-000000000008** supplies a 16-bit result. The function stores it at offset `0x0c` of an address supplied through an auxiliary register-result channel.

### Second paired result and context-relative writes

**F-000000000009** supplies a second paired result. The function retains its low word and uses its high word as an address base.

The following stores occur in order:

1. A zero halfword at high-word address plus `0x2c`.
2. A word at address-context offset `0x26c`, containing the low word logically shifted right by 26.

The second store therefore contains a value from 0 through 63.

After **F-000000000010**, the function reads a word-sized destination base from address-context offset `0x320`. **F-000000000011** and **F-000000000012** execute before a word store to captured destination offset `0x54`. Its value is the second pair’s low word shifted left by two as an unsigned 32-bit value.

**F-000000000013**, **F-000000000014**, **F-000000000015**, **F-000000000016**, and **F-000000000017** execute in that order.

The function then reads an unsigned halfword from absolute address `0x21c48994`. Its zero-extended value becomes a base for subsequent word reads. No additional base or scaling is shown for those uses.

After **F-000000000018**, the function reads a word from address-context offset `0x280` and passes it to **F-000000000019**.

A software breakpoint with immediate `0x92` follows that call. No explicit memory access intervenes between its return and the breakpoint.

### Post-breakpoint reads and halfword stores

If execution resumes after the breakpoint:

- The result of F-000000000019 is shifted left by two for later address formation.
- The fixed constant `0x21c48960` is logically shifted right by 18 and multiplied by four, producing `0x21c4`.
- A word is read at offset `0x44` of the previously captured halfword-derived base. That word becomes the retained record base.

The first calculation involves a reconstructed signed operand; negative or overflowing cases require machine semantics not established here.

**F-000000000020** executes, followed by **F-000000000021**, which supplies a word result. The function then reads another word at offset `0x04` of the halfword-derived base. **F-000000000022** receives the result of F-000000000021 and this freshly read word.

Next:

1. A destination base is read from the second fixed global word.
2. **F-000000000023** executes.
3. A halfword is stored at that captured base plus the shifted result of F-000000000019. The stored value is the captured base’s low 16 bits.
4. **F-000000000024** executes.
5. A word is read from absolute address `0x2218`, obtained from `0x21c4 + 0x54`.
6. **F-000000000025** supplies an address.
7. Halfword `0x21c4` is stored at that address plus `0x04`.
8. **F-000000000026** and **F-000000000027** execute.
9. An unsigned halfword is read at offset `0x34` of the base obtained from `0x2218`.

The last halfword is retained as an unsigned index.

### Final distribution and address-context adjustment

**F-000000000028** supplies the third paired result.

Before changing the address context, the function:

1. Stores the pair’s low word at address-context offset `0x258`.
2. Reads an unsigned halfword at offset `0x14` of the pair’s high-word address.
3. Forms a left-shift-by-six value from the low word.
4. Forms a four-times-high-word value.

The latter two calculations use reconstructed signed values. Their behavior beyond representable signed ranges is unresolved without the machine instruction semantics.

The address context is then decreased by `0xf4`, or 244 bytes. No restoration is shown before repetition.

The third pair’s low word is stored at absolute address:

**four times the retained index, plus `0x74`.**

This address does not incorporate the address context.

The remaining sequence is:

1. **F-000000000029** executes; the zero-extended halfword read through the third pair’s high word is stored as a word at record-base offset `0x54`.
2. **F-000000000030** executes; that same value is stored as a word at record-base offset `0x64`.
3. **F-000000000031** executes; a halfword containing the record base’s low 16 bits is stored at the sum of the low-word-shifted-by-six value and an auxiliary address-register result from this call.
4. **F-000000000032** and **F-000000000033** execute.
5. A byte is read from absolute address five times the retained index.
6. **F-000000000034** executes.
7. The captured byte is stored at the sum of the zero-extended halfword obtained after F-000000000028, the incoming auxiliary register value, and `0x19`.
8. **F-000000000035** executes.
9. The four-times-high-word value is stored as a word at record-base offset `0x44`.
10. **F-000000000036** executes.
11. The selector is assigned `0x08712031`.
12. A word is read at the four-times-high-word address plus `0x44`.
13. **F-000000000037** is invoked with that word.
14. Execution repeats without an ordinary exit test.

The selector assignment precedes the final word read. If that read faults, the assignment has already been reached, but F-000000000037 has not.

For the unsigned 16-bit index:

- Four times the index plus `0x74` ranges from `0x74` through `0x40070`.
- Five times the index ranges from zero through `0x4fffb`.

These arithmetic bounds do not establish address validity.

## Side effects

### Memory and ownership

Explicit effects include 8-bit, 16-bit, and 32-bit stores through global-derived bases, context-derived bases, dependency results, auxiliary register channels, and absolute calculated addresses.

No explicit allocation, release, reference counting, ownership transfer, or buffer-bound validation is shown. Receiving or passing an address does not establish ownership or lifetime guarantees.

Aliasing is not excluded. Earlier stores or dependency effects may change storage used by later reads, including address-selection data. Later reads occur at their specified positions and must not be treated as if all inputs were captured at entry.

Conversely, an explicitly captured value is reused without an explicit reread of its original storage. This does not imply that the original storage remains unchanged or that every physical register survives arbitrary dependency behavior.

### State and control

- The address context decreases by 244 bytes whenever execution reaches its adjustment.
- The selector becomes `0x08712031` after F-000000000036 and before the final explicit read.
- The breakpoint occurs once per iteration reaching it.
- No ordinary exit, rollback, or cleanup is present.

The represented next-iteration first-target expression, using the assigned selector, evaluates to `0x0871302f`. This is a static selector calculation, not a guarantee that unknown dependencies or exception handling preserve all relevant machine state.

## Dependencies

### Established target relationships

The target arithmetic establishes the following equal target values:

| Call occurrences | Established relationship |
|---|---|
| F-000000000005 and F-000000000006 | Equal targets |
| F-000000000009 and F-000000000013 | Equal targets |
| F-000000000017 and F-000000000024 | Equal targets |
| F-000000000019 and F-000000000031 | Equal targets |
| F-000000000025 and F-000000000032 | Equal targets |

These relationships are statically known, not unresolved implementation guesses. They do not establish identical arguments, results, side effects, or behavior across invocations.

The first occurrence has the selector-dependent target expression described above. The supplied constant arithmetic also determines target values for occurrences 2 through 37. What remains unknown is the actual implementation and interface behind those target values, not the existence of the listed arithmetic equalities.

### Visible interfaces

| Occurrences | Established use |
|---|---|
| F-000000000001 | Two zero arguments; selector-derived target |
| F-000000000002–F-000000000004 | No explicit arguments |
| F-000000000005 | First paired result |
| F-000000000006 | Low-word and high-plus-low arguments; subsequent auxiliary byte channel |
| F-000000000007 | Precedes zero-byte store |
| F-000000000008 | Halfword result and auxiliary destination channel |
| F-000000000009 | Second paired result |
| F-000000000010–F-000000000018 | Calls interleaved with the accesses specified above |
| F-000000000019 | Word argument from context offset `0x280`; result later shifted |
| F-000000000020 | Follows record-base read |
| F-000000000021 | Word result |
| F-000000000022 | Two word arguments |
| F-000000000023–F-000000000024 | Surround global-derived halfword store |
| F-000000000025 | Address result |
| F-000000000026–F-000000000027 | Precede index read |
| F-000000000028 | Third paired result |
| F-000000000029–F-000000000030 | Precede record word stores |
| F-000000000031 | Auxiliary address contribution |
| F-000000000032–F-000000000036 | Interleaved with final byte transfer and word update |
| F-000000000037 | Final word argument after selector assignment and read |

“No explicit arguments” does not exclude hidden inputs. No dependency is assumed to preserve globals, unrelated registers, memory contents, or allocation ownership.

The breakpoint is a separate machine operation, not an ordinary dependency call.

## Acceptance examples

These examples specify conditional behavior, not observed execution.

1. **Selector arithmetic.** If the incoming selector is `0x00001000`, the first target expression produces `0x00001ffe`. After the explicit assignment to `0x08712031`, the corresponding opening-target expression produces `0x0871302f`, provided execution reaches it without an external alteration of that state.

2. **Equal targets, different invocations.** F-000000000005 and F-000000000006 must have equal target values under the supplied arithmetic. Acceptance must not require equal results: the latter invocation supplies explicit arguments and occurs later.

3. **Top-six-bit extraction.** If the second pair’s low word is `0xfc000003`, the store at context offset `0x26c` contains 63. Its unsigned left-shift-by-two store contains `0xf000000c`.

4. **Zero extension.** If the halfword read through the third pair’s high word is `0xffff`, the subsequent word stores at record offsets `0x54` and `0x64` contain `0x0000ffff`.

5. **Halfword truncation.** If the second global supplies base `0x12345678`, its address-derived halfword store contains `0x5678`. The destination calculation still uses the full captured base.

6. **Captured versus fresh values.** Changing the storage holding a previously captured destination base does not introduce a reread in this sequence. A later explicitly scheduled read may observe intervening writes or dependency effects.

7. **Breakpoint does not resume.** If breakpoint `0x92` prevents continuation, the post-breakpoint reads, context adjustment, selector assignment, and final calls are not reached.

8. **Absolute index addressing.** For retained index 3, the low-word store uses absolute address `0x80`; the later byte source is absolute address `0x0f`. Neither acquires an implicit context or table base.

9. **Final read faults.** If F-000000000036 returns and the final word read faults, the selector assignment to `0x08712031` precedes that fault. F-000000000037 is not invoked unless the read completes.

10. **Repeated context adjustment.** If two adjustments execute without external alteration or arithmetic wrap, the context is 488 bytes below its initial value.

11. **Store events, not permanent contents.** A dependency may overwrite an earlier destination. Acceptance concerns the explicit store and its ordering, not unsupported persistence of the resulting memory image.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data-structure meanings.
- Actual implementations behind calculated targets.
- Calling conventions, hidden inputs, and auxiliary result-channel guarantees.
- Dependency memory effects, register preservation, ownership behavior, and return behavior.
- Provenance and validity of the address context and incoming auxiliary register.
- Whether the context adjustment faithfully represents runtime state.
- Signed-overflow, signed-shift, and effective-address semantics.
- Endianness, alignment, memory permissions, and volatile or device-memory behavior.
- Breakpoint handling and resumability.
- Allocation lifetimes, concurrency, and synchronization.

Small absolute addresses and address-derived stored values are established expressions, but their practical meaning is unresolved. They are not repaired into more conventional table accesses or data structures.

Inferred intent is limited to descriptive grouping: the accesses resemble distribution of dependency results across several destinations. No application-level interpretation is accepted as established behavior.

## Evidence

The supplied static representation establishes:

- An unconditional repeated sequence containing 37 indirect-call occurrences.
- The selector-plus-`0x0ffe` first-target expression.
- The selector assignment `0x08712031`, before the final word read.
- The five equal-target relationships listed above.
- Three paired-result uses and explicit byte, halfword, and word accesses.
- Breakpoint immediate `0x92`.
- A 244-byte address-context decrement.
- No explicit validation, ordinary return, allocation, release, or structured error path.

The constant reductions, subsequent selector-derived target, and unsigned-index bounds are arithmetic deductions from that evidence. Dependency implementations and application intent are not established.

No runtime tests have been performed.
