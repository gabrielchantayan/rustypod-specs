<!-- Trusted provenance: F-f161a6bf8fd9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f161a6bf8fd9

## Purpose

F-f161a6bf8fd9 represents an unconditional loop containing indirect calls, memory reads, memory writes, a software breakpoint, and changes to implicit execution state. No application-level purpose is established.

A complete iteration contains 37 indirect call occurrences and 15 explicit memory stores. Completion is conditional: a dependency, invalid memory access, or breakpoint handling may prevent execution from reaching later operations. The representation contains no normal return, explicit exit condition, validation branch, or recovery path.

This contract describes the represented operations and their ordering. It does not establish that the complete sequence is executable in an unknown memory and calling environment.

## Interface

**Explicit parameters:** None represented.

**Return:** No ordinary return or return value represented.

**Implicit inputs and state:**

- A link-register-like word used to calculate the first indirect-call target.
- An implicit frame base used for word accesses and later decreased by `0xf4`.
- An incoming register-like value used in a late byte-store address.
- Two global-held words, at represented locations `0x087120b4` and `0x08712120`.
- Memory contents at fixed, frame-relative, and computed addresses.
- Ordinary dependency results and additional register outputs associated with particular calls.

These are represented inputs, not a recovered calling convention. In particular, an additional register output is not necessarily a separately declared return value.

**Widths and layout:**

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Paired result: 64 bits, interpreted as low and high 32-bit words.
- Unsigned halfword reads produce values from 0 through 65535.
- Halfword stores retain the low 16 bits.
- Word stores of zero-extended halfwords retain the full numerical value without sign extension.

Offsets identify byte displacements. No object type, array extent, structure boundary, alignment guarantee, or non-aliasing guarantee is established. The frame base is an implicit address base, not proof of an ordinary stack frame.

For numerical acceptance examples, word arithmetic uses a 32-bit bit-pattern interpretation: left shifts and additions retain the low 32 bits. This interpretation does not resolve the original execution semantics of signed-looking overflow or pointer wraparound.

No ownership transfer, allocation responsibility, lifetime guarantee, or release obligation is established.

## Behavior

### Execution and ordering

F-00000001 through F-00000037 identify successive call occurrences within one iteration. They are documentation IDs, not recovered symbol identities. Their target calculations are specified separately under Dependencies.

All ordering below concerns operations directly represented in F-f161a6bf8fd9. Dependencies may introduce additional effects. A value explicitly captured before a call is subsequently used as that captured value in the representation; this does not imply that the callee preserves the memory from which it was read.

### Opening calls and byte writes

A byte-valued temporary is initialized to zero before F-00000001. F-00000001 receives two explicit zero arguments and targets the current link-like word plus `0xffe`.

A separate addition byte is initialized to zero after that call. F-00000002 and F-00000003 follow. The word held at `0x087120b4` is then read and retained as an address base before F-00000004.

F-00000005 produces a paired result. F-00000006 receives its low word and the word-sized sum of its high word, low word, and the zero addition byte. This is not a 64-bit concatenation or addition. An additional byte output associated with F-00000006 is stored at the retained global-derived base plus `0x11`.

F-00000007 follows that store. The previously initialized zero byte is then stored at the same retained base plus `0x15`. Neither store reloads the global-held base.

### Halfword initialization and first retained pair

F-00000008 supplies a halfword result and an additional address-like register output. The halfword is stored at that additional output plus `0x0c`.

F-00000009 supplies another paired result, retained for several later operations:

- A zero halfword is stored at its high word plus `0x2c`.
- Its low word, logically shifted right by 26 bits, is stored as a word at frame-base offset `0x26c`. The stored value is between 0 and 63.
- After F-00000010, a word is read from frame-base offset `0x320` and retained as an address.
- F-00000011 and F-00000012 occur before the earlier pair’s low word, shifted left by two bits, is stored at that retained address plus `0x54`.

The final store in this group uses the result of F-00000009, not a result from F-00000010 through F-00000012.

The fixed word `0x21c48960` is established before F-00000010 and retained for the subsequent fixed-address calculations.

### Fixed-address read and breakpoint

F-00000013 through F-00000017 occur in order. An unsigned halfword is then read at `0x21c48960 + 0x34`, namely `0x21c48994`.

F-00000018 follows that read. A word is then read at frame-base offset `0x280` and passed to F-00000019. Its word result is retained.

A software breakpoint with immediate `0x92` occurs next. Continuation is not established. Every operation below is conditional on execution proceeding beyond it.

### Post-breakpoint addressing and captured global

The result of F-00000019 is shifted left by two bits for use as a byte displacement. Separately, logically shifting `0x21c48960` right by 18 bits and multiplying by four produces `0x21c4`.

The unsigned halfword read before the breakpoint is used directly as an address component. A word at that halfword value plus `0x44` is read and retained as a record address. No represented operation restores any omitted high address bits.

F-00000020 and F-00000021 follow. A word at the same zero-extended halfword value plus `0x04` is then read. F-00000022 receives the result of F-00000021 and this newly read word, in that order.

Next, the word held at `0x08712120` is read and retained. F-00000023 occurs before a halfword store whose:

- Destination is the captured global word plus the shifted result of F-00000019.
- Payload is the low 16 bits of that captured global word.

The payload is not the displacement or the destination’s low half. The global is not reloaded after F-00000023.

F-00000024 follows. A word is read from `0x21c4 + 0x54`, namely `0x2218`, and retained as another address base. F-00000025 then supplies an address-like word result. The halfword `0x21c4` is stored at that result plus four.

After F-00000026 and F-00000027, an unsigned halfword is read at the retained address obtained from `0x2218`, plus `0x34`. This halfword later drives two absolute index-derived accesses.

### Final pair, frame adjustment, and record updates

F-00000028 produces a paired result. Its low word is stored at frame-base offset `0x258`, using the frame base active before the explicit decrease. An unsigned halfword is then read at the pair’s high word plus `0x14`.

The remaining calculations use:

- The low word of this pair shifted left by six bits.
- Four times the high word of this pair.
- The zero-extended halfword loaded through the high word.
- The unsigned halfword loaded after F-00000027.

The implicit frame base is decreased by `0xf4`, or 244 bytes. No matching restoration is represented.

The pair’s low word is stored at four times the halfword loaded after F-00000027, plus `0x74`. This is an absolute derived address; no separate array or table base is represented.

The retained record address from the first post-breakpoint word read receives several writes, separated by calls:

- F-00000029 precedes a word store at record offset `0x54`.
- F-00000030 precedes a word store at record offset `0x64`.
- Both stores contain the same zero-extended halfword loaded through F-00000028’s high word.

After F-00000031, the low 16 bits of the retained record address are stored at the sum of the pair’s low word shifted left by six and an additional address-like register output associated with F-00000031.

F-00000032 and F-00000033 follow. One byte is then read at five times the halfword loaded after F-00000027. Again, no separate table base is represented.

F-00000034 separates that byte read from its write. The byte is written at the sum of the zero-extended halfword loaded through F-00000028’s high word, the incoming register-like value, and `0x19`.

F-00000035 follows. Four times F-00000028’s high word is stored as a word at retained record offset `0x44`. F-00000036 then occurs.

The link-like state is explicitly assigned `0x08712031`. After that assignment, a word is read at four times F-00000028’s high word plus `0x44` and passed to F-00000037.

If control reaches the loop continuation, another iteration begins. The explicit link assignment would make its first target `0x0871302f` if no intervening dependency effect changes the relevant state. Likewise, absent an unresolved restoration or modification, the next iteration begins with a frame base 244 bytes lower.

## Side effects

### Direct memory stores

There are **15 explicit stores per complete iteration**, comprising:

| Width | Destination or role |
|---|---|
| Byte | First captured global-derived base plus `0x11`; additional output from F-00000006 |
| Byte | Same base plus `0x15`; zero |
| Halfword | Additional output from F-00000008 plus `0x0c` |
| Halfword | F-00000009 high word plus `0x2c`; zero |
| Word | Frame offset `0x26c`; high six bits of F-00000009 low word |
| Word | Frame-loaded address plus `0x54`; F-00000009 low word shifted left by two |
| Halfword | Second captured global plus displacement; low half of that global |
| Halfword | F-00000025 result plus four; `0x21c4` |
| Word | Frame offset `0x258`; F-00000028 low word |
| Word | Four times a loaded halfword plus `0x74`; F-00000028 low word |
| Word | Retained record plus `0x54`; zero-extended halfword |
| Word | Retained record plus `0x64`; same zero-extended halfword |
| Halfword | Shifted low result plus additional output from F-00000031 |
| Byte | Loaded halfword plus incoming register-like value plus `0x19` |
| Word | Retained record plus `0x44`; four times F-00000028 high word |

The two explicit frame-relative stores precede the explicit frame-base decrease.

Addresses may alias each other, globals, frame-relative storage, dependency-visible memory, or later read locations. The writes therefore cannot be treated as an unordered set of independent field updates. Endianness and alignment requirements are unresolved.

### Other state and control effects

A complete iteration also includes 37 indirect calls, the breakpoint, the frame-base decrease, and the fixed link-like assignment.

No dependency is established to preserve global state, memory, implicit registers, or ownership. Absence of a visible allocation or release operation does not establish that dependencies allocate or release nothing.

## Dependencies

The following table preserves the represented target arithmetic. All numbers are hexadecimal. Constant expressions are target calculations, not guesses about symbol identity. No instruction-set mode-bit clearing, target normalization, or alternative address interpretation is established.

| Occurrence | Represented target | Established interface role |
|---|---|---|
| F-00000001 | Current link-like word + `0xffe` | Two explicit zero arguments |
| F-00000002 | `0x08711f67 + 0xe9a` | Unrecovered |
| F-00000003 | `0x08711f6d + 0xe9a` | Unrecovered |
| F-00000004 | `0x08711f71 + 0xe9a` | Unrecovered |
| F-00000005 | `0x08711f75 + 0xe84` | Paired result |
| F-00000006 | `0x08711f79 + 0xe80` | Two word arguments; additional byte output |
| F-00000007 | `0x08711f7d + 0xedc` | Unrecovered |
| F-00000008 | `0x08711f81 + 0xeda` | Halfword result; additional address-like output |
| F-00000009 | `0x08711f85 + 0xeee` | Paired result |
| F-00000010 | `0x08711f89 + 0xed6` | Unrecovered |
| F-00000011 | `0x08711fa5 + 0xed4` | Unrecovered |
| F-00000012 | `0x08711fa9 + 0xed2` | Unrecovered |
| F-00000013 | `0x08711fad + 0xec6` | Unrecovered |
| F-00000014 | `0x08711fb1 + 0xeb2` | Unrecovered |
| F-00000015 | `0x08711fb5 + 0xe8a` | Unrecovered |
| F-00000016 | `0x08711fb9 + 0xeac` | Unrecovered |
| F-00000017 | `0x08711fbd + 0xecc` | Unrecovered |
| F-00000018 | `0x08711fc1 + 0xf08` | Unrecovered |
| F-00000019 | `0x08711fc5 + 0xf18` | Frame-loaded word argument; word result |
| F-00000020 | `0x08711fc9 + 0xed4` | Unrecovered |
| F-00000021 | `0x08711fdd + 0xed0` | Word result used by F-00000022 |
| F-00000022 | `0x08711fe1 + 0xece` | Two explicit word arguments |
| F-00000023 | `0x08711fe5 + 0xe98` | Unrecovered |
| F-00000024 | `0x08711fe9 + 0xea0` | Unrecovered |
| F-00000025 | `0x08711fed + 0xece` | Address-like word result |
| F-00000026 | `0x08711ff1 + 0xef0` | Unrecovered |
| F-00000027 | `0x08711ff5 + 0xf04` | Unrecovered |
| F-00000028 | `0x08711ff9 + 0xf0c` | Paired result |
| F-00000029 | `0x08711ffd + 0xeca` | Unrecovered |
| F-00000030 | `0x08712011 + 0xec8` | Unrecovered |
| F-00000031 | `0x08712015 + 0xec8` | Additional address-like output |
| F-00000032 | `0x08712019 + 0xea2` | Unrecovered |
| F-00000033 | `0x0871201d + 0xe82` | Unrecovered |
| F-00000034 | `0x08712021 + 0xea4` | Unrecovered |
| F-00000035 | `0x08712025 + 0xede` | Unrecovered |
| F-00000036 | `0x08712029 + 0xec4` | Unrecovered |
| F-00000037 | `0x0871202d + 0xece` | Word argument read through scaled high result |

“Unrecovered” describes the dependency contract, not its represented target arithmetic. A call without explicit arguments may still consume implicit register, global, or memory inputs. Distinct occurrence IDs do not establish distinct dependency implementations.

## Acceptance examples

These are conditional static cases, not executed tests. Each assumes execution reaches the relevant operation, required accesses are valid, and intervening dependency effects do not invalidate the stated conditions.

1. **Store count.** If one iteration reaches its loop continuation, it has executed 15 directly represented stores. Dependency-internal stores are outside that count.

2. **First-call target.** If the incoming link-like word is `0x08712031`, F-00000001 targets `0x0871302f` and receives two zeros. The first target is not generally a fixed constant.

3. **Next-iteration target.** After the explicit late assignment, the link-like word is `0x08712031`. If F-00000037 and subsequent control transfer leave that state unchanged, the next first-call target is `0x0871302f`. Without that preservation condition, the next target is unresolved.

4. **Shifted paired result.** If F-00000009 supplies low word `0xfc000003`, frame offset `0x26c` receives 63. Under the stated 32-bit interpretation, the later left-shifted store receives `0xf000000c`.

5. **Unsigned address component.** If the halfword read at `0x21c48994` is `0xfff0`, the subsequent word-read addresses derived from it are `0x10034` and `0xfff4`. It is not interpreted as −16.

6. **Captured global payload.** If the second global read yields `0x12340000` and F-00000019 returns 3, the indexed halfword destination is `0x1234000c` and the payload is zero. Changing the global’s memory afterward does not cause an explicit reload for this store.

7. **Fixed derived value.** The fixed-word shift and multiplication produce `0x21c4`; the corresponding word read is at `0x2218`, and the halfword written through F-00000025 is `0x21c4`.

8. **Zero extension and absolute indexing.** If the halfword loaded through F-00000028’s high word is `0xffff`, both record word stores contain `0x0000ffff`. Independently, if the halfword loaded after F-00000027 is 7, the indexed word destination is `0x90` and the later byte-read address is `0x23`.

9. **Breakpoint stop.** If the breakpoint permanently stops execution, no post-breakpoint operations occur. Neither a complete iteration nor all 15 stores may be claimed.

10. **Frame progression.** If one iteration continues without external frame modification or restoration, the next begins 244 bytes lower. No normal-return value is produced.

## Uncertainties

**documented_only — purpose and intent:** No subsystem, field meaning, or application-level objective is established. “Record” and “frame” are descriptive conveniences, not recovered types.

**documented_only — dependency contracts:** Symbol identities, calling conventions, hidden inputs, complete outputs, failure behavior, memory effects, state preservation, and ownership effects remain unknown. The target expressions above are nevertheless established by the representation.

**documented_only — execution environment:** Executability of the targets, address mapping, alignment rules, endianness, device-memory behavior, and breakpoint disposition are unknown.

**documented_only — arithmetic execution:** Recovered widths support the stated bit-pattern examples. They do not independently establish original signed-overflow, signed-looking shift, multiplication-overflow, or pointer-wrap behavior.

**documented_only — implicit state:** The physical meaning of the frame base and incoming register-like state is unresolved. Cross-call preservation and subsequent-iteration state cannot be assumed.

**documented_only — errors and ownership:** No explicit null check, bounds check, status check, rollback, allocation contract, or release contract is represented. Invalid accesses and unavailable targets have environment-dependent consequences; no substitute error result is established.

## Evidence

Established static evidence consists of the supplied function representation: its unconditional loop, ordered call occurrences, target expressions, explicit argument and result uses, memory-access widths, 15 stores, constants, breakpoint, frame-base change, and link-like assignment.

The occurrence IDs, phase boundaries, and descriptive names in this contract are documentation devices. They do not establish original abstractions or dependency identities.

Application intent remains unknown. Arithmetic examples using truncation are conditional on the stated 32-bit interpretation, rather than runtime observations.

No runtime tests, execution traces, dependency implementations, or memory-map inspection have been performed or supplied.
