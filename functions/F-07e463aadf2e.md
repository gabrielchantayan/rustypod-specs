<!-- Trusted provenance: F-07e463aadf2e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-07e463aadf2e

## Purpose

F-07e463aadf2e represents an indefinitely repeating sequence of indirect calls, memory accesses, ambient-state updates, and a software breakpoint. It has no represented normal return.

No application-level purpose is established. The operations do not, by themselves, identify a device, message format, queue, allocator, or other subsystem. The contract therefore describes observable sequencing and data dependencies without assigning such roles.

The supplied static representation establishes an unconditional loop. It does not establish that any execution can complete an iteration: an indirect call, invalid memory access, or breakpoint may prevent continuation.

## Interface

### Explicit interface

There are no explicit parameters and no returned value.

No success value, error value, or ordinary completion condition is represented. If all operations complete and control continues normally, execution begins another iteration.

### Ambient inputs

The function depends on:

- Incoming link state used to construct the first indirect-call target.
- An ambient memory base used for several fixed-offset word accesses.
- An additional ambient offset used in a late byte-write address.
- Two global slots containing 32-bit address values.
- Memory reached through global values, dependency results, and computed addresses.
- Secondary result state supplying a byte or an address after particular calls.
- The execution environment’s handling of indirect calls, memory accesses, and the breakpoint.

The global slots are described below as the **early-base slot** and the **indexed-base slot**. These labels identify their uses, not source-level types or ownership.

### Widths and arithmetic

The represented data model uses 8-bit bytes, 16-bit halfwords, 32-bit words and addresses, and 64-bit results split into low and high 32-bit portions. All offsets below are byte offsets.

Unsigned halfword reads produce values from 0 through 65,535. Their subsequent word stores are zero-extended. Byte and halfword stores retain the low 8 or 16 bits, respectively.

The unsigned 32-bit right shift by 26 produces a value from 0 through 63. The explicitly unsigned 32-bit left shift by two retains the low 32 bits of the shifted value.

Other calculations involve signed-looking 32-bit operands, including result addition, address addition, a left shift by two, a left shift by six, and multiplication by four. Overflow and negative-operand behavior for those calculations are not resolved by the supplied representation. Examples requiring a truncating 32-bit target interpretation state that condition explicitly.

Byte order, alignment requirements, accessible address ranges, and object boundaries are not established.

## Behavior

### Dependency identities and sequencing

F-D01 through F-D37 are opaque identifiers assigned in call-occurrence order. They identify occurrences, not necessarily distinct callees.

The following pairs resolve to identical target expressions after constant arithmetic:

| Shared target | Separate occurrences |
|---|---|
| First shared target | F-D05 and F-D06 |
| Second shared target | F-D09 and F-D13 |
| Third shared target | F-D17 and F-D24 |
| Fourth shared target | F-D19 and F-D31 |
| Fifth shared target | F-D25 and F-D32 |

These occurrences remain separate calls. In particular, F-D25 supplies a destination address used by a halfword store; F-D32 has no explicitly consumed ordinary result.

“No explicit arguments” means that none are displayed in the representation. It does not exclude implicit register inputs or other ambient dependencies.

### Opening calls and early stores

At the beginning of each iteration, a working byte is set to zero. F-D01 is then called with two explicit zero arguments. Its target is formed from the current link state plus `0xFFE`.

A second working byte is set to zero before F-D02. F-D02 and F-D03 execute in that order. The early-base slot is then read, capturing a 32-bit base for two later byte stores.

F-D04 executes, followed by F-D05. F-D05 supplies a 64-bit result. F-D06 receives two explicit 32-bit arguments:

- The low word of that result.
- The sum of its high word, its low word, and the second working byte, which is zero.

After F-D06, a byte from secondary result state is stored at the captured early base plus `0x11`. F-D07 then executes, followed by a zero-byte store at the same captured base plus `0x15`.

These stores use the captured base. They do not reread the global slot after the intervening calls.

F-D08 executes next. Its 16-bit result is stored at the secondary result address associated with that call plus `0x0C`.

### First result-derived fields and breakpoint

F-D09 supplies another 64-bit result. Before any subsequent dependency call:

1. A zero halfword is stored at its high word plus `0x2C`.
2. Its unsigned low word, shifted right by 26, is stored as a word at the ambient memory base plus `0x26C`.

A working value is then established as `0x21C48960`. F-D10 executes. A word is read at the ambient memory base plus `0x320` and captured as a field destination.

F-D11 and F-D12 execute in order. The low word previously returned by F-D09, shifted left by two as an unsigned 32-bit value, is then stored at the captured field destination plus `0x54`. Neither the F-D09 result nor the field destination is reread.

F-D13, F-D14, F-D15, F-D16, and F-D17 execute in order.

An unsigned halfword is read at absolute address `0x21C48994`. This captured halfword is subsequently used as a numeric address component, not as a documented table index.

F-D18 executes. A word is then read at the ambient memory base plus `0x280` and passed explicitly to F-D19. Its 32-bit result is retained for a later displacement.

A software breakpoint with immediate `0x92` follows F-D19. Every remaining operation in the iteration requires continuation beyond this breakpoint.

### Indexed store and constant-derived accesses

After continuation, the F-D19 result is shifted left by two. This becomes the displacement for an indexed halfword store. Its behavior for negative or overflowing signed-looking operands remains unresolved.

The constant-derived expression consisting of `0x21C48960` shifted right by 18 and then multiplied by four equals `0x21C4`.

A word is read at the earlier captured halfword value plus `0x44`. This captures a destination base used by several late stores.

F-D20 executes, followed by F-D21. The F-D21 result is retained. A word is then read at the same captured halfword value plus `0x04`. F-D22 receives the F-D21 result and this newly read word as its two explicit arguments.

The indexed-base slot is read after F-D22. F-D23 executes. A halfword is then stored at:

- Address: captured indexed base plus the shifted F-D19 result.
- Value: low 16 bits of that same captured indexed base.

F-D24 executes next. A word is read at absolute address `0x2218`, obtained from `0x21C4 + 0x54`, and captured as another base.

F-D25 supplies an address. The halfword `0x21C4` is stored at that address plus `0x04`.

F-D26 and F-D27 execute in order. An unsigned halfword is then read at the base captured from address `0x2218` plus `0x34`. This halfword controls two later numeric addresses.

The read at `0x2218` is literal under the supplied arithmetic. The representation does not use `0x21C48960` itself as the base of this read.

### Final result pair and late updates

F-D28 supplies the final 64-bit result used in the iteration.

Its low word is first stored at the ambient memory base plus `0x258`. An unsigned halfword is then read at its high word plus `0x14`.

The representation derives:

- The final low word shifted left by six.
- The final high word multiplied by four.
- The halfword captured after F-D27 multiplied by four.

It then reduces the represented ambient memory base by `0xF4`. This is a represented state update, not merely an alternative displacement for the preceding store. No restoration appears before the loop repeats.

The final low word is stored at four times the halfword captured after F-D27, plus `0x74`.

F-D29 executes. The halfword captured through the final high word is stored, zero-extended to a word, at the late destination base plus `0x54`.

F-D30 executes. The same captured halfword value is stored as a word at the same destination base plus `0x64`.

F-D31 executes. A halfword containing the low 16 bits of the late destination base is stored at the sum of:

- The final low word shifted left by six.
- The secondary result address associated with F-D31.

F-D32 and F-D33 execute in order. A byte is read at five times the halfword captured after F-D27. F-D34 executes, after which that captured byte is stored at the sum of:

- The halfword captured through the final high word.
- The additional ambient offset.
- `0x19`.

F-D35 executes. The final high word multiplied by four is stored as a word at the late destination base plus `0x44`.

F-D36 executes. The represented link state is assigned `0x08712031`. A word is read at the scaled final high word plus `0x44`, and F-D37 receives that word as its explicit argument.

Control then returns to the beginning of the loop. Both working bytes are reset during the opening sequence; the late copied byte is not carried forward as either initial byte.

The fixed link-state assignment precedes F-D37. It does not establish that an unknown callee preserves the corresponding physical machine state. Under the represented state flow, the next F-D01 target is formed from that fixed value plus `0xFFE`; its concrete ABI interpretation remains unresolved.

## Side effects

### Direct effects per completed iteration

Each completed iteration contains:

- 37 indirect dependency-call occurrences.
- One software breakpoint.
- **15 explicit memory writes: three byte writes, five halfword writes, and seven word writes.**
- Reads from both global slots and the computed addresses described above.
- A represented ambient-memory-base reduction by `0xF4`.
- A represented link-state assignment before F-D37.

The direct-store inventory, in execution order, is:

| Order | Width | Destination and stored value |
|---:|---|---|
| 1 | Byte | Early base + `0x11`; secondary byte after F-D06 |
| 2 | Byte | Early base + `0x15`; zero |
| 3 | Halfword | Secondary address after F-D08 + `0x0C`; F-D08 result |
| 4 | Halfword | F-D09 high word + `0x2C`; zero |
| 5 | Word | Ambient base + `0x26C`; F-D09 low word shifted right by 26 |
| 6 | Word | Captured field destination + `0x54`; F-D09 low word shifted left by two |
| 7 | Halfword | Indexed base + scaled F-D19 result; low 16 bits of indexed base |
| 8 | Halfword | F-D25 address + `0x04`; `0x21C4` |
| 9 | Word | Ambient base + `0x258`, before its reduction; F-D28 low word |
| 10 | Word | Four times the halfword captured after F-D27 + `0x74`; F-D28 low word |
| 11 | Word | Late destination base + `0x54`; final captured halfword, zero-extended |
| 12 | Word | Late destination base + `0x64`; same captured halfword, zero-extended |
| 13 | Halfword | Shifted F-D28 low word + secondary address after F-D31; low 16 bits of late destination base |
| 14 | Byte | Final captured halfword + additional ambient offset + `0x19`; captured byte |
| 15 | Word | Late destination base + `0x44`; F-D28 high word multiplied by four |

Dependency-internal writes and breakpoint-handler effects are additional unknown effects, not included in these totals.

### Ordering, aliasing, and ownership

No non-aliasing guarantee is established. Computed accesses may overlap each other, global slots, or locations read later. Such overlap can affect later fresh reads.

Captured values remain distinct from fresh reads in this contract. For example, F-D30 lies between two stores of the same captured halfword value. It may change memory, but the second store does not reload that halfword.

No ownership transfer, allocation, release, locking, or resource lifetime is established. Returned addresses are used without visible validation. Dependencies may nevertheless allocate, release, mutate global state, or transfer control nonlocally.

### Failures

There are no visible null checks, bounds checks, alignment checks, status checks, recovery branches, rollback operations, or cleanup paths.

A dependency that does not return, a memory fault, or a breakpoint that does not resume can stop progression. Earlier effects are not visibly reversed.

## Dependencies

All F-D01–F-D37 implementations are unavailable. The following are their explicitly consumed arguments or results:

| Dependency | Established use |
|---|---|
| F-D01 | Two zero arguments; target depends on link state |
| F-D05 | 64-bit result consumed by F-D06 |
| F-D06 | Two 32-bit arguments; secondary byte consumed afterward |
| F-D08 | 16-bit result and secondary destination address |
| F-D09 | 64-bit result used for three stores |
| F-D19 | One loaded word argument; 32-bit result becomes a displacement |
| F-D21 | 32-bit result passed to F-D22 |
| F-D22 | Two explicit 32-bit arguments |
| F-D25 | 32-bit address result used for a halfword store |
| F-D28 | 64-bit result used for stored values and address calculations |
| F-D31 | Secondary result supplies part of a halfword-store address |
| F-D32 | Same target as F-D25; no explicitly consumed ordinary result |
| F-D37 | One freshly loaded word argument |

Other occurrences have no explicitly consumed ordinary result. This does not imply absence of outputs, implicit inputs, memory effects, or register effects.

The breakpoint facility is a separate environmental dependency. Only its placement and immediate are established; exception, debugger, termination, and resumption behavior are unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes execution reaches the relevant operation and all required accesses are valid.

### Pair arithmetic

If F-D05 returns low word `3` and high word `7`, F-D06 receives `3` and `10`.

If the signed-looking addition exceeds its representable range, this example does not establish wraparound behavior.

### Captured early base

If the early-base slot supplies address `A`, the early byte stores target `A + 0x11` and `A + 0x15`. A later change to that slot alone does not redirect either store. The second store writes zero.

### Unsigned shift fields

If F-D09 returns low word `0xFC000003` and a valid high-word address:

- A zero halfword is stored at the high-word address plus `0x2C`.
- The word stored at ambient base plus `0x26C` is `63`.
- The later word stored at the captured field destination plus `0x54` is `0xF000000C`, from the unsigned 32-bit left shift.

### Indexed halfword store

If F-D19 supplies `3` and the indexed-base slot supplies `0x12345000`, the indexed halfword store targets `0x1234500C` and stores `0x5000`, provided execution resumes after the breakpoint.

### Constant-derived address and shared call target

The constant-derived word read targets `0x2218`. If F-D25 returns address `P`, the following halfword store writes `0x21C4` at `P + 0x04`.

F-D32 later invokes the same target as F-D25. That fact does not imply it returns `P`, and its ordinary result is not used by the represented sequence.

### Late captured values

Suppose the halfword captured after F-D27 is `2`, the halfword read through the final high word is `9`, and F-D28 returns low word `5` and high word `0x1000`. Then:

- Word value `5` is stored at address `0x7C`.
- Word value `9` is stored at late destination offsets `0x54` and `0x64`.
- The late byte read targets address `10`.
- The scaled high-word value is `0x4000`.
- That value is stored at late destination offset `0x44`.
- The final word argument is read at `0x4044`.
- The late byte destination is the additional ambient offset plus `0x22`.

These low numeric addresses are arithmetic consequences, not accessibility guarantees.

### Breakpoint interruption

If the breakpoint does not resume, none of the post-breakpoint operations occurs on that path. Earlier calls and stores remain prior effects; no reversal is specified.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and valid object states.
- Concrete identities and ownership of the two global slots.
- The physical meaning and validity of the ambient memory-base update.
- The ABI interpretation of link state and secondary results.
- Signed-looking overflow, negative shifts, and address-overflow behavior.
- Byte order, alignment policy, permissions, and accessible ranges.
- Implicit dependency arguments and register conventions.
- Dependency preservation or mutation of globals and ambient state.
- Dependency allocation, release, ownership transfer, and nonlocal control flow.
- Breakpoint handling and whether execution reaches later operations.
- Fidelity of the supplied representation to executable control flow.

The term “base” describes how a value participates in addressing. It does not establish a valid pointer, allocated object, or structured record.

## Evidence

The sole evidence is the supplied static function representation. The prior draft and review feedback are not independent evidence.

The representation supports the call order, explicit operands, access widths, captured-value dependencies, breakpoint placement, and unconditional repetition described here.

Independent enumeration of the direct stores yields three byte stores, five halfword stores, and seven word stores: **15 total**. Constant-target arithmetic establishes all five shared-target pairs, including F-D25 and F-D32. Constant data-address arithmetic establishes `0x21C4`, `0x2218`, and `0x21C48994`.

Application purpose and successful indefinite execution are not established. No runtime execution, smoke run, or tests have been performed.
