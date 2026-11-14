<!-- Trusted provenance: F-9ee192ced3b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ee192ced3b0

## Purpose

The supplied static representation describes a routine that interleaves indirect dependency calls with computed-address memory accesses. It propagates 32-bit values, stores selected portions of retained results as bytes, and repeatedly clears a word at absolute address `0x24`.

No application-level purpose is established. The observed offsets and repeated stores do not establish allocation, initialization, serialization, object construction, or cleanup.

The recovered path terminates at an invalid-instruction boundary. No ordinary return, success indication, or completed higher-level operation is established.

This contract distinguishes established static evidence from conditional machine-level interpretation and unresolved behavior. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First input | 32 bits | No explicit use in the recovered body. |
| Second input | 32 bits | Passed twice to the first indirect invocation. |
| Third input | Unsigned 32 bits | Passed to the first invocation, used in an address calculation after a logical right shift by three, and stored twice as a full word. |

The first input’s lack of explicit use does not establish irrelevance under the actual entry convention.

### Additional required state

The visible body also consumes:

- An incoming control-flow value used to determine the initial indirect target.
- An incoming unsigned 32-bit auxiliary value, shifted right by three for the first invocation.
- An incoming 32-bit payload, written after the first five-call group.

The three explicit inputs therefore do not independently specify a reproducible invocation.

A secondary result used substantially later is exposed by an intervening dependency call; it is not established as an entry input.

### Outputs and execution conditions

The displayed interface has no return value, and the recovered path contains no normal return. Observable outputs are memory accesses, dependency invocations, dependency-generated effects, and whatever occurs at the invalid-instruction boundary.

Continuation through the represented path requires readable and writable computed addresses, executable indirect targets, compatible calling conventions, and dependencies that return when continuation is expected. These conditions are not visibly checked.

No recoverable error protocol, rollback mechanism, output-buffer contract, or ownership-transfer result is established.

## Behavior

### Widths, addresses, and terminology

A **byte** is eight bits; a **word** is 32 bits. One dependency result is represented as 64 bits and consumed as separate lower and upper 32-bit halves.

The descriptions below use these mathematical operations:

- **Fourfold:** multiplication by 4.
- **Sixteenfold:** multiplication by 16.
- **Reduced form:** clear the top two bits of a 32-bit value, then shift logically right by one.
- **High-derived byte:** clear the top two bits, shift logically right by 20, then retain the low eight bits.
- **Byte-selected destination:** read an unsigned selector byte and use its numeric value plus `0x11` as the destination address.

A byte-selected destination ranges from `0x11` through `0x110`. It is not the selector’s address plus an offset, nor a pointer loaded from the selector’s location.

Unsigned calculations have their represented 32-bit behavior. In particular, unsigned products and left shifts can discard high bits. Several other expressions are represented using signed 32-bit arithmetic. Their overflow and shift behavior is not established as portable language semantics; descriptions involving those expressions are conditional on the recovered machine-width interpretation. Executable address width remains unresolved.

Word stores write the full 32-bit value; byte stores retain only the resulting low eight bits under the machine interpretation. Endianness is not established.

### Dependency sequencing

A **five-call group** means one invocation with the specified explicit arguments, followed by four invocations having no explicit arguments in the representation. Unless stated otherwise, the fifth invocation supplies the group result.

This is an ordering convention, not a dependency identity or signature. The five invocations need not share a target. “No explicit arguments” does not prove absence of machine-level inputs. Calls whose results are unused still occur and can have observable effects.

The phases below are consecutive portions of the same recovered path.

### 1. Initial propagation

The initial five-call group receives, in order, the second input, the second input again, the third input, and the incoming auxiliary value shifted logically right by three. Its result selects a destination at fourfold that result plus `0x14`, where the incoming payload is stored as a word.

Three subsequent groups establish this propagation chain:

- The next group begins with fourfold the initial result. Fourfold its own result is stored at sixteenfold that result plus `0x14`.
- The following group begins with that stored fourfold value. The same value is stored at fourfold this new group result plus `0x14`.
- The following group begins with fourfold the preceding result. Fourfold its own result is stored at sixteenfold that result plus `0x14`.

A separate invocation receives fourfold the last result. A no-explicit-argument invocation follows; its result becomes the leading argument of another five-call group. At fourfold this group’s result plus `0x14`, the routine stores the reduced form of the fourfold value supplied to the separate invocation.

Another group begins with fourfold the preceding group result. Fourfold its own result is stored at sixteenfold that result plus `0x14`.

The routine then reads a word at this last fourfold value plus `0x14`. Its unsigned contents, shifted right by three, begin another group. Afterward, the same saved fourfold value is stored at fourfold the new result plus `0x14`.

The read is fresh and is not from the sixteenfold destination merely because both use offset `0x14`. Intervening dependencies may also have changed either location.

### 2. Base seed, input writes, and shift seed

A group beginning with fourfold the preceding result produces the **base seed**, a retained 32-bit value.

Before the next dependency call, the routine stores the third input as a word at:

\[
((\text{third input}\gg3)\times0x80000)+0x24.
\]

This is unsigned 32-bit arithmetic; high product bits can be discarded.

Two quantities are derived from the base seed:

- The **scaled base**, equal to sixteenfold the base seed.
- The **reduced seed**, equal to the reduced form of the base seed.

A five-call group begins with two explicit arguments: the scaled base and fourfold the base seed. Its final result is unused. After the group, the third input is stored again, at scaled base plus `0x24`.

The routine next reads a word at scaled base plus `0x74`. That word begins another five-call group, whose result is the **shift seed**.

The base seed and shift seed are distinct retained results:

- The base seed determines the scaled base, reduced seed, and first high-derived-byte write.
- The shift seed supplies a subsequent fourfold argument and the later shifts by 23 and 25.

Neither result is established to equal the word read at scaled base plus `0x74`.

### 3. Byte-selected updates and the first fixed clear

The routine reads a selector byte at scaled base plus one, then stores the **base seed’s high-derived byte** at selector plus `0x11`.

A separate invocation receives fourfold the **shift seed**. A no-explicit-argument invocation follows, and its result begins a five-call group. After that group, the selector at scaled base plus one is read again and zero is stored at the newly selected byte destination.

A further group begins with fourfold the preceding group result; its result is unused. The caller then:

- Stores a zero byte at reduced seed plus `0x0f`.
- Derives the **second base** as reduced seed multiplied by `0x400`.
- Reads a selector at second base plus one and stores zero at selector plus `0x11`.

A special invocation receives four arguments derived from the **shift seed**: its value shifted left by 23 in each of the first three positions, and shifted left by 25 in the fourth position. A no-explicit-argument invocation follows, and its result begins another five-call group.

The selector at second base plus one is then read again. Its numeric value is retained, and a zero byte is stored at that value plus `0x11`.

Another group begins with fourfold the preceding group result; its result is unused. The routine then stores a zero word at the **retained selector** plus `0x44`, followed by a zero word at absolute address `0x24`.

The retained selector is not reread for the word store. Its selected word address ranges from `0x44` through `0x143` and need not be aligned.

### 4. Further word clearing and split-result use

A special invocation receives four explicit arguments: zero, `0x98`, the second base, and `0x98`. The constants do not establish a size or allocation contract.

A no-explicit-argument invocation returns a value passed to a separate invocation. Another no-explicit-argument invocation then returns a value that begins a five-call group.

After this group, the caller clears a word at fourfold its result plus `0x44`, then clears the word at absolute address `0x24`.

The next group begins with fourfold the preceding group result. Its final result is represented as 64 bits:

- Fourfold its lower 32 bits is passed to a separate dependency.
- After that invocation, the caller clears a word at twice the saved upper 32 bits plus `0x44`.
- The caller then clears the word at absolute address `0x24`.

The upper half is retained from the 64-bit result, not obtained from the intervening invocation.

That intervening invocation exposes a secondary result. The next special invocation receives zero, that secondary result, and fourfold that secondary result. A no-explicit-argument invocation follows; its result is unused.

Define the **masked reduced seed** as the low 22 bits of the reduced seed. A five-call group begins with this masked value shifted left by seven. Afterward, the caller clears a word at fourfold the group result plus `0x44`, then clears the word at absolute address `0x24`.

Together with phase 3, this accounts for four explicit word clears at absolute address `0x24`. Dependencies intervene between them; they are separate observable events.

### 5. Zero stores and resumed propagation

A group begins with fourfold the preceding group result. Fourfold its result is passed to a separate dependency. This dependency’s return is the **first zero-store selection**. The caller stores zero at fourfold that selection plus `0x14`.

The next group receives four explicit arguments: the first zero-store selection three times, followed by the masked reduced seed shifted left by seven. Its result is the **second zero-store selection**. A zero word is stored at fourfold that result plus `0x14`.

Three propagation groups follow, with these explicit leading inputs and stores:

| Group | Leading argument | Subsequent word store |
|---|---|---|
| First propagation group | Fourfold the second zero-store selection | Fourfold this group’s result, at sixteenfold its result plus `0x14` |
| Second propagation group | Fourfold the first propagation group’s result | The first group’s saved fourfold result, at fourfold this second result plus `0x14` |
| Third propagation group | Fourfold the second propagation group’s result | Fourfold this third result, at sixteenfold its result plus `0x14` |

A special invocation receives fourfold and sixteenfold the third propagation result. A no-explicit-argument invocation follows.

Its result begins a five-call group with a second explicit argument equal to the masked reduced seed shifted left by four. Afterward, the caller stores the masked reduced seed multiplied by `0x80` at fourfold the group result plus `0x14`.

The masked value is at most `0x003fffff`. Its multiplication by `0x80`, equivalently its left shift by seven, is therefore at most `0x1fffff80`. Its left shift by four is at most `0x03fffff0`.

### 6. Final byte-selected updates

A group beginning with fourfold the preceding group result produces the **first tail seed**. The caller derives the **tail base** as reduced seed multiplied by `0x4000000`, with unsigned 32-bit truncation.

The remaining operations preserve these dependencies:

1. The caller reads a selector at tail base plus one and stores the low byte of fourfold the first tail seed at selector plus `0x11`.

2. A special invocation receives fourfold the first tail seed. A no-explicit-argument invocation follows, and **its result is passed as the leading argument of the following five-call group**. Retain that group’s result as the **tail continuation**. The caller rereads the selector at tail base plus one and stores the first tail seed’s high-derived byte at the newly selected destination.

3. The next five-call group begins with **fourfold the tail continuation**. Its result becomes the **second tail seed**. The caller rereads and retains the selector at tail base plus one, then stores the low byte of fourfold the second tail seed at selector plus `0x11`.

4. A special invocation receives the retained selector as an unsigned argument. A no-explicit-argument invocation follows, and **its result begins the following five-call group**. Retain that group’s result as the **final continuation**. The caller then reads a selector at eightfold the **second tail seed** plus one and stores the **second tail seed’s high-derived byte** at selector plus `0x11`.

Finally, fourfold the final continuation begins one last five-call group. Its final result is unused. The recovered path then reaches the invalid-instruction boundary.

The second tail seed, not the final continuation, determines the last selector-read address and byte contents.

## Side effects

The caller visibly performs eight-bit and 32-bit memory accesses. Offsets `0x14`, `0x24`, `0x44`, and `0x74` do not establish record layouts, object boundaries, or allocation sizes.

Observable ordering includes:

- Each selector read precedes its selected write.
- Repeated selector accesses are distinct reads, except where retention is explicitly stated.
- Computed word clears precede their associated fixed-address clears.
- Dependencies occur between the specified caller accesses and may observe or change memory.

Destinations may alias selectors, earlier destinations, absolute address `0x24`, or dependency-managed state. Even the caller’s own writes can affect later reads if addresses alias. No disjointness is guaranteed.

Writes are not transactional. Earlier effects need not be undone if a dependency fails to return, a memory access faults, or execution reaches the invalid-instruction boundary.

No allocation, release, retention, reference counting, or ownership transfer is established. Values passed to dependencies are not thereby proven to be pointers, handles, sizes, or owned resources.

The represented sequence establishes neither atomicity nor barriers, volatile semantics, thread safety, or ordering against concurrent observers.

## Dependencies

**F-unknown** designates unresolved indirect dependencies. It is an opaque placeholder, not a claim that different invocation sites call one function.

The supplied identity mapping establishes only F-9ee192ced3b0 itself. Dependency identities and contracts remain unresolved.

Visible invocations have zero through four explicit arguments. Consumed results include 32-bit values, one represented 64-bit value, and one secondary result exposed outside the ordinary primary return.

The initial target depends on incoming control-flow state. Later targets use embedded control-flow-related values and offsets. Their expressions do not establish shared identity or purpose.

No F-unknown invocation is assumed to preserve global memory, preserve selector contents, return normally, allocate memory, or release memory.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial arguments:** If the initial invocation is reached, its first two explicit arguments both equal the second input. Substituting the first input changes the contract.

2. **Zero third input:** A zero third input causes the input-derived store to write a zero word at `0x24`. The four later explicit fixed-address clears remain separate events.

3. **Distinct seeds:** If the base seed is `0x12345678` and the shift seed is `1`, the first selected high-byte store writes `0x23`. The subsequent special fourfold argument is `4`; the later shifted arguments are `0x00800000` three times and `0x02000000` once.

4. **Fresh selector:** A selector of `0x20` selects `0x31`. If an intervening effect changes it to `0x40`, a later reread selects `0x51`.

5. **Retained selector:** If the second-base selector is retained as `0x20` before the intervening group, the subsequent word clear targets `0x64`, even if that group changes the selector’s memory location.

6. **Propagation inputs:** If the second zero-store selection is `3`, the first propagation group begins with `12`. If its result is `5`, the second group begins with `20`; if that result is `7`, the third begins with `28`.

7. **Byte truncation:** A tail seed of `0x41` produces byte `0x04` for its fourfold byte store, not a word store of `0x104`.

8. **Tail dependencies:** If the first tail no-explicit-argument invocation returns `9`, the following group begins with `9`. If that group returns `7`, the next group begins with `28`. The analogous no-explicit-argument result after the selector-argument invocation likewise begins its following group.

9. **Split result:** An upper 32-bit half of `3` selects address `0x4a` for the split-result word clear, conditional on the recovered arithmetic. The lower half does not select that destination.

10. **Interrupted execution:** A nonreturning dependency prevents subsequent caller operations. Preceding writes may already be observable.

## Uncertainties

The following remain **documented_only**:

- Accuracy and completeness of instruction recovery.
- Actual entry convention and hidden-state requirements.
- Executable address width and signed overflow or shift semantics.
- Alignment requirements, endianness, and validity of low-address mappings.
- Dependency identities, signatures, side effects, and ownership policies.
- Machine-level meaning of the represented 64-bit and secondary results.
- Fault, exception, and invalid-instruction handling.
- Whether another path returns normally.
- Intended data structures and application-level purpose.

Initialization-like or packed-field interpretations are inferred possibilities only, not acceptance requirements.

## Evidence

This contract is based solely on the supplied static representation, which reports unreliable instruction recovery and overlapping global-symbol interpretations.

Established evidence includes the represented access widths, explicit argument dependencies, retained-value uses, repeated reads and writes, and the absence of a recovered normal return. In particular, the base seed remains the source of the first high-derived byte, while the later shift seed supplies the fourfold and 23/25-bit-shift arguments.

Dependency semantics and valid memory objects are not established. Conditional examples describe consequences of the represented operations, not observed execution.

No independent implementation inspection, runtime execution, smoke test, or regression test has been performed.
