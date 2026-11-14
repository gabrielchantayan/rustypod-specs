<!-- Trusted provenance: F-ad69d3db63a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad69d3db63a5

## Purpose

F-ad69d3db63a5 performs a linear sequence of indirect calls, computed-address memory accesses, and extraction or placement of fields from 32-bit values. The supplied static representation does not establish an application-level purpose such as serialization, allocation, device configuration, or object construction.

The represented path ends at an invalid-instruction boundary, without an ordinary return. Successful completion and any intended continuation beyond that boundary are not established.

This contract describes the visible behavior conditionally: each effect requires execution to reach it, prior dependencies to return, and the relevant memory accesses to succeed. It does not validate the recovered control flow, low numeric addresses, or implicit register inputs.

## Interface

### Explicit inputs

| Input | Represented width | Established direct use |
|---|---:|---|
| First argument | 32 bits, uninterpreted | Supplied to the first indirect call. |
| Second argument | 32 bits, uninterpreted | No direct use is visible. |
| Third argument | 32 bits, uninterpreted | No direct use is visible. |
| Fourth argument | 32 bits, represented as signed | Shifted left by ten and supplied twice to one call. |

The absence of a direct use does not prove that an argument is irrelevant under the actual calling convention. Dependencies may consume machine state not expressed as arguments.

### Additional incoming state

The operation also depends on:

- An incoming link-register value used to construct the first call target.
- An implicit base-address register used for an early byte read.
- An implicit 32-bit payload register used for the first word write.
- An auxiliary result register referenced after indirect calls.

The auxiliary register is not a declared argument or a separately explained return value. Its producer, lifetime, and preservation across calls remain unresolved.

### Outputs and completion

There is no established explicit return value. Observable results are direct memory accesses, dependency invocations and their displayed arguments, dependency effects, and behavior at the terminal invalid-instruction boundary.

No status reporting, validation failure result, recovery path, or rollback is visible.

### Width and arithmetic conventions

The address and bit-pattern examples below use a conditional 32-bit machine interpretation:

- Additions, multiplications, and left shifts retain their low 32 bits.
- Byte reads produce unsigned values from `0` through `255`.
- Byte writes retain the low eight bits of their value.
- Word accesses transfer 32 bits.
- Right shifts applied to represented unsigned values are logical.
- The represented 64-bit result is separated into low and high 32-bit halves.

This convention describes the apparent machine-level calculations. It does not establish defined signed-language behavior for overflow or signed left shifts.

Endianness, alignment restrictions, memory mappings, access permissions, and volatile or device-memory semantics are not supplied.

## Behavior

### Sequencing and terminology

The recovered path has no visible branch or loop. Calls displayed without arguments still represent observable execution boundaries; they are not assumed to consume no state or have no effects.

“Token,” “selector,” “scaled token,” and “retained word” are descriptive terms, not established application types. A token multiplied by four is its *scaled token*. All formulas below use the width convention above.

All unresolved indirect dependencies are identified as **F-UNRESOLVED**. This denotes unresolved call sites, not a claim that every invocation reaches one function.

### 1. Initial word publication

The first F-UNRESOLVED invocation receives the first explicit argument. Further indirect calls lead to a call producing an initial 32-bit token.

The operation writes the implicit payload word at:

- Address: four times the initial token, plus `0x14`.
- Value: the implicit payload register’s 32-bit contents.

The next call receives four times the initial token.

There is no visible check that the token identifies an allocated object, that the destination is writable, or that the payload represents an address.

### 2. First selector and two byte publications

After intervening calls, a dependency produces the first selector.

The operation reads one byte at the implicit base address plus one. It then writes the low eight bits of four times the selector at the read byte plus `0x11`. A call receives four times the selector.

The destination has no visible additional base. Since the index is one byte, its numeric range is `0x11` through `0x110`.

After further calls, another dependency produces a token. Before forwarding that token, the operation:

1. Reads a byte at 64 times the first selector, plus one.
2. Retains that byte as the first byte index.
3. Writes at the first byte index plus `0x11`.
4. Stores the low eight bits of the selector masked with `0x3FFFFFFF`, then shifted right by 20.

The stored byte therefore contains selector bits 20 through 27. Although the masked and shifted intermediate includes bits corresponding to original bits 28 and 29, byte truncation discards them.

The next call receives the newly produced token shifted left by two. Another token-producing sequence follows. A later two-argument call receives, in order:

1. The newer token shifted left by two.
2. The retained first byte index.

### 3. Fourth-argument call and zero-byte publication

Following another indirect call, a dependency receives the fourth explicit argument shifted left by ten in both displayed argument positions.

After the next call, the operation reads a byte at numeric address `1` and writes zero at that byte plus `0x11`.

The read address is `1` because its represented multiplier operand is zero. The zero store does not eliminate the byte read, and the representation does not establish a valid null-based object.

The following call receives three displayed values derived from the auxiliary result register:

1. The auxiliary value.
2. The auxiliary value again.
3. Sixteen times the auxiliary value.

### 4. Auxiliary-result byte publication

After additional calls and a token-producing invocation, the operation reads a byte at sixteen times the represented auxiliary value, plus one. It retains this byte as the second byte index.

It writes at the second byte index plus `0x11`, storing the low byte of the auxiliary value shifted right by 22. The resulting byte selects original auxiliary-value bits 22 through 29.

The next call receives the newly produced token shifted left by two. After another token-producing sequence, a two-argument call receives:

1. The retained second byte index.
2. The newer token shifted left by two.

This argument order differs from the earlier two-argument call.

The repeated auxiliary-register reference does not establish a preservation guarantee across the intervening dependencies. Its actual value and provenance at each relevant point remain unresolved.

### 5. Split result and four word publications

A subsequent dependency receives zero in both displayed argument positions. The next dependency is represented as returning 64 bits.

Let the result’s upper and lower 32-bit portions be its high half and low half. The following direct effects occur in order, separated by the visible dependency calls:

| Order | Word destination | Stored value | Following material call argument |
|---:|---|---|---|
| 1 | Four times the high half, plus `0x14` | Zero | The high half, unscaled |
| 2 | Four times a fresh token, plus `0x14` | The high half | Four times that token |
| 3 | Low half times `0x10000000`, plus `0x14` | Low half times `0x04000000` | The stored value |
| 4 | Four times another fresh token, plus `0x14` | Low half masked with `0x1FFFFFFF`, then shifted right by nine | Four times that token |

The fourth stored value contains original low-half bits 9 through 28 and is at most `0x000FFFFF`.

The third destination and value have different multipliers. Neither should be replaced by the other or interpreted as an allocation-relative access without additional evidence.

### 6. Four scaled-token stages

Each of these stages begins with a fresh dependency-produced token. Its first write stores the scaled token at sixteen times the token plus `0x14`.

That destination is generally different from the scaled token plus `0x14`.

#### First stage: two argument reads

After the initial write, the operation reads a word at the scaled token plus `0x14` for a logical right shift by 12. It also reads a word at that same address for a left shift by two.

A three-argument call receives:

1. The right-shifted read value.
2. The scaled token.
3. The left-shifted read value.

Both reads precede the call. Their relative evaluation order is not established by the supplied representation. They are two accesses, not one guaranteed shared sample.

After intervening calls, a fresh destination token is obtained. The stage’s scaled token is written at four times this destination token plus `0x14`. The following call receives four times the destination token.

#### Second stage: low-half argument

A fresh stage token produces a new scaled token. After the initial sixteen-times-token-address write, a call receives:

1. The scaled token.
2. The scaled token again.
3. The original split result’s low half shifted left by 26.

After further calls, a fresh destination token identifies a word destination at four times that token plus `0x14`. The current scaled token is written there, and the next call receives four times the destination token.

#### Third stage: scaled-token argument

A new stage token produces a scaled token and the same initial form of word write.

The next material call receives the scaled token. After intervening calls and another destination-token result, the operation writes the scaled token at four times the destination token plus `0x14`. The following call receives four times that destination token.

#### Fourth stage: retained word

A further stage token produces a scaled token and the initial word write.

The operation then reads one word at the scaled token plus `0x14` and retains the value before the next call. That call receives the scaled token.

After intervening dependencies, a fresh destination token is obtained. Call this the **retained destination token**, because it also remains relevant to later calculations.

The operation writes the retained word logically shifted right by 12 at four times the retained destination token plus `0x14`. The next call receives four times the retained destination token.

This write uses the saved read value; no replacement read is visible after the intervening calls.

### 7. Fifth scaled-token stage and bit-field call

Another dependency produces a stage token. The operation writes four times that token at sixteen times the token plus `0x14`.

The next material call receives:

1. Four times the retained destination token, masked with `0x3F`, then shifted left by 14.
2. The current stage’s scaled token.
3. The retained destination token shifted left by 30.

The first argument can contain bits only in positions 16 through 19: multiplication by four clears its two lowest bits before masking. The third argument retains only the retained destination token’s original lowest two bits, placed in positions 30 and 31.

After further calls, a new destination token is produced. The current scaled token is written at four times this new token plus `0x14`. The following call receives four times the new token.

This newer destination token does **not** replace the retained destination token in the final byte-read address described next.

### 8. Final selector and two byte publications

After additional calls, a dependency produces the final selector.

The operation reads a byte at:

- The retained destination token from the fourth scaled-token stage, multiplied by `0x10000000`, plus one.

It writes the low byte of the final selector shifted left by two at the read byte plus `0x11`.

The next call receives:

1. The final selector shifted left by two.
2. The same shifted value.
3. The final selector shifted left by four.

This call target is constructed using an offset of `0xFEC`, unlike the `0xFFE` offset used by the other displayed target constructions.

A subsequent dependency returns an uninterpreted 32-bit value, which is forwarded to the next dependency. Further calls lead to another token result.

The operation then rereads the same byte address based on the retained destination token. It writes at this newly read byte plus `0x11`, storing the low byte of the final selector masked with `0x3FFFFFFF`, then shifted right by 20.

The two byte reads are distinct and separated by dependency calls. Their results need not match.

The next call receives the latest token shifted left by two. After another token-producing sequence, a two-argument call receives:

1. The newer token shifted left by two.
2. The byte index obtained by the second final read.

One further indirect call precedes the invalid-instruction boundary.

## Side effects

Conditional on reaching every direct write, the representation contains **15 explicit 32-bit word writes and six explicit byte writes: 21 writes total**.

The word-write count consists of:

- One initial payload write.
- Four split-result-related writes.
- Eight writes across the first four scaled-token stages.
- Two writes in the fifth scaled-token stage.

The byte-write count consists of:

- Two first-selector writes.
- One zero write.
- One auxiliary-result write.
- Two final-selector writes.

These counts exclude all unknown dependency effects. They describe access occurrences, not distinct destinations; aliasing can make several writes target the same location.

The representation also contains six explicit byte reads and three explicit word reads. Two of the word reads occur in the first scaled-token stage, and one occurs in the fourth.

No access is visibly guarded by bounds, null, alignment, permission, or overflow checks. An invalid access or nonreturning dependency may prevent later effects. Previously completed writes have no visible rollback.

Aliasing is possible through equal inputs, truncation, or 32-bit wraparound. Earlier writes and dependency mutations may affect later reads. No atomicity, isolation, or concurrency guarantee is established.

No explicit allocation, deallocation, ownership transfer, reference counting, or lifetime-management operation is identifiable. Passing an address-like value does not establish that a dependency owns, retains, or releases memory.

## Dependencies

F-ad69d3db63a5 is the only resolved function identity supplied.

**F-UNRESOLVED** denotes unresolved indirect dependencies. Different call sites may reach different functions. Similar target construction does not establish common identity.

Most displayed targets add `0xFFE` to the incoming link-register value or an embedded code-location value. One late three-argument call instead uses `0xFEC`.

Visible dependency roles include:

- Receiving one, two, or three displayed 32-bit arguments.
- Returning tokens, selectors, or an uninterpreted word.
- Producing a represented 64-bit result.
- Potentially contributing auxiliary register state.
- Executing with no arguments displayed.

Actual signatures, executable target mappings, hidden inputs, preservation rules, error behavior, and ownership effects are unknown. No dependency is established as pure, nonthrowing, normally returning, or preserving globals or memory.

## Acceptance examples

These are conditional static criteria, not runtime test results.

1. **Write accounting.** If the complete represented path reaches all explicit stores, it performs 15 word writes and six byte writes. Unknown dependency writes are additional and uncounted.

2. **Selector truncation.** If the first selector is `0x12345678`, its first byte publication stores `0xE0`; its later masked, right-shifted byte publication stores `0x23`.

3. **Byte-index destination.** If a byte-index read returns `0xFF`, the corresponding destination is numeric address `0x110`, without an additional base.

4. **Zero still requires a read.** If execution reaches the zero-byte publication and address `1` yields `5`, zero is written at `0x16`. A failed read does not establish that the store occurs.

5. **Split-result arithmetic.** If the low half is `0xFFFFFFFF`, the large-scale publication writes `0xFC000000` at `0xF0000014`. The masked right-shift publication stores `0x000FFFFF` at its separately token-derived destination.

6. **Distinct scaled addresses.** With stage token `3`, the scaled token is `12`. The initial stage write stores `12` at `0x44`; a read at scaled token plus `0x14` accesses `0x20`.

7. **Retained destination provenance.** If the fourth stage’s retained destination token is `1` and the fifth stage’s destination token is `2`, both final byte reads use `0x10000001`, not `0x20000001`.

8. **Mutable reread.** If that final byte address yields `2` first and `7` later, the final byte writes target `0x13` and `0x18`, respectively.

9. **Partial effects.** If a dependency does not return, subsequent direct accesses are not required. The visible operation provides no undo of earlier writes.

## Uncertainties

The following remain **documented_only**:

- Accuracy of recovered control flow and call boundaries near invalid instruction data.
- Actual calling convention and provenance of implicit inputs.
- Auxiliary-register production and validity across intervening calls.
- Dependency identities, signatures, preservation rules, errors, and ownership behavior.
- Memory mappings, endianness, alignment, permissions, and hardware side effects.
- Intended semantics where represented signed arithmetic overflows or shifts left.
- Relative order of the first scaled stage’s two argument reads.
- Whether the terminal boundary is a real trap, unreachable data, or missing continuation.
- Any application meaning for tokens, selectors, offsets, or extracted fields.

“Publication” and “bit-field” describe visible writes and arithmetic. They do not establish synchronization semantics or domain intent.

## Evidence

This contract is based solely on the supplied static representation: four explicit arguments, implicit register state, indirect calls, explicit byte and word accesses, masks and shifts, and termination at a reported invalid-instruction boundary.

The direct-write inventory establishes 15 word stores and six byte stores, conditional on reaching all of them. No dependency effects are included in that inventory.

No dependency bodies, runtime traces, memory maps, or external interface definitions were available. No runtime tests have been performed. Acceptance examples are arithmetic and sequencing consequences of the stated static interpretation, not observations of execution.
