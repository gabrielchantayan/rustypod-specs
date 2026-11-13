<!-- Trusted provenance: F-d9da7eba6922; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9da7eba6922

## Purpose

F-d9da7eba6922 processes a request consisting of a target, property, and value against a context obtained from F-7fba0909c452.

Established behavior:

- Context acquisition precedes request dispatch.
- A null context ends processing immediately.
- An accepted request performs one local state write: either one byte or one 32-bit word.
- A rejected request with a non-null context calls F-375c8199e7c2.
- No explicit result distinguishes acceptance, rejection, or absence of a context.

The token comparisons and compact stored values suggest an enumeration-setting interface. That is **inferred intent**. No domain-specific meaning is assigned to the targets or fields.

External values are observed at separate points during dispatch. Equality failure against one observation does not establish inequality against a later observation of the same external storage. The behavior below preserves that distinction.

## Interface

### Inputs and result

| Input | Representation | Role |
|---|---|---|
| Target | 32-bit integer | Selects a request family |
| Property | 32-bit integer | Selects a field or field group |
| Value | 32-bit integer | Supplies an enumeration token, raw word, or truth value |

Ordered comparisons in the supplied representation are signed. Equality compares the corresponding 32-bit values.

The function has no explicit return value. Dependency failures or nonlocal exits are not characterized by the supplied evidence.

### Context and destinations

Let **C** be the non-null context address returned by F-7fba0909c452. Before target dispatch, the function forms a base address at C + `0x944`.

Each accepted path reads a 32-bit signed selection index at C + `0x24C`. For that observation **n**, define:

**O = C + `0x944` + n × `0x24`**

| Destination | Width | Role |
|---|---:|---|
| O + `0x25` | 1 byte | First enumeration field |
| O + `0x26` | 1 byte | Second enumeration field |
| O + `0x27` | 1 byte | Third enumeration field |
| O + `0x28` through O + `0x2A` | 1 byte each | Group A destinations |
| Property-dependent addresses | 1 byte | Groups B and C |
| O + `0x31` through O + `0x33` | 1 byte each | Group D destinations |
| O + `0x34` | 4 bytes | Raw-value field |
| O + `0x38` | 4 bytes | Raw-value field |
| O + `0x42` | 1 byte | Boolean field |

The offsets exceed the `0x24` selection stride. They describe the supplied address expressions, not a proven array-of-records layout. Allocation boundaries, field disjointness, and valid index ranges are unknown.

### External observations

The following symbols identify external storage by its role, not by a presumed constant value.

| Symbol | Role |
|---|---|
| **P** | Main property boundary and group A/B token source |
| **Q** | Raw-field property selector and lower-dispatch boundary |
| **R** | Token source for the first three enumeration fields |
| **S** | Token source and basis of a derived token |
| **T** | Group C/D token source |
| **U** | Later property selector and boundary |
| **D** | Alternate-family diagnostic argument |

Each reached reference denotes its own observation unless explicitly described as retained. In particular, repeated observations of P, Q, S, T, or U need not agree. A comparison table must not be interpreted as using an entry-time snapshot.

The observations participate in 32-bit operations in the supplied representation. Actual external values, declarations, synchronization, and immutability are unavailable.

## Behavior

### Entry and target selection

F-7fba0909c452 is called once without explicit arguments.

If it returns zero, processing ends without local dispatch, an index read, a state write, or a call to F-375c8199e7c2.

For a non-null context, the base address described above is formed and the target determines further behavior:

| Target | Behavior |
|---|---|
| `0x2300` | Property and value dispatch described below |
| `0x8861` | Boolean-family handling |
| Any other value | Call F-375c8199e7c2 with C and `0x500` |

For target `0x8861`, D is read and retained **before** Property is checked.

- Property `0x8862` accepts every Value and writes one byte at O + `0x42`: zero becomes `0`; every nonzero value becomes `1`.
- Every other property calls F-375c8199e7c2 with C and the retained D.

### Target `0x2300`: property routing

The following stages describe comparison outcomes and precedence. Each stage is reached only after the preceding alternative has failed.

| Stage | Comparison or outcome | Route |
|---|---|---|
| 1 | Property equals a first observation of P plus `0x11` | Group B |
| 2 | Stage 1 fails; Property is less than a fresh observation of P plus `0x11` | Lower-property routing |
| 3 | Stage 2 comparison is false; Property equals an observation of U | Group C |
| 4 | Stage 3 equality fails; Property is less than a fresh observation of U | Literal middle routing |
| 5 | Stage 4 comparison is false | Literal final routing |

Literal middle routing recognizes:

- `0x8589` and `0x858A`: group B.
- `0x8590` and `0x8591`: group C.

Literal final routing recognizes `0x8598`, `0x8599`, and `0x859A` as group D.

An unrecognized property is rejected. Once a handler is selected, an unrecognized Value is also rejected; dispatch does not resume at a later property alternative.

Under stable, non-overflowing external values, stages 3–5 can be simplified using the usual strict-order consequences of preceding equality failures. That simplification does **not** apply to changing observations. For example, stage 5 does not generally establish Property greater than the U observed there: Property may equal that later U after failing equality against an earlier U.

### Lower-property routing

This routing is entered only after the stage-2 comparison succeeds.

First, Property is compared for equality with an observation of Q:

- Equality selects the four-byte raw write at O + `0x34`.
- On inequality, Q is observed again for the ordered comparison **Q < Property**.

The ordered comparison controls the remaining routes:

| Outcome of the second Q comparison | Eligible handling |
|---|---|
| Q < Property is true | `0x8580`, `0x8581`, or `0x8582` selects group A; other properties reject |
| Q < Property is false | The raw-field and first-three-field routes below |

The false branch establishes **Property ≤ the second Q observation**, not necessarily Property < Q. The failed equality used an earlier observation.

Within this false branch:

- Property `0x0D1C` selects the four-byte raw write at O + `0x38`.
- Property `0x2200` selects the first enumeration field.
- For every other property, S is read and **K = that observation of S + `0x8D`** is computed and retained.
  - Property `0x8571` selects the second enumeration field.
  - Property `0x8572` selects the third enumeration field.
  - Other properties reject, but only after the S observation and K calculation.

Both raw writes preserve all 32 input bits and perform no enumeration validation.

If Q is stable, failure of equality followed by failure of Q < Property does establish Property < Q. This is a conditional simplification, not the general branch contract.

### First enumeration field

This handler writes one byte at O + `0x25`.

| Condition, with earlier alternatives excluded | Stored byte |
|---|---:|
| Value equals `0x2100` | 4 |
| Value < `0x2101`, and Value equals `0x0104` | 3 |
| Value < `0x2101`, and Value equals `0x0BE2` | 2 |
| Value < `0x2101`, and Value equals the reached R observation | 1 |
| Value ≥ `0x2101`, and Value equals `0x2101` | 0 |
| Value ≥ `0x2101`, and Value equals `0x8570` | 5 |

Other values reject. R is observed only if both earlier comparisons in the lower-value branch fail. Its mapping is therefore range-limited and subordinate to the literal-token checks.

### Second enumeration field

This handler writes one byte at O + `0x26`. K is the retained value computed before this property was selected.

| Condition, with earlier alternatives excluded | Stored byte |
|---|---:|
| Value equals K | 3 |
| K < Value, and Value equals `0x8575` | 4 |
| K < Value, and Value equals `0x86AE` | 6 |
| K < Value, and Value equals `0x86AF` | 7 |
| K < Value is false, and Value equals `0x0104` | 2 |
| K < Value is false, and Value equals the reached R observation | 0 |
| K < Value is false, and Value equals `0x2100` | 1 |
| K < Value is false, and Value equals a fresh S observation | 5 |

Other values reject.

Here, unlike repeated external comparisons, the failed equality and ordered comparison use the **same retained K**. Consequently, the final four rows require Value < K. The later S comparison nevertheless uses a fresh observation; it need not equal the S from which K was derived.

### Third enumeration field

This handler writes one byte at O + `0x27`. It also receives the previously retained K.

First, Value is compared with a fresh S observation. Equality stores `5`. Otherwise, S is observed again for the ordered comparison **S < Value**.

| Outcome after the initial equality fails | Value checks, in precedence order | Stored byte |
|---|---|---:|
| Second S observation < Value | Value equals K | 3 |
| Same true branch | Value equals `0x8575` | 4 |
| Second S observation < Value is false | Value equals `0x0104` | 2 |
| Same false branch | Value equals the reached R observation | 0 |
| Same false branch | Value equals `0x2100` | 1 |

Other values reject.

The false branch requires **Value ≤ the second S observation**, not necessarily Value < S. Equality may have failed against the earlier observation and still hold against the later one. Thus Value `0x0104` can store `2` when the second S observation is exactly `0x0104`.

With stable S, the earlier failed equality makes this branch equivalent to Value < S. There are no dedicated `0x86AE` or `0x86AF` cases here, although either bit pattern could coincide with an external token or K.

### Enumeration groups A and B

Both groups use the following token precedence:

| Check, in order | Stored byte |
|---|---:|
| Value equals `0x1702` | 0 |
| Value equals a reached P observation minus 1 | 1 |
| Value equals a fresh P observation | 2 |
| Value equals `0x8578` | 3 |

Other values reject. A match stops later checks and their external reads.

| Group | Destination |
|---|---|
| A | O + `0x28` + (Property − `0x8580`) |
| B | O + `0x2B` + (Property − `0x8588`) |

Group A properties yield offsets `0x28`–`0x2A`. Group B’s literal properties yield offsets `0x2C` and `0x2D`.

A property selected through the initial P-derived equality uses its actual input value in the same group B address formula. It is not recalculated from a later P observation.

### Enumeration group C

| Check, in order | Stored byte |
|---|---:|
| Value equals `0x0300` | 0 |
| Value equals `0x0301` | 1 |
| Value equals a reached T observation minus 1 | 2 |
| Value equals a fresh T observation | 3 |

Other values reject.

The destination is:

**O + `0x2E` + (Property − `0x8590`)**

Literal properties `0x8590` and `0x8591` yield offsets `0x2E` and `0x2F`. Selection by equality with U uses the actual Property in this formula; the destination is not necessarily one of those two offsets.

### Enumeration group D

Only the reachable literal properties `0x8598`, `0x8599`, and `0x859A` enter this group.

| Check, in order | Stored byte |
|---|---:|
| Value equals a reached T observation minus 1 | 2 |
| Value equals a fresh T observation | 3 |

Other values reject. The destination is O + `0x31` + (Property − `0x8598`), producing offsets `0x31`–`0x33`.

The group has no dedicated `0x0300` or `0x0301` cases.

### Rejection

Every rejected target-`0x2300` request calls F-375c8199e7c2 with C and `0x500`. This includes a recognized property whose value fails its handler’s checks.

`0x500` is an established argument, not an established public error-code meaning.

## Side effects

### Observable ordering

For an accepted request, the local sequence is:

1. Context acquisition.
2. Base-address formation and applicable dispatch, including reached external observations.
3. One selected-index read.
4. One destination write.
5. Return without calling F-375c8199e7c2.

No destination-field read or read-modify-write is shown. Enumeration and boolean writes affect one byte; raw writes affect four bytes.

Rejected requests perform no local selected-index read or destination write. They may still read external values, and the rejection dependency may have its own effects. The context-acquisition dependency may also have effects even when it returns null.

### Ownership and validity

No local allocation, release, reference-count update, or explicit ownership transfer appears. The returned context is used directly. This does not establish ownership behavior inside either dependency.

Successful execution requires readable index storage and writable storage covering the computed destination and write width. No local index-sign, bounds, allocation-extent, or alignment checks are present.

### Arithmetic

The supplied representation uses 32-bit integer operations, including:

- P + `0x11`, P − 1, S + `0x8D`, and T − 1.
- Selection-index multiplication by `0x24`.
- Base, property-relative, and destination-address additions.

The equations in this contract describe non-overflowing address relationships. Portable signed-overflow semantics, machine wrapping, pointer provenance, alignment tolerance, endianness, and atomicity are **documented_only**. They are not inferred from the displayed expressions.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-7fba0909c452 | Called once at entry without explicit arguments; returns the context address or zero | Ownership, allocation, global mutation, synchronization, failure mechanisms |
| F-375c8199e7c2 | Called on rejection with C and either `0x500` or retained D | Diagnostic meaning, context mutation, memory ownership effects, exceptions or nonlocal exits |

Neither dependency is assumed to preserve globals or retain, release, or transfer memory ownership.

## Acceptance examples

These are conditional static deductions, not executed tests. Successful-write examples assume valid storage and non-overflowing arithmetic.

1. **Null context:** If F-7fba0909c452 returns zero, no local dispatch, selected-index read, state write, or rejection call follows.

2. **Boolean normalization:** Target `0x8861`, Property `0x8862`, and Value `−7` write byte `1` at O + `0x42`. D is read even though no rejection call follows.

3. **Retained diagnostic:** For target `0x8861` and any other property, the rejection call receives the D observed before the property check, not a later replacement value.

4. **Raw word:** If lower-property routing is reached and Property equals the first Q observation, Value `−1` writes all-one bits at O + `0x34`.

5. **Changing Q reaches equality in the later false branch:** Suppose lower-property routing is reached with Property `0x0D1C`. The first Q observation is `0x2200`, so equality fails. The second is `0x0D1C`, so Q < Property is false. The request writes the raw Value at O + `0x38`. Property is equal to, not less than, the second Q observation.

6. **Changing S permits the third-field byte-2 write:** Suppose the third enumeration field is reached with Value `0x0104`. Its initial equality observes S as `0x2000` and fails. Its ordered comparison observes S as `0x0104`; S < Value is false. The function writes `2` at O + `0x27`.

7. **Group A destination:** If lower routing reaches the true Q < Property branch with Property `0x8581`, Value `0x1702` writes `0` at O + `0x29`.

8. **Selector precedence:** If lower routing is reached with Property `0x2200` and the first Q observation is `0x2200`, the request selects the raw write at O + `0x34`, not the first enumeration field.

9. **Retained-K restriction:** In the second field, if retained K exceeds `0x86AE`, Value `0x86AE` cannot reach its dedicated byte-6 case. It rejects unless an eligible comparison in the other branch accepts it.

10. **Group D restriction:** If group D is reached with Value `0x0300`, and neither reached T-derived comparison matches, rejection occurs without a local index read or state write.

## Uncertainties

- **documented_only:** External values, their declarations, mutability, and synchronization are unavailable.
- **documented_only:** The cause and feasibility of changing observations are unknown. The contract describes their branch consequences, not a guarantee that any particular concurrent execution is defined.
- **documented_only:** Context allocation extent, valid indices, field boundaries, and cross-selection overlap are unknown.
- **documented_only:** Overflow, address validity, alignment, endianness, atomicity, and dependency failure behavior remain unresolved.
- **Inferred intent:** Enumeration translation and selected-state mutation are suggested by the comparisons and writes; domain meaning is not established.

## Evidence

Established claims derive from the supplied static representation: call placement, comparison order, separate external references, retained derived values, address expressions, write widths, stored constants, early returns, and rejection arguments.

In particular, Q equality and Q ordering use separate external observations. The third field’s S equality and S ordering also use separate observations. Neither pair supports an unconditional strict-inequality conclusion after both comparisons fail. By contrast, the second field’s equality and ordering both use retained K.

No runtime tests have been performed. No external storage contents, dependency implementations, surrounding layout declarations, or machine-level overflow evidence were supplied.
