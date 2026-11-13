<!-- Trusted provenance: F-8cc5bebf9647; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8cc5bebf9647

## Purpose

F-8cc5bebf9647 performs a represented straight-line sequence of indirect calls, global assignments, and memory accesses of several widths. Two dependencies are identified as F-93c4ce5fe2dd and F-7d50b94821c2. The supplied representation ends at an invalid-instruction boundary; it does not demonstrate a normal return.

A higher-level application purpose is **documented_only**. The evidence does not establish object initialization, format processing, device control, or another coherent application operation.

This contract describes conditional behavior: later operations require earlier accesses to succeed and dependencies to return. It does not establish that the complete represented path is executable.

## Interface

### Explicit inputs and output

| Input | Represented interpretation |
|---|---|
| A | Signed 32-bit value, right-shifted by 32 for the first call. The shift result is unresolved. |
| B | Opaque 32-bit value with no explicit use on this path. |
| C | Signed 32-bit address-like value; a word at offset `0x28` is read before the first call. |
| D | Pointer to a 32-bit unit whose contents supply an argument to the first call. |

No return value or successful completion is represented.

### Ambient inputs and stack-associated evaluations

Additional inputs include an incoming link-register value, an incoming register-associated address/value, global storage, stack-associated storage through offset `0x3ec`, and a register-associated byte available after the final indirect call.

Here, **S[o] means an evaluation of the stack-associated value at byte offset o at the stated point in the sequence**. It does not mean an immutable entry-time snapshot. Each occurrence is a separate represented evaluation unless explicitly retained under another name.

This distinction applies across calls and intervening writes. For example:

- S[`0x324`] is evaluated before U-02 and again after U-03.
- S[`0x134`] is evaluated after U-05.
- S[`0x384`] is evaluated repeatedly, including after F-7d50b94821c2.
- S[`0x3ec`] is not frozen before U-01, which receives a reference to nearby stack-associated storage at `0x3e0`.

The correspondence between these evaluations and physical stack loads is **documented_only**. Neither preservation nor aliasing can be assumed from the recovered stack notation.

### Widths, arithmetic, and preconditions

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Address offsets are bytes. The first call has a represented 64-bit result, divided into low and high 32-bit halves. Pointer values are represented as 32-bit quantities.

Unsigned right shifts are logical. Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Narrow writes retain the low 8 or 16 bits. Signed additions, multiplication, left shifts, and conversions do not establish portable behavior outside their representable ranges. In particular, converting an eventual result to unsigned does not resolve an earlier signed overflow.

A right shift of a represented 32-bit operand by 32 has no established result here. No masked count, zero result, sign replication, or wider hidden operand is assumed.

Endianness, alignment rules, memory permissions, and invalid-address behavior are **documented_only**. Following the complete path requires all encountered accesses and call targets to be valid. No represented validation establishes those conditions.

## Behavior

The order below is the represented order between operations. Argument evaluation order and the relative order of multiple reads within one expression are not established unless stated.

### 1. Initial call and globals

1. Read and retain the signed word at C plus `0x28` as a displacement.
2. Read the word through D while preparing U-01.
3. Call U-01 through the incoming link-register value with its lowest two bits cleared. Its four represented arguments are:
   - A right-shifted by 32;
   - a reference to stack-associated storage at `0x3e0`;
   - the word read through D;
   - a reference to stack-associated storage at `0x28`.
4. From the returned low 32-bit half plus the retained displacement, read a byte. Zero-extend it into an address **P**, necessarily between 0 and 255.
5. Read and retain a word **W** at the returned high half plus `0x30`.
6. Read and retain a word **V** at P.
7. Write V as a word at P plus W plus 4.

The W read precedes the V read. P is derived from a byte, not from a pointer-sized memory read.

Next:

1. Read and retain a global word as an address base.
2. Assign `0x2848` to another global location.
3. At that location plus `(V AND W)`, write the low byte of the incoming register-associated value.
4. Assign `0xe5` to one global location, then `0xe5` to another.
5. Form and retain unsigned **G** from the earlier address base plus `0xf6`.
6. Write G logically shifted right by two as a word at that earlier base plus `0x12a`.

The declarations do not establish the widths of the three global assignments. Global-symbol overlap is explicitly reported; the byte write can overlap global storage. G’s formation includes signed addition before its unsigned destination interpretation.

### 2. Early calls and shifted-address writes

1. Call U-02 with `(S[0x324] >> 2) << 4` and fixed value `0x089827c8`. These shifts operate on an unsigned 32-bit value.
2. Call U-03 with no explicit arguments.
3. Twice, in order, write G shifted right by two at `(G >> 2) + 0x30`.
4. Write S[`0x324`] shifted right by two at S[`0x18`] plus `0x14`.
5. Evaluate S[`0x324`] again; retain its logical right shift by two as **Q**.
6. Call U-04 with Q and the then-evaluated S[`0xc0`].
7. Call U-05 with no explicit arguments.
8. Evaluate signed S[`0x134`], shift it left by two, and retain the unsigned 32-bit result as **H**.
9. Call U-06 with:
   - a byte read at `(S[0x324] >> 4) + 0x17`;
   - S[`0xec`];
   - H;
   - Q shifted right by two.
10. Perform a word write whose destination is a byte read at the incoming register-associated address plus `0x14`, plus `0x18`. Its value is a separate byte read at that same source address, zero-extended.
11. Retain **J** as signed S[`0x194`] multiplied by four, and **R** as H logically shifted right by four.
12. Call U-07 with fresh evaluations of S[`0x194`] and S[`0x1dc`].
13. Call U-08 with no explicit arguments.
14. Write `R >> 2` at R plus `0x34`.
15. Call U-09 with two separate evaluations of `S[0x214] >> 2`, followed by S[`0x260`].
16. Call U-10 with no explicit arguments.
17. Assign `R >> 2` to stack-associated S[`0x78`], then repeat the write at R plus `0x34`.
18. Twice write `R >> 6` at `(R >> 4) + 0x24`.
19. Evaluate and retain **F** as `S[0x304] >> 2`; write F at F plus `0x14`.
20. Write `R >> 6` at `(R >> 4) + 0x18`.
21. Write the then-evaluated `S[0x384] >> 2` at F plus `0x14`.
22. Read and retain byte **C8** at `(R >> 4) + 0x1b`.
23. Read and retain signed word **D32** at `(S[0x384] >> 2) + 0x2f`, using another stack-associated evaluation.

The two byte reads in step 10 are distinct; their relative order is unresolved. The two S[`0x214`] evaluations are likewise not a single established snapshot. Q, H, R, J, F, C8, and D32 are retained scalar results, unlike subsequent S evaluations.

### 3. Call chain and halfword writes

Call U-11 with `(S[0x384] >> 2) + 0x25`, evaluating S[`0x384`] again. Then call U-12 through U-19 in order, with no explicit arguments.

U-20 supplies a retained signed 32-bit result. U-21 receives that result and its signed left shift by four. U-22 has no explicit arguments. U-23 supplies another signed 32-bit result.

Define retained values:

- **N**: U-23’s result minus `0x35`.
- **M**: unsigned 32-bit destination value of the signed product `N × (D32 + 0x31)`.
- **E**: signed product `N × 2`.
- **L**: M’s low 16 bits.

Subject to the signed-arithmetic limitations, perform these halfword writes in order:

| Destination | Value |
|---|---|
| M + `0x0a` | C8 + `0x33` |
| M + `0x0c` | L |
| M + `0x0e` | L |
| `(M >> 2) + 0x0e` | L |
| M + `0x12` | L |
| E + `0x12` | L |
| S[`0x288`] + `0x14` | L |
| M + `0x16` | L |
| J + `0x16` | L |
| M + `0x18` | L |
| M + `0x1a` | Low 16 bits of `S[0x2c4] >> 2` |
| E + `0x1c` | L |
| S[`0x3ec`] + `0x20` | Low 16 bits of `S[0x218] >> 2` |
| `(S[0x1dc] >> 2) + 0x22` | Low 16 bits of a separate `S[0x1dc] >> 2` evaluation |
| `(S[0x64] >> 2) + 0x24` | Low 16 bits of S[`0x3ec`] |

C8 plus `0x33` ranges from 51 through 306 and fits in 16 bits. Each table occurrence of S is evaluated for its own operation; intervening writes may alias the associated storage.

### 4. Word writes and F-93c4ce5fe2dd

Perform these operations in order:

1. Read E plus `0x3c` twice: one word supplies a destination base, the other supplies the word written at that base plus `0x18`.
2. Write `S[0x3ec] >> 2` at M plus `0x3c`.
3. Write `S[0x39c] >> 2` at S[`0x39c`] plus `0x18`.
4. Perform the analogous write using S[`0x3d8`].
5. Write S[`0x3ec`] at S[`0x3ec`] plus E.
6. Write `S[0x3ec] >> 2` at `(S[0x3ec] >> 2) + 0x14`.
7. Call F-93c4ce5fe2dd with the then-evaluated `S[0x3ec] >> 2`.
8. Write its opaque 32-bit result at `S[0x3ec] × 4 + 0x14`, evaluating S[`0x3ec`] after the call.
9. Evaluate S[`0x3ec`] again and retain **Y** as its logical right shift by two.
10. Read and retain zero-extended byte **B8** at Y plus `0x1f`.
11. Call U-24 with B8 and a fresh evaluation of `S[0x3ec] >> 2`.

There is no established invariant equating U-24’s second argument with Y. The call, result write, or intervening aliases can affect subsequent stack-associated evaluations. In steps containing repeated S expressions, their equality and relative evaluation order are not independently established.

### 5. Retained-byte expansion

Let **Z** be retained Y logically shifted right by two, and **K** be retained B8 shifted left by eight.

After U-24:

1. Write Z at Y plus `0x34`, then at Y plus `0x18`, then at Z plus `0x14`.
2. Write Z’s low halfword at the then-evaluated `S[0x3ec] × 4`.
3. Call U-25 with `B8 << 2` as both represented arguments.
4. Write Z again at Y plus `0x18`.
5. Write Z at `(B8 >> 2) + 0x24`, then again at Z plus `0x14`.
6. Write `B8 << 6` at `(B8 >> 2) + 0x18`.
7. Write the low halfword of `B8 × 64` at that product plus 4.
8. Write K at `(B8 AND 0x3fffffff) + 0x18`.
9. Write Z’s low halfword at Y plus 4.
10. Twice write Z at `((B8 AND 0xffffff) × 64) + 0x24`.
11. Assign Y to stack-associated S[`0x2c`].
12. Call U-26 with K and `(B8 AND 0xffffff) << 6`.
13. Write its opaque word result at `K × 4 + 0x14`.
14. Write Z’s low halfword at Z plus 2.
15. Write Z at `(K >> 2) + 0x24`.
16. Write the low halfword of `K >> 2` at `(K >> 2) + 4`.
17. Call U-27 with K and `K >> 2`.
18. Write Z’s low halfword at K plus 6.

B8 is between 0 and 255, so both masks leave it unchanged. K ranges from zero through `0xff00`; the byte-expansion calculations fit in 32 bits.

### 6. Final effects and boundary

1. Repeat the two-read, pointer-directed word write involving E plus `0x3c`. These are new reads, not reuse of earlier contents.
2. Write `K >> 2` at `K × 4 + 0x3c`.
3. Write `K >> 2` at S[`0x220`] plus `0x20`.
4. Write `K >> 2` at `(K >> 2) + 0x14`.
5. Write the then-evaluated `S[0x384] >> 2` at `(K >> 2) + 0x24`.
6. Call F-7d50b94821c2 with `K >> 4` and S[`0x178`].
7. Evaluate S[`0x384`] again, and read a byte at its logical right shift by two plus `0x18`; retain this byte as **T8**.
8. Evaluate S[`0x258`] and retain signed address-like value **A32** as that value plus `0x31`. Separately form the signed product `N × 4` for the final byte-write address.
9. Write byte 100 at a fresh evaluation of S[`0x258`] plus `0x45`.
10. Read and retain word-sized destination pointer **P32** at another evaluation of S[`0x258`] plus `0xa5`.
11. Write A32 as a word at T8 plus `0x38`.
12. At P32 plus offsets 0, 4, 8, and 12, write respectively: 100, A32, A32, and P32.
13. Call U-28 with 100, A32, and `(K >> 2) + 0x25`.
14. Write a register-associated byte at `(N × 4) + 0x1e`, using the product formed before U-28.
15. Encounter the invalid-instruction boundary.

The post-F-7d50b94821c2 byte read does not use a retained earlier S[`0x384`] value. Likewise, the three S[`0x258`] evaluations are not unconditionally equal.

The final register-associated byte is not established as U-28’s formal return value.

## Side effects

Direct effects include global mutations, two stack-associated assignments, and computed-address byte, halfword, and word writes. Repeated writes remain separate ordered effects even when their addresses and values match.

No disjointness guarantee exists. Destinations can alias inputs, globals, stack-associated storage, other destinations, or dependency-accessible memory. Such aliasing can affect later evaluations. A final-memory-only description would omit potentially observable intermediate writes.

Retained scalar results are distinguished from later memory evaluations. Dependencies must not be assumed to preserve accessible memory, globals, ownership, or termination behavior. They may have additional effects not recoverable here.

No allocation or release is directly represented. This does not establish that dependencies neither allocate nor release memory. Ownership transfer, synchronization, atomicity, rollback, and error-reporting contracts are **documented_only**.

## Dependencies

| Opaque ID | Represented interface |
|---|---|
| U-01 | Incoming-context-selected target; four arguments; 64-bit result. |
| U-02–U-10 | Early calls described above; no consumed results. |
| U-11–U-19 | U-11 has one represented argument; others have none. |
| U-20 | No explicit arguments; consumed signed 32-bit result. |
| U-21 | Two represented arguments; no consumed result. |
| U-22 | No explicit arguments; no consumed result. |
| U-23 | No explicit arguments; consumed signed 32-bit result. |
| F-93c4ce5fe2dd | One 32-bit argument; consumed opaque 32-bit result. |
| U-24–U-27 | Two represented arguments each; U-26 supplies a consumed word result. |
| F-7d50b94821c2 | Two represented arguments; no consumed result. |
| U-28 | Three represented arguments; subsequent register-associated byte has unresolved provenance. |

U-series IDs label call occurrences, not proven distinct functions. Their targets are represented through address expressions. Complete ABI, hidden arguments, ownership effects, failure behavior, and termination are **documented_only**. “No explicit arguments” does not establish a true zero-argument interface.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial read order.** If the selected byte produces P = 7, the routine first reads W at the returned high half plus `0x30`, then reads V at address 7.
2. **Retained displacement.** If U-01 changes C plus `0x28`, the byte-address calculation still uses the displacement read before U-01.
3. **Non-snapshot stack evaluation.** If U-03 changes the storage represented by S[`0x324`], later evaluations are not required to equal the value used for U-02. Q is retained only when explicitly formed afterward.
4. **Call-modified S[`0x3ec`].** If F-93c4ce5fe2dd changes that stack-associated value, its result-store destination and the later formation of Y use their respective post-call evaluations, not its argument’s earlier value.
5. **Post-call final byte.** If F-7d50b94821c2 changes S[`0x384`], the subsequent byte-read address follows the post-call evaluation. Earlier uses do not constrain it to the old value.
6. **Halfword narrowing.** If M’s low halfword is `0xabcd`, every L-valued write stores those 16 bits.
7. **Maximum expansion.** For B8 = 255, K is `0xff00`; U-26 receives `0xff00` and `0x3fc0`.
8. **Zero expansion.** For B8 = 0, low-address operations remain represented; no null guard skips them.
9. **Repeated destination.** The location `(K >> 2) + 0x24` receives Z and later a fresh `S[0x384] >> 2`. Both writes matter even if their values coincide.
10. **Final layout and failure.** If reached successfully, P32 receives the four ordered words specified above. A fault or nonreturning dependency prevents promises about later effects; no rollback or status result is established.

## Uncertainties

The following remain **documented_only**:

- Whether this is a valid contiguous executable routine and whether the terminal boundary is reachable.
- Complete calling context and physical realization of stack-associated evaluations.
- Preservation and aliasing across calls and direct writes.
- Shift-by-32 behavior, signed overflow, signed shifts, and relevant conversions.
- Incompletely declared global widths and exact overlap.
- Endianness, alignment, valid mappings, permissions, and access-fault behavior.
- Dependency semantics, ownership, hidden effects, and final-byte provenance.
- Behavior at or beyond the invalid-instruction boundary.

No validation branches or loops appear in the supplied straight-line representation. This does not exclude control flow inside dependencies or unrecovered behavior.

## Evidence

**Established static evidence:** represented access widths where explicit, arithmetic expressions, call ordering and arguments, consumed results, repeated stack-associated evaluations, direct assignments, global-overlap warnings, and the invalid-instruction boundary.

**Inferred intent:** no application-level purpose is adopted as a requirement.

**Unknowns:** retained explicitly above rather than replaced with preservation, ownership, or execution assumptions.

Only the supplied static representation and dependency mapping were used. No implementation access, runtime execution, or runtime tests were performed.
