<!-- Trusted provenance: F-d810ceb01fa7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d810ceb01fa7

## Purpose

F-d810ceb01fa7 performs stateful indirect dispatch, a conditional retry, branch-dependent processing, table reads, and ordered memory mutation.

Its application-level purpose is **documented_only**. The supplied static evidence does not establish allocation, loading, decoding, object construction, or another domain-level service. Embedded labels do not establish those meanings.

This contract describes effects conditional on reaching them: earlier accesses must succeed and earlier dependencies must return. The early-dispatch path visibly permits a normal return. The other visible terminal paths stop at invalid instruction data; they do not establish successful completion.

## Interface

### Explicit inputs and output

| Input | Width | Established role |
|---|---:|---|
| Primary value | 32 bits | Stored in state; replaced by the retry result when retry occurs. |
| Auxiliary argument | 32 bits | No explicit use is visible. |
| State address | 32 bits | Used for stores, dependency arguments, and a signed address-derived comparison. |
| Control value | 32 bits | Used in arithmetic and shifts; its low byte is stored in state. |

The nominal interface has no return value. No returned status or exception contract is established.

### Implicit inputs

Behavior also depends on:

- An incoming carry flag.
- A 32-bit unsigned value serving as a source address and shift state.
- An address through which a signed 16-bit quantity is loaded.
- A dispatch marker.
- A 32-bit link-derived value used to calculate dependency targets.
- Incoming stack contents, including a branch-specific argument.
- Secondary register results from dependencies.
- Global words, an embedded base word, and low-address memory.

Initialization and ABI treatment of these inputs are **documented_only**. Explicit arguments alone do not determine behavior.

### Direct state layout

| State-relative offset | Visible operation |
|---:|---|
| `0x10` | 32-bit primary-value store at the start of every loop iteration. |
| `0x90` | Address passed to a branch-entry dependency; no access width follows from this use. |
| `0x99` | 8-bit store of the control value’s low byte after loop exit. |

No enclosing object size is established. No null, bounds, alignment, or permission checks are visible.

Address-like values and pointer-valued stores are represented at 32-bit width. Other direct accesses have 8-, 16-, or 32-bit widths. Some dependency results are represented as 64 bits and subsequently split into 32-bit halves or used for bit extraction. Byte order and unaligned-access behavior are **documented_only**.

## Behavior

### Initial write, dispatch, and retry

Every loop iteration first stores the primary value at state offset `0x10`. This write precedes both the dispatch-marker test and the carry-dependent exit test.

A nonzero marker selects early dispatch:

- A preliminary dependency target is the link-derived value with its lowest two bits cleared, plus `0x34C`.
- Its 32-bit result becomes the first argument to an unresolved indirect dispatch target.
- The second argument is a zero-extended byte from source address plus one.
- The third argument is the control value arithmetically shifted right by three, then decreased by `0x55`.
- The fourth argument is a separately expressed zero-extended byte load from source address plus one, shifted left by 30 at 32-bit width.
- If the dispatch target returns, F-d810ceb01fa7 returns.

Only the source byte’s lowest two bits survive the fourth-argument shift. The two byte-load expressions do not establish their relative evaluation order or whether they are coalesced. Equivalence must not require memory to remain stable between them.

With a zero marker, clear carry or a zero primary value exits the loop. Otherwise, retry occurs:

1. The retry target is calculated from the previous, **unmasked** link-derived value plus `0xCCC`.
2. Before invocation, the carried link-derived value is replaced by `0x083FF749`.
3. The already-calculated target is invoked, and its result replaces the primary value.
4. Carry becomes bit 29 of the current source/shift value.
5. That value is shifted left by three at unsigned 32-bit width.
6. The marker becomes the new primary value.

The replacement link-derived value is retained. It affects later target calculations.

The next iteration writes the retry result before testing it. A nonzero result selects dispatch; zero exits the loop. Consequently, the visible updates permit at most one retry invocation.

Target alignment is selective: early dispatch masks two low bits, retry does not, and the two subsequent branch-entry calculations use different masking rules.

### Preparation and branch selection

After loop exit, the function forms state address plus `0x12`, stores the control value’s low byte at state offset `0x99`, and then loads and sign-extends a 16-bit quantity through the implicit halfword address.

The byte store precedes the halfword load. Aliasing can therefore affect the loaded quantity.

The branch condition compares state address plus `0x12`, interpreted as signed, against `0xD2`. It does not compare the loaded halfword and is not established as a length check. Signed-overflow behavior in the address-derived arithmetic is **documented_only**.

### Lower-comparison branch

When the comparison value is signed-less-than `0xD2`, the control value is shifted left by two and an ordered chain of indirect dependencies executes.

The first target is the current, unmasked link-derived value plus `0xFFE`; it receives state address plus `0x90`. A prior retry therefore changes this destination.

Subsequent targets are derived from internal continuation addresses. Most use unmasked addition of `0xFFE`; one uses `0xF0E`. Later shared stages explicitly mask low target bits. These forms must not be normalized into a universal alignment rule.

Observable dataflow includes:

- Passing an opaque code address to a dependency, without establishing exception semantics.
- Passing one returned word both doubled and unchanged.
- Passing several returned words as equal argument pairs.
- Forwarding returned words through successive calls.
- Splitting a represented 64-bit result into low and high 32-bit words.
- Extracting bit 28 from a high word, bit 24 at two later stages, and bit 29 at two later stages. Each extraction yields one bit.
- Reading a byte at absolute address `0x10` for a nonnegative signed result, or `0x11` for a negative result.
- Later reading a byte at `0x10` plus an extracted bit 29.

The branch supplies a 32-bit result to shared processing. The initially loaded signed halfword remains the branch-dependent value used by a later 16-bit store.

These calls are ordered external interactions. Similar argument patterns do not establish purity or redundancy. Calls with no displayed arguments do not establish a zero-input ABI.

### Other-comparison branch

When the signed comparison is false, the branch-entry target is the link-derived value with its lowest two bits cleared, plus `0xA34`. It receives state address plus `0x90` and an incoming stack-derived argument.

The next visible order is:

1. Assign `0x7E` to one global location.
2. Invoke another dependency.
3. Assign `0x3F0` to another global location.
4. Read a byte at a secondary call-result address plus the comparison value.
5. Assign the comparison value and loaded signed halfword to global locations.
6. Invoke a dependency with an opaque code address.

The widths of the overlapping global assignments are not fully established.

After this call, a global base word is read and saved. A signed 32-bit word at comparison value plus `0x24` is loaded and arithmetically shifted right by 13 to form a table address. Table words are then read at offsets zero, eight, and four, in that order.

The offset-four word is multiplied by four to form the **fourfold value**, represented as a 32-bit quantity. A stop at invalid instruction data occurs when both conditions hold:

- The fourfold value is nonzero.
- Its signed-negative predicate equals the signed-addition overflow flag for the control value plus the implicit halfword address interpreted as a signed 32-bit operand.

This is not a returned validation error. Arithmetic outside the representable signed range is not given a portable overflow contract by the supplied representation.

If execution continues, a base-relative 32-bit store uses **fresh reads of the global base**. Its destination is a freshly read base plus the implicit halfword address; its stored value also reads that global. The relative order of these two reads is not established. Neither may be replaced with the earlier saved base merely because the values might agree.

The subsequent ordered effects are:

- Invoke a dependency with table address plus 12 and the saved offset-zero entry.
- Store the 32-bit implicit halfword address at fourfold value plus `0x1C`.
- Store the earlier saved base’s low 16 bits at a secondary call-result address plus two.
- Change the represented stack base.
- Perform the second 16-bit store described below.
- Invoke another dependency.
- Load a 32-bit word at the earlier saved base plus the saved offset-eight entry.

**The second 16-bit store’s destination is twice the sum of the fourfold value and `0xA5`, equivalently twice the fourfold value plus `0x14A`, wherever the represented arithmetic is defined.** The stored value is the low 16 bits of the fourfold value plus `0xA5`. The address is not twice the fourfold value plus `0xA5`: the addition occurs before doubling.

The fresh-base store and saved-base operations remain distinct even when their bases happen to match.

The control value is replaced by the implicit halfword address plus the saved offset-eight entry and a one-bit adjustment derived from the earlier byte. The adjustment tests bit 29 after adding `0x62` to that byte’s promoted value using unsigned 32-bit arithmetic. A signed byte from −128 through −99 produces one; an unsigned byte produces zero. The byte’s signedness is **documented_only**.

The saved offset-eight entry replaces the initial halfword as the branch-dependent value for the shared 16-bit store.

### Shared processing and low-address table

Shared processing propagates 32-bit and represented 64-bit dependency results. It includes:

- Passing low words, sometimes twice as equal arguments.
- Extracting bit 31 after unsigned 32-bit subtraction of seven from a high word.
- Separately selecting a byte at `0x10` or `0x11` according to the sign of a signed high-word-minus-seven expression.
- Extracting bit 63 from two later 64-bit results.
- Extracting bit 22 from a later 32-bit result.

The unsigned subtraction-and-extraction is not universally equivalent to a mathematical “high word less than seven” predicate. The separate signed subtraction retains unresolved overflow behavior.

A 16-bit value `0x002F` is stored at twice the branch-dependent value: the initial signed halfword on the lower-comparison branch, or the saved offset-eight table entry on the other branch.

Following further calls, the current control value is decreased by `0x33` using unsigned 32-bit arithmetic and logically shifted right by 24. The result, from zero through 255, is a literal table address, not a scaled index.

F-2cc1255261f3 receives the top four bits of a preceding 32-bit result, giving an argument from zero through 15. Additional indirect calls occur before table reads; their effects must not be assumed absent.

The low-address table is then read as four 32-bit entries at offsets zero, four, eight, and 16, in that order. Offset 12 is not explicitly read.

### Record construction and snapshots

Four ordered 32-bit stores construct a low-address record:

| Address | Stored value |
|---:|---|
| `0xC3` | Secondary register result from a preceding dependency. |
| `0xC7` | Record pointer `0xC3`. |
| `0xCB` | Saved low-table entry from offset eight. |
| `0xCF` | Saved low-table entry from offset 16. |

These potentially unaligned addresses are literal represented effects, not evidence of a conventional object layout.

The following ordering is significant:

1. After record construction, read and save the embedded base word.
2. Invoke the record-consuming dependency with the saved offset-zero entry, the earlier secondary result, record pointer `0xC3`, and the saved offset-four entry.
3. After that call, read and save the unsigned 32-bit threshold global.
4. Store the call’s secondary result at the saved embedded base plus `0x68`.
5. Compare the saved threshold unsigned with nine.

The record-consuming call may change the embedded base, but the subsequent destination uses its pre-call snapshot. The secondary-result store may alias the threshold global, but the comparison uses the threshold sampled before that store.

### Terminal effects

A sampled threshold of nine or greater reaches an invalid-instruction stop after the secondary-result store.

For a sampled threshold below nine, another dependency yields a represented 64-bit result. Four 32-bit stack-relative stores follow in this exact order:

| Active stack-relative destination | Stored value |
|---:|---|
| Base minus four | Saved low-table entry from offset 16. |
| Base minus eight | Sampled threshold. |
| Base minus twelve | Record pointer logically shifted right by 14. |
| Base minus sixteen | Saved embedded base. |

The record pointer is `0xC3`, so its logical right shift by 14 is zero. The active represented stack base includes the other branch’s stack-base change, if that branch executed.

Next, the threshold’s low byte is stored at saved embedded base plus `0x0F`. A 16-bit store at sampled threshold plus `0x28` receives the low 16 bits of the stack-relative address at base minus twelve, increased by `0x398` and retained at 16-bit width.

A final dependency receives the terminal result’s low and high 32-bit halves and `0x74`. Execution then reaches invalid instruction data.

The sampled threshold, saved offset-16 entry, and saved embedded base are reused rather than re-read.

## Side effects

Direct effects include state writes, overlapping-global assignments, table reads, pointer-valued stores, low-address record writes, stack-base mutation, terminal stack writes, and external calls.

Aliasing makes ordering observable. Relevant boundaries include the state-byte store before the halfword load, record construction before the embedded-base snapshot, the record-consuming call before threshold sampling, and threshold sampling before the secondary-result store. On the other-comparison branch, stack-base mutation precedes the 16-bit store at twice the sum of the fourfold value and `0xA5`.

No ownership transfer is established. Passing an address proves neither allocation nor retention, release, or borrowing. Dependencies are not established to preserve globals, shared memory, or unspecified machine state.

No rollback, atomicity, reentrancy, or thread-safety guarantee is visible. Faults, nonreturning dependencies, and invalid-instruction stops may follow completed mutations.

## Dependencies

**F-2cc1255261f3** is the only external dependency with a supplied stable opaque identity. It receives one visible four-bit unsigned argument. Its displayed return value is unused. Side effects, ownership behavior, failures, and nonreturning behavior are **documented_only**.

All other dependency identities are unresolved, **documented_only**. Their described roles—retry, dispatch, branch processing, record consumption, and terminal processing—are not invented identities.

Some targets use low-bit masking and others do not. Some dependencies expose secondary register results or represented 64-bit results. Displayed argument lists do not establish complete ABI contracts.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume preceding accesses succeed, dependencies return, and relevant arithmetic is defined.

1. **Immediate dispatch:** A nonzero initial marker causes the primary-value store before dispatch. If dispatch returns normally, the later state-byte store is not reached.

2. **Loop exit:** A zero marker with clear carry or zero primary value avoids retry while retaining the initial state write.

3. **Retry link state:** Retry invokes the old unmasked link-derived value plus `0xCCC`, but carries `0x083FF749` afterward. A result of five is written on the next iteration before dispatch uses the replacement link state.

4. **Byte widths:** Control value `0x123456AB` stores `0xAB` at state offset `0x99`. A dispatch source byte of three produces fourth argument `0xC0000000`.

5. **Corrected halfword destination:** On the other-comparison branch, a saved offset-four table word of one gives fourfold value four. If the invalid-instruction guard permits continuation, the second 16-bit store writes `0x00A9` at address `0x152`: twice `0xA9`, equivalently eight plus `0x14A`. Address `0xAD` is not the represented destination.

6. **Zero fourfold value:** A fourfold value of zero makes the branch’s invalid-instruction guard false. If subsequent effects are reached, the second 16-bit store writes `0x00A5` at address `0x14A`.

7. **Low-table arithmetic:** Current control values `0x32` and `0x33` derive numeric table addresses `0xFF` and zero, respectively.

8. **Saved versus fresh base:** Changes to the global base affect the fresh-base 32-bit store, while the later saved-base halfword value and word-load address continue to use the earlier snapshot.

9. **Embedded-base mutation:** If the record-consuming dependency changes the embedded base word, the secondary-result destination and terminal base-dependent effects still use its pre-call snapshot.

10. **Threshold aliasing:** If the sampled threshold is eight and the secondary-result store overwrites its global location with nine, the below-nine path still uses eight.

11. **Terminal ordering:** That path stores the saved offset-16 entry, sampled threshold, zero, and saved embedded base at the four descending stack-relative destinations, in order. Aliasing does not authorize reordering.

12. **Threshold boundary:** A sampled value of nine skips the additional terminal dependency and stack writes. Neither threshold outcome establishes successful return.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and valid entry context.
- Authentic function boundaries and control-flow completeness.
- Initialization and ABI treatment of implicit inputs.
- Unresolved dependency identities, effects, and preservation guarantees.
- Widths of overlapping global assignments.
- Byte signedness where extension is not explicit.
- Signed-overflow behavior, including multiplication, address arithmetic, subtraction, and negative left shifts.
- Byte order, alignment requirements, and invalid-address behavior.
- Whether invalid-instruction stops represent authentic execution or recovery artifacts.
- Behavior after faults or nonreturning calls.
- Successful completion beyond early dispatch return.

The corrected halfword destination establishes grouping, not a general signed-overflow rule. Its addition precedes doubling; behavior outside the defined arithmetic range remains unresolved.

## Evidence

**Established static evidence:** The supplied representation supports the stated access widths, dispatch/retry ordering, retained link replacement, selective target masking, branch-dependent snapshots, table offsets, record layout, threshold sampling, and terminal write order. It places addition of `0xA5` before doubling in the other-comparison branch’s second 16-bit store destination, while storing only the low 16 bits of the undoubled sum.

**Inferred intent:** Dispatch and table-driven mutation characterize visible mechanics only. They do not establish a domain-level purpose or justify repairing suspicious addresses.

**Evidence limitations:** The supplied material reports invalid instruction data, removed unreachable regions, unresolved indirect control flow, and overlapping globals. These limit conclusions about completeness and executable behavior.

No runtime tests have been performed.
