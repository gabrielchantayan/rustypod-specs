<!-- Trusted provenance: F-de9d707f2dc0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de9d707f2dc0

## Purpose

The routine represents an indefinitely repeated sequence of indirect calls, memory reads and writes, address calculations, and a software breakpoint. There is no visible normal return or loop-exit condition. Repetition requires every intervening call to return, every memory access to complete, and execution to resume after the breakpoint.

No application-level purpose is established. The observable contract is the ordering and data dependence of these operations, not an interpretation as initialization, device control, decoding, or another domain operation.

This specification distinguishes the supplied static representation from verified machine behavior. In particular, implicit register inputs, secondary call outputs, frame-like state, and arithmetic involving signed values have unresolved machine-level semantics.

## Interface

### Explicit interface

- No explicit parameters.
- No explicit return value.
- No visible normal completion, status result, or local error-reporting mechanism.

### Implicit inputs and retained state

Execution depends on:

- An incoming link-like value that determines the first iteration’s initial call target.
- A frame-like base used for word-sized accesses and subsequently decreased.
- An incoming address component used in the final byte-store address.
- Two global 32-bit pointer-like values, read at different points in each iteration.
- Memory reached through these values, dependency results, and integer-derived addresses.
- Secondary register-like outputs associated with three dependency calls.

None of these inputs is locally validated.

The **frame base** below means the represented frame-like value before the current iteration’s explicit decrease. The **incoming address component** means the implicit value used in the final byte-store address. Neither description establishes a conventional parameter or an actual stack pointer.

### Widths and layout

| Quantity or access | Represented width |
|---|---:|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word | 32 bits |
| Address-bearing integer | 32 bits |
| Combined dependency result | 64 bits, divided into low and high 32-bit halves |

Unsigned halfword reads produce values from 0 through 65,535. Word stores of those values are zero-extended. Byte and halfword stores retain only the low 8 or 16 bits, respectively.

All offsets and addresses are byte offsets or byte addresses. No complete object layout is established; the field offsets below describe accesses, not a validated record schema.

Unsigned word shifts have the represented 32-bit behavior. For calculations involving signed address-bearing values, this document uses a 32-bit bit-pattern interpretation when stating derived addresses or examples. That interpretation is conditional where overflow or signed shifting is involved; exact executable behavior is not established.

Endianness, alignment rules, accessible address ranges, and fault handling are unknown. Successful continuation requires the actual environment to permit the accesses and calls encountered, but no narrower valid-input domain can be derived.

## Behavior

### Call-site notation and iteration entry

`F-unknown-01` through `F-unknown-37` are opaque dependency call-site IDs in execution order. They do not imply 37 distinct target functions. Known equal-target relationships are specified under Dependencies.

A zero byte intended for the first object’s later zero store is established before `F-unknown-01`. That call receives two zero arguments. Its represented target is the incoming link-like value plus `0xFFE`.

A second zero byte, used in an argument calculation, is established after that call and before `F-unknown-02`. These zero values are re-established on every iteration.

### Initial captured object and early results

The initial call order is `F-unknown-01`, `F-unknown-02`, and `F-unknown-03`. After the third call, the routine reads and retains the first global pointer-like value. It then calls `F-unknown-04` and `F-unknown-05`.

The 64-bit result from `F-unknown-05` supplies two arguments to `F-unknown-06`:

- Its low 32-bit half.
- The sum of its high half, low half, and the second zero byte.

Thus the latter argument has the represented bit pattern of the sum of the two halves, subject to the signed-arithmetic qualification above.

After `F-unknown-06`, its represented secondary byte output is stored at offset `0x11` of the captured global-derived object. `F-unknown-07` follows, then a zero byte is stored at offset `0x15` of that same object.

Both stores use the previously captured pointer. The global is not visibly reread between these operations, even if a dependency changes it.

`F-unknown-08` next supplies a halfword result and a represented secondary destination address. The halfword is stored at that address plus `0x0C`.

`F-unknown-09` supplies a second 64-bit result. Its halves are called the **early low word** and **early high word**. The routine then:

- Stores a zero halfword at early high word plus `0x2C`.
- Stores the early low word logically shifted right by 26 at frame base plus `0x26C`.
- Establishes the constant `0x21C48960`.
- Calls `F-unknown-10`.
- Reads a word-sized pointer from frame base plus `0x320`.
- Calls `F-unknown-11` and `F-unknown-12`.
- Stores the early low word shifted left by two at offset `0x54` of that captured pointer.

The right-shifted value lies between 0 and 63. The unsigned left shift discards the original top two bits. Both stores use the same retained early low word despite intervening calls.

### First selector, breakpoint, and indexed store

After `F-unknown-13` through `F-unknown-17`, in order, the routine reads an unsigned halfword at address `0x21C48994`. This zero-extended value is the **first selector**.

`F-unknown-18` follows. A word is then read from frame base plus `0x280` and passed to `F-unknown-19`. The routine executes software breakpoint `0x92` after that call.

All subsequent behavior is conditional on resumption after the breakpoint.

The result of `F-unknown-19`, shifted left by two, supplies an **indexed byte offset**. A separate calculation, logically shifting `0x21C48960` right by 18 and multiplying by four, yields `0x21C4`.

The routine reads a word-sized pointer at first selector plus `0x44` and retains it as the **destination object**. It then calls `F-unknown-20` and `F-unknown-21`. The result of the latter and a newly read word at first selector plus `0x04` are passed to `F-unknown-22`, in that argument order.

The first selector is used directly as an unsigned integer address base. There is no visible table lookup or additional object base for either selector-relative read.

After `F-unknown-22`, the second global pointer-like value is read and retained. `F-unknown-23` intervenes before a halfword store whose:

- Address is the retained second global value plus the indexed byte offset.
- Stored value is the low 16 bits of that same retained global value.

A dependency’s later modification of the global does not cause a visible reread for this store.

### Fixed-derived pointer and late result

After `F-unknown-24`, a word-sized pointer is read at address `0x2218`, obtained from `0x21C4 + 0x54`. This is the **selector-bearing object**.

`F-unknown-25` returns an address. The routine stores halfword `0x21C4` at that address plus `0x04`, then calls `F-unknown-26` and `F-unknown-27`.

An unsigned halfword is read at selector-bearing object plus `0x34`. This zero-extended value is the **second selector**.

`F-unknown-28` supplies the **late low word** and **late high word** as the halves of another 64-bit result. The following accesses and state changes occur before `F-unknown-29`:

- The late low word is stored at frame base plus `0x258`.
- An unsigned halfword is read at late high word plus `0x14` and retained as the **copied halfword**.
- Address components are formed from the late low word shifted left by six and the late high word multiplied by four.
- The frame-like base is decreased by `0xF4`.
- The second selector is multiplied by four.
- The late low word is stored at four times the second selector plus `0x74`.

The frame-relative store uses the base before the decrease. The selector-derived store has no additional visible object base.

### Final updates and loop-carried state

The remaining dependency calls occur in numerical order, with the following intervening accesses:

| Position | Observable access |
|---|---|
| After `F-unknown-29` | Store the copied halfword as a zero-extended word at destination object plus `0x54`. |
| After `F-unknown-30` | Store the same retained value as a word at destination object plus `0x64`. |
| After `F-unknown-31` | Store the low 16 bits of the destination-object address at the sum of the late low word shifted left by six and the represented secondary address output of this call. |
| After `F-unknown-33`, before `F-unknown-34` | Read a byte at five times the second selector. |
| After `F-unknown-34` | Store that retained byte at copied halfword plus incoming address component plus `0x19`. |
| After `F-unknown-35` | Store the late high word multiplied by four at destination object plus `0x44`. |

`F-unknown-32` occurs between the halfword store following `F-unknown-31` and `F-unknown-33`.

After `F-unknown-36`, the represented link-like state is assigned `0x08712031`. A word is then read at four times the late high word plus `0x44` and passed to `F-unknown-37`.

If the final call returns and execution continues, the next iteration begins without a visible restoration of the frame-like base.

The link assignment gives the next iteration’s first represented call target as `0x0871302F`, from `0x08712031 + 0xFFE`. This is distinct from the first iteration’s target, which depends on incoming link-like state. The assignment and resulting represented target are statically known; whether the corresponding machine register retains that value through the final call remains unresolved.

## Side effects

### Direct writes

Each completed iteration contains **15 explicit memory writes: three byte writes, five halfword writes, and seven word writes**.

| Width | Destinations |
|---|---|
| Byte: 3 | First captured object plus `0x11`; that object plus `0x15`; copied halfword plus incoming address component plus `0x19`. |
| Halfword: 5 | Secondary address from `F-unknown-08` plus `0x0C`; early high word plus `0x2C`; indexed second-global address; result of `F-unknown-25` plus `0x04`; shifted late-low address plus secondary output from `F-unknown-31`. |
| Word: 7 | Frame base plus `0x26C`; frame-loaded pointer plus `0x54`; pre-decrease frame base plus `0x258`; four times second selector plus `0x74`; destination object plus `0x54`, `0x64`, and `0x44`. |

These counts describe write operations, not distinct memory locations. Aliasing may cause destinations to overlap.

Dependency calls may have additional effects. Earlier writes or dependency effects may influence later reads. Conversely, already captured values are not visibly refreshed merely because calls intervene.

### Control, state, and failure

Each iteration that reaches its end contains 37 indirect calls and one software breakpoint. The frame-like base decreases by `0xF4` once per iteration reaching that update. The link-like assignment occurs near the end.

The destination-object updates are separate stores with calls between them, not an atomic transaction. If execution stops, faults, traps, or fails to return from a dependency, only effects already reached are represented as having occurred. No rollback, cleanup, retry, or recovery path is visible.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or reference-count operation is established.

Captured pointers are not guaranteed to remain valid across calls. Unknown dependencies may mutate memory, change globals, allocate, release, or invalidate referenced storage. Neither borrowing discipline nor ownership can be inferred from pointer capture alone.

## Dependencies

The following are visible call-site contracts, not complete calling conventions.

| Opaque call-site IDs | Visible inputs or results |
|---|---|
| `F-unknown-01` | Two zero arguments; target derived from link-like state. |
| `F-unknown-02`–`F-unknown-04` | No visible explicit arguments; ordinary results unused. |
| `F-unknown-05` | Supplies the first 64-bit combined result. |
| `F-unknown-06` | Receives two derived words; associated secondary byte output is stored. |
| `F-unknown-07` | No visible explicit arguments; ordinary result unused. |
| `F-unknown-08` | Supplies a halfword and associated secondary destination address. |
| `F-unknown-09` | Supplies the early low and high words. |
| `F-unknown-10`–`F-unknown-18` | No visible explicit arguments; ordinary results unused. |
| `F-unknown-19` | Receives the frame-loaded word; supplies the indexed-offset source. |
| `F-unknown-20` | No visible explicit arguments; ordinary result unused. |
| `F-unknown-21` | Supplies the first explicit argument to `F-unknown-22`. |
| `F-unknown-22` | Receives that result and a word read using the first selector. |
| `F-unknown-23`–`F-unknown-24` | No visible explicit arguments; ordinary results unused. |
| `F-unknown-25` | Supplies the destination for halfword `0x21C4`. |
| `F-unknown-26`–`F-unknown-27` | No visible explicit arguments; ordinary results unused. |
| `F-unknown-28` | Supplies the late low and high words. |
| `F-unknown-29`–`F-unknown-30` | Intervene before destination-object word stores. |
| `F-unknown-31` | Associated secondary output supplies a halfword-store address component. |
| `F-unknown-32`–`F-unknown-36` | No visible explicit arguments; ordinary results unused. |
| `F-unknown-37` | Receives the word read through the scaled late high word. |

The represented target expressions establish these equalities:

- `F-unknown-05` and `F-unknown-06` target the same address.
- `F-unknown-09` and `F-unknown-13` target the same address.
- `F-unknown-17` and `F-unknown-24` target the same address.
- `F-unknown-19` and `F-unknown-31` target the same address.
- `F-unknown-25` and `F-unknown-32` target the same address.

These are established target relationships, despite unknown dependency identities and behavior. Separate call-site IDs preserve ordering and distinct uses of results; they do not negate those equalities.

“No visible explicit arguments” does not establish an argument-free machine interface. Hidden register inputs, global inputs, return conventions, preservation rules, purity, and termination remain unspecified.

## Acceptance examples

These are conditional static examples, not runtime tests. Each assumes execution reaches the relevant operation and that dependencies and memory permit continuation.

1. **Early unsigned shifts.** If the early low word is `0xFC000003`, the frame-relative store at offset `0x26C` receives 63. The later pointer-relative store at offset `0x54` receives `0xF000000C`.

2. **Unsigned selector addressing.** If the first selector is `0xFFFF`, the destination-pointer read uses address `0x10043`; the later argument read uses `0x10003`. The selector is not interpreted as negative one.

3. **Indexed halfword store.** If `F-unknown-19` returns 3 and the captured second global value is `0x12345678`, the store targets `0x12345684` and writes `0x5678`.

4. **Retained, zero-extended halfword.** If the late-result halfword read produces `0xFEDC`, both destination-object word stores receive `0x0000FEDC`. An intervening change to the original halfword does not visibly change either retained value.

5. **Absolute selector-derived accesses.** If the second selector is 2, the late low-word store targets `0x7C`, and the later byte read targets `0x0A`. Neither includes an additional object base.

6. **Frame-base transition.** With a represented pre-decrease frame base of `0x1000`, the late low-word frame store targets `0x1258`. The explicit decrease produces `0x0F0C`, with no visible restoration before repetition.

7. **Breakpoint boundary.** If breakpoint `0x92` does not resume, subsequent indexed and final stores are not reached. Earlier effects are not visibly reversed.

8. **Loop-carried target.** If the iteration reaches the link assignment and final call, and execution follows the represented loop, the next initial target is `0x0871302F`. The first iteration need not use that target. Corresponding physical register preservation is not proven.

9. **Equal targets, distinct observations.** `F-unknown-05` and `F-unknown-06` must be described as calls to the same represented address, while retaining their different visible argument and result uses. Target equality does not establish a pure or conventional callable interface.

## Uncertainties

### Established static evidence

The representation establishes call order, explicit access widths and offsets, captured-value reuse, the 15-write inventory, breakpoint placement, frame-like decrease, fixed link assignment, represented next-iteration target, listed target equalities, and unconditional loop continuation after the final call.

### Inferred intent

Repeated offsets and selector-derived accesses suggest structured-state manipulation. That suggestion is non-normative: no application meaning, complete layout, or valid selector range is established.

A modular 32-bit interpretation is useful for describing address calculations, but it is not independent evidence of executable signed-overflow or signed-shift behavior.

### documented_only unknowns

- Actual dependency identities and behavior, beyond the established represented target equalities.
- The calling convention for combined results and secondary outputs.
- Physical register preservation across calls, including the loop-carried link-like state.
- Whether the frame-like base is a stack pointer, context pointer, or representation artifact.
- Validity and lifetime of referenced memory across dependencies.
- Endianness, alignment, memory mapping, and access-fault behavior.
- Breakpoint handling and whether execution resumes.
- Exact executable behavior for signed shifts, overflow, and address conversion.
- Whether repeated frame-like decreases correspond directly to executable state changes.

No specific error status, exception type, recovery guarantee, or ownership policy is established.

## Evidence

This contract is based solely on the supplied static representation of `F-de9d707f2dc0`. The prior draft and reviewer feedback are not treated as independent behavioral evidence.

No implementation access, dependency-body inspection, instruction-level verification, or runtime tests have been performed. The acceptance examples are conditional arithmetic and ordering consequences of the supplied representation, not observed executions.
