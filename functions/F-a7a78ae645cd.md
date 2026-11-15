<!-- Trusted provenance: F-a7a78ae645cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7a78ae645cd

## Purpose

The supplied static representation depicts a state-mutating routine that repeatedly invokes unresolved dependencies, derives addresses from dependency results and incoming register state, and performs interleaved byte and 32-bit memory writes.

A higher-level purpose is **not established**. In particular, the evidence does not establish that the routine constructs objects, manages a collection, serializes data, or implements a recognizable application operation. Offset repetition and frequent multiplication by four are insufficient to assign those meanings.

This contract describes the represented observable behavior, conditional on the supplied control-flow interpretation being accurate. That condition is material: the supplied representation reports invalid instruction data and ends at a truncated control-flow boundary. It therefore does not establish a complete executable function or a normal return.

## Interface

### Explicit inputs

The represented interface has two explicit arguments:

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, otherwise untyped | No visible use in the supplied body |
| Second argument | 32 bits, represented as signed | Passed directly to dependencies and used in an initial left shift by three |

An unused explicit argument does not establish that the argument is irrelevant to the actual executable. That conclusion depends on the accuracy and completeness of the supplied representation.

### Implicit inputs

The represented body also consumes:

- An incoming 32-bit register value, called the **entry context** below. It participates in address calculations, unsigned right shifts, masks, and dependency arguments.
- An incoming link-register value used to calculate the first dependency target.
- Three auxiliary register outputs associated with earlier dependency activity. Their exact producing calls and ABI meanings are not established.
- Memory reached through the entry context, byte-valued selectors, dependency results, and words loaded from those locations.
- The behavior of every unresolved dependency.

These are substantive inputs, not optional implementation details. The explicit arguments alone do not determine the represented execution.

### Outputs and completion

The represented interface has no explicit return value. Observable outputs consist of memory writes, dependency invocations, and any effects of those dependencies.

The supplied body does not show a normal return. Its final boundary is an invalid-instruction termination marker. Whether execution traps there, continues through differently decoded instructions, or follows another path is **documented_only**.

### Arithmetic and memory conventions

For describing the represented operations:

- A byte is 8 bits; a word is 32 bits.
- Byte reads used as selectors are zero-extended to the range 0–255.
- Byte stores retain only the low eight bits of the supplied value.
- Explicit unsigned right shifts insert zeros.
- A represented 64-bit dependency result is split into low and high 32-bit halves.
- Address arithmetic, multiplication, and left shifts are described by their low-32-bit results where a 32-bit machine interpretation is required.

The last convention is conditional, not proof of executable semantics. Some shifts and multiplications are presented on signed values without overflow checks. The evidence does not establish source-language overflow behavior or independently validate the machine instructions.

Word access endianness, alignment requirements, address-space mapping, and fault behavior are unknown.

## Behavior

### General ordering rules

The visible body is linear: it contains no explicit conditional branch or loop. Dependencies may nevertheless branch, fail, mutate memory, or fail to return.

All following stages are conditional on every preceding access succeeding and every preceding dependency returning to the represented continuation.

A **selector write** below means:

1. Read one byte from a stated address.
2. Zero-extend that byte.
3. Write a byte at the resulting numerical address plus `0x11`.

It does **not** mean indexing a demonstrated object or adding an omitted base address. Selector-write destinations therefore lie between `0x11` and `0x110`.

A **word-to-byte transfer** means reading a full word at a base plus `0x74`, then storing its low byte at that base plus one. The full word read remains observable even though only one byte is written.

Dependency runs described as having no displayed arguments must still occur at their represented positions. Their lack of displayed arguments does not prove that they consume no register or memory state.

### 1. Initial selector clearing and entry-derived calls

The routine first reads the byte at entry context plus one and clears the selected byte at selector plus `0x11`.

It then invokes the entry-relative dependency with four displayed arguments:

- The second explicit argument shifted left by three, twice.
- The unchanged second argument.
- Zero.

Four further dependency invocations follow; the last supplies a word result.

The routine then rereads the byte at entry context plus one, retains that selector, and clears its selected byte. This second read must not be replaced with the first selector: dependencies intervene.

The captured dependency result, shifted left by two, is passed onward. Four more calls follow.

### 2. Entry-derived transfer and paired result

Using the retained selector as a numerical base, the routine transfers the word at base plus `0x74` into the byte at base plus one.

It invokes a dependency with:

- Twice the entry context.
- The second explicit argument.
- Entry context logically shifted right by two.
- The retained selector.

The next dependency supplies a represented 64-bit result. Another receives its low half, high half, low half again, and the entry context masked with `0x7fffffff` and shifted right by one.

Four further calls culminate in a word result. Four times that result becomes a new base. A word-to-byte transfer occurs at that base, followed by a call receiving the base and four further calls.

### 3. Doubled-context selector updates

The routine reads a selector at twice the entry context plus one and clears the selected byte. It next passes an auxiliary register output to a dependency.

The following call yields a value retained here as the **anchor**. That value is passed to another dependency; four subsequent calls culminate in another result.

The selector at twice the entry context plus one is read again. Its selected byte receives the top byte of the doubled entry context.

The latest dependency result shifted left by two is passed onward. Four further calls culminate in a result whose fourfold value is retained as the **scaled handle**.

### 4. Anchor-derived selector chain

The routine reads a word at anchor plus four. That word is used as an address: a byte is read at word plus one and its selected byte is cleared.

A dependency receives the scaled handle three times. Four subsequent calls culminate in a result.

Next, the previously loaded word is logically shifted right by two and incremented by one to form a byte-read address. The byte read there becomes a retained **small base**, and its selected byte is cleared.

The latest dependency result shifted left by two is passed onward. Four further calls culminate in another result.

### 5. Captured words and shifted arguments

In order, the routine:

- Reads a word at scaled handle plus four.
- Reads a word at small base plus `0x74`, retaining it as the **captured payload**.
- Stores the payload’s low byte at small base plus one.

A dependency receives the payload, the scaled handle shifted left by 19 twice, and the latest result shifted left by two.

The next call yields a word passed to another dependency alongside the payload logically shifted right by two. Four further calls culminate in a result.

Separately, the word read at scaled handle plus four is logically shifted right by 22. This retained **extracted field** is not reread from memory later.

### 6. Word stores derived from dependency results

Four times the latest result becomes a base. The routine reads its word at `+0x74`, retains that word, and writes its low byte at `+1`.

After a call receiving that base and four further calls, the retained full word is stored at four times an auxiliary register output plus `0x14`. That auxiliary output is then passed to a dependency.

The next result is passed onward together with two copies of:

- The low nine bits of the extracted field, shifted left by 21.

Four further calls culminate in a result. The routine stores the extracted field shifted left by 23 at four times that result plus `0x14`.

A call receives four times the result; four subsequent calls yield a new unsigned word.

### 7. Masked-value stores

For that unsigned result, the routine first stores its fourfold value at sixteen times the original result plus `0x14`.

Only afterward does it retain:

- The result masked with `0x3fffffff`, called the **first masked value**.
- That masked value logically shifted right by two, called the **retained quarter**.

The first masked value is passed onward. Four subsequent calls yield a value retained as the **index**. The first masked value is stored at four times the index plus `0x14`.

A call receives four times the index; four subsequent calls yield another unsigned result.

### 8. Repeated writes around paired results

The routine stores the low byte of eight times the index at captured payload plus one. It then stores four times the latest unsigned result at eight times the index plus `0x34`.

A dependency receives that fourfold result. Another yields a word passed onward. The unsigned result masked with `0x3fffffff` is retained as the **second masked value**.

Four subsequent calls culminate in a represented 64-bit result. The routine then:

- Stores the low byte of eight times the index at second masked value plus one.
- Overwrites the word at eight times the index plus `0x34` with the result’s high half.
- Passes the low half shifted left by two onward.

Four further calls culminate in another result.

### 9. Sixteenfold-index byte updates

The routine stores:

- The low byte of sixteen times the index at `(second masked value >> 20) + 0x11`.
- The low byte of four times the earlier unsigned result from stage 7 at sixteen times the index plus `0x0d`.

A dependency receives two copies of the latest result shifted left by two. The next result is passed onward alongside its logical right shift by three. Four further calls yield another result.

The routine then writes the low byte of sixteen times the index directly to address `0x11`, repeats the byte store at sixteen times the index plus `0x0d`, and passes the latest result shifted left by two onward.

Four further calls culminate in another unsigned result.

### 10. Second masked value and thirty-twofold base

Before deriving new addresses from that latest result, the routine writes:

- Four times the earlier stage-7 unsigned result to second masked value plus `0x14`.
- The low byte of twice the second masked value to second masked value plus `0x0d`.

It then retains:

- Thirty-two times the latest unsigned result as the **wide base**.
- The same result shifted left by seven.
- The result masked with `0x07ffffff`, then shifted right by 17, as the **small field**.

Three further calls culminate in a result. The routine stores the retained seven-bit-left-shifted value at wide base plus `0x14`, and the low byte of four times the latest result at wide base plus `0x0d`.

That fourfold result is passed onward, followed by four calls.

### 11. Small-field and selector stores

The routine writes the low byte of twice the retained quarter at small field plus seven.

It then reads a selector at wide base plus one and stores the low byte of the small field at selector plus `0x11`.

A dependency receives three copies of the earlier stage-10 result shifted left by 21. The next result, retained as the **late captured value**, is passed onward. Four further calls culminate in another result.

The routine reads and retains the selector at wide base plus one, clears its selected byte, passes the latest result shifted left by two onward, and performs four more calls.

### 12. Low-address updates and fixed arguments

The routine retains the low byte of the late captured value logically shifted right by three. It stores that byte at wide base plus `0x0d`.

It writes the retained selector directly to address `0x0d`.

A dependency then receives twice that selector, `0x98`, and `0x260`. After another call, a further dependency receives an auxiliary register output logically shifted right by three.

Four following calls culminate in a result. Another dependency receives that result shifted left by two and returns a replacement result.

The routine then writes the retained byte at twice the retained selector plus `0x0d`, writes `0xc0` directly to address `0x0d`, and passes the replacement result shifted left by two onward.

Four further calls yield another result.

### 13. Loaded-pointer stores

The routine stores four times this result at twice the retained selector plus `0x24`, then reads a word at twice the selector plus `0x74`. That word is retained as the **loaded base**.

A dependency receives:

- The loaded base.
- Twice the loaded base.
- The preceding result shifted left by five.
- The loaded base masked with `0x7fffffff` and logically shifted right by two.

Four following calls occur. The routine stores the retained five-bit-left-shifted result at loaded base plus `0x24`.

It reads a word at loaded base plus `0x74` and passes that word to another dependency. Four further calls culminate in a represented 64-bit result.

### 14. Final paired results and repeated selector writes

The routine stores the late captured value logically shifted right by three at four times the paired result’s high half plus `0x14`.

It passes three copies of the low half shifted left by two, followed by the high half logically shifted right by three.

The next call supplies another paired result. A subsequent dependency receives its low half, high half, low half again, and high half logically shifted right by three.

Four further calls yield a result. At four times that result plus `0x14`, the routine stores the previously retained shifted low half.

A call receives four times the result; four subsequent calls yield a new unsigned result, called the **final seed**.

The routine reads a selector at loaded base plus one and writes the low byte of four times the final seed at selector plus `0x11`.

Three intervening calls culminate in another result. The routine rereads the selector at loaded base plus one and writes the low byte of `(final seed & 0x3fffffff) >> 20` at its selected address.

The latest dependency result shifted left by two is passed onward, followed by four calls.

### 15. Terminal selector clears

The routine reads a selector at 1024 times the fourfold final seed plus one and clears its selected byte.

A dependency receives:

- Eight times the fourfold final seed, twice.
- The fourfold final seed.
- The latest dependency result shifted left by two.

Four subsequent calls culminate in another result.

The same selector source address is read again and its selected byte cleared again. The latest result shifted left by two is passed onward, followed by four calls.

The supplied representation then ends at its invalid-instruction boundary.

## Side effects

### Memory and aliasing

The routine visibly performs byte and word writes to computed addresses, including literal low addresses `0x0d` and `0x11`. No null checks, range checks, alignment checks, or access checks appear.

There is no demonstrated record layout. Offsets `+1`, `+7`, `+0x0d`, `+0x14`, `+0x24`, `+0x34`, and `+0x74` describe accesses, not established field types or object boundaries.

Any computed locations may alias. Consequently:

- A write may affect a later selector or word read.
- Repeated reads must remain repeated reads.
- Captured words must remain captured values rather than later memory reloads.
- Intermediate writes cannot be omitted merely because a later store may overwrite them.

Dependencies intervene between many accesses and may themselves observe or change memory.

### Ownership and errors

No allocation, ownership transfer, retention rule, or release operation is established. Dependency calls must not be labeled allocation or cleanup without additional evidence.

No explicit status check, rollback, retry, or error return is visible. If an access faults or a dependency does not return, later represented actions are not established to occur. Atomicity, synchronization, and recovery behavior are unknown.

## Dependencies

The dependency inventory is **documented_only**.

- **F-unknown-001** identifies the first call target, calculated from incoming link-register state plus `0xfec`.
- **F-unknown-002** identifies the unresolved dependency-call population at all subsequent call positions.

F-unknown-002 is a documentation placeholder for unresolved identities, **not** a claim that those positions invoke one function. The supplied dependency map provides no verified external identities.

Displayed argument lists and captured result widths are established only at individual call positions. The evidence does not establish complete signatures, register preservation, memory effects, ownership effects, or whether represented paired outputs are genuine 64-bit returns.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial selector behavior.** If entry context plus one contains `0x23`, the first represented write clears the byte at `0x34`. It does not write at entry context plus `0x34`.

2. **Dependency-sensitive reread.** If an intervening dependency changes that selector to `0x40`, the second initial clear targets `0x51`. Reusing the original selector would violate the represented order.

3. **Full-word read with byte truncation.** If a word-to-byte transfer reads `0x89abcdef`, its byte store writes `0xef`; it does not copy four bytes. Memory interpretation remains subject to the unknown endianness.

4. **Shift truncation.** Under the conditional 32-bit arithmetic model, shifting `0x40000001` left by two produces `0x00000004`. Widening the address calculation to preserve the discarded high bits would differ.

5. **Mask boundaries.** An extracted field can retain ten bits after a right shift by 22. The later nine-bit mask therefore matters: shifting the unmasked field left by 21 is not generally equivalent.

6. **Intermediate word visibility.** The stage-8 word at eight times the index plus `0x34` first receives a fourfold result and later a paired result’s high half. An intervening dependency may observe the first value.

7. **Literal low-address writes.** If execution reaches stage 12, address `0x0d` receives the retained selector before receiving `0xc0`. There is no displayed object base for either write.

8. **Nonreturning dependency.** If a dependency does not return, subsequent stores are not required by this conditional trace. The evidence supplies no recovery path.

## Uncertainties

- **Established static evidence:** displayed access widths, masks, shifts, literal offsets, call ordering, repeated reads, captured values, and absence of visible validation.
- **Inferred intent:** repeated scaling may reflect encoded addresses or handles. This is not established and imposes no object-model requirement.
- **documented_only:** instruction validity, complete control flow, calling convention, dependency identities, auxiliary-output provenance, memory mapping, alignment, endianness, ownership, error semantics, and actual termination.

No unknown callee is assumed to preserve global state, preserve referenced memory, allocate storage, or release storage.

## Evidence

This contract is based solely on the supplied static representation and its limited dependency map. Comments and names are not treated as behavioral authority.

The representation itself warns of invalid instruction data and truncated control flow. Accordingly, the detailed trace above specifies what that representation depicts, not independently verified executable behavior.

No runtime tests, execution traces, memory observations, or independent implementation inspection have been performed.
