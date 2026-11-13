<!-- Trusted provenance: F-0c7c950fbe7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0c7c950fbe7e

## Purpose

F-0c7c950fbe7e describes an ordered sequence of indirect calls, byte stores, word stores, and pointer-like address calculations. Its application-level purpose is **unknown**. The supplied static representation does not establish that it implements a valid complete routine: it contains an instruction-decoding warning and ends at a truncated control-flow boundary.

The contract below therefore specifies the **conditionally observable trace represented by the supplied material**, not a verified executable implementation. Each operation is conditional on reaching it, successful completion of preceding operations, and the validity of the recovered control flow.

Repeated field offsets and scaled values suggest structured state manipulation, but they do not establish object types, allocation boundaries, ownership, or field meanings.

## Interface

### Explicit inputs

Four explicit inputs are represented:

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | Passed to the first indirect dependency. |
| Second input | 32 bits | Stored at the ambient primary base plus `0x34`. |
| Third input | 32 bits | Stored as a word and, at other locations, truncated to a byte. |
| Fourth input | 32 bits | Shifted left by one, then truncated to a byte for a store. |

The fourth input is represented as signed, but its visible use does not establish any signed numeric interpretation.

### Additional entry state

The explicit inputs are insufficient to determine the trace. Four additional ambient machine values supply:

- **Primary base:** used as an address, a byte value, and the source of a doubled address.
- **Initial byte base:** supplies the address of the first byte store.
- **Selector:** shifted right by 22 bits to obtain a small-address destination.
- **Quarter-base source:** shifted logically right by two bits to form another address.

An additional ambient control value participates in selecting the first indirect call target. These values are not established as ordinary formal parameters or valid object pointers.

### Outputs and termination

No explicit return value or normal return is represented.

The visible terminal event is a bad-instruction termination marker at a truncated boundary. This establishes neither an application-defined error result nor a reliable hardware exception classification.

### Width and arithmetic conventions

The representation uses:

- 8-bit byte reads and writes;
- 32-bit word reads, writes, and scalar values;
- two dependency results represented as 64-bit aggregates, plus an earlier aggregate result;
- logical right shifts where the source is represented as unsigned;
- truncation to the low eight bits for byte stores.

For compactness, the trace below describes left shifts, multiplication, and address addition using 32-bit bit-pattern arithmetic. This is a **conditional machine-width interpretation**, not a guarantee derived from portable signed-language arithmetic. In particular, several represented signed shifts and multiplications require confirmation from actual instructions.

Endianness, alignment requirements, address-space behavior, and the physical atomicity of word accesses are not established.

## Behavior

### Trace notation

Dependencies receive opaque, specification-local IDs **F-C001** through **F-C097**, in visible invocation order. These are call-event identifiers, not claims that there are 97 distinct functions. Target identity, aliasing between targets, and target validity remain unknown.

A dependency listed without arguments has no explicit arguments in the supplied representation. This does not establish that it receives no meaningful register or stack state.

A result not explicitly retained below is unused by the visible expressions. That does not make its call side-effect-free.

### 1. Initial stores and first result chain

Before any dependency invocation:

1. Store the low byte of the primary base at the initial byte base plus `1`.
2. Store the second input as a 32-bit word at the primary base plus `0x34`.

Then:

| Dependency | Explicit arguments | Retained result |
|---|---|---|
| F-C001 | First input | None |
| F-C002 | None | A 32-bit forwarding value |
| F-C003 | Forwarding value | None |
| F-C004–F-C006 | None | None |
| F-C007 | None | First aggregate |

The **quarter base** is the quarter-base source logically shifted right by two. Its calculation occurs after F-C002 and before F-C003 in the representation.

After F-C007:

- Store the primary base’s low byte at the quarter base plus `1`.
- Store the first aggregate’s upper 32 bits at the primary base plus `0x34`, overwriting the earlier second-input word if the address remains the same.
- Invoke F-C008 with the first aggregate’s lower 32 bits shifted left by two.

F-C009–F-C011 have no explicit arguments. F-C012 has no explicit arguments and supplies a 32-bit **first scaled-call seed**.

### 2. Doubled-primary destinations and literal-address store

Form the **doubled primary base** by multiplying the primary base by two.

Before F-C013:

- Store its low byte at `(selector logically shifted right by 22) + 0x11`.
- Store the third input’s low byte at the doubled primary base plus `0x0d`.

The next calls are:

| Dependency | Explicit arguments | Retained result |
|---|---|---|
| F-C013 | First scaled-call seed shifted left by two, twice | None |
| F-C014 | None | A 32-bit forwarding word |
| F-C015 | Forwarding word; forwarding word logically shifted right by three | None |
| F-C016–F-C018 | None | None |
| F-C019 | None | Second scaled-call seed |

After F-C019:

- Store the doubled primary base’s low byte at literal address `0x11`.
- Again store the third input’s low byte at the doubled primary base plus `0x0d`.

Invoke F-C020 with the second scaled-call seed shifted left by two. F-C021–F-C023 have no explicit arguments. F-C024 returns the **layout seed**.

The repeated byte store is observable and must not be collapsed: intervening dependencies can modify the destination or observe its earlier value.

### 3. Layout-derived writes and indexed byte destination

After F-C024:

- Store the third input as a 32-bit word at the quarter base plus `0x14`.
- Store the low byte of twice the quarter base at the quarter base plus `0x0d`.

Derive three retained values from the layout seed:

- **Layout base:** layout seed multiplied by `0x20`.
- **Layout word:** layout seed shifted left by seven.
- **Compact index:** layout seed masked with `0x07ffffff`, then logically shifted right by 17.

F-C025 and F-C026 have no explicit arguments. F-C027 returns the **layout call seed**.

Then:

- Store the layout word at the layout base plus `0x14`.
- Store the low byte of the layout call seed shifted left by two at the layout base plus `0x0d`.
- Invoke F-C028 with that shifted call seed.
- Invoke F-C029–F-C032 without explicit arguments.

Next:

1. Store the low byte of the fourth input shifted left by one at the compact index plus `7`.
2. Read one unsigned byte from the layout base plus `1`.
3. Store the compact index’s low byte at the address formed by adding `0x11` to that byte.
4. Invoke F-C033 with three identical arguments: the layout call seed shifted left by 21.

The selector byte is an unsigned 8-bit quantity. Its resulting destination is therefore between `0x11` and `0x110`, inclusive.

### 4. Saved selector, retained byte, and low-address updates

F-C034 returns a 32-bit **retained word**. F-C035 receives that word. F-C036–F-C038 have no explicit arguments, and F-C039 returns a new scaled-call seed.

After F-C039:

- Read the layout base plus `1` again as an unsigned byte.
- Save this value as the **saved selector**.
- Store zero at the saved selector plus `0x11`.

This second read is distinct from the earlier read. No equality is guaranteed across the intervening dependencies.

Invoke F-C040 with F-C039’s result shifted left by two, then F-C041–F-C044 without explicit arguments.

Next:

- Form the **small base** as twice the saved selector.
- Form the **retained byte** as the low byte of the retained word logically shifted right by three.
- Store the retained byte at the layout base plus `0x0d`.
- Store the saved selector at literal address `0x0d`.

Invoke:

| Dependency | Explicit arguments | Retained result |
|---|---|---|
| F-C045 | Small base; `0x98`; `0x260` | None |
| F-C046 | None | Secondary outgoing machine value used next |
| F-C047 | That secondary value logically shifted right by three | None |
| F-C048–F-C050 | None | None |
| F-C051 | None | Conversion seed |
| F-C052 | Conversion seed shifted left by two | Converted value |

The provenance and ABI meaning of the secondary outgoing value associated with F-C046 are not otherwise established.

After F-C052:

- Store the retained byte at the small base plus `0x0d`.
- Store `0xc0` at literal address `0x0d`.
- Invoke F-C053 with the converted value shifted left by two.

### 5. Link-like state propagation

F-C054–F-C056 have no explicit arguments. F-C057 returns the **link seed**.

Then, in order:

1. Store the link seed shifted left by two as a word at the small base plus `0x24`.
2. Read a 32-bit **link base** from the small base plus `0x74`.
3. Retain the link seed shifted left by five as the **link payload**.
4. Invoke F-C058 with:
   - link base;
   - link base shifted left by one;
   - link payload;
   - link base masked with `0x7fffffff`, then logically shifted right by two.
5. Invoke F-C059–F-C062 without explicit arguments.
6. Store the link payload at the link base plus `0x24`.
7. Read a word from the link base plus `0x74` and pass it to F-C063.

The read supplied to F-C063 occurs after the preceding dependencies and word store. It is not an earlier cached field value.

### 6. Aggregate results and scaled word destinations

F-C064–F-C066 have no explicit arguments. F-C067 returns the **second aggregate**.

After F-C067:

- Retain its lower half shifted left by two as the **aggregate payload**.
- Store the retained word from F-C034, logically shifted right by three, at four times the second aggregate’s upper half plus `0x14`.
- Invoke F-C068 with the aggregate payload three times, followed by the second aggregate’s upper half logically shifted right by three.

F-C069 returns the **third aggregate**. F-C070 receives its lower half, upper half, lower half again, and upper half logically shifted right by three.

F-C071–F-C073 have no explicit arguments. F-C074 returns a **destination seed**.

Then:

- Store the aggregate payload at four times the destination seed plus `0x14`.
- Invoke F-C075 with four times the destination seed.
- Invoke F-C076–F-C078 without explicit arguments.
- Retain F-C079’s result as the **final selector seed**.

### 7. Final byte-directed updates and termination boundary

Form the **final scaled seed** by shifting the final selector seed left by two.

Immediately after F-C079:

- Read one unsigned byte from the link base plus `1`.
- Store the final scaled seed’s low byte at that byte plus `0x11`.

F-C080 and F-C081 have no explicit arguments. F-C082 returns another scaled-call seed.

Then:

- Read the link base plus `1` again.
- Store the low byte of the final selector seed masked with `0x3fffffff` and logically shifted right by 20 at that byte plus `0x11`.
- Invoke F-C083 with F-C082’s result shifted left by two.

F-C084–F-C086 have no explicit arguments. F-C087 returns a final argument seed.

Next:

1. Read one unsigned byte from `(final scaled seed multiplied by 0x400) + 1`.
2. Store zero at that byte plus `0x11`.
3. Invoke F-C088 with the final scaled seed shifted left by three twice, the final scaled seed, and F-C087’s result shifted left by two.

F-C089–F-C091 have no explicit arguments. F-C092 returns the final forwarded result.

Read the same computed selector location again and store zero at the newly read byte plus `0x11`. Invoke F-C093 with F-C092’s result shifted left by two, then F-C094–F-C097 without explicit arguments.

The supplied representation then reaches its bad-instruction termination marker. No subsequent behavior is established.

## Side effects

### Memory and ordering

The trace contains both computed-address and literal-address writes. In particular, literal addresses `0x0d` and `0x11` are written without visible validation.

These addresses must not be silently reinterpreted as object-relative fields, omitted as implausible, or redirected into a conventional data structure.

All visible stores and reads retain their ordering relative to dependency invocations. Repeated reads must remain separate because dependencies may mutate their source locations. Repeated stores must remain separate because dependencies may observe or overwrite them.

Aliasing is unrestricted. A write can affect a later read, another apparent field, a low-address destination, or storage used by a dependency.

### Ownership and state

No allocation, release, ownership transfer, locking protocol, or lifetime extension is explicitly established.

Unknown dependencies may change global state, mutate accessible memory, allocate or release storage, alter execution state, or fail to return. Their apparent lack of explicit arguments supplies no preservation guarantee.

### Preconditions and errors

Continuing through the represented trace requires:

- valid control flow and callable indirect targets;
- accessible storage for every reached memory access;
- compatible alignment or platform support for each word access;
- valid ambient machine state;
- dependency behavior that permits continuation.

There are no visible null checks, bounds checks, status checks, retries, or rollback operations. Failure midway can leave earlier writes observable. Fault delivery, error reporting, and recovery behavior are unknown.

## Dependencies

F-C001–F-C097 are opaque invocation IDs assigned solely for this contract.

Established dependency properties are limited to invocation order, visible explicit arguments, and selected result consumption described above. The supplied material does not establish:

- resolved public APIs or application roles;
- whether separate call sites reach the same target;
- calling convention or complete register inputs;
- preservation of memory, globals, registers, or ownership;
- whether aggregate results are genuine 64-bit returns or recovered register pairs;
- whether calls return normally.

The call graph therefore remains **documented_only** beyond the visible invocation trace.

## Acceptance examples

These examples are conditional static acceptance criteria, not observed executions.

1. **Initial write truncation.** With primary base `0x12345678` and a valid initial byte destination, the first byte store writes `0x78`. The second input is separately written as all 32 bits at the primary base plus `0x34`.

2. **Upper-half overwrite.** If F-C007 returns lower half `3` and upper half `0x11223344`, the later primary-base word store writes `0x11223344`, and F-C008 receives `12`, under the stated width interpretation.

3. **Compact-index boundary.** A layout seed of `0xffffffff` produces compact index `0x3ff`. Its fourth-input-derived byte destination is `0x406`. A selector byte of `0xff` read from the layout base plus `1` produces destination `0x110`, not a sign-extended address.

4. **Selector mutation between reads.** If the layout selector read before F-C033 yields `2`, but the read after F-C039 yields `5`, the earlier compact-index byte goes to `0x13`, while the later zero goes to `0x16`. Subsequent small-base calculations use `10`.

5. **Retained word versus retained byte.** If F-C034 returns `0x00000800`, the retained byte is zero, while the later word store derived from that same result writes `0x00000100`. Truncating the retained word early would be incorrect.

6. **Literal-address ordering.** Subject to aliasing, dependencies, and continued execution, literal address `0x0d` first receives the saved selector and later receives `0xc0`. The earlier write cannot be discarded merely because a later explicit write exists.

7. **Fresh final selector reads.** If the final two selector reads yield different bytes, their zero stores target different addresses. Reusing the first byte would violate the represented trace.

8. **Partial completion.** If F-C001 does not return, the two initial stores have already occurred. No later explicit operation is required to occur.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered region is valid executable code throughout.
- The actual entry convention and origins of ambient values.
- Exact machine semantics for signed-looking arithmetic, overflow, and address wrapping.
- Endianness, access alignment, memory mapping, and device-memory behavior.
- Complete dependency arguments and effects.
- Aggregate-result ABI interpretation.
- The secondary outgoing value’s precise origin and width.
- Whether any apparent structure offsets represent real fields.
- Runtime behavior at the truncated terminal boundary.
- Any application-level purpose or ownership model.

Inferred intent is limited to the observation that repeated offsets and scaling resemble state-field updates. No semantic labels such as allocation, initialization, serialization, or cleanup are justified.

## Evidence

The supplied static representation establishes the visible sequence of explicit memory operations, indirect invocations, arithmetic expressions, retained result uses, and the terminal decoding boundary.

Its instruction-decoding warning materially limits confidence in treating that sequence as a complete executable routine. This contract preserves those limits rather than repairing control flow or assigning meanings to unknown dependencies.

No runtime tests have been performed. Acceptance examples are arithmetic and ordering consequences conditional on the represented trace, not claims of successful execution.
