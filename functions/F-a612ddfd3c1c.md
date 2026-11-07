<!-- Trusted provenance: F-a612ddfd3c1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a612ddfd3c1c

## Purpose

F-a612ddfd3c1c conditionally invokes two update dependencies and returns two 32-bit words packed into a 64-bit result. Branch selection depends on a 32-bit field at offset `0x15c` from the first input. Later calls depend on zero-versus-nonzero dependency results.

The supplied representation establishes two main paths:

- An initially nonzero field selects a path whose return words start as the first two inputs.
- An initially zero field selects a path that reads the field again, initializes four working words, and conditionally calculates differences with doubled adjustments.

**Inferred intent:** paired values and adjusted differences suggest coordinate-like or extent-like processing. Their meanings and units are not established. This contract uses neutral word and field descriptions instead.

All behavior below is based on static evidence. Acceptance examples are conditional expectations, not executed tests.

## Interface

### Inputs

The visible interface has four 32-bit integer inputs:

| Input | Observed role |
|---|---|
| A | Address-like base for field reads; argument to dependencies; initial low word in some paths. |
| B | Initial high return word on the initial-nonzero path; initial high word of another pair in the alternate initialization. |
| C | Used only when the first field observation is zero and the second is nonzero. |
| D | Subtraction operand on the initial-nonzero path; initial working word in the alternate initialization. |

A must permit every field read reached during the invocation. No null check, accessible-size check, alignment check, or address validation is visible.

The integer-typed representation of A does not establish a portable pointer interface. Native pointer width, integer-to-address conversion, address-overflow behavior, and calling convention remain **documented_only**.

### Directly accessed fields

| Offset relative to A | Width | Observed access |
|---|---:|---|
| `0x15c` | 32 bits | Initial branch selection; additional branch selection or dependency argument reads on specified paths. |
| `0x168` | 32 bits | Read only when the first predicate-like call on the initial-nonzero path returns nonzero. |

The `0x15c` branch tests distinguish zero from nonzero; they do not distinguish positive from negative values.

The function contains no explicit write to either field. Dependency calls may change them. A field value passed to a dependency is the value obtained at that particular read, not necessarily the value observed earlier.

### Working words and address exposure

This contract names four working words:

- **R-low** and **R-high**.
- **S-low** and **S-high**.

On the initial-nonzero path, only R-low and R-high are explicitly initialized and used for the return. The address of R-low may be passed to F-59fa3fd3efc5.

On the initial-zero path, all four words are explicitly initialized. In its alternate initialization branch, the address of S-low is passed to F-59fa3fd3efc5.

The supplied representation suggests this relative storage order:

| Displacement from S-low | Suggested word |
|---|---|
| `+0` | S-low |
| `+4` | S-high |
| `+8` | R-low |
| `+12` | R-high |

This is not a confirmed buffer declaration or access-permission contract. The dependency’s legal access extent and the authoritative storage layout are unresolved.

An address passed to F-59fa3fd3efc5 exposes storage belonging to the current invocation. Whether that dependency may retain the address, access additional words, or make the address available to later dependencies is unknown. Consequently, a dependency leaving the words unchanged during its own call does not establish their preservation during subsequent calls.

### Output

The output is a 64-bit bit pattern constructed from two 32-bit words:

- Bits 31–0 contain the selected low word.
- Bits 63–32 contain the selected high word.

The initial-nonzero path selects the final R-low and R-high. The initial-zero path selects the final S-low and S-high.

Packing preserves each word’s bit pattern. It does not sign-extend the low word. The signed interpretation of the resulting 64-bit value is not established.

## Behavior

### Initial field observation

The function first reads the 32-bit field at A plus `0x15c`.

- A nonzero observation selects the initial-nonzero path.
- A zero observation selects the initial-zero path.

The latter path performs a separate read of the same field. The representation does not justify merging these reads or treating them as one snapshot. Whether different observations are possible under valid surrounding execution rules remains unresolved.

### Initial-nonzero path

The function initializes R-low to A and R-high to B.

It then invokes F-3d16c4397c76. **No explicit argument is shown for this invocation.** The same dependency is shown receiving A on the other main path; this discrepancy is not resolved here.

If F-3d16c4397c76 returns nonzero:

1. The function reads the 32-bit field at A plus `0x168`.
2. It invokes F-065fd89ae7c4 with A and that field value.

The `0x168` read occurs after the predicate-like call returns. Changes made by that call can therefore affect the value read. A zero predicate result skips both this read and this update call.

The function next invokes F-5e490153287b with A.

If that result is nonzero:

1. The function reads the current 32-bit field at A plus `0x15c`.
2. It invokes F-59fa3fd3efc5 with that value and the address of R-low.
3. After the call returns, it obtains the then-current R-high and computes D minus R-high.
4. It invokes F-d25031d00f00 with A and the computed difference.

There is no fresh nonzero test of the field value supplied to F-59fa3fd3efc5. That value may be zero even though the initial observation was nonzero.

If F-5e490153287b returns zero, both F-59fa3fd3efc5 and F-d25031d00f00 are skipped.

The function finally packs the then-current R-low and R-high. It does not explicitly restore their initial values after passing the working-storage address. In particular, the return words are observed after F-d25031d00f00 returns when that call occurs. Their values cannot be fixed from F-59fa3fd3efc5’s immediate effects alone if later access through an escaped address is permitted.

### Initial-zero path: second observation and initialization

After the initial field observation yields zero, the function reads the field at `0x15c` again.

If this second observation is zero, it initializes all four working words to zero. This branch does **not** call F-59fa3fd3efc5.

If the second observation is nonzero, it initializes:

- R-low to C.
- R-high to D.
- S-low to A.
- S-high to B.

It then performs a third read of the field at `0x15c` and invokes F-59fa3fd3efc5 with that freshly read value and the address of S-low.

The third observation is not tested for nonzero before being passed. It need not equal the second observation under a model permitting changes between reads.

The zero-then-nonzero branch requires differing observations. It is unreachable under an assumption that the field remains stable across those reads, but such stability is not established unconditionally.

### Initial-zero path: first conditional update

After either initialization branch completes, the function invokes F-3d16c4397c76 with A.

If its result is nonzero:

1. The function computes R-low minus S-low using the working-word values available after the predicate-like call.
2. It retains that difference.
3. It invokes F-163aa6442bb1 with A.
4. It adds twice that dependency’s result to the retained difference.
5. It invokes F-065fd89ae7c4 with A and the resulting argument.

The difference is computed before F-163aa6442bb1 is invoked. Any working-word changes during that call do not cause the already-computed difference to be recomputed.

Conversely, initialization values alone do not determine the difference if an intervening dependency can modify the relevant words. This matters particularly after the address of S-low has escaped: preservation must cover later calls, not merely the initial F-59fa3fd3efc5 invocation.

A zero predicate-like result skips both F-163aa6442bb1 and F-065fd89ae7c4 for this update.

### Initial-zero path: second conditional update

The function next invokes F-5e490153287b with A.

If its result is nonzero:

1. The function computes R-high minus S-high using the values available after this predicate-like call.
2. It retains that difference.
3. It invokes F-163aa6442bb1 with A.
4. It adds twice that invocation’s result to the retained difference.
5. It invokes F-d25031d00f00 with A and the resulting argument.

This difference is computed after the entire first conditional-update sequence. Earlier dependency effects may therefore affect it wherever access to the relevant storage is permitted.

The two adjustment calls are independent. Their results need not agree. As with the first difference, later word changes do not alter the retained second difference.

A zero result from F-5e490153287b skips both calls in this sequence.

Finally, the function packs the then-current S-low and S-high. These are final-read values, not necessarily their initialization values or their values immediately after F-59fa3fd3efc5.

### Arithmetic and observation boundaries

The visible calculations use 32-bit integer operands:

- D minus R-high.
- R-low minus S-low.
- R-high minus S-high.
- Twice a result from F-163aa6442bb1.
- A retained difference plus the doubled result.

For cases where each intermediate and final result fits the signed 32-bit range, these mathematical expressions determine the corresponding update arguments.

No saturation, clamping, overflow check, or arithmetic error return is visible. The authoritative behavior of overflowing signed expressions is not established by the supplied representation and remains **documented_only**.

The call boundaries and placement of calculations are established. The relative read order of the two working-word operands within an individual subtraction is not established. The contract does not assign a coherent snapshot guarantee to accesses that might be affected by concurrent activity.

## Side effects

The function directly performs field reads, working-storage initialization, dependency calls, and final packing. It contains no explicit allocation, deallocation, persistent registration, or object-field write.

That observation does not limit dependency behavior:

- Dependencies may change state reachable through A.
- Dependencies may modify globals.
- F-59fa3fd3efc5 may modify storage through its address argument.
- If permitted by the unknown dependency contracts, retained working-storage addresses may enable changes during later calls.
- Dependency ownership, allocation, retention, and release behavior are not established.

No ownership transfer or memory-release guarantee can be inferred merely from the call signatures. Retention beyond this invocation would require a separate lifetime contract; none is supplied.

No explicit error status, validation branch, or rollback is visible. If a field access fails or a dependency does not return normally, subsequent calls and a final packed return are not guaranteed by the visible control flow.

## Dependencies

| Opaque ID | Observed use |
|---|---|
| F-3d16c4397c76 | Its zero/nonzero result gates the first update. Shown without an explicit argument on the initial-nonzero path and with A on the initial-zero path. |
| F-065fd89ae7c4 | Receives A and either the `0x168` field value or an adjusted low-word difference. Its return value is not consumed. |
| F-5e490153287b | Receives A. Its zero/nonzero result gates the second update sequence. |
| F-59fa3fd3efc5 | Receives a fresh `0x15c` field value and a working-storage address. Its return value is not consumed. |
| F-d25031d00f00 | Receives A and either D minus R-high or an adjusted high-word difference. Its return value is not consumed. |
| F-163aa6442bb1 | Receives A. Its visible 32-bit result is doubled and added to a previously retained difference. Invoked separately for each enabled update on the initial-zero path. |

“Predicate-like” describes how a result controls this function. “Update” describes the role of a call here. Neither label establishes purity, state preservation, or a complete dependency contract.

## Acceptance examples

These examples are conditional static acceptance cases. All reached field accesses must be valid, and all mentioned calls must return normally.

### Initial nonzero field; both predicates false

Assume the initial field observation is nonzero, F-3d16c4397c76 returns zero, and F-5e490153287b returns zero. Assume the calls leave R-low and R-high unchanged.

Expected behavior:

- No `0x168` read.
- No F-065fd89ae7c4, F-59fa3fd3efc5, or F-d25031d00f00 call.
- Return low word A and high word B.

For A = `0x00001000` and B = 7, the result is `0x0000000700001000`.

### Initial nonzero field; second sequence enabled

Assume F-5e490153287b returns nonzero. Suppose F-59fa3fd3efc5 is permitted to set R-low to 12 and R-high to 9, does so, and every subsequent dependency leaves those words unchanged. Let D be 20.

Expected behavior:

- F-59fa3fd3efc5 receives the field value read immediately before its invocation, even if that value is zero.
- F-d25031d00f00 receives A and 11.
- The result is `0x000000090000000c`.

The subtraction uses R-high after the address-passing call, not necessarily B.

### Both initial observations zero; both updates enabled

Assume both field observations are zero, both predicate-like results are nonzero, and every dependency leaves all four working words unchanged.

If the successive F-163aa6442bb1 results are 3 and 5:

- F-59fa3fd3efc5 is not called.
- F-065fd89ae7c4 receives A and 6.
- F-d25031d00f00 receives A and 10.
- The packed result is zero.

### Field changes between the first two observations

Assume the first field observation is zero and the second is nonzero. Let A = `0x1000`, B = 10, C = `0x1014`, and D = 40.

Assume F-59fa3fd3efc5 **and every subsequent dependency** leave all four working words unchanged throughout their calls. Both updates are enabled, and each adjustment result is 2.

Expected behavior:

- The first difference is 20, so F-065fd89ae7c4 receives A and 24.
- The second difference is 30, so F-d25031d00f00 receives A and 34.
- The returned low word is `0x1000`, and the high word is 10.
- The packed result is `0x0000000a00001000`.

Preservation only during F-59fa3fd3efc5 would not establish these results. If permitted retained-address access lets the subsequent predicate change R-low before the first difference, that difference changes. Once calculated, however, it remains retained across the following adjustment call.

### Adjustment call changes a word after difference capture

Assume the alternate initialization branch is reached and the first predicate returns nonzero. At the first difference calculation, let R-low be 30 and S-low be 10. Suppose F-163aa6442bb1 then changes R-low to 100 through access permitted by its dependency contract and returns 2.

F-065fd89ae7c4 receives A and 24, not 94. The captured difference is 20; the later word change does not trigger recomputation. Final return values still depend on the words observed at return.

### High return word has its sign bit set

If the selected final low word is 1 and high word is `0xffffffff`, the packed result is `0xffffffff00000001`. This establishes bit placement only.

## Uncertainties

The following remain **documented_only**:

- Authoritative calling convention and the inconsistent visible arguments to F-3d16c4397c76.
- Native address width, conversion rules, alignment requirements, and address arithmetic.
- Complete object layout and field meanings.
- Authoritative working-storage layout and permitted dependency access extent.
- Working-address retention, sharing, and lifetime rules.
- Whether repeated field reads may legitimately disagree.
- Synchronization, volatility, concurrency, and snapshot guarantees.
- Overflow semantics outside representable signed 32-bit calculations.
- Dependency state preservation, ownership, allocation, and release behavior.
- Invalid-address, exception, fault, and nonreturning-call behavior.
- Semantic units of inputs, differences, and adjustment results.

## Evidence

**Established static evidence:** the supplied representation shows the field-read branches, working-word assignments, dependency-call order, address arguments, retained differences, arithmetic expressions, and final two-word packing described here.

**Inferred intent:** coordinate-like or extent-like adjustment and predicate/update roles. These interpretations do not replace the neutral contract.

**Evidence limits:** the supplied representation warns that processing was restarted to delay dead-code elimination involving stack state. Apparent storage layout and the unusual repeated-read branch therefore warrant particular caution. Complete dependency contracts and authoritative source-language semantics are unavailable.

No implementation was executed, no dependencies were inspected, and no runtime tests were performed.
