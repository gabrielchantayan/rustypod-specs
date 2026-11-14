<!-- Trusted provenance: F-fb88a966bfd5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fb88a966bfd5

## Purpose

The established behavior is an ordered sequence of indirect dependency calls, computed-address memory accesses, and state-clearing writes. Three nominal 32-bit arguments and additional incoming machine state contribute to call arguments, stored values, and effective addresses.

No application-level purpose is established. Link-like writes and repeated clears suggest state reconstruction, but do not establish object initialization, collection management, allocation, resource release, or a protocol.

The supplied static representation ends at an invalid-instruction termination marker and reports incomplete control-flow decoding. This contract describes the visible path, not a proven complete executable routine.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use on the visible path. |
| Second argument | 32 bits | Argument to the first indirect call. |
| Third argument | 32 bits | Stored at two computed word destinations, with dependency calls between the stores. |

The unused first argument is not proven irrelevant to the original calling convention or implicit machine-state inputs.

### Additional incoming state

The visible path also consumes:

- An unsigned 32-bit value, logically shifted right by three for the first word store.
- A signed integer value, multiplied by `0x80000` to derive one publication destination.
- A link-related machine value used to compute the first indirect target.

Their origins and validity requirements are unknown. They are incoming-state requirements, not independently recovered public parameters.

### Outputs and preconditions

The nominal interface has no return value. No ordinary return is visible.

Observable effects include dependency invocations, explicit byte and word accesses, dependency-produced effects, and the terminal invalid-instruction marker if execution reaches it.

To traverse the entire visible path, every reached call target must support execution and every explicit access must be permitted. Arithmetic and calling-convention interpretations must also support the represented operations. No checks enforce these conditions.

### Widths and address boundaries

A **byte** is eight bits; a **word** is 32 bits. One dependency result is represented as 64 bits and consumed as low and high 32-bit halves. This does not establish a true 64-bit API return.

Selector bytes are zero-extended before address formation:

- A selector plus `0x11` produces a byte-store address from `0x11` through `0x110`, inclusive.
- The retained-selector plus `0x44` produces a **word-store starting address** from `0x44` through `0x143`, inclusive.
- Under ordinary byte-addressed, contiguous word-storage semantics, that word store touches four bytes. Across all selector values, its possible footprint extends through `0x146`.

These bounds apply only to those selector-derived accesses. They do not bound selector-read addresses, token-derived accesses, or dependency accesses. Selector zero and selector `0xff` receive no visible special treatment. Selector-derived word addresses need not be aligned.

Unsigned 32-bit shifts and arithmetic use word-width results; unsigned multiplication can discard high bits. Some later values are held in signed integer intermediates. Signed overflow, overflowing signed left shifts, and out-of-range unsigned-to-signed conversions are not independently resolved. Address interpretation, endianness, alignment requirements, and atomicity remain unknown.

Byte publications retain only the low eight bits of the computed value.

## Behavior

### Call notation

Dependency IDs identify call occurrences, not proven distinct implementations:

- **Five-call group:** occurrence 1 receives the stated arguments; occurrences 2–5 have no explicit arguments. Occurrence 5 supplies any stated result.
- **Seven-call forwarding group:** occurrence 1 receives the stated arguments; occurrence 2 returns a word supplied to occurrence 3; occurrences 4–7 have no explicit arguments. Occurrence 7 supplies any stated result. Additional occurrence-3 arguments are identified explicitly.
- **Nine-call forwarding group:** occurrence 2 returns a word supplied to occurrence 3, and occurrence 4 returns a word supplied to occurrence 5. Occurrences 6–9 have no explicit arguments. Occurrence 9 supplies the stated result.

“No explicit arguments” does not establish absence of implicit machine inputs. Unused results do not make calls removable.

### 1. Initial linked-word updates

Seven-call forwarding group **F-d01.1–F-d01.7** begins with the second explicit argument and yields a **first index**. After the group, the incoming unsigned value shifted logically right by three is stored as a word at four times that index plus `0x14`.

Five-call group **F-d02.1–F-d02.5** begins with four times the first index and yields a **second index**. Four times the second index is retained as a **link value** and stored at sixteen times the second index plus `0x14`.

Only after that store, a word is read from link value plus `0x14`. Its logical right shift by three begins five-call group **F-d03.1–F-d03.5**. The resulting **third index** selects a word destination at four times that index plus `0x14`, where the retained link value is stored.

The post-store load is an actual memory read. The two addresses may alias; no earlier value may replace the load without additional assumptions.

### 2. Principal token and third-argument stores

Five-call group **F-d04.1–F-d04.5** begins with four times the third index and yields an unsigned **principal token**.

The third explicit argument is then stored as a word at the incoming signed integer multiplied by `0x80000`, plus `0x24`.

Two retained quantities are derived:

- **Principal base:** principal token multiplied by sixteen, with the unsigned product subsequently represented in an integer address intermediate.
- **Reduced token:** principal token masked with `0x3fffffff`, then logically shifted right by one. Its range is zero through `0x1fffffff`.

Five-call group **F-d05.1–F-d05.5** begins with two arguments: principal base and principal token shifted left by two. After all five calls, the third explicit argument is stored at principal base plus `0x24`.

A word is then read at principal base plus `0x74`. That value begins five-call group **F-d06.1–F-d06.5**, whose result is retained as the **shift seed**.

The two argument stores are not an atomic pair. Dependencies execute between them, and either destination can overlap other relevant storage.

### 3. Principal selector publications

After F-d06.5, a byte is read at principal base plus one. At the zero-extended selector plus `0x11`, the routine stores the low byte of the principal token masked with `0x3fffffff` and shifted right by twenty. The stored byte corresponds to principal-token bits 20–27.

Seven-call forwarding group **F-d07.1–F-d07.7** begins with the shift seed shifted left by two and yields a **clear token**.

The selector at principal base plus one is then reread, and zero is stored as a byte at that selector plus `0x11`. Dependencies may have changed the selector; this destination need not match the preceding publication.

Five-call group **F-d08.1–F-d08.5** begins with the clear token shifted left by two. Its last result is not explicitly used.

### 4. Reduced-token clears and retained selector

After F-d08.5, the explicit access order is:

1. Store a zero byte at reduced token plus `0x0f`.
2. Read a selector byte at **middle base** plus one, where middle base is reduced token multiplied by `0x400`.
3. Store a zero byte at that selector plus `0x11`.

No dependency call separates these accesses. Aliasing can therefore make the first store affect the selector load, or make a selector-directed store affect subsequent accesses.

Seven-call forwarding group **F-d09.1–F-d09.7** begins with four arguments: three copies of the shift seed shifted left by twenty-three, followed by the shift seed shifted left by twenty-five. It yields a **middle token**.

The selector at middle base plus one is reread and retained. Zero is stored as a byte at retained selector plus `0x11`.

Five-call group **F-d10.1–F-d10.5** begins with middle token shifted left by two. After all five calls, the routine stores a zero word at **retained selector plus `0x44`**, without another selector read. It then clears a word at absolute address `0x24`.

The retained-selector word destination ranges from `0x44` to `0x143`, not from `0x11` to `0x110`.

### 5. Word-clearing progression

Nine-call forwarding group **F-d11.1–F-d11.9** begins with zero, `0x98`, middle base, and `0x98`. Its result is a **first clearing index**.

A zero word is stored at four times that index plus `0x44`, followed by a zero word at absolute address `0x24`.

Five-call group **F-d12.1–F-d12.5** begins with four times the first clearing index and produces the represented 64-bit result.

**F-d13.1** receives its low 32-bit half interpreted as an integer and shifted left by two. After that call, a word is cleared at twice the high 32-bit half interpreted as an integer, plus `0x44`. The word at absolute address `0x24` is then cleared.

Group **F-d14.1–F-d14.7** is not the ordinary forwarding pattern:

- F-d14.1 receives zero, a secondary machine result associated with F-d13.1, and that secondary result shifted left by two.
- F-d14.2 has no explicit arguments.
- F-d14.3 receives reduced token masked with `0x3fffff` and shifted left by seven.
- F-d14.4–F-d14.7 have no explicit arguments.

The secondary result’s ABI meaning is unknown. F-d14.7 yields a **second clearing index**. A word at four times that index plus `0x44` is cleared, followed by another clear at absolute address `0x24`.

### 6. Link-word clears

Five-call group **F-d15.1–F-d15.5** begins with four times the second clearing index. Its result shifted left by two is passed to **F-d16.1**.

F-d16.1 yields a **third clearing index**. A zero word is stored at four times that index plus `0x14`.

Five-call group **F-d17.1–F-d17.5** begins with four arguments: three copies of the third clearing index, followed by reduced token masked with `0x3fffff` and shifted left by seven.

Its result is a **fourth clearing index**. A zero word is stored at four times that index plus `0x14`.

### 7. Link-like reconstruction

Five-call group **F-d18.1–F-d18.5** begins with four times the fourth clearing index and yields a **first reconstruction index**.

Four times that index is retained as a **reconstructed link** and stored at sixteen times the index plus `0x14`.

Five-call group **F-d19.1–F-d19.5** begins with the reconstructed link and yields a **second reconstruction index**. The reconstructed link is stored at four times this second index plus `0x14`.

Five-call group **F-d20.1–F-d20.5** begins with four times the second reconstruction index and yields a **third reconstruction index**. Four times that index is stored at sixteen times that index plus `0x14`.

These writes establish neither pointer validity nor ownership transfer.

### 8. Masked-value publication

Seven-call forwarding group **F-d21.1–F-d21.7** begins with two arguments: four times and sixteen times the third reconstruction index.

F-d21.3 receives both the word forwarded from F-d21.2 and an additional argument: reduced token masked with `0x3fffff` and shifted left by four.

The final result is a **publication index**. Reduced token masked with `0x3fffff` and multiplied by `0x80` is stored as a word at four times that index plus `0x14`.

Five-call group **F-d22.1–F-d22.5** begins with four times the publication index and yields an unsigned **first terminal token**.

### 9. Terminal byte publications

A **terminal base** is derived from reduced token multiplied by `0x4000000`. The multiplication is unsigned word arithmetic before representation in the integer address intermediate.

A selector byte is read at terminal base plus one. The low byte of first terminal token shifted left by two is stored at selector plus `0x11`.

Seven-call forwarding group **F-d23.1–F-d23.7** begins with first terminal token shifted left by two and yields a **terminal forwarding index**.

The selector at terminal base plus one is reread. At selector plus `0x11`, the routine stores the low byte of first terminal token masked with `0x3fffffff` and shifted right by twenty.

Five-call group **F-d24.1–F-d24.5** begins with terminal forwarding index shifted left by two and yields an unsigned **second terminal token**.

The selector at terminal base plus one is reread and retained. The low byte of four times second terminal token is stored at retained selector plus `0x11`.

Seven-call forwarding group **F-d25.1–F-d25.7** begins with the retained selector zero-extended to a word, **not** the second terminal token. It yields a **final forwarding index**.

A new selector is read at eight times second terminal token plus one, using unsigned word arithmetic. The low byte of second terminal token masked with `0x3fffffff` and shifted right by twenty is stored at that selector plus `0x11`.

Five-call group **F-d26.1–F-d26.5** begins with final forwarding index shifted left by two. Its last result is unused. The visible path then reaches the invalid-instruction marker.

## Side effects

Explicit accesses comprise byte loads and stores and 32-bit word loads and stores. No wider explicit memory store is visible.

Absolute address `0x24` receives four separate explicit word clears: after the retained-selector word clear and after each of three subsequent clearing operations. Dependencies and other accesses separate these writes; they cannot be treated as one initialization.

All computed regions may overlap each other or dependency-maintained storage. No non-aliasing, synchronization, memory-barrier, or concurrent-access guarantee is established.

No explicit allocation, deallocation, reference-count action, or resource transfer is identified. Unknown dependencies may perform such actions; their absence from the visible operations is not evidence of their absence overall.

No visible validation, error return, exception translation, rollback, or recovery path exists. If an access or dependency prevents progress, preceding effects have already occurred.

## Dependencies

F-d01.1 through F-d26.5, with group sizes specified above, are opaque occurrence identifiers. They do not identify known APIs or prove that different occurrences invoke different implementations.

The first target depends on incoming link-related state; later targets are formed from represented address anchors and offsets. Target formation does not establish an application-level dependency contract.

Dependencies may change memory, globals, selector locations, or other machine state. They are not assumed to preserve globals, retain or release passed resources, return normally, or obey any particular ownership convention.

Values described as retained across calls follow the supplied representation. Correct recovery of the original register-preservation convention remains unverified.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial shift.** With incoming unsigned value `0x40` and first index `3`, the first explicit word store writes `8` at `0x20`, assuming permitted access.

2. **Aliasing load.** With second index zero, the link store and subsequent word load both address `0x14`. Under ordinary memory semantics, the load observes the just-written zero before its right shift.

3. **Publication order.** If the third-argument destinations are distinct, F-d05 can observe the first destination already written while the second has not yet received this routine’s later store. Dependency writes can alter either observation.

4. **Selector reread.** If the principal selector is `2` for its first publication and becomes `7` during F-d07, the byte destinations are respectively `0x13` and `0x18`.

5. **Retained selector.** If the selector retained before F-d10 is `5`, the later word clear starts at `0x49`, even if F-d10 changes the original selector location.

6. **Boundary distinction.** Selector `0xff` selects byte address `0x110` for a selector-plus-`0x11` publication. The retained-selector word clear instead starts at `0x143`; under contiguous four-byte word semantics, it touches `0x143` through `0x146`. Selector zero produces corresponding starting addresses `0x11` and `0x44`.

7. **Byte truncation.** First terminal token `0x41` produces byte `0x04` for its low-byte-times-four publication, while the unsigned word argument beginning F-d23 is `0x104`.

8. **Masked bounds.** Reduced token never exceeds `0x1fffffff`. Masking it with `0x3fffff` and shifting left by seven produces at most `0x1fffff80`.

9. **Unsigned product.** Reduced token `0x40` produces unsigned terminal-base product zero after word-width truncation. If the represented address arithmetic applies, the selector load is at address one. Accessibility is not established.

10. **Endpoint.** Reaching the final marker does not satisfy a successful-return requirement. No ordinary return is visible.

## Uncertainties

The following remain **documented_only**:

- Completeness and executability of the visible control flow.
- Original instruction-set state, calling convention, pointer width, and endianness.
- Origins and preservation rules of additional incoming state.
- Signed overflow, signed-shift behavior, and out-of-range integer conversions.
- Dependency identities, failure modes, ownership rules, and global effects.
- Whether the represented 64-bit result is a true API return or recovered multi-register state.
- Meaning of the secondary machine result after F-d13.1.
- Memory mapping, access permissions, alignment behavior, and device-memory semantics.
- Whether apparent offsets describe fields, tables, or unrelated storage.
- Actual behavior beyond the decoding failure and the machine manifestation of the terminal marker.

Initialization and reconstruction remain inferred intent, not established purpose.

## Evidence

Established static evidence consists of the supplied representation’s explicit argument uses, call ordering, arithmetic widths, memory accesses, selector rereads, retained selector, four absolute-address clears, and terminal marker.

The selector-bound distinction follows directly from separate eight-bit-selector additions: `0x11` for byte-store destinations and `0x44` for the retained-selector word-store destination.

Control-flow and overlapping-storage warnings limit claims about a complete executable contract. No runtime tests, executable observations, or external verification have been performed.
