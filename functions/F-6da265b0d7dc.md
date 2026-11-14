<!-- Trusted provenance: F-6da265b0d7dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6da265b0d7dc

## Purpose

F-6da265b0d7dc performs an unconditional repeating sequence of indirect calls, calculated-address word stores, selector-based byte stores, direct byte clears, and word reads.

The supplied static representation establishes these operations and their data dependencies. It does not establish an application-level purpose. Descriptions such as allocator, serializer, device initializer, or object manager would require evidence not supplied here.

Three properties govern this contract:

- There is no explicit normal exit from the repeating body.
- Opaque call results determine addresses, stored values, and state carried into the next iteration.
- Repeated reads separated by opaque calls remain distinct observations. An earlier selector value or global value cannot substitute for a later read unless the representation explicitly retains it.

All behavior below is conditional on execution reaching the relevant operation. No runtime tests have been performed.

## Interface

### Explicit inputs and result

The represented interface contains four 32-bit input positions and no declared return value.

| Input position | Established explicit use |
|---|---|
| First | No explicit use in the displayed body. |
| Second | No explicit use in the displayed body. |
| Third | Supplies a shifted argument early in the iteration. Later replaced by a call result, used for a word store, and carried into the next iteration. |
| Fourth | Supplies the first word-store address and a shifted call argument. Later replaced by four times the low word of a wide call result and carried into the next iteration. |

The absence of explicit uses for the first two inputs does not prove that their machine-level values are irrelevant to dependencies. The recovered indirect-call interfaces are incomplete.

If the body and all dependencies complete normally, execution repeats rather than returning. Dependency nontermination, abnormal control transfer, and memory faults are not specified by the available evidence.

### Additional entry state

The first iteration also depends on:

- A preexisting register-derived base used for its first selector-byte read.
- A preexisting return-address-derived basis for its first indirect target.
- Memory at every calculated access address.
- A global storage location read as a 32-bit word late in the iteration.
- Any implicit inputs consumed by opaque dependencies.

The four explicit inputs alone therefore do not determine the first iteration.

### Widths and arithmetic model

The visible operations use 8-bit bytes, 32-bit words, and several 64-bit dependency results split into low and high 32-bit words.

For the formulas in this contract:

- Word additions, multiplications, and left shifts retain their low 32 bits.
- Byte stores retain only the low eight bits.
- Selector reads produce unsigned values from 0 through 255.
- Right shifts of unsigned words are logical.
- “Bits 20–27” includes both endpoints, numbering the least significant bit as bit zero.

These rules describe the apparent fixed-width dataflow. They do not resolve language-level signed-overflow questions, executable instruction semantics, or address-space rules.

Recurring access forms include word accesses at a calculated base plus `0x14`, selector-byte reads at a calculated base plus `1`, and byte stores at the numeric selector value plus `0x11`. These forms do not establish a validated structure layout.

A selector-based byte destination lies between `0x11` and `0x110`. The late word destination based on a saved selector lies between `0x14` and `0x113`. Neither alignment nor mapping is guaranteed.

## Behavior

The following stages describe one iteration in order. Descriptive names identify values within this contract, not implementation identifiers.

Opaque calls between accesses are part of the behavior. Calls with no explicit recovered arguments are not presumed argument-free, pure, or removable. Dependency-only sequences summarized below still contain observable calls whose effects are unresolved.

### 1. Initial payload and paired word stores

Three indirect calls occur before the first explicit memory store. The first target uses the incoming return-address-derived basis. The third call returns a word.

Define the **first payload** as four times that returned word.

The function then:

1. Stores the first payload as a word at fourth input times `0x1000`, plus `0x14`.
2. Calls a dependency with the fourth input shifted left by 14 bits.
3. Performs another dependency sequence ending in a returned address factor.
4. Stores the same first payload at four times that factor, plus `0x14`.
5. Calls a dependency with four times that factor.

The two stores are separate writes. There is no explicit read-back, comparison, or success check.

### 2. Initial selector update and two selected clears

A later dependency returns a 64-bit value. Define the **first scaled base** as four times its low word.

Before forwarding the high word, the function reads a selector byte at the incoming register-derived base times `0x1000`, plus `1`. It writes bits 20–27 of the first payload to selector plus `0x11`.

The displayed extraction masks the payload with `0x3fffffff` and shifts right by 20. That intermediate result may contain ten bits, but the byte store retains only eight.

The wide result’s high word is then passed to a dependency. Further calls obtain and forward a word, obtain another wide result, and forward that result’s low and high words together.

Two separate selected clears follow, with opaque calls between them:

- Read a selector at first scaled base times `0x10000000`, plus `1`; store a zero byte at selector plus `0x11`.
- Later, read a selector at four times the first scaled base, plus `1`; store a zero byte at selector plus `0x11`.

A call following the first clear receives four times an intervening returned word. A call following the second clear receives the first scaled base shifted left by 10 bits.

The selector address in the second clear recurs later. Its contents are reread, not implicitly retained.

### 3. Third-input argument and second payload pair

A returned word is passed to a dependency together with the current third input shifted left by 28 bits.

After another dependency sequence, the function rereads the selector at four times the first scaled base, plus `1`, and clears the selected byte at offset `0x11`. A following call receives four times an intervening returned word.

A word obtained earlier supplies the **second address factor**. Define:

- **Second scaled base:** four times the second address factor.
- **Second payload:** second address factor shifted left by 14 bits.

The function stores the second payload at second scaled base times `0x1000`, plus `0x14`.

It calls a dependency with the second scaled base shifted left by 14 bits. After another call, a dependency receives the second payload shifted left by 28 bits and an auxiliary register-derived value.

Under the stated 32-bit model, the shifted payload argument is zero: the payload already has at least 14 low zero bits. The call remains significant; its auxiliary argument and side effects remain unresolved.

Further calls produce another address factor. The function stores the second payload at four times that factor, plus `0x14`, then calls a dependency with four times the factor.

### 4. Second payload selector update and third payload pair

A later call supplies another factor. Define the **third scaled base** as four times that factor.

The function rereads the selector at four times the first scaled base, plus `1`, and stores bits 20–27 of the second payload at selector plus `0x11`.

The following dependency sequence includes:

- A call receiving the second address factor shifted left by 26 bits.
- A call receiving a subsequently returned word and the third scaled base shifted left by 28 bits.
- A call receiving the low and high words of another wide result.

After further calls, the function rereads the same selector address and clears its selected byte. A following call receives four times an intervening result.

Another returned word, multiplied by four, becomes the **third payload**. The function then:

1. Stores this payload at third scaled base times `0x1000`, plus `0x14`.
2. Calls a dependency with four explicit values: third scaled base times `0x400`, that same value again, the factor underlying the third scaled base shifted left by 14 bits, and the third payload.
3. Performs further calls that produce another address factor.
4. Stores the third payload at four times that factor, plus `0x14`.
5. Calls a dependency with four times that factor.

### 5. Two self-derived word pairs

Two successive groups each use a fresh returned factor.

Within each group:

- Store four times the factor at factor times `0x10`, plus `0x14`.
- Call a dependency with the stored value in three explicit argument positions.
- Perform further calls yielding a fresh destination factor.
- Store the same value at four times that destination factor, plus `0x14`.
- Call a dependency with four times the destination factor.

These are four word stores in total, not one operation reused twice.

The original factor in the second group also defines the **terminal base** as that factor times `0x40000000`. Only its bottom two bits influence the terminal base under 32-bit arithmetic.

The destination factor used for the second group’s copied word is retained for a later one-bit payload.

### 6. Direct clearing pairs and indexed word read

Another dependency sequence returns an **index word**. The function then performs two direct byte stores in this order:

1. Zero at terminal base plus `0x11`.
2. Zero at terminal base plus `1`.

These stores use the terminal base directly. They do not read selectors.

A dependency receives four times the index word. Further calls obtain and forward another word. After additional calls, the same ordered pair of direct zero-byte stores occurs again. A subsequent dependency receives four times another returned word.

A later call supplies the **final scaling factor**. The function stores a word at final scaling factor times `0x10`, plus `0x14`. The value is the retained destination factor from the second self-derived group shifted left by 31 bits. Only that retained factor’s least significant bit survives.

Immediately afterward, with no intervening call, the function reads a word at:

`0x14 + ((index word masked by 0x3fffffff) shifted right by 20)`.

The added displacement ranges from zero through 1023. It is not multiplied by four. The word read can be unaligned.

A dependency receives four times the final scaling factor. The next returned word replaces the third input.

Another dependency receives the indexed word shifted right by 22 bits and then left by 28 bits. Under 32-bit arithmetic, only original bits 22–25 contribute.

### 7. Final payload updates and saved-selector word store

After further calls, the function stores a zero word at four times a newly returned factor, plus `0x14`, then calls a dependency with four times that factor.

A later returned word becomes the **final selector payload**.

The function reads the selector at terminal base plus `1` and stores the low byte of four times the final selector payload at selector plus `0x11`.

Subsequent calls receive:

- Four times the final selector payload.
- A returned word together with the final selector payload shifted left by 30 bits.

After further calls, the function rereads terminal base plus `1` and saves this selector value. It stores bits 20–27 of the final selector payload at saved selector plus `0x11`.

A dependency receives four times another returned word, followed by additional calls. The function then stores four times the updated third input as a word at saved selector plus `0x14`.

This word destination uses the saved selector. It does not reread terminal base plus `1`. The intervening calls may alter memory, but the represented dataflow retains the earlier selector value.

### 8. Global snapshot and loop-carried updates

Immediately after the saved-selector word store, the function reads the global word and retains its value.

A dependency receives four times the final scaling factor. Further calls supply a wide result. Four times its low word replaces the fourth input.

The function rereads the selector at terminal base plus `1`. At selector plus `0x11`, it stores the low byte of the global snapshot shifted right by 22 bits: original global bits 22–29.

The wide result’s high word is forwarded to a dependency. Another returned word is forwarded together with the updated fourth input shifted left by 28 bits.

After additional calls, the function again rereads terminal base plus `1` and clears the selected byte at offset `0x11`.

The remaining sequence includes a call receiving four times an intervening result, further calls, a call receiving four times another result in two explicit argument positions, and one final call.

Before the two-argument call, the represented register-derived base is updated to four times the new fourth input. Before the final call, the return-address-derived basis is replaced with a fixed internal-continuation-derived value. Execution then repeats.

## Side effects

Each normally completed iteration contains the following explicit memory operations:

| Operation | Count |
|---|---:|
| 32-bit word stores | 13 |
| Selector-based byte stores | 10 |
| Direct byte stores | 4 |
| **All byte stores** | **14** |
| Selector-byte reads | 10 |
| Calculated-address word reads | 1 |
| Global-word reads | 1 |

The ten selector-based stores each use one selector-byte read. In execution order, they store:

1. Bits 20–27 of the first payload.
2. Zero through the selector at the highly scaled first base.
3. Zero through the recurring first-base selector.
4. Zero through that recurring selector again.
5. Bits 20–27 of the second payload.
6. Zero through the recurring selector.
7. The low byte of four times the final selector payload.
8. Bits 20–27 of the final selector payload, saving this selector.
9. Global snapshot bits 22–29.
10. Zero through a final fresh selector read.

The four direct byte stores are exactly two ordered clearing pairs: terminal base plus `0x11`, then terminal base plus `1`. They introduce no additional selector reads.

Counts exclude dependency effects and do not imply distinct addresses. Aliasing is unrestricted by the evidence. Word stores may overlap later selector reads, selected destinations, the global storage, or other word stores. In particular, the global snapshot occurs after the saved-selector word store and can observe its effects if they overlap.

No explicit allocation, release, ownership transfer, synchronization, rollback, or error reporting is established. Dependencies may perform such operations; their absence from the visible body does not constrain them.

## Dependencies

**F-opaque-0001** denotes the unresolved dependency set. It is not a claim that all indirect calls invoke one function.

This set includes calls with entry-state-relative or internal-continuation-relative targets; calls with no explicit recovered arguments; calls returning words or wide pairs; and calls receiving shifted values, forwarded results, repeated arguments, or auxiliary register state.

Different occurrences may resolve to different functions. Similar target calculations do not establish identical behavior.

No dependency is assumed to:

- Preserve global memory.
- Preserve memory reachable through calculated addresses.
- Validate addresses or arithmetic.
- Acquire, retain, transfer, or release ownership.
- Return normally.

Exact identities, complete signatures, implicit arguments, machine-level preservation rules, side effects, and failure modes remain documented_only.

## Acceptance examples

These are conditional static consequences, not executed tests. Memory examples assume the relevant accesses are valid and execution reaches them.

1. **Initial multiplication truncation.** If the first payload-producing call returns `0x40000001`, the first payload is `0x00000004`. Both initial word stores use that value.

2. **Byte extraction width.** For a payload of `0x3ff00000`, masking and shifting yields `0x3ff`, but the selected byte store writes `0xff`.

3. **Fresh selectors after calls.** If a recurring selector is 7 for one update and becomes 9 before the next read, the later selected destination is `0x1a`, not `0x18`.

4. **Saved selector retention.** If the saved late selector is 7, the late word store targets `0x1b`, even if a dependency subsequently changes terminal base plus `1` to 9. Byte ordering and unaligned-access behavior remain unresolved.

5. **Direct clears are not selected clears.** If the terminal-base factor is 5, the terminal base is `0x40000000`. Each direct clearing pair targets `0x40000011` before `0x40000001`, without first reading either address as a selector.

6. **Indexed-read upper boundary.** If index bits 20–29 are all set, the calculated word-read address is `0x413`. No word-size scaling is added.

7. **One-bit payload.** An even retained destination factor produces zero for the later left-by-31 word payload. An odd factor produces `0x80000000`.

8. **Global snapshot.** A captured global value of `0x00400000` produces a later selected byte value of 1, regardless of subsequent changes to the global itself.

9. **Per-iteration access totals.** A completed iteration contains 14 explicit byte stores: ten selected stores and four direct stores. It contains ten selector reads, not a read for every byte store.

10. **Continuation.** If every dependency and access completes normally, execution starts another iteration with the represented updated state. Normal return is not an outcome shown by this body.

## Uncertainties

The following remain documented_only:

- Application-level purpose and intended data model.
- Validity and identity of indirect call targets.
- Complete calling convention and implicit input flow.
- Initial register-derived and return-address-derived state.
- Exact executable arithmetic and address semantics.
- Endianness, alignment handling, mappings, and permissions.
- Whether low numerical addresses denote ordinary memory, device state, or reconstruction artifacts.
- Semantics of overlapping global storage symbols.
- Fault handling, recovery, dependency failures, and nonlocal control transfers.
- Whether the recovered unconditional loop accurately represents intended executable control flow.

The warning about an omitted unreachable block supplies no behavior for that block and does not justify adding an exit path.

## Evidence

**Established static evidence:** the supplied representation shows a four-position interface, an unconditional repeating body, indirect dependencies, fixed-width arithmetic, 13 word stores, ten selector-based byte stores, four direct byte stores, ten selector-byte reads, one calculated-address word read, one global-word read, and explicit loop-carried updates. The direct byte stores appear as two ordered pairs without selector dereferences.

**Inferred intent:** recurring offsets and paired payload stores suggest repeated layout or calling conventions. No particular object model, protocol, ownership scheme, or application role is adopted.

**Unknown:** dependency implementations, complete machine-level interfaces, environmental validity, and successful execution.

No implementation execution, runtime observation, or tests have been performed.
