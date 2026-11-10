<!-- Trusted provenance: F-5ba9ec2a02d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ba9ec2a02d6

## Purpose

F-5ba9ec2a02d6 consumes a mutable context and three two-component integer inputs, selects a subdivision depth, and issues coordinate pairs to F-7ed325e66f98.

Its structure is consistent with processing a cubic curve: it uses four coordinate pairs, a depth-selection calculation, a subdivision dependency, and a depth-first traversal. That interpretation is **inferred intent**, not an established geometric guarantee. In particular, one depth-selection expression uses the first component of the second argument where a symmetric coordinate formula might suggest its second component. The contract preserves that asymmetry.

Two execution paths are established:

- At selected depth one, issue two coordinate pairs directly, without initializing the traversal workspace.
- At greater depth, initialize context-resident records and traverse them, invoking F-507b27e06f8e on records that require subdivision and are not rejected by a vertical-range condition.

The function has no explicit return value or error result. Its externally visible behavior consists of memory accesses, context writes, dependency calls, and any effects of those dependencies.

## Interface

### Arguments

The function accepts four arguments:

| Argument | Contract designation | Required access |
|---|---|---|
| First | Context | A valid, suitably aligned address for the context fields and workspace described below |
| Second | Pair A | Two readable 32-bit signed integer components, A.x and A.y |
| Third | Pair B | Two readable 32-bit signed integer components, B.x and B.y |
| Fourth | Pair E | Two readable 32-bit signed integer components, E.x and E.y |

Pair components occupy consecutive four-byte words. The first component is at byte offset zero and the second at byte offset four.

The supplied representation uses a 32-bit integer to hold the context address. This contract does not establish portability to a wider-address ABI.

### Context layout

All offsets below are byte offsets from the context address. They describe observable access locations, not an externally established structure definition.

| Offset | Width | Established use |
|---|---:|---|
| 0x18 | 32 bits | Signed vertical limit, read during traversal |
| 0x38 | 32 bits | First component of a context-held coordinate pair, designated S.x |
| 0x3c | 32 bits | Second component, designated S.y |
| 0x44 onward | Four-byte words | Subdivision-record workspace |
| 0x34c onward | Four-byte words | Depth-stack workspace |
| 0x4e0 | 32 bits | Second argument to the second F-870c4fcf7295 call |
| 0x4e4 | 32 bits | Second argument to the first F-870c4fcf7295 call |

A subdivision record contains eight consecutive 32-bit words, interpreted by this function as four coordinate pairs. Successive active record addresses are separated by 24 bytes, so adjacent records overlap by two words.

The context must therefore provide more than a single eight-word record when subdivision occurs. Merely allocating through the two parameter words at offsets 0x4e0 and 0x4e4 does not establish that an opaque dependency’s additional storage requirements have been met.

### Numeric model and preconditions

The visible operations use 32-bit integer quantities, signed comparisons, and signed right shifts. The coordinate operations include multiplication, addition, negation, and left shifts.

For ordinary signed-arithmetic interpretation, inputs and intermediate values must remain representable. Notable hazards include:

- Negating the most-negative 32-bit signed value.
- Adding coordinate components before multiplying them.
- Multiplying coordinates by 3, 4, or 12.
- Left-shifting negative values or values whose scaled result is not representable.
- Summing the terms used for emitted coordinates.

The supplied representation does not establish a portable overflow contract. Wrapped machine arithmetic must not be substituted silently for mathematically exact arithmetic or for defined signed-language behavior.

The following descriptions of signed right shifts assume sign propagation, as represented by the supplied operations. Exact overflow behavior remains documented_only.

## Behavior

### Initial observations and depth inputs

Before calling any dependency, the function captures values in this order:

1. Context S.x.
2. E.x.
3. E.y.
4. A.x.
5. Context S.y.
6. A.y.
7. B.x.
8. B.y.

Arithmetic calculations are interspersed with these reads, but no explicit context or input writes occur during this phase.

Define two temporary coordinates:

- U.x = E.x + the arithmetic right shift of S.x by two bits.
- U.y = E.y + the arithmetic right shift of S.y by two bits.

The first depth input is the greater of:

- The absolute value of U.x − 2 × A.x.
- The absolute value of U.y − 2 × A.y.

The second depth input is the greater of:

- The absolute value of U.x − 3 × (A.x + B.x).
- The absolute value of U.y − 3 × (A.x + B.y).

The use of **A.x**, rather than A.y, in the second expression’s vertical term is established by the supplied representation. Whether intentional or erroneous is unknown.

Each “absolute value” is implemented by a signed negative check followed by negation. It is not a saturating absolute-value operation.

The function then makes two calls, in order:

1. F-870c4fcf7295 receives the first depth input and the 32-bit context word read at offset 0x4e4.
2. F-870c4fcf7295 receives the second depth input and the 32-bit context word read at offset 0x4e0.

The second context parameter is read after the first call returns. If the first dependency changes that word through other reachable state, the second call can observe the changed value.

No meaning, normalization rule, or valid range for these context parameters is established here.

### Depth selection

The selected depth begins at one.

While at least one of the two signed dependency results is positive, the function:

- Arithmetic-right-shifts the first result by two bits.
- Arithmetic-right-shifts the second result by three bits.
- Increments the selected depth.

Zero and negative results do not independently keep the loop running. In particular, two nonpositive results select depth one.

Assuming ordinary sign-propagating 32-bit shifts, the selected depth is between one and seventeen inclusive. The first result can require at most sixteen shifts to cease being positive; the second can require at most eleven.

### Direct path: selected depth one

The function computes one coordinate pair from the initially captured values:

- First component: S.x + 4 × E.x + 12 × (A.x + B.x), divided by eight with truncation toward zero.
- Second component: S.y + 4 × E.y + 12 × (A.y + B.y), divided by eight with truncation toward zero.

The divide-by-eight behavior is represented using a sign-dependent adjustment followed by a signed shift. For a representable numerator, negative nonmultiples of eight truncate toward zero, rather than toward negative infinity.

The function calls F-7ed325e66f98 twice:

1. With the context and that computed pair.
2. With the context and the pair (4 × E.x, 4 × E.y).

Both calls use coordinates derived from the initial captures. This function does not reread the pair inputs or S between those calls.

No explicit traversal-workspace stores occur on this path. That does not imply that the dependencies leave the workspace or any other context field unchanged.

### Subdivision-path initialization

For selected depth greater than one, the function writes the initial record in this order:

| Destination offset | Stored value |
|---|---|
| 0x44 | Initially captured E.x, shifted left by two bits |
| 0x48 | A fresh read of E.y, shifted left by two bits |
| 0x4c | A fresh read of B.x, shifted left by two bits |
| 0x50 | A fresh read of B.y, shifted left by two bits |
| 0x54 | A fresh read of A.x, shifted left by two bits |
| 0x58 | A fresh read of A.y, shifted left by two bits |
| 0x5c | A fresh read of context S.x |
| 0x60 | A fresh read of context S.y |

It then writes the selected depth at context offset 0x34c.

The mixture of captured and freshly read values is observable. The two F-870c4fcf7295 calls occur before these writes. If they modify inputs or context through reachable aliases, the initialized record need not describe the same values used to select the depth.

Stores and subsequent reads can also interact when arguments alias the workspace. The function does not stage all initialization values before writing them.

### Traversal and terminal records

Traversal begins at record offset 0x44 and depth-stack index zero.

At each iteration, the function reads the current depth-stack entry. If it is less than two, the current record is terminal.

For a terminal record, designate its first three pairs R0, R1, and R2. The function captures R0.x and R0.y and forms:

- First numerator: 3 × (R1.x + R2.x) + a current read of context S.x + R0.x.
- Second numerator: 3 × (R1.y + R2.y) + a current read of context S.y + R0.y.

It calls F-7ed325e66f98 with those numerators divided by eight, truncating toward zero. It then calls F-7ed325e66f98 with the captured R0 pair.

The context coordinate pair used here is read anew for each terminal record. It is **not** read from the record’s fourth pair. Effects of earlier dependency calls on context S may therefore affect later output coordinates.

The second call for a terminal record uses the captured R0 values, even if the first call changes the record.

After both calls, traversal moves back one depth-stack entry and back 24 bytes in the record workspace. Processing ends when this movement takes the depth index below zero.

### Vertical-range condition and subdivision

When the current depth entry is at least two, the function reads the second components of all four record pairs, in record order. It computes their signed minimum and maximum.

It treats the record as terminal, without subdivision, when either condition holds:

- The current signed context value at offset 0x18 is less than or equal to the minimum component arithmetic-right-shifted by eight bits.
- The maximum component arithmetic-right-shifted by eight bits is negative.

Equality at the first boundary is terminal. A maximum shifted value of zero does not satisfy the second condition.

This condition does **not** discard the record’s outputs. It routes the record through the same two-call terminal behavior described above.

If neither condition holds, the function:

1. Calls F-507b27e06f8e with the address of the current record.
2. Increments the depth-stack index.
3. Stores the previously read depth minus one into the parent depth entry.
4. Advances the current record address by 24 bytes.
5. Stores that same decremented depth into the child depth entry.

The decremented depth is based on the value read before the dependency call. The function does not reload it after that call.

This produces a depth-first traversal structure. The geometry and contents of generated records remain dependent on F-507b27e06f8e.

## Side effects

Explicit writes are confined to the context’s record and depth workspaces on the subdivision path. The function does not explicitly restore their earlier contents.

On the direct path, there are exactly two F-7ed325e66f98 calls, provided earlier calls return normally. Each processed terminal record on the subdivision path likewise produces two such calls.

The supplied representation contains no allocation, deallocation, ownership-transfer operation, or explicit input-pair store. Pair storage remains caller-provided. Opaque dependency effects cannot be used to establish stronger ownership guarantees.

There are no explicit checks for null pointers, insufficient storage, invalid alignment, arithmetic overflow, or dependency failure. There is no status propagation or recovery branch. Normal completion requires dependency calls to return and all required memory operations to be valid.

Reentrancy, thread safety, asynchronous mutation safety, and preservation of unrelated context fields are not established.

## Dependencies

| Opaque ID | Observed interface | Established role |
|---|---|---|
| F-870c4fcf7295 | Two 32-bit arguments; a result subsequently treated as signed 32-bit | Called twice to obtain depth-selection values |
| F-7ed325e66f98 | Context address and two signed coordinate values | Receives each emitted pair |
| F-507b27e06f8e | Current record address | Called before descending to an overlapping child record |

The dependency implementations are unavailable. None is assumed to preserve global state, context contents, aliased inputs, or workspace. None is assumed to allocate, release, retain, or transfer ownership of memory.

The descriptions “emitted pair” and “subdivision” identify how this function uses the calls; they do not establish rendering, geometry, memory-management, or persistence behavior inside the dependencies.

## Acceptance examples

These are conditional acceptance criteria, not observed executions. Unless stated otherwise, arithmetic is representable, dependencies return normally, and dependency calls do not mutate inputs or context.

1. **Nonpositive depth results.** If the two F-870c4fcf7295 results are zero and −1, selected depth is one. No explicit workspace initialization occurs, and exactly two coordinate calls follow.

2. **Shift thresholds.** Results of 4 and 0 select depth three: the first result changes from 4 to 1 to 0. Results of 0 and 8 also select depth three: the second changes from 8 to 1 to 0. Results of 1 and 1 select depth two.

3. **Asymmetric depth input.** Let S and E be zero, A = (2, 7), and B = (3, 11). The first depth input is 14. The second is 39, obtained from the greater of 15 and 39. Replacing A.x with A.y in its vertical term would produce a different contract.

4. **Direct coordinates.** Let S = (8, −8), E = (2, −2), A = (1, −1), and B = (3, −3). If depth one is selected, the coordinate calls receive (8, −8), then (8, −8).

5. **Negative truncation.** Let S = (−1, 0), with A, B, and E all zero. If depth one is selected, the first emitted pair is (0, 0), not (−1, 0).

6. **Upper vertical boundary.** For a nonterminal record whose minimum second component is 256, a context limit of one forces terminal processing. The record still produces two coordinate calls; it is not silently omitted.

7. **Lower vertical boundary.** If a nonterminal record’s maximum second component is −1, its shifted maximum is negative and subdivision is bypassed. If the maximum is zero, this lower-bound condition alone does not bypass subdivision.

8. **Fresh initialization reads.** If the second depth dependency changes B.x before subdivision initialization, the record’s B.x slot uses the changed value. The depth calculation still used the earlier captured B.x.

## Uncertainties

The following are documented_only:

- Exact arithmetic behavior outside representable signed ranges.
- The semantic meaning of the two context parameter words.
- The internal contract and effects of each dependency.
- Whether the asymmetric vertical depth expression is intentional.
- The unit and scale of coordinates and the vertical limit.
- Additional workspace required or modified by F-507b27e06f8e.
- Valid aliasing arrangements beyond the explicitly described access order.
- Behavior under dependency nonreturn, invalid memory, or concurrent mutation.

Curve processing, fixed-point scaling, and visibility-related subdivision avoidance are inferred interpretations. They are not substitutes for the established arithmetic and call behavior.

## Evidence

Established static evidence is limited to the supplied representation of F-5ba9ec2a02d6 and the supplied opaque dependency mapping.

That evidence supports the field offsets, 32-bit operations, initial captures and later rereads, asymmetric depth input, depth-reduction rules, direct-path coordinates, workspace-write order, overlapping record stride, terminal-call sequence, vertical boundary conditions, and dependency-call order described above.

No dependency implementation, independent ABI definition, runtime trace, or memory-ownership specification was supplied. No runtime tests have been performed.
