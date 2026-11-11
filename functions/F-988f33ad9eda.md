<!-- Trusted provenance: F-988f33ad9eda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-988f33ad9eda

## Purpose

F-988f33ad9eda repeatedly invokes indirect dependencies and performs ordered memory reads and writes. The supplied representation has no normal-return path and no loop-exit condition.

The established behavior is low-level state manipulation. An application-level purpose—such as initialization, device servicing, or table maintenance—is not established.

Each iteration reaches a software-breakpoint operation with immediate value `0x92` after its first 19 indirect calls, provided those calls and preceding accesses complete. Operations after that breakpoint, including eventual repetition, are conditional on execution continuing.

## Interface

### Explicit interface

- No explicit parameters.
- No explicit return value.
- No visible success or error result.
- No normal completion condition.

This apparent interface does not describe all required execution state. The function also depends on ambient address values, global pointer slots, absolute-address memory, and results conveyed by indirect calls.

### Ambient inputs and descriptive names

The following names describe roles in this contract, not application-level types:

| Name | Role |
|---|---|
| **Frame base** | Ambient address used for word accesses; explicitly decreased by `0xF4` during each completed iteration. |
| **Entry link value** | Ambient value used to calculate the first indirect destination on initial entry. |
| **Ambient index base** | Incoming register-derived value contributing to a late byte-store address; no explicit assignment appears in the body. |
| **Global slot A** | Word-sized global location read early in each iteration to capture a destination base. |
| **Global slot B** | Different word-sized global location read after the breakpoint to capture a table base. |

Initial values, validity, and provenance are not established.

### Widths and arithmetic

- Byte accesses transfer 8 bits.
- Halfword accesses transfer 16 bits.
- Word accesses and represented addresses use 32 bits.
- Paired results contain 64 bits, split into low and high 32-bit words.
- Unsigned halfword loads are zero-extended.
- Halfword stores retain the low 16 bits of their source.
- Byte stores transfer one byte.

Arithmetic descriptions use the low-32-bit machine interpretation for address additions, word sums, left shifts, and multiplication. Right shifts explicitly described as logical insert zero bits. No checked-overflow behavior or overflow error path is established.

Some expressions have signed-looking types in the supplied representation. Their language-level overflow semantics are not established; this contract does not introduce exceptions or validation on that basis.

Memory byte order, alignment requirements, permissions, address-space mapping, and access-fault behavior remain unspecified.

## Behavior

### Dependency notation and destination equality

**F-D01** through **F-D37** identify indirect call sites in execution order. They do not identify 37 necessarily distinct implementations.

Constant destination arithmetic establishes these numerical equalities:

| Call sites | Equal numerical destination |
|---|---|
| F-D05 and F-D06 | `0x08712DF9` |
| F-D09 and F-D13 | `0x08712E73` |
| F-D17 and F-D24 | `0x08712E89` |
| F-D19 and F-D31 | `0x08712EDD` |
| F-D25 and F-D32 | `0x08712EBB` |

These equalities are established, not merely possible aliasing. They do not establish callable implementation identity, a shared signature, or identical behavior under different execution state. For example, F-D05 and F-D06 use the same numerical destination, but only F-D06 has two explicit arguments in the representation.

Except where stated below, argument state, results, and dependency effects are unknown. A call without displayed arguments is not proof of a parameterless machine-level interface.

### Early calls and captured-base stores

Each iteration begins with **F-D01**, whose destination is the current link-derived value plus `0xFFE`. It receives two explicit zero arguments. **F-D02** and **F-D03** follow.

The function then reads global slot A as a word and retains that captured address for two byte stores. **F-D04** runs after this capture.

**F-D05** supplies a paired result. **F-D06**, at the same numerical destination, receives:

1. The low word of that result.
2. The word-width sum of its high word, low word, and a locally represented zero byte.

A register-derived byte output from F-D06 is stored at captured base A plus `0x11`. **F-D07** runs next, followed by a zero-byte store at captured base A plus `0x15`.

Both stores use the captured address, not a fresh read of global slot A. An intervening dependency may change the slot without changing these represented destinations.

**F-D08** then supplies a halfword result and a register-derived destination base. The halfword is stored at that destination plus `0x0C`. This base is not established as equal to captured base A.

### Paired-result processing and pre-breakpoint reads

**F-D09** supplies another paired result. Its high word becomes a base for a zero-halfword store at offset `0x2C`. After that store, its low word, logically shifted right by 26, is stored as a word at frame base plus `0x26C`. The stored extraction lies between 0 and 63.

**F-D10** runs before a word is loaded from frame base plus `0x320`. That loaded word is retained as a destination base. **F-D11** and **F-D12** run before the low word from F-D09, shifted left by two at word width, is stored at the retained destination plus `0x54`.

**F-D13**, **F-D14**, **F-D15**, **F-D16**, and **F-D17** then run in order. F-D13 has the same numerical destination as F-D09, but this call site does not supply the paired value used by the preceding stores.

An unsigned halfword is read from absolute address `0x21C48994`. Its zero-extended value is the **first halfword base**. This is a 16-bit memory value used as an address base, not a pointer obtained by a word load.

After **F-D18**, a word is read from frame base plus `0x280` and passed to **F-D19**. The word result of F-D19 is retained.

The software-breakpoint operation with immediate `0x92` occurs next. No unconditional progress past it is established.

### Post-breakpoint addressing and table update

If execution continues, the retained F-D19 result is multiplied by four at word width to form a table-store offset.

A separate address quantity is derived from the fixed word `0x21C48960`: logical right shift by 18, followed by multiplication by four. The result is `0x21C4`.

A word is read at first halfword base plus `0x44` and retained as the **retained object base**. **F-D20** runs, followed by **F-D21**, which supplies a word result. A word is then read at first halfword base plus `0x04`. **F-D22** receives the F-D21 result and this loaded word.

Global slot B is read as a word, capturing the **table base**. After **F-D23**, a halfword is stored at:

- Captured table base plus four times the F-D19 result, with word-width arithmetic.

The value written is the low 16 bits of the captured table base itself.

**F-D24** runs after this store. Its numerical destination equals that of F-D17.

A word is then read at absolute address `0x2218`, which is `0x21C4 + 0x54`. The loaded word is the **secondary object base**. This access is not to `0x21C489B4`.

**F-D25** supplies a destination address. The halfword `0x21C4` is stored at that address plus `0x04`.

After **F-D26** and **F-D27**, an unsigned halfword is read at secondary object base plus `0x34`. Its zero-extended value is the **table index**.

**F-D28** supplies the final paired result used in the iteration.

### Frame-base transition and final writes

The low word of F-D28 is stored at the current frame base plus `0x258`. Next, an unsigned halfword is read at the high word of F-D28 plus `0x14`. Its zero-extended value is the **propagated halfword**.

Two word-width quantities are retained:

- The low word of F-D28 shifted left by six.
- The high word of F-D28 multiplied by four.

The represented frame base is then decreased by `0xF4`. Thus the store at offset `0x258` uses the pre-decrement base.

The table index is multiplied by four. A word store writes the low word of F-D28 at four times the table index plus `0x74`. No additional object or table pointer participates in this destination.

The following effects occur in order:

- **F-D29** runs.
- The propagated halfword is stored as a zero-extended word at retained object base plus `0x54`.
- **F-D30** runs.
- The same zero-extended value is stored as a word at retained object base plus `0x64`.
- **F-D31** runs.
- The low 16 bits of the retained object base are stored at the sum of the shifted F-D28 low word and a register-derived address contribution from F-D31.

F-D31 has the same numerical destination as F-D19. This does not establish that its address contribution equals F-D19’s earlier word result.

**F-D32** and **F-D33** run next. F-D32 has the same numerical destination as F-D25.

A byte is read at five times the table index, formed by adding the index to its previously calculated fourfold value. **F-D34** runs between this read and the corresponding write. The captured byte is then stored at:

- Propagated halfword plus ambient index base plus `0x19`.

After **F-D35**, the fourfold high word of F-D28 is stored as a word at retained object base plus `0x44`.

**F-D36** runs before a word is read at four times the high word of F-D28 plus `0x44`. The represented link-derived state is explicitly assigned `0x08712031` before **F-D37**, which receives this final loaded word.

Control repeats unconditionally if execution reaches the loop back edge. In the supplied representation, the next F-D01 destination is consequently `0x0871302F`, calculated from `0x08712031 + 0xFFE`, rather than from the initial incoming link value. Architectural interpretation of that represented link state remains unresolved.

## Side effects

### Direct memory effects

An iteration reaching every explicit operation performs:

- Three byte stores.
- Five halfword stores.
- Seven word stores.
- Reads from both global pointer slots.
- Additional byte, halfword, and word reads through frame-relative, absolute, and computed addresses.

These counts exclude dependency-internal accesses and breakpoint effects.

There are no visible bounds checks, null checks, alignment checks, or explicit access-failure recovery. Continued execution requires every reached access to be supported by the actual environment. The behavior of a failing access is not established.

### Ordering, capture, and aliasing

The order described in **Behavior** is significant:

- Global slot A is captured before F-D04; its two byte destinations are not reloaded.
- The frame-relative destination at offset `0x320` is captured before F-D11 and F-D12.
- Global slot B is captured before F-D23.
- The zero-halfword store through F-D09’s high word precedes the shifted frame-relative word store.
- The F-D28 low-word store at frame offset `0x258` precedes the frame-base decrement.
- F-D29 and F-D30 separate the late index-derived and propagated-value stores.
- F-D34 separates the late byte read from its write.
- F-D36 separates the retained-object update from the final word read.

No disjointness guarantee exists for data addresses. Earlier stores or dependency effects may therefore affect later reads.

Numerically equal indirect destinations are a separate, established fact: the five pairs listed above must not be treated as five merely hypothetical aliases.

### Ownership and lifetime

The visible body contains no explicit allocation or release operation. It establishes no ownership transfer for addresses passed to, or obtained from, dependencies.

Dependencies must not be assumed to preserve globals, retain allocated storage, or leave referenced memory alive. Their allocation, release, retention, and mutation behavior remains unknown. Capturing an address does not guarantee its continued validity.

## Dependencies

| Call sites | Established interaction |
|---|---|
| F-D01 | Link-derived destination; receives two explicit zeros. |
| F-D02–F-D04 | Ordered calls surrounding capture of global slot A. |
| F-D05 / F-D06 | Same numerical destination `0x08712DF9`; paired result at F-D05, explicit word arguments and subsequent byte output at F-D06. |
| F-D07–F-D08 | Separate early stores; F-D08 supplies a halfword and destination contribution. |
| F-D09 / F-D13 | Same numerical destination `0x08712E73`; the F-D09 result drives earlier stores. |
| F-D10–F-D12 | Surround frame-relative destination capture and a shifted-word store. |
| F-D14–F-D18 | Ordered calls before the frame-relative argument read for F-D19. |
| F-D17 / F-D24 | Same numerical destination `0x08712E89`, reached on opposite sides of the breakpoint. |
| F-D19 / F-D31 | Same numerical destination `0x08712EDD`; different represented result uses. |
| F-D20–F-D22 | Ordered calls ending with two explicit word arguments to F-D22. |
| F-D23–F-D24 | Surround the captured-table-base halfword store. |
| F-D25 / F-D32 | Same numerical destination `0x08712EBB`; only F-D25 supplies the represented destination for the `0x21C4` store. |
| F-D26–F-D28 | Precede the table-index read and supply the final paired result. |
| F-D29–F-D30 | Separate late word stores. |
| F-D33–F-D36 | Interleave the late byte transfer and final word operations. |
| F-D37 | Receives the final loaded word before repetition. |

Callable implementation identities, full calling conventions, termination, error behavior, and internal side effects are not supplied. No dependency is established as F-988f33ad9eda itself.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Each assumes that all relevant calls and accesses complete and that execution continues past the breakpoint when required.

1. **Established destination equalities.**  
   The numerical destinations of F-D05/F-D06, F-D09/F-D13, F-D17/F-D24, F-D19/F-D31, and F-D25/F-D32 must match their respective values in the equality table. Treating these equalities as unresolved would contradict the supplied constant arithmetic. Matching destinations do not require matching results across calls.

2. **Word-width argument sum.**  
   If F-D05 supplies low word `0xFFFFFFFF` and high word `2`, F-D06 receives `0xFFFFFFFF` and `1` as its two explicit arguments.

3. **Logical extraction and truncating shift.**  
   If F-D09’s low word is `0xFC000001`, the word stored at frame offset `0x26C` is `63`. The later word stored at the captured destination plus `0x54` is `0xF0000004`.

4. **Unsigned halfword as an address base.**  
   If the read at `0x21C48994` yields `0xFFFF`, the subsequent word reads through that value target `0x10043` and `0x10003`, in that order. No negative-base sign extension occurs.

5. **Table-base truncation.**  
   If global slot B supplies `0x12345678` and F-D19 returns `3`, the table-derived halfword store targets `0x12345684` and writes `0x5678`.

6. **Fixed-address derivation.**  
   The fixed shift-and-scale calculation produces `0x21C4`. The secondary-object word read therefore targets `0x2218`, and the halfword store through F-D25 writes `0x21C4`.

7. **Maximum table index.**  
   For table index `65535`, its fourfold value is `262140`. The index-derived word store targets `262256`; the late byte read targets `327675`. Neither calculation overflows 32 bits.

8. **Unsigned propagation.**  
   If the propagated halfword is `0xFFFF`, the word stores at retained object offsets `0x54` and `0x64` write `0x0000FFFF`, not `0xFFFFFFFF`.

9. **Frame-base transition.**  
   If the frame base is `0x1000` when the F-D28 low-word store occurs, that store targets `0x1258`. The explicit decrement changes the represented base to `0x0F0C`. Subsequent frame-relative accesses use that changed base insofar as the represented execution state remains applicable.

10. **Breakpoint boundary.**  
    If execution stops permanently at the breakpoint, none of the post-breakpoint accesses or later iterations occurs.

## Uncertainties

The following remain **documented_only**:

- Intended application-level purpose.
- Initial values and provenance of ambient execution inputs.
- Callable implementation identities and semantics at indirect destinations.
- Any additional destination coincidences involving the initial, ambient-dependent F-D01 target.
- Full calling conventions and meanings of register-derived outputs.
- Whether paired results represent ordinary scalar returns or paired register values.
- Architectural accuracy of represented frame-base and link-derived state.
- Breakpoint handling, resumption, and external trap effects.
- Memory mapping, byte order, alignment, permissions, and fault semantics.
- Dependency effects on globals, memory, ownership, lifetimes, and execution state.
- Reachability of the represented repeating execution in a valid environment.

The five constant destination equalities are not among these uncertainties. No validation, fallback, cleanup, or recovery behavior is established for unresolved conditions.

## Evidence

**Established static evidence:** The supplied representation contains an unconditional loop, 37 indirect call sites, a software-breakpoint operation, explicit byte/halfword/word accesses, paired-result splitting, and a frame-base decrement. Constant address arithmetic establishes the five destination-equality pairs listed in this contract.

**Inferred intent:** Repeated state or table manipulation is plausible. No application semantics are adopted.

**Runtime evidence:** None. No runtime tests have been performed. Acceptance examples describe conditional static consequences, not observed executions.
