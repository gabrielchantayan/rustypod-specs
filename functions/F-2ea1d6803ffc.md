<!-- Trusted provenance: F-2ea1d6803ffc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ea1d6803ffc

## Purpose

F-2ea1d6803ffc performs an ordered, short-circuit consistency check over selected scalar fields addressed relative to one caller-supplied base address.

On normal completion, it returns a 32-bit `0` only when every required comparison succeeds and F-7f14f1ab6851 returns a value whose low 32 bits are zero. Otherwise, it returns a 32-bit `1`.

This is not whole-region equality. The checks use different field widths, reuse two previously read values, and reload another field after the dependency call. Some memory within the overall address range is never directly inspected.

The apparent purpose is to compare related representations of data. That interpretation is inferred; the supplied evidence establishes neither field meanings nor a public object layout.

## Interface

### Input and addressing

The function accepts one 32-bit argument used as a base address. All offsets in this contract are byte offsets from that base.

The supplied representation types the argument as a signed 32-bit integer and forms addresses by adding constant offsets. It provides no length parameter, null check, bounds check, alignment check, or explicit address-overflow handling.

The caller must provide valid address arithmetic and accessible memory for every read reached on the selected path. A successful path directly reads through offset `0x39B`. A region extending at least `0x39C` bytes from the base covers the direct-read extent, but does not establish alignment, address validity, or satisfaction of the dependency’s requirements.

An intermediate base-plus-`0x200` address calculation occurs after the first read at `0x008` and before the read at `0x20A`. It does not itself dereference memory. The contract does not infer wraparound behavior for address calculations that exceed the valid range of the supplied arithmetic representation.

### Scalar layout

The direct accesses use these widths:

| Interpretation | Offsets |
|---|---|
| Unsigned 16-bit | `0x008`, `0x20A`, `0x216`, `0x330` |
| Signed 16-bit | `0x214` |
| 32-bit | Every other directly read field listed under Behavior |

The two unsigned 16-bit fields at `0x216` and `0x330` are extended to 32 bits without sign extension before comparison.

The evidence does not establish a serialized byte order. Fields are interpreted through native scalar accesses. Concrete alignment requirements are likewise not supplied.

### Output and errors

Normal returns are exactly:

| Result | Meaning |
|---|---|
| `0` | All required conditions succeeded. |
| `1` | A required condition failed. |

The result does not identify the failing condition.

Invalid reads, invalid address arithmetic, dependency exceptions, and failure of the dependency to return are not translated into this binary result by any visible handling.

### Ownership

The function contains no explicit allocation, release, ownership transfer, or returned memory address.

F-7f14f1ab6851 receives two interior addresses. Its ownership, retention, mutation, and release behavior are unknown. This contract does not assume that those addresses are merely borrowed or that the supplied region remains unchanged.

## Behavior

### Ordered gates

The following gates are reached in order. A failed gate prevents subsequent gates and produces `1` on normal completion.

“Before” in the read-order column denotes sequencing established by separate source-level statements. It is not a promise of exact compiled memory-access order.

| Gate | Required condition | Established read sequencing |
|---|---|---|
| 1 | Unsigned 16-bit `0x008` equals unsigned 16-bit `0x20A`. | `0x008` is read before `0x20A`. |
| 2 | 32-bit `0x018` equals 32-bit `0x20C`. | `0x018` is read before `0x20C`. |
| 3 | 32-bit `0x01C` equals 32-bit `0x210`. | Both reads occur within one comparison expression; their relative order is not established. |
| 4 | The condition involving 32-bit `0x00C` and signed 16-bit `0x214` succeeds, as detailed below. | `0x00C` is read before the expression containing the read at `0x214`. |
| 5 | 32-bit `0x028` equals zero-extended unsigned 16-bit `0x216`. | `0x216` is read before `0x028`. |
| 6 | 32-bit `0x044` equals 32-bit `0x21C`. | Relative operand-read order is not established. |
| 7 | 32-bit `0x048` equals 32-bit `0x220`. | Relative operand-read order is not established. |
| 8 | 32-bit `0x020` equals 32-bit `0x328`. | Relative operand-read order is not established. |
| 9 | 32-bit `0x04C` equals 32-bit `0x32C`. | `0x04C` is read before `0x32C`. |
| 10 | 32-bit `0x02C` equals zero-extended unsigned 16-bit `0x330`. | `0x330` is read before `0x02C`. |
| 11 | The retained value from `0x028` equals 32-bit `0x334`. | Only `0x334` introduces a new field read. |
| 12 | The retained value from `0x02C` equals 32-bit `0x338`. | Only `0x338` introduces a new field read. |
| 13 | F-7f14f1ab6851 returns a value with zero low 32 bits. | The call follows gate 12 and precedes the remaining field reads. |
| 14 | 32-bit `0x030` equals 32-bit `0x368`. | Both reads occur within one value-construction expression; their relative order is not established. |
| 15 | 32-bit `0x034` equals 32-bit `0x36C`. | Relative operand-read order is not established. |
| 16 | 32-bit `0x038` equals 32-bit `0x370`. | Relative operand-read order is not established. |
| 17 | 32-bit `0x03C` equals 32-bit `0x374`. | Relative operand-read order is not established. |
| 18 | 32-bit `0x024` equals 32-bit `0x380`. | Relative operand-read order is not established. |
| 19 | Both terminal-pair comparisons succeed, as detailed below. | The complete `0x054`/`0x38C` comparison precedes the `0x050`/`0x388` comparison. |
| 20 | A newly read 32-bit `0x018` equals 32-bit `0x398`. | Relative operand-read order within this comparison is not established. |

Success at gate 20 produces `0`.

Equal-width 32-bit comparisons require matching 32-bit representations. Signed versus unsigned presentation does not change their equality result.

### Gate 4: precedence and signed promotion

Gate 4 is not a conventional masked-field comparison against a sentinel.

Under the supplied operator precedence, the signed 16-bit value at `0x214` is first compared for inequality with integer `65535`. Its promoted value lies between `−32768` and `32767`, so that inequality is always true. The resulting integer one is combined by bitwise AND with the previously read 32-bit value at `0x00C`. The gate requires the combined result to be zero.

Thus, under the supplied widths and signedness:

- An even value at `0x00C` passes gate 4.
- An odd value at `0x00C` fails gate 4.
- The stored value at `0x214` does not affect the Boolean outcome.

A 16-bit pattern of `0xFFFF` at `0x214` is interpreted as signed `−1`, not positive `65535`. It creates no special acceptance case.

The read at `0x00C` occurs in a separate statement before the expression containing `0x214`. The latter remains a source-level read operand even though its value cannot affect the outcome under these type assumptions. Whether a compiled implementation retains that redundant read is not established.

### Mixed-width constraints and retained values

Gate 5 compares the full 32-bit value at `0x028` with the zero-extended unsigned 16-bit value from `0x216`. Gate 11 then compares that same retained 32-bit value with `0x334`.

Acceptance therefore requires the value originally read from `0x028` to:

- Be in the inclusive range `0` through `65535`.
- Equal the value read from `0x216`.
- Equal the later value read from `0x334`.

The corresponding requirements apply to `0x02C`, `0x330`, and `0x338`.

Neither group truncates its 32-bit operand to 16 bits. Matching low 16 bits is insufficient. Gates 11 and 12 do not reread `0x028` or `0x02C`.

### Dependency boundary and return width

After gates 1–12 succeed, the function calls F-7f14f1ab6851 exactly once, supplying these arguments in order:

1. Base plus `0x058`.
2. Base plus `0x33C`.
3. Integer `0x20`, decimal `32`.

The supplied representation treats the result as 64 bits. Only its low 32 bits determine whether execution can proceed to gate 14.

If the low half is zero, the result container is replaced with newly read fields from `0x030` and `0x368`; those fields are then compared. The dependency result’s original high half does not participate in that field comparison.

If the low half is nonzero, an intermediate comparison of the dependency result’s halves is still evaluated, but cannot permit acceptance or cause the later field reads. The function rejects regardless of whether those halves happen to match.

The two addresses and integer `32` suggest a fixed-size comparison, but this is inferred intent only. The caller does not establish that the dependency compares bytes, reads exactly 32 bytes, or has conventional comparison-function semantics.

### Terminal pair and final reload

Gate 19 consists of two separately sequenced comparisons:

1. `0x054` against `0x38C`.
2. `0x050` against `0x388`.

The second comparison is evaluated even when the first comparison reports inequality, provided execution continues normally. Within each comparison, the relative order of its two field reads is not established.

Only if both comparisons succeed is gate 20 reached.

Gate 20 rereads `0x018`. It does not reuse the value read at gate 2. Consequently, acceptance does not require `0x20C` and `0x398` to match if the contents of `0x018` change between its two reads.

## Side effects

### Direct memory effects

The supplied body contains no explicit memory writes.

Its source-level reads follow the gated structure above. In particular, the following sequences are established by separate statements, not merely by the written position of operands:

- `0x008` before `0x20A`.
- The initial `0x018` before `0x20C`.
- `0x00C` before the expression containing `0x214`.
- `0x216` before `0x028`.
- `0x04C` before `0x32C`.
- `0x330` before `0x02C`.
- The complete `0x054`/`0x38C` comparison before the complete `0x050`/`0x388` comparison.

Unspecified operand ordering applies only where the relevant reads occur within one expression. It does not erase sequencing between separate statements.

These statements describe the supplied static representation. They do not establish an instruction trace, volatile-access contract, or guarantee that every apparent scalar read survives compilation unchanged.

### Effects across the dependency call

Any effects of F-7f14f1ab6851 occur between the pre-call checks and the post-call checks. The caller contains no visible save-and-restore operation for the supplied region or global state.

Therefore:

- Post-call comparisons use the values read after the dependency returns.
- Earlier successful comparisons are not repeated.
- The final read of `0x018` can observe a different value from the initial read.
- No preservation guarantee is established for caller memory, globals, ownership, or external resources.

The function supplies no visible synchronization or snapshot mechanism. Its result should not be interpreted as a comparison of one immutable object state without an external stability guarantee.

## Dependencies

### F-7f14f1ab6851

**Established caller-side requirements**

- Invoked at most once per invocation of F-2ea1d6803ffc.
- Reached only after gates 1–12 succeed.
- Receives the two interior addresses and integer `32` specified above.
- Must return normally for subsequent checks to occur.
- Must return zero in its low 32 result bits for acceptance.

**Unknown behavior — documented_only**

- Parameter types and validity requirements beyond the supplied argument values.
- Meaning of the integer `32`.
- Read and write extents.
- Ownership, retention, allocation, or release behavior.
- Effects on globals, supplied memory, or external state.
- Error handling and exceptional control flow.
- Result semantics beyond this caller’s low-half test.

No familiar library contract is substituted for these unknowns.

## Acceptance examples

These are conditional static consequences, not observed executions. They assume valid reached accesses and normal execution wherever required.

### Complete acceptance

If every listed equality succeeds, `0x00C` is even, both mixed-width groups match without truncation, and the dependency returns zero in its low 32 bits, the result is `0`.

All post-call equalities must hold when their reads occur, including the final reload of `0x018`.

### Earliest mismatch

If the unsigned 16-bit values at `0x008` and `0x20A` differ, the result is `1`. The read at `0x008` precedes the read at `0x20A`, with the intermediate address calculation between them. No later field read or dependency call is reached.

This path does not require later fields to be readable, but still requires validity of the accesses and arithmetic actually reached.

### Reversed mixed-width read order

If gates 1–4 succeed, gate 5 first reads unsigned 16-bit `0x216`, then reads 32-bit `0x028`.

If these yield zero and `65536`, respectively, gate 5 rejects. Their low 16 bits match, but their compared values differ.

The same source-level ordering principle applies at gate 10: `0x330` is read before `0x02C`.

### Least-significant-bit condition

If gates 1–3 succeed and `0x00C` contains `3`, gate 4 rejects regardless of the signed 16-bit value at `0x214`.

If `0x00C` contains `2`, gate 4 succeeds. This does not establish success of any later gate.

### Dependency result halves

A dependency result with high half one and low half zero permits gate 14.

A result with both halves one rejects. Equality between its halves does not override the nonzero low half.

### Terminal mismatch still permits the second comparison

If gates 1–18 succeed but `0x054` differs from `0x38C`, the comparison between `0x050` and `0x388` is still evaluated afterward. The final comparison involving `0x398` is not reached.

This describes source-level sequencing, not an observed machine-access trace.

### Mutation across the call

Suppose the initial reads of `0x018` and `0x20C` both yield `7`. If the dependency changes `0x018` to `9`, the final comparison requires the newly read value `9` to match `0x398`.

Acceptance remains conditional on all other gates. This example does not assert that the dependency actually performs such a mutation.

## Uncertainties

- **Established static evidence:** Direct field offsets and widths, source-level sequencing between separate statements, gate conditions, retained-value reuse, dependency arguments, final reload, and normal return values.
- **Inferred intent:** Consistency checking between related representations and a possible fixed-size comparison by the dependency.
- **Documented_only:** Field meanings, public object layout, byte order, concrete alignment requirements, and dependency implementation behavior.
- **Documented_only:** Exact compiled memory-access order, optimization effects, and whether the redundant read at `0x214` remains.
- **Documented_only:** Behavior for invalid addresses, insufficient memory, invalid address arithmetic, unsuitable alignment, or exceptional dependency execution.
- **Type-sensitive conclusion:** Gate 4’s least-significant-bit interpretation depends on the supplied signed 16-bit field and integer-promotion rules. Different underlying types require reassessment.
- **No snapshot guarantee:** Stability under concurrent or dependency-mediated mutation is not established.

## Evidence

This contract is derived from the supplied static function body and opaque dependency mapping. The dependency implementation, caller contracts, public structure definitions, compiled instruction trace, and runtime behavior were not provided.

The read-order distinctions follow separate source-level statements where present; unspecified ordering is retained for reads occurring within a single expression.

No runtime tests have been performed. The acceptance examples express conditional consequences of the static evidence, not measured results.
