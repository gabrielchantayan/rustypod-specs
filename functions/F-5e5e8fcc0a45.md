<!-- Trusted provenance: F-5e5e8fcc0a45; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5e5e8fcc0a45

## Purpose

F-5e5e8fcc0a45 repeatedly performs an ordered sequence of indirect calls, memory reads, memory writes, and a software breakpoint. The supplied representation contains an unconditional loop and no normal return.

Its application-level purpose is **documented_only**. The evidence does not establish that the accessed storage represents particular objects, tables, devices, or allocated regions. Values described below as bases, selectors, or indices are descriptive labels, not recovered application types.

This contract distinguishes:

- **Established static evidence:** explicit operations, access widths, call-target relationships, and data dependencies in the supplied representation.
- **Inferred interpretation:** 32-bit machine-value arithmetic used to describe address calculations and truncating operations.
- **Unknown behavior:** dependency implementations, complete calling conventions, ownership, fault handling, and correspondence to execution on a particular platform.

Every later operation is conditional on earlier calls, accesses, and the breakpoint allowing execution to continue.

## Interface

### Explicit interface

The represented routine has no explicit arguments and no explicit return value. It has no represented successful completion or finite iteration limit.

### Implicit inputs

Execution consumes:

- An incoming link-like value used to calculate the first indirect-call target.
- A frame-like base used for word reads, word writes, and a repeated downward adjustment.
- An additional incoming base used in a late byte-store address.
- Two separately read global 32-bit values.
- Memory addressed through these values, dependency results, and narrow integer loads.
- Secondary register results associated with F-d006, F-d008, and F-d031.
- Any additional state required by the dependencies.

The representation supplies no validation of these inputs. Their initialization, alignment, calling-convention obligations, and lifetime requirements remain **documented_only**.

### Widths and arithmetic

| Quantity | Represented width or interpretation |
|---|---|
| Byte access | 8 bits |
| Halfword access | 16 bits |
| Word access | 32 bits |
| Paired dependency result | 64 bits, separated into low and high 32-bit words |
| Address-like values and arithmetic | Apparently 32 bits |
| Unsigned halfword promoted for calculation | Zero-extended to 32 bits |

Narrow stores retain only the destination width. A halfword store of an address-like value writes its low 16 bits, not a full address.

Calculations below use a **32-bit bit-vector interpretation**: addition, multiplication, and left shifts retain the low 32 bits. This is an inferred machine-level model, not a portable source-language arithmetic guarantee. In particular, apparently signed additions, multiplications, and left shifts do not establish defined behavior for all values in a source-language model.

Right shifts of the explicitly unsigned quantities are logical. Splitting a paired result into high and low words describes numeric portions of that result; it does not establish memory endianness.

### Preconditions for continued execution

Every indirect-call target must be executable under the applicable calling convention. Every memory access must support the stated width and direction at its computed address. No null, bounds, alignment, permission, or lifetime checks are represented.

Continuation beyond the software breakpoint additionally requires platform behavior that resumes execution.

## Behavior

### Dependency identity and address relationships

F-d001 through F-d037 identify call sites in execution order. They do not assert distinct implementations or known interfaces beyond the explicitly consumed values.

The following equalities are statically established by the target expressions:

| Call sites | Equal numeric target |
|---|---:|
| F-d005 and F-d006 | `0x08712df9` |
| F-d009 and F-d013 | `0x08712e73` |
| F-d017 and F-d024 | `0x08712e89` |
| F-d019 and F-d031 | `0x08712edd` |
| F-d025 and F-d032 | `0x08712ebb` |

These are equal target addresses, not merely possible sharing. They do not establish identical argument interpretation, preserved state, results, or state-dependent behavior at the two visits. The same numeric target is insufficient to infer a complete dependency contract.

F-d001 uses the current link-like value plus `0xffe`. The remaining sites use fixed numeric target expressions.

### Initial calls and captured record writes

At the beginning of each iteration, a byte-valued quantity is set to zero. F-d001 is then invoked with two explicit zero arguments. A separate byte-valued quantity is set to zero after F-d001 and before F-d002.

F-d002 and F-d003 execute in that order. The first global 32-bit value is then read and retained as the **initial record base**. This capture precedes F-d004 and is not repeated for the subsequent record writes.

After F-d004, F-d005 supplies a 64-bit **first pair**. F-d006 receives two explicit 32-bit arguments:

- The first pair’s low word.
- The sum of its high word, its low word, and the separate zero-valued byte quantity.

An 8-bit secondary result associated with F-d006 is written at initial record base plus `0x11`.

F-d007 executes next. The byte quantity initialized before F-d001 is then written at initial record base plus `0x15`; its represented value is zero.

F-d008 supplies a 16-bit result and an associated secondary address-like result. The 16-bit result is written at that secondary address plus `0x0c`.

The representation retains the initial record base across these calls. It does not establish that the dependencies preserve the validity or contents of the addressed storage.

### Second pair and frame-relative accesses

F-d009 supplies a 64-bit **second pair**. A zero halfword is written at the address represented by its high word plus `0x2c`. Next, its low word is logically shifted right by 26 bits and written as a 32-bit value at the current frame-like base plus `0x26c`. This written value is between 0 and 63.

The constant `0x21c48960` is established for later calculations. After F-d010, a 32-bit value is read from frame-like base plus `0x320` and retained as the **second record base**.

F-d011 and F-d012 execute before a word is written at second record base plus `0x54`. That word is the second pair’s low word shifted left by two bits. Under the bit-vector interpretation, the original top two bits are discarded.

F-d013, F-d014, F-d015, F-d016, and F-d017 then execute in that order. F-d013 revisits the same numeric target as F-d009, but no result from this visit is explicitly consumed.

### Narrow selector and breakpoint

An unsigned halfword is read at `0x21c48994`, equivalent to the established constant plus `0x34`. It is retained as the **first selector**.

After F-d018, a word is read from frame-like base plus `0x280`. F-d019 receives that word as its explicit argument and supplies a 32-bit result.

The routine then executes a software breakpoint with immediate `0x92`. No bypass is represented. Whether this traps, stops, resumes through a handler, or has another effect is **documented_only**.

If execution resumes, the F-d019 result shifted left by two bits becomes the **scaled result index**. The established constant, logically shifted right by 18 bits and multiplied by four, becomes a **constant-derived offset**. Its numeric value is `0x21c4`.

A word is read at first selector plus `0x44` and retained as the **primary record base**. F-d020 executes, followed by F-d021, whose 32-bit result is captured. A word is then read at first selector plus `0x04`. F-d022 receives the F-d021 result and this newly read word, in that argument order.

The first selector is a zero-extended 16-bit value used directly in these addresses. No wider pointer reconstruction is represented.

### Global-based and returned-address halfword writes

The second global 32-bit value is read and retained as the **table base**. F-d023 executes before a halfword store at table base plus scaled result index. The stored value is the low 16 bits of the captured table base.

F-d024 follows that store. A word is then read at constant-derived offset plus `0x54`, numerically address `0x2218`, and retained as the **linked record base**.

F-d025 supplies a 32-bit result interpreted as an address. A halfword is written at that address plus `0x04`; its value is the low 16 bits of the constant-derived offset, namely `0x21c4`.

F-d026 and F-d027 execute in order. An unsigned halfword is then read from linked record base plus `0x34` and retained as the **second selector**.

Neither halfword store has an explicit bounds or validity check.

### Third pair, frame adjustment, and captured short value

F-d028 supplies a 64-bit **third pair**. Its low word is written at the current frame-like base plus `0x258`.

An unsigned halfword is then read from the address represented by the third pair’s high word plus `0x14`. This is retained, zero-extended, as the **short value**.

Subsequent calculations derive:

- A **wide scaled index**: the third pair’s low word shifted left by six bits.
- A **scaled address**: the third pair’s high word multiplied by four.
- A **selector offset**: the second selector multiplied by four.

The frame-like base is decreased by `0xf4`. The earlier write at offset `0x258` uses the base before this adjustment.

The third pair’s low word is written as a word at selector offset plus `0x74`. F-d029 then executes, followed by a word store of the short value at primary record base plus `0x54`.

After F-d030, the same captured short value is written as a word at primary record base plus `0x64`. There is no intervening reread of the original halfword. F-d030 may nevertheless change memory between the two stores.

### Final updates and loop-carried state

F-d031 executes. A halfword is then written at wide scaled index plus the secondary address-like result associated with that call. The stored value is the low 16 bits of primary record base.

F-d032 and F-d033 execute before a byte is read at second selector plus selector offset. Because selector offset is four times the second selector, this address is five times that zero-extended 16-bit selector.

F-d034 executes before the captured byte is written at short value plus the additional incoming base plus `0x19`. The byte is not reread after F-d034.

F-d035 executes next. The scaled address is then written as a word at primary record base plus `0x44`.

After F-d036, the represented link-like state is assigned `0x08712031`. A word is subsequently read at scaled address plus `0x44`, and F-d037 receives that word as its explicit argument. This read occurs after F-d036 and must not be replaced with an earlier captured value.

The body repeats unconditionally. On reaching the next iteration, F-d001’s target is formed from the assigned link-like value, producing `0x0871302f` under the 32-bit interpretation.

The frame-like base has no represented restoration. It is decreased once on every traversal that reaches the adjustment.

## Side effects

The body contains **15 explicit memory stores per iteration**, provided execution reaches all of them:

| Order | Width | Destination | Value |
|---:|---:|---|---|
| 1 | 8 bits | Initial record base + `0x11` | F-d006 secondary byte |
| 2 | 8 bits | Initial record base + `0x15` | Zero |
| 3 | 16 bits | F-d008 secondary address + `0x0c` | F-d008 result |
| 4 | 16 bits | Second pair high word + `0x2c` | Zero |
| 5 | 32 bits | Frame-like base + `0x26c` | Second pair low word logically shifted right by 26 |
| 6 | 32 bits | Second record base + `0x54` | Second pair low word shifted left by two |
| 7 | 16 bits | Table base + scaled result index | Low 16 bits of table base |
| 8 | 16 bits | F-d025 result + `0x04` | `0x21c4` |
| 9 | 32 bits | Pre-adjustment frame-like base + `0x258` | Third pair low word |
| 10 | 32 bits | Selector offset + `0x74` | Third pair low word |
| 11 | 32 bits | Primary record base + `0x54` | Zero-extended short value |
| 12 | 32 bits | Primary record base + `0x64` | Same captured short value |
| 13 | 16 bits | Wide scaled index + F-d031 secondary address | Low 16 bits of primary record base |
| 14 | 8 bits | Short value + additional incoming base + `0x19` | Captured byte |
| 15 | 32 bits | Primary record base + `0x44` | Scaled address |

Other direct effects are the ordered dependency calls, memory reads, breakpoint, and mutation of represented frame-like and link-like state.

No direct allocation or release is identifiable. This does not establish ownership, borrowing, retention, or release behavior. Dependencies may allocate, release, invalidate captured addresses, modify globals, or perform external effects.

Addresses may alias. Earlier stores and dependency effects may therefore affect later reads. No disjointness, atomicity, synchronization, rollback, or transaction guarantee is established.

There is no explicit error return or recovery path. An inaccessible address, nonreturning call, or nonresuming breakpoint can prevent all subsequent operations. Exact failure behavior is unknown.

## Dependencies

The dependency IDs are local opaque call-site references. Published identities and complete contracts are **documented_only**.

| Dependency IDs | Explicitly consumed interface |
|---|---|
| F-d001 | Two zero arguments; state-derived target |
| F-d002–F-d004 | No explicit result consumed |
| F-d005 | 64-bit paired result |
| F-d006 | Two 32-bit arguments; secondary byte result |
| F-d007 | No explicit result consumed |
| F-d008 | 16-bit result and secondary address |
| F-d009 | 64-bit paired result |
| F-d010–F-d018 | No explicit result consumed |
| F-d019 | One 32-bit argument; 32-bit result |
| F-d020 | No explicit result consumed |
| F-d021 | 32-bit result |
| F-d022 | Two 32-bit arguments |
| F-d023–F-d024 | No explicit result consumed |
| F-d025 | Address-like 32-bit result |
| F-d026–F-d027 | No explicit result consumed |
| F-d028 | 64-bit paired result |
| F-d029–F-d030 | No explicit result consumed |
| F-d031 | Secondary address |
| F-d032–F-d036 | No explicit result consumed |
| F-d037 | One 32-bit argument |

“No explicit result consumed” does not imply no result, no side effects, or preservation of machine state. Equal target addresses do not justify merging call-site observations or assuming that a result consumed at one site is meaningful at another.

## Acceptance examples

These are conditional consequences of static interpretation, not executed tests.

1. **Second-pair truncation.** If F-d009 supplies low word `0x80000001`, the frame-relative right-shift store writes `0x00000020`, while the later left-shift store writes `0x00000004`, under the bit-vector model.

2. **Unsigned halfword expansion.** If the halfword read after F-d028 is `0xffff`, the two primary-record word stores write `0x0000ffff`. A change to the original halfword by F-d030 does not change the second store’s captured value.

3. **Truncated table-base storage.** If the captured table base is `0x12345678` and the scaled result index is `0x10`, the halfword store targets `0x12345688` and writes `0x5678`. This assumes that address is writable.

4. **Constant-derived write.** If F-d025 returns a writable address, the halfword at that address plus four receives `0x21c4`, not `0x8960` or the original 32-bit constant.

5. **Equal targets, different observations.** F-d019 and F-d031 target `0x08712edd`. The former consumes an explicit argument and supplies the value used for a scaled index; the latter supplies a secondary address used for a halfword store. Target equality does not establish that their observed outputs are interchangeable.

6. **Breakpoint interruption.** If the breakpoint does not resume, the primary-record-base read and every later operation in that iteration are unreached. The first six explicit stores precede the breakpoint; completion of all 15 is not guaranteed.

7. **Dependency-visible final read.** If F-d036 changes the word at scaled address plus `0x44`, F-d037 receives the word read after that change, assuming execution continues and the address remains readable.

8. **Repeated execution.** If every operation continues indefinitely, another iteration begins after F-d037. The next first-call target uses the assigned link-like value, and the represented frame-like base carries its decrement forward. There is no finite successful completion result.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object layouts, and domain meaning of stored fields.
- Valid entry state and full calling convention.
- Dependency implementations, implicit arguments, and state-dependent effects.
- ABI meaning of paired and secondary results.
- Whether the frame-like adjustment corresponds to an actual stack operation.
- Endianness, alignment enforcement, memory mapping, and access-fault semantics.
- Breakpoint handling and resumability.
- Ownership, lifetimes, and dependency-induced invalidation.
- Volatility, device-memory behavior, concurrency, and atomicity.
- Fidelity of the supplied representation to executable machine behavior.

These unknowns do not erase the represented operation order, the 15 explicit stores, or the five established call-target equalities. Conversely, those static facts do not establish safe execution on any particular platform.

## Evidence

This contract is based solely on the supplied static representation of F-5e5e8fcc0a45: its control flow, target arithmetic, access widths, constants, captured values, and ordered reads and writes.

The store count is obtained by enumerating the explicit memory destinations listed above. The five target equalities follow from evaluating the corresponding fixed target expressions; they do not depend on assumptions about dependency implementations.

No independent implementation access, runtime tests, breakpoint observations, memory traces, or dependency execution was performed. Acceptance examples are conditional specifications, not observed results.
