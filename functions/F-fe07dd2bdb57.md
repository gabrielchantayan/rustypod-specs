<!-- Trusted provenance: F-fe07dd2bdb57; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe07dd2bdb57

## Purpose

F-fe07dd2bdb57 exposes an ordered sequence of dependency calls, memory sampling, and memory mutation. Its established result is an execution trace, not a returned value or a verified application-level outcome.

The supplied static representation does not establish whether this sequence implements initialization, dispatch, serialization, allocation, or another operation. Recurring offsets and scaling factors are observable address conventions; they do not establish object types, array bounds, or ownership.

This contract distinguishes:

- **Established static evidence:** explicitly represented inputs, arithmetic, access widths, calls, and their relative order.
- **Inferred interpretation:** a conditional 32-bit machine model for overflow-sensitive calculations.
- **documented_only:** behavior requiring unavailable dependency definitions, calling-convention details, address-space information, or execution evidence.

The final action before the displayed return is a call through address zero with three zero arguments. Normal return therefore requires that call to execute and return.

## Interface

### Positional inputs and output

| Position | Represented width | Established direct use |
|---|---:|---|
| First | 32 bits, otherwise untyped | No explicit reference in the body |
| Second | Unsigned 32 bits | Supplied twice to the first dependency; subsequently shifted logically right by nine and used in address calculations |
| Third | 32 bits, otherwise untyped | No explicit reference in the body |
| Fourth | 32 bits, otherwise untyped | Supplied as the third argument to the first dependency |

No return value is represented.

The absence of direct references to the first and third inputs does not prove their irrelevance to the underlying execution convention. Many dependency calls lack complete argument descriptions.

### Additional entry state

Three quantities are consumed without an established origin among the positional inputs:

- An unsigned 32-bit entry word.
- An entry address from which a selector byte is read at offset 1.
- A control quantity that determines the first call target after addition of 0xFFE.

The entry word supplies two byte-store values and a field obtained by logical right shift by 19. Its provenance, and the caller’s obligation to establish these quantities, are **documented_only**.

Later processing also uses secondary dependency results represented separately from ordinary returns. Their producer relationships and ABI guarantees are not fully established.

### Execution preconditions

Every reached access must be permitted, and every reached call target must be executable under the actual execution environment. The body contains no explicit validation of:

- Address validity, alignment, bounds, or arithmetic overflow.
- Low or zero addresses.
- Returned storage capacity or lifetime.
- Index ranges.
- Dependency availability or successful completion.

Selector-driven stores can target absolute addresses from 0x11 through 0x110 inclusive. Other operations read address 1, use a separately represented 24-bit quantity at address 9, or access addresses derived from dependency results. Ordinary process-memory assumptions do not establish safe execution.

## Behavior

### Widths, arithmetic, and retained values

Direct byte, halfword, and word accesses have widths of 8, 16, and 32 bits respectively. Some dependencies return represented 64-bit quantities whose low and high 32-bit halves are used independently.

Unsigned right shifts are logical. Byte stores retain the low eight bits of the value being stored. Explicit unsigned widening of a sampled byte produces a value from 0 through 255. The full argument-extension convention for untyped byte and halfword samples is **documented_only**.

For overflow-sensitive multiplication, addition, and left shifts, address formulas below have a **conditional low-32-bit machine interpretation**. This is not a claim of portable signed-arithmetic semantics. Exact instruction-level overflow and signed-character behavior remain unresolved.

A retained scalar is distinct from a later memory read. Unknown dependencies may alter memory and globals, but that alone does not make an explicitly retained scalar a fresh observation of its original memory source.

### Selector-driven writes

A recurring operation reads an unsigned selector byte, adds 0x11 to its value, and writes one byte at that absolute destination. It does not write at offset 0x11 within the object from which the selector was obtained.

Each represented occurrence performs its own selector read unless the description explicitly identifies a retained selector. Calls and previous writes may change the next selector observation. Equal store values do not make separate occurrences interchangeable.

### Entry and initial calls

The first memory event is a selector read at offset 1 from the hidden entry address. The function then stores the low byte of the entry word at the selected absolute destination.

The initial dependency receives the second positional input twice and the fourth input once. Four more calls follow. The last of those produces a word subsequently supplied, multiplied by four, to another dependency.

Before that scaled-argument call, the entry selector is read again. The second entry store writes the low byte of the entry word logically shifted right by 22. The destination may differ from the first entry destination.

Later in this opening sequence, the second positional input is logically shifted right by nine. Selector accesses based on this retained quantity use the shifted value, not the original input.

An early dependency receives seven represented arguments: four copies of one returned word, the entry word shifted right by 19, the shifted second input, and a control-derived word. A subsequent paired return supplies two 32-bit halves.

The function then performs a selector-driven zero store using the shifted second input as the selector-address base. Calls intervene before a second such zero store. Both selector reads and both stores remain observable.

### Early sampling and stack exposure

The subsequent sequence intersperses calls and selector-driven stores with these memory samples, in order:

1. An unsigned byte at twice the low half of an earlier paired return.
2. A 32-bit word at another paired-result base plus either zero or 0x80000000. Under the conditional 32-bit interpretation, the displacement is selected by bit 19 of that retained base.
3. A byte at offset 0x0E from a secondary-result base.
4. An unsigned byte at a paired-result base plus a secondary-result displacement.
5. A 16-bit value at an address obtained by adding a retained extracted component to the preceding unsigned byte sample.

Each of these samples is subsequently supplied to a dependency, commonly as four identical arguments. They are not collected in a pure read phase: dependency calls and byte writes occur between them.

A dependency also receives four copies of the same address within the active stack frame. The static evidence does not establish the accessible extent, initial contents, initialization status, or permitted retention of that region.

### Indexed publication and the offset-0x0D/offset-0x44 interleaving

After the halfword sample, a paired return supplies a retained base and an index. The function writes an extracted field as a 32-bit word at four times that index plus 0x14. Following intervening calls, another indexed word store publishes the same field.

The next byte-field sequence has a specific observable interleaving:

- A byte is stored at offset 0x0D from the retained base. Its value is the low byte of twice that base.
- A 32-bit word is then read at **four times a separately retained returned word, plus 0x44**. That sample is immediately supplied as the argument of the next dependency call.
- Four further dependency calls occur.
- The literal byte value 4 is stored at offset 0x0D from the same retained base.
- The next dependency call receives two represented arguments: four times the result of the last intervening call, and a freshly read 32-bit word at **the low 30 bits of the separately retained word, shifted right by 20, plus 0x44**.

The first offset-0x44 read therefore occurs between the two offset-0x0D stores. The second occurs after the literal-4 store. The second address formula contains no multiplication by four.

These events must not be modeled as two byte stores followed by two reads. Unknown dependencies may observe the first byte-field state or modify either read destination. Aliasing may also make the literal-4 store affect the second sample.

Subsequent indexed word destinations include:

- Four times an externally supplied index, plus 0x14.
- Sixteen times an externally supplied index, plus 0x14.
- 256 times a returned word, plus 0x14.

The stored words include scaled returns, extracted fields, widened byte samples, retained earlier results, and zero. A store at sixteen times an index is not interchangeable with one at four times that index.

Several sequences publish a scaled value and then call a dependency with that or another related scaled value. They establish ordered stores and calls, not an atomic update or preservation of published memory.

### Linked field accesses and offset-0x15 samples

A later sequence uses a retained index scaled by sixteen. It first stores a zero word at offset 0x24 from that scaled base, then reads a word at offset 0x74 from the same base. The loaded word becomes a new base.

Calls intervene before a returned unsigned word, shifted right by 20, is stored at offset 0x24 from the new base. A word at offset 0x74 from that new base is then sampled and supplied to a dependency.

Further selector-driven writes follow, including one derived from a retained masked field and a later zero write. They perform separate selector reads.

Three later unsigned byte samples come from offset 0x15 of externally supplied bases. The first two feed dependency-call sequences. The third is also widened and published through a four-times-index-plus-0x14 word destination.

Some subsequent arguments are necessarily zero under the represented widths. In particular, shifting a 32-bit unsigned word right by 20 leaves at most twelve significant bits; shifting that retained result right by another 21 yields zero. This conclusion does not depend on preservation of global memory by intervening calls.

### Zero-derived addresses and global processing

A later retained unsigned scalar is explicitly set to zero. Calculations based on it consequently produce zero bases and selector reads at address 1. These are real represented accesses, not evidence that low-address operations may be ignored.

The function clears a first displayed global, here called **global A**. Later reads of global A are not guaranteed to return zero. Dependencies intervene, and the supplied representation warns that global symbols may overlap smaller storage.

The middle-to-late sequence includes:

- Zero-word publications to indexed destinations.
- Word samples at offset 0x14 from both direct and transformed bases.
- A byte clear at an already-fourfold-scaled return multiplied by 0x2000, plus 9.
- A byte store at offset 9 from another retained base, using that base’s low byte.
- A word sample at offset 0x34 from that base, supplied to a dependency.
- Additional indexed stores of retained values and extracted fields.

One chain publishes a retained base through an indexed destination, reads a word at offset 0x14 from a secondary-result base, then uses transformed and loaded values for further offset-0x14 reads and indexed publications. The direct and scaled bases are distinct address formulas; no common object layout is established.

A retained read of global A is logically shifted right by 23. A further right shift of that retained result by 23 is necessarily zero for a 32-bit source. It is not a second read of global A.

### Late global publication and field copying

Near the end, a dependency return is written to a second displayed global, here called **global B**, before being supplied to another dependency.

Before that publication, an unsigned selector byte has been read from address 1 and retained. A zero byte is stored at that retained selector value plus 0x11. Later, after global B is written and dependencies execute, the function reads a 32-bit field at the retained selector value plus 0x74 and stores the field’s low byte at the retained selector value plus 1.

The field read uses the retained selector value. It does not reread address 1.

Related field-copy sequences use bases formed by multiplying later returns by four. At one such scaled base, the offset-0x74 word is sampled and its low byte copied to offset 1 twice, with calls between the copies. These are separate word reads and byte stores; the second copy may differ from the first.

A subsequently sampled offset-0x74 word is also published to indexed word destinations. Another publication writes a fourfold-scaled return at a destination formed by clearing the low two bits of a retained word and adding 0x14.

Still later, global A is read again and retained as a base. After intervening operations, a word at that retained base plus 0x14 is sampled. Subsequent selector-driven byte writes use the low byte of this sample and the low byte of the sample logically shifted right by 22. Earlier clearing of global A does not determine this late sample.

### Terminal events

The terminal sequence, in order, is:

- A dependency produces an ordinary result represented as one byte and a secondary pointer result.
- A zero byte is stored through that secondary pointer.
- Global B is written with a 32-bit concatenation: a separately read 24-bit quantity occupies bits 31 through 8, and the returned byte occupies bits 7 through 0.
- A zero halfword is stored at the address obtained by retaining the low 30 bits of an earlier retained word and shifting right by seven.
- F-00000004 is called through address zero with three zero arguments.

The 24-bit source is read after the secondary-pointer byte clear in the represented sequence. If storage overlaps, the clear can affect the quantity subsequently incorporated into global B.

The displayed return is reachable only if the zero-address call returns.

## Side effects

Direct side effects include byte, halfword, and word writes; writes to globals A and B; publication of dependency results; and exposure of a stack-frame address to a dependency.

There is no visible rollback, error-status return, recovery branch, or exception handler. A failing access or nonreturning dependency can leave all earlier completed effects in place. Actual fault signaling and unwinding are **documented_only**.

No allocation, release, ownership transfer, or storage-lifetime guarantee is established. Passing an address does not prove that a dependency preserves, retains, owns, or releases the referenced memory.

Aliasing is behaviorally significant. A store may change a later selector or sample, and dependencies may mutate storage between represented accesses. The contract specifies represented access order; it does not establish atomicity, synchronization, or hardware memory-barrier guarantees.

## Dependencies

These opaque IDs describe unresolved dependency roles, not recovered implementation identities.

| Opaque ID | Role | Established contract boundary |
|---|---|---|
| F-00000001 | Entry-derived target | Receives the second positional input twice and the fourth once |
| F-00000002 | Family of derived targets | Consumes memory samples, retained and scaled words, paired-result halves, secondary results, and a stack address |
| F-00000003 | Separately designated late target | Receives two copies of one returned word |
| F-00000004 | Zero-address target | Receives three zero arguments at the terminal call |

F-00000002 denotes multiple call targets, not a single callee with a shared signature. Calls displayed without arguments do not prove argument-free interfaces.

No dependency is assumed to preserve globals or published memory, follow a particular ownership policy, or return normally.

## Acceptance examples

These are conditional static criteria, not executed tests. Every example assumes execution reaches the relevant events and that required accesses are permitted.

1. **Entry truncation and fresh selection.** With entry word 0x123456AB and initial selector 2, the first store writes 0xAB at address 0x13. If dependencies change the selector to 7 before its second observation, the second entry store targets 0x18.

2. **Shifted input base.** A second input of 0x00000600 becomes 3 after logical right shift by nine. The corresponding selector read uses address 4, not address 0x00000601.

3. **Unsigned byte widening.** An explicitly unsigned sample of 0xFE supplies the numeric word value 254, not −2.

4. **Required byte-field/read interleaving.** Suppose the retained byte-field base is 0x203 and the separate retained word is 0x00100000. The first offset-0x0D store writes 0x06 at 0x210. The next word sample comes from 0x00400044 and is supplied to a dependency. Four further calls precede the store of 4 at 0x210. Only then is the second word sampled, from 0x45, for the next call. This criterion is conditional on the environment permitting that word access.

5. **Intervening mutation of the second sample.** In the preceding example, if an intervening dependency changes the word at 0x45, the second read must use its then-current contents. It must not reuse a value sampled before the literal-4 store.

6. **Global mutation after clearing.** If a dependency changes global A after its clear, a later global read observes the then-current storage value. Acceptance must not require zero solely because of the earlier clear.

7. **Terminal concatenation.** If the separately sampled 24-bit quantity is 0x123456 and the terminal returned byte is 0xAB, the represented write to global B has numeric value 0x123456AB. Its byte layout depends on the unresolved byte order.

8. **Terminal halfword clear.** A retained word of 0x00000100 produces address 2 after the low-30-bit mask and right shift by seven. The clear at that address has width 16 bits.

9. **Non-executable terminal target.** If address zero cannot execute, normal return is not required. No visible mechanism rolls back earlier completed writes.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid callers.
- Completeness of the positional interface and hidden entry-state requirements.
- Dependency identities, signatures, secondary-result conventions, and side effects.
- Exact pointer widths and overflow-sensitive instruction behavior.
- Signed-character treatment where not explicitly unsigned.
- Byte order, alignment requirements, address-space mappings, and fault behavior.
- Global extents and overlap, including the separately represented 24-bit source.
- Storage ownership, lifetime, and permitted retention of the exposed stack address.
- Dependency nonreturn, exception, and unwinding behavior.
- Meaning and executability of address zero.

Repeated offsets suggest recurring storage conventions only as inferred intent. They do not establish legal layouts or safe ranges.

## Evidence

This contract is based solely on the supplied static representation. The prior draft and reviewer feedback are not independent evidence of behavior.

The offset-0x0D/offset-0x44 ordering is established by the represented sequence: doubled-base byte store, scaled-address word sample and consuming call, four further calls, literal-4 byte store, then masked-and-shifted-address word sample as an argument to the next call.

The body contains no explicit conditional branch or loop. Conditional acceptance concerns reachability through dependency returns, permitted accesses, aliasing, and the stated arithmetic interpretation.

No runtime execution or runtime tests have been performed. The acceptance examples are derived conditions, not observed outcomes.
