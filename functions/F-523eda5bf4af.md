<!-- Trusted provenance: F-523eda5bf4af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-523eda5bf4af

## Purpose

F-523eda5bf4af repeatedly performs an ordered sequence of indirect calls, memory accesses, arithmetic, and state updates. A fully traversed iteration contains 37 indirect call occurrences, one software breakpoint, and 15 explicit memory writes.

The static evidence does not establish an application-level purpose. Descriptions such as “working base” and “address cell” identify observable roles, not object types, ownership, or device semantics.

No normal return, termination condition, completion result, or recovery path is represented. Reaching any later operation requires earlier calls and accesses to complete. Reaching the second part of an iteration additionally requires execution to continue after the breakpoint.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no return value. Its implicit inputs include:

- An entry control value used to form the first indirect call target.
- Working-base state read independently at several points.
- An inherited address contribution used by a late byte store.
- Two shared cells containing address words.
- Memory reached through constants, loaded addresses, narrow values, and dependency results.
- Auxiliary machine state consumed after certain calls.

The absence of formal parameters does not establish that arbitrary entry state is valid.

Observable outputs consist of explicit memory writes, a working-base update, a control-state assignment, the breakpoint, and dependency effects. Dependency effects are not bounded by the explicit write inventory.

### Widths and arithmetic

| Quantity | Represented width |
|---|---|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word or address word | 32 bits |
| Result pair | 64 bits, consumed as low and high 32-bit words |

Unsigned halfword reads are zero-extended. Byte and halfword stores retain the low 8 and 16 bits respectively. Word stores retain 32 bits.

Unsigned word shifts discard bits above bit 31. Other expressions include signed addition, left shifts, multiplication, and address arithmetic. Their overflow behavior is not established as a portable guarantee. Examples involving such overflow are conditional on the stated machine-level interpretation.

Endianness, alignment requirements, atomicity, and memory-region attributes are unknown.

### Descriptive notation

The following labels are contract notation, not implementation identifiers:

- **G-A**, **G-B:** distinct shared cells containing address words.
- **K:** the word constant `0x21C48960`.
- **L1/H1**, **L2/H2**, **L3/H3:** low/high words of three captured result pairs.
- **W₁:** working-base value read for the store at offset `0x26C`.
- **W₂:** working-base value read for the address load at offset `0x320`.
- **W₃:** working-base value read for the argument load at offset `0x280`.
- **W₄:** working-base value read for the store at offset `0x258`.
- **W₅:** working-base value read for the explicit decrement.
- **H:** inherited address contribution consumed by the late byte store.

W₁ through W₅ are separate observations. They are **not** an iteration-entry snapshot and are not established to be equal. Dependency calls intervene between several of these observations, and their preservation behavior is unknown.

F-U01 through F-U37 identify indirect call occurrences in iteration order. They do not identify 37 distinct target addresses.

## Behavior

### Ordering and captured values

The sequence below specifies local ordering. A memory read following a dependency remains after that dependency. A value explicitly captured from memory is subsequently used as that captured value; the local sequence does not automatically reread its source.

This distinction is particularly important for the working base: each designated working-base access reads the state at that point. Capturing an address from G-A or from W₂ + `0x320` does not create a working-base snapshot.

Captured-value descriptions do not grant unknown callees preservation, ownership, or memory-lifetime guarantees. Auxiliary and inherited machine-state lifetimes remain qualified where their calling convention is unavailable.

### Initial calls and byte stores

Each iteration establishes a zero byte for a later store and invokes F-U01 with two explicit zero arguments. On first entry, its target is formed from the entry control value plus `0xFFE`.

A carry-like byte is then set to zero. F-U02 and F-U03 execute in order. The address word in G-A is read and captured as **A**.

F-U04 executes, followed by F-U05, which supplies L1 and H1. F-U06 receives:

1. L1 as its first word argument.
2. The represented sum of signed-word H1, signed-word L1, and the zero carry-like contribution as its second argument.

For representable signed arithmetic, this is their ordinary sum. A low-word wraparound interpretation for overflowing cases requires an additional machine-execution assumption.

After F-U06, an auxiliary byte associated with that call is stored at A + `0x11`. F-U07 executes, followed by the zero-byte store at A + `0x15`.

Both stores use captured A. Changes to G-A during intervening calls do not cause a local reread. The auxiliary byte is not established as an ordinary return value of F-U06.

### Halfword stores and second result pair

F-U08 supplies a halfword result and auxiliary address state. The halfword is stored at that auxiliary address plus `0x0C`.

F-U09 supplies L2 and H2. The operation then:

1. Stores a zero halfword at H2 + `0x2C`.
2. Reads the working-base state as W₁ and stores the unsigned word L2 logically shifted right by 26 at W₁ + `0x26C`.

K is established as `0x21C48960`, and F-U10 executes. Only afterward is the working-base state read independently as W₂. The word at W₂ + `0x320` is captured as **B**.

F-U11 and F-U12 execute in order. The low 32 bits of unsigned L2 shifted left by two are then stored at B + `0x54`.

B is a captured address word. The later store does not reread either the working-base state or the word at W₂ + `0x320`. No named packing format is established for the two shifted stores derived from L2.

### Fixed-root read and breakpoint

F-U13 through F-U17 execute in order. An unsigned halfword is read at K + `0x34` and zero-extended as **S**.

F-U18 executes. The working-base state is then read independently as W₃, and the word at W₃ + `0x280` is passed to F-U19. Its word result is captured as **J**.

The operation executes a software breakpoint with immediate `0x92`. Exactly six explicit memory writes precede it, including the word store at B + `0x54`.

If execution continues, the operation forms:

- **I:** J shifted left by two using a represented signed expression.
- **Q:** K logically shifted right by 18, then multiplied by four; exactly `0x21C4`.
- **C:** the word read at S + `0x44`.

S is the zero-extended halfword value itself. K is not added to S when forming this address.

### Intermediate calls and truncated stores

F-U20 executes, followed by F-U21, whose word result is captured as **V**. A word is then read at S + `0x04`. F-U22 receives V first and this newly read word second.

The address word in G-B is captured as **D**. F-U23 executes. The low 16 bits of D are then stored at D + I. The stored value is the truncated base D, not the index I or the result J.

F-U24 executes. The word at Q + `0x54` is captured as **E**. This read address is `0x2218`.

F-U25 supplies address result **F**. A halfword containing the low 16 bits of Q is stored at F + `0x04`; its value is `0x21C4`.

F-U26 and F-U27 execute. Only afterward is an unsigned halfword read at E + `0x34` and zero-extended as **T**. Although E was captured earlier, these intervening dependencies can affect memory subsequently read through E.

### Third result pair and working-base update

F-U28 supplies L3 and H3. The working-base state is independently read as W₄, and L3 is stored as a word at W₄ + `0x258`.

An unsigned halfword is then read at H3 + `0x14` and zero-extended as **U**.

The operation forms:

- **X:** L3 shifted left by six using a represented signed expression.
- **Y:** signed-word H3 multiplied by four.

It then independently reads the working-base state as W₅ and updates that state to the represented result of W₅ − `0xF4`.

After the update, it forms **Z** as unsigned T multiplied by four and stores L3 as a word at Z + `0x74`.

T ranges from zero through `0xFFFF`; therefore Z ranges from zero through `0x3FFFC` without unsigned 32-bit overflow. The destination Z + `0x74` does not depend on the working base.

No dependency call is shown between the W₄ access and the decrement, but they remain separate state reads. No explicit restoration is represented before repetition. The evidence does not establish that this state is a valid stack or frame pointer.

### Final stores and repetition

F-U29 executes, followed by a word store of U at C + `0x54`. F-U30 executes, followed by a word store of U at C + `0x64`.

Both stores are 32-bit writes of the zero-extended halfword U. F-U30 separates them and may have independent effects.

F-U31 executes. The low 16 bits of C are stored at an address formed from X plus auxiliary address state associated with F-U31.

F-U32 and F-U33 execute. A byte is then read at T + Z and captured as **B8**. Since Z is 4T, this address is 5T, ranging from zero through `0x4FFFB`.

F-U34 executes before B8 is stored at U + H + `0x19`. The local sequence does not reread B8’s source after F-U34. Preservation and lifetime of the inherited contribution H are not established by a dependency contract.

F-U35 executes, followed by a word store of Y at C + `0x44`. F-U36 executes.

The control value used to form a subsequent F-U01 target is assigned `0x08712031`. The word at Y + `0x44` is then read and passed to F-U37. The iteration repeats unconditionally.

The final store uses C as its base; the final argument load uses Y. The store does not by itself make Y the argument to F-U37.

The explicit control assignment precedes F-U37. Consequently, it does not establish that F-U37 preserves that control state. Subsequent first-call targeting depends on the state actually available after that call.

## Side effects

A fully traversed iteration has **15 explicit memory writes: three byte, five halfword, and seven word writes**, in this order:

| # | Width | Destination | Value |
|---|---|---|---|
| 1 | Byte | A + `0x11` | Auxiliary byte associated with F-U06 |
| 2 | Byte | A + `0x15` | Zero |
| 3 | Halfword | F-U08 auxiliary base + `0x0C` | F-U08 halfword result |
| 4 | Halfword | H2 + `0x2C` | Zero |
| 5 | Word | W₁ + `0x26C` | L2 logically shifted right by 26 |
| 6 | Word | B + `0x54` | Low word of unsigned L2 shifted left by two |
| 7 | Halfword | D + I | Low 16 bits of D |
| 8 | Halfword | F + `0x04` | `0x21C4` |
| 9 | Word | W₄ + `0x258` | L3 |
| 10 | Word | Z + `0x74` | L3 |
| 11 | Word | C + `0x54` | Zero-extended U |
| 12 | Word | C + `0x64` | Zero-extended U |
| 13 | Halfword | X + F-U31 auxiliary contribution | Low 16 bits of C |
| 14 | Byte | U + H + `0x19` | B8 |
| 15 | Word | C + `0x44` | Y |

W₁ and W₄ are separately sampled working-base values. The inventory does not imply equality with each other, W₂, W₃, or W₅.

At the breakpoint, the completed explicit-write prefix contains two byte, two halfword, and two word writes. Writes 7–15 have not been reached.

These counts exclude dependency effects, breakpoint effects, and register-state updates. They count operations rather than distinct locations; destinations can alias.

Writes are incremental. No transaction, rollback, validation, synchronization, allocation, ownership transfer, reference-count operation, or release is explicitly represented. Dependencies may perform such operations. No preservation or ownership rule may be inferred from their absence locally.

All accesses require suitable storage and permissions to complete. No explicit rejection of null addresses, invalid indices, insufficient storage, or overflow is represented. Fault handling and partial effects of unsuccessful accesses are unknown.

## Dependencies

Dependency bodies and semantic contracts are unavailable. The locally consumed interfaces are:

| Occurrences | Local interface evidence |
|---|---|
| F-U01 | Two explicit zero arguments; implicit control-derived target |
| F-U02–F-U04 | No ordinary result consumed |
| F-U05 | First 64-bit result pair |
| F-U06 | Two word arguments; auxiliary byte subsequently consumed |
| F-U07 | Separates the early byte stores |
| F-U08 | Halfword result and auxiliary destination state |
| F-U09 | Second 64-bit result pair |
| F-U10–F-U18 | Ordered calls interleaved with the specified accesses |
| F-U19 | Memory-loaded word argument; word result J |
| F-U20–F-U21 | F-U21 supplies V |
| F-U22 | V and the word read at S + `0x04` |
| F-U23–F-U24 | Calls surrounding the indexed halfword store |
| F-U25 | Address result F |
| F-U26–F-U27 | Precede the read producing T |
| F-U28 | Third 64-bit result pair |
| F-U29–F-U30 | Precede separate word stores of U |
| F-U31 | Auxiliary address contribution |
| F-U32–F-U34 | Calls surrounding the late byte read and write |
| F-U35–F-U36 | Calls surrounding the final word store |
| F-U37 | Word read at Y + `0x44` |

The fixed target expressions establish these equal target addresses:

- **F-U05 = F-U06**
- **F-U09 = F-U13**
- **F-U17 = F-U24**
- **F-U19 = F-U31**
- **F-U25 = F-U32**

These are target-address equalities, not preservation guarantees. Different occurrences can consume different arguments or result components even when their target addresses are equal.

Dependency semantics, calling conventions, effects on working-base state and globals, and storage ownership remain unresolved. F-U01’s target additionally depends on implicit control state.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Each assumes the relevant calls return and accesses complete.

1. **Captured early address.** If G-A supplies A and F-U06 supplies auxiliary byte `0xAB`, the sequence stores `0xAB` at A + `0x11`, invokes F-U07, then stores zero at A + `0x15`. An intervening change to G-A does not introduce a reread.

2. **Representable first-pair arithmetic.** If L1 has word bits `0xFFFFFFFF`, interpreted as signed −1, and H1 is 2, F-U06’s second argument is 1. This case does not require signed-overflow wraparound.

3. **Separate working-base observations.** If W₁ is `0x1000` and F-U10 leaves the working-base state at `0x2000`, write 5 targets `0x126C`, while B is read from `0x2320`. An iteration-entry snapshot must not replace these observations.

4. **Unsigned shifts.** If L2 is `0xFC000001`, write 5 stores 63 at W₁ + `0x26C`, and write 6 stores 4 at B + `0x54`.

5. **Breakpoint stop.** If the breakpoint stops execution without resumption, six explicit writes have occurred. Writes 7–15 have not been reached.

6. **Conditional signed-shift truncation.** If J has bits `0x40000001` and its signed shift executes with 32-bit truncation, I is 4. Without that execution assumption, the overflowing shift remains documented_only.

7. **Stored base rather than index.** If D is `0x12348000` and I is 4, write 7 stores halfword `0x8000` at `0x12348004`.

8. **Narrow address base.** If S is `0x1234`, the reads through S occur at `0x1278` and `0x1238`, without adding K.

9. **Zero extension and distinct events.** If U is `0xFFFF`, writes 11 and 12 each store word `0x0000FFFF`. F-U30 remains between them.

10. **Qualified working-base progression.** If the working-base state is preserved from the W₄ observation through the W₅ observation, both observations yield W, and subtraction is representable, write 9 targets W + `0x258` and the update produces W − `0xF4`. Extending this relationship across iterations additionally requires preservation by all intervening dependencies.

11. **Distinct final addresses.** If C + `0x44` and Y + `0x44` are distinct, the final explicit store does not determine F-U37’s argument. That argument comes from the later read through Y.

12. **Known target equality without preservation.** F-U19 and F-U31 have equal fixed target addresses. This does not establish that either call preserves the working base, globals, or storage used by the other occurrence.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and dependency semantics.
- Calling conventions and auxiliary-result lifetimes.
- Dependency effects on globals, working-base state, inherited state, and addressed memory.
- Whether separately observed working-base values are equal in an actual execution.
- Additional runtime target relationships involving the control-derived F-U01 target.
- Storage validity, permissions, alignment, lifetime, and ownership.
- Endianness, concurrency, atomicity, and asynchronous observation.
- Breakpoint handling and reachability of its continuation.
- Overflow behavior of signed arithmetic and address calculations.
- Runtime interpretation of the working-base update.
- Fault propagation, unsuccessful-access effects, and dependency nonreturn.

The five established fixed-target equalities are not unresolved target-aliasing questions. Their semantic consequences remain unknown.

These uncertainties do not justify adding validation, reconstructing wider pointers from narrow values, assuming memory retention, removing the breakpoint, or introducing termination.

## Evidence

**Established static evidence:** unconditional repetition; 37 indirect call occurrences per fully traversed iteration; the five fixed-target equalities; one breakpoint with immediate `0x92`; the ordered accesses and 15-write inventory; three consumed result pairs; truncation and zero-extension operations; five distinct working-base reads; and the explicit working-base update.

The representation does not show an iteration-entry working-base capture. Its accesses at offsets `0x26C`, `0x320`, `0x280`, and `0x258`, and its decrement, independently consume working-base state.

**Inferred intent:** no application-level intent is adopted. Address-role names are descriptive. Overflowing signed-expression examples require explicit machine-level assumptions.

**Unknown:** dependency contracts, ownership rules, runtime validity, and execution-environment behavior.

No runtime tests have been performed. Acceptance examples express conditional static expectations, not observed execution.
