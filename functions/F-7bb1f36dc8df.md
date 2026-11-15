<!-- Trusted provenance: F-7bb1f36dc8df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7bb1f36dc8df

## Purpose

F-7bb1f36dc8df performs a visibly unconditional, repeating sequence of indirect calls, byte accesses, and 32-bit memory accesses. Returned values and working machine state determine later call arguments, addresses, and stored bit patterns.

No application-level purpose is established. The accessed locations are not established to be objects, allocated buffers, hardware registers, or valid memory.

This contract distinguishes:

- **Established static evidence:** operations, ordering, explicit value dependencies, and repetition visible in the supplied representation.
- **Conditional interpretation:** consequences of 32-bit truncating machine arithmetic and the represented division of a wider result into two words.
- **Documented-only unknowns:** dependency behavior, entry conventions, address validity, ownership, and the producer of an unresolved extra-result channel used later in the iteration.

No runtime tests have been performed. The contract describes the supplied static representation, not a verified executable interface.

## Interface

### Explicit inputs

The visible interface contains four 32-bit input slots.

| Input | Visible use |
|---|---|
| First | Passed to the iteration’s first indirect call. Replaced at the back edge by a final returned word shifted left by two. |
| Second | No explicit use appears. |
| Third | No explicit use appears. |
| Fourth | Its low eight bits are captured at iteration entry and later stored twice. The working input is set to zero before the iteration ends. |

The absence of explicit uses for the second and third inputs does not establish that the underlying execution ignores their machine-level argument locations. The actual calling convention remains unresolved.

### Implicit entry state

The representation consumes four additional values before assigning them locally:

- A base for byte accesses at offsets 1 and 17.
- A word supplying the first shifted word-store value.
- An unsigned word supplying a right-shift-derived read address.
- A link-like value supplying the first indirect call target.

These are implicit entry requirements of the represented execution. Their provenance, validity, and relationship to a callable interface are undocumented.

A separate extra-result channel appears at a later two-argument call. **It is not established to be an entry input.** Its producer and availability are unknown at that point of use; the representation does not establish that the caller must initialize it.

### Outputs and termination

There is no visible ordinary return value or normal return path. Reaching the back edge begins another iteration.

Observable outputs include indirect calls and arguments, byte and word writes, and changes to working state carried into the next iteration. Reads are also observable operations and determine subsequent behavior.

A dependency may fail to return. An invalid access or indirect target may interrupt execution according to the surrounding platform. No local error reporting, recovery, or rollback is visible.

### Widths and layout

A **word** is 32 bits; a **byte** is eight bits. Represented paired results contain 64 bits divided into low and high 32-bit words. This division describes value bits, not memory byte order.

Throughout the behavioral description:

- Byte stores retain the low eight bits.
- Selector bytes are unsigned, ranging from 0 through 255.
- Right shifts of explicitly unsigned words are logical.
- Address offsets 1, 17, and 20 are byte offsets.
- A word access occupies four bytes under the stated width interpretation.

Unless otherwise qualified, scaling and left-shift formulas below use a **conditional 32-bit truncating interpretation**: bits above bit 31 are discarded. The supplied representation includes signed arithmetic whose portable-language overflow and shift semantics are not established. These formulas therefore do not assert portable arithmetic validity.

Endianness, alignment requirements, atomicity, and address-space behavior are unknown.

## Behavior

### Execution and dependency notation

The phases below occur in order within each iteration. There is no visible conditional branch between them.

**F-UNRESOLVED** denotes unresolved indirect dependency occurrences. It is an opaque documentation label, not an assertion that different occurrences call the same function. “Four calls” means four distinct call occurrences in sequence, not four invocations of a proven common target.

A “fresh result” belongs to the stated call occurrence. It does not imply allocation or object creation. Descriptive names such as “first packed word” identify contract values, not implementation identifiers.

Calls must remain observable boundaries. Unknown dependencies may change memory, globals, or external state. The description of a retained value follows its explicit dependency in the representation; it does not establish ABI-level preservation by unknown callees.

### 1. Captured byte and opening calls

The low byte of the working fourth input is captured before the first call.

The opening sequence is:

1. Call a target derived from the current link-like state plus 4086, passing the working first input.
2. Make a call producing a word.
3. Forward that word to another call.
4. Make four further calls; the fourth produces an index-like word.
5. Store the captured byte at current base plus 17.
6. Store the same captured byte at current base plus 1.
7. Call a dependency with the index-like word shifted left by two.

The two stores have no intervening visible call. The initial base comes from implicit entry state; a later phase constructs the next iteration’s base.

### 2. Initial word store, read, and retained word

Four calls follow. Let the last result be the first word-address index.

The function then:

- Stores the incoming shift-source word shifted left by 28 at `16 × first word-address index + 20`.
- Reads an unsigned word at `(incoming unsigned word >> 22) + 20`.
- Calls a dependency with `4 × first word-address index`.
- Calls another dependency and retains its returned word.
- Calls a dependency with `(read word >> 22) << 28`.

The store precedes the read. No non-aliasing guarantee separates them.

Under 32-bit truncation, the final call argument contains only original read-word bits 22–25, placed in bits 28–31. The retained word is used in phases 3 and 7.

### 3. Zero store and first packed word

Four calls follow; the last returns another address index. The function stores a zero word at `4 × index + 20`, then calls a dependency with `4 × index`.

Four more calls produce the **first packed word**.

Next:

1. Read a selector byte at current base plus 1.
2. Store the low byte of `first packed word << 2` at selector plus 17.
3. Call a dependency with `first packed word << 2`.
4. Obtain a fresh word from another call.
5. Pass that fresh word and `first packed word << 30`, in that order, to a two-argument call.

Four further calls produce a fresh word used only for the next stated shifted argument. The function rereads current base plus 1, saves that selector, and stores the low byte of `(first packed word AND 0x3fffffff) >> 20` at saved selector plus 17.

It then calls with the fresh word shifted left by two and makes four further calls. After those calls, it stores the phase-2 retained word shifted left by two at **saved selector plus 20**.

This word store uses the saved selector. It does not reread the selector source. The earlier and later selector reads remain separate operations.

### 4. Global snapshot and first paired result

After the saved-selector word store, the function reads a global-like 32-bit storage location. Its identity and relationship to other storage are unresolved.

A call receives `4 × first word-address index`, reusing the phase-2 value. Four additional calls follow; the fourth produces a represented paired result.

Define the **first scaled result** as the pair’s low word shifted left by two.

The function then:

- Rereads current base plus 1.
- Stores the low byte of `global snapshot >> 22` at selector plus 17.
- Calls with the paired result’s high word.
- Obtains a fresh word from another call.
- Passes that word and `first scaled result << 28` to a two-argument call.

Four further calls produce a fresh word. Another selector read from current base plus 1 selects a byte destination where zero is stored. A call then receives the fresh word shifted left by two.

The global-derived byte uses the earlier snapshot, not a visible reread after the intervening calls.

### 5. First duplicated word

Four calls produce a fresh word. A two-argument call receives that word shifted left by two in both positions.

Four additional calls produce another word. Define the **first duplicated word** as this result shifted left by two.

The function stores it at `4096 × first scaled result + 20`, then calls with `first scaled result << 14`.

Four further calls produce a fresh address index. The function stores the same first duplicated word at `4 × index + 20`, then calls with `4 × index`.

Both stores carry the same represented value. Their destinations may coincide or overlap; neither store is omitted on that account.

### 6. Second paired result and selector-source construction

Four calls produce a second paired result. Define the **second scaled result** as its low word shifted left by two.

Before forwarding the high word, the function reads a selector byte at `16384 × first scaled result + 1`. It stores the low byte of `(first duplicated word AND 0x3fffffff) >> 20` at selector plus 17.

Next:

1. Call with the second pair’s high word.
2. Obtain a fresh word from another call.
3. Forward that word to a call.
4. Obtain a further paired result from a call.
5. Forward that pair’s low and high words, in that order, to a two-argument call.
6. Make two further calls; retain the second result for a shifted argument.

The function reads a selector at `268435456 × second scaled result + 1`, writes zero at selector plus 17, and calls with the retained fresh result shifted left by two.

Four more calls produce the **middle result**.

The function then establishes the **repeated selector source** at `4 × second scaled result + 1`, reads a byte there, and stores zero at that byte’s value plus 17.

### 7. Repeated selector reads and second duplicated word

A call receives `second scaled result << 10`. Another returns a word. A following call receives that word and the phase-2 retained word shifted left by 28.

Four further calls produce a fresh result. The function rereads the repeated selector source, writes zero at selected byte plus 17, and calls with the fresh result shifted left by two.

Four additional calls occur. Their explicit results do not supply the next stored value.

Define the **second duplicated word** as `middle result << 14`. The function stores it at `4096 × (middle result << 2) + 20`, then calls with `(middle result << 2) << 14`.

One further call occurs. The next two-argument call receives:

1. `second duplicated word << 28`.
2. An unresolved extra-result-channel value.

The second argument’s producer, width convention beyond its represented 32-bit slot, and availability are **documented_only**. The representation does not establish that it originates at function entry or that the immediately preceding call produces it.

Under 32-bit truncation, the first argument is zero: shifting the middle result left by 14 and then by 28 discards every original bit. This does not make the call removable.

Four further calls produce an address index. The function stores the second duplicated word at `4 × index + 20`, then calls with `4 × index`.

### 8. Final packed-byte sequence

Four calls produce a word. Define the **third scaled result** as that word shifted left by two, retaining the unscaled producer for phase 9.

The function rereads the repeated selector source and stores the low byte of `(second duplicated word AND 0x3fffffff) >> 20` at selector plus 17.

It then:

- Calls with `middle result << 26`.
- Obtains a fresh word from another call.
- Passes that word and `third scaled result << 28` to a two-argument call.
- Obtains another paired result.
- Forwards its low and high words to a two-argument call.
- Makes two further calls, retaining the second result.

Another read of the repeated selector source selects a byte destination for a zero store. A call then receives the retained result shifted left by two.

### 9. Final three word-store groups

**Group A:** Four calls produce a word, which is shifted left by two and retained as the group’s stored value. That value is written at `4096 × third scaled result + 20`.

A four-argument call receives, in order:

1. `1024 × third scaled result`.
2. The same value again.
3. The unscaled producer of the third scaled result shifted left by 14.
4. The stored value.

Four calls produce an address index. The stored value is written again at `4 × index + 20`, followed by a call with `4 × index`.

**Group B:** Four calls produce a fresh result. Its value multiplied by four is stored at `16 × result + 20`. The working fourth input is then set to zero.

A three-argument call receives the stored value in all three positions. Four calls produce another address index. The same stored value is written at `4 × index + 20`, followed by a call with `4 × index`.

**Group C:** Four calls produce another result. Its value multiplied by four is stored at `16 × result + 20`. The next iteration’s base is then assigned `1073741824 × result`.

A three-argument call receives this group’s stored value in all three positions. Four calls produce another address index. The same stored value is written at `4 × index + 20`, followed by a call with `4 × index`.

The group-C second-write index is retained for the back-edge update.

### 10. Back edge and carried state

Four final calls occur. Immediately before the fourth, the link-like working state is assigned a fixed internal continuation value. The fourth call produces the final result.

The function then, in order:

1. Sets the working first input to `final result << 2`.
2. Sets the incoming shift-source state to `group-C second-write index << 3`.
3. Stores zero at next base plus 17.
4. Stores zero at next base plus 1.
5. Sets the incoming unsigned state to the updated first input.
6. Repeats unconditionally.

The final zero stores follow from the earlier assignment of zero to the working fourth input. Subsequent iterations therefore begin with a captured zero byte and with explicitly constructed working state rather than the initial implicit values.

## Side effects

There are no visible bounds, null, alignment, permission, or target-validity checks.

A selector byte is promoted to an absolute address component; it is not added back to its source address. Consequently:

- Selector-based byte destinations range from addresses 17 through 272.
- The saved-selector word destination ranges from addresses 20 through 275.
- That four-byte word access can extend through address 278.
- Selector-based word destinations need not be aligned.

Scaled addresses may wrap under the conditional arithmetic interpretation.

Reads, writes, and calls retain their stated ordering even when addresses coincide or a later write restores a previous value. Calls can change the contents subsequently read from a selector source. Stores can overlap selector sources, the global-like location, or other stores; separation is not established.

No allocation, release, ownership transfer, synchronization, or locking operation is established. Address-like values passed to dependencies do not imply ownership. Unknown dependencies are not assumed to preserve globals or to acquire, retain, or release memory.

## Dependencies

F-UNRESOLVED covers unresolved indirect call occurrences without assigning them a common identity.

Visible interactions include zero through four explicit arguments, forwarding of word results, extraction and forwarding of paired-result halves, and consumption of an extra-result channel with unknown provenance.

The first target depends on incoming link-like state. Later targets derive from fixed continuation-like values and offsets. Apparent zero-argument calls are not proven to consume no implicit machine state.

Actual identities, complete signatures, argument conventions, clobbers, result conventions, memory effects, failure behavior, and termination behavior are **documented_only**.

## Acceptance examples

These are conditional consequences, not executed tests. They assume returning dependencies, valid accesses, and the specified arithmetic interpretation where needed.

1. **Captured byte:** With fourth input `0x123456ab`, the opening stores write `0xab` at base plus 17 and then base plus 1, using the pre-call capture.

2. **Opening call count:** After forwarding the opening returned word, four further calls precede the two opening byte stores. The fourth supplies the result shifted for the following call.

3. **Absolute selector:** A selector value of 7 selects byte address 24. Its source address is not included in that destination.

4. **Packed-byte truncation:** A first packed word of `0xffffffff` produces `0xfc` for its left-shift-derived byte store and `0xff` for its masked, right-shift-derived byte store.

5. **Saved selector:** If phase 3 saves selector 7 and later calls change the source byte to 9, its saved-selector word store still starts at address 27. A subsequent fresh selector read of 9 selects byte address 26.

6. **Duplicate stores:** The two phase-5 word stores contain the same first duplicated word. Equal destinations still require both stores.

7. **Extra-result uncertainty:** Knowing all explicit inputs and the middle result does not establish phase 7’s unresolved second argument. The contract does not require an extra caller-initialized entry value.

8. **Truncated argument:** Under 32-bit truncation, phase 7’s first argument at the unresolved-channel call is zero for every middle result.

9. **Next-base reduction:** Group C’s base becomes `0`, `0x40000000`, `0x80000000`, or `0xc0000000`, according to the producer’s low two bits.

10. **Continuation limit:** If a dependency does not return, later operations are not reached. No local recovery is visible.

## Uncertainties

- **Entry convention — documented_only:** initialization and validity requirements for the four implicit entry values are unknown.
- **Extra-result channel — documented_only:** its producer and availability at the phase-7 use are unresolved; entry provenance is not established.
- **Arithmetic fidelity — documented_only:** signed shifts, overflow, and address formation need machine-level confirmation.
- **Memory model — documented_only:** validity, byte order, alignment, atomicity, volatility, concurrency, and device effects are unspecified.
- **Global layout — documented_only:** the global-like read may overlap other storage.
- **Dependency contracts — documented_only:** identities, ABI behavior, ownership effects, exceptions, and termination remain unresolved.
- **Completeness — documented_only:** a warning reports removal of an unreachable block; the representation does not establish completeness for other entry or control-flow interpretations.
- **Intent — inferred only:** packing, scaling, and repeated stores suggest structured state manipulation, but no domain-specific purpose is justified.

## Evidence

The supplied static representation supports the explicit access sequence, four opening calls after the first forwarding call, value dependencies, store widths, selector rereads, saved selector, paired-result handling, back-edge assignments, and unconditional repetition described above.

It shows four consumed implicit entry values and a later unresolved extra-result-channel use. It does not establish an entry origin for that channel.

Warnings concerning overlapping global storage and an omitted unreachable block limit interpretation. No dependency inspection, implementation execution, runtime tests, or observed acceptance results support this contract.
