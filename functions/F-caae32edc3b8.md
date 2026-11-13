<!-- Trusted provenance: F-caae32edc3b8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-caae32edc3b8

## Purpose

F-caae32edc3b8 conditionally updates one field in a context-selected storage region. Category and selector inputs determine whether the supplied integer is stored directly, normalized to a Boolean, or translated into a byte-sized encoding.

Unsupported combinations invoke F-375c8199e7c2. The evidence does not establish that dependency’s purpose, error-reporting policy, side effects, or ability to return.

Six externally backed integer values participate in dispatch and translation. Their immutability is **not established**. Separate source-visible reads must therefore remain separate observations: a later read need not equal an earlier read of the same backing location.

The established operation is dispatch followed by a store or dependency invocation. Application-specific meanings for the fields remain inferred intent.

## Interface

### Inputs and result

| Input | Role |
|---|---|
| Value | Integer payload to store or translate. |
| Category | Main dispatch family. |
| Selector | Destination and interpretation selector. |
| Context | Base address for the index and destination storage. |

Only Value appears as an explicit parameter. The other inputs arrive through incoming machine state; their portable calling convention is unknown.

There is no explicit return value or local status result. An accepted path performs one destination write and returns. A rejected path invokes F-375c8199e7c2 and returns only if that dependency returns normally.

Integer values, comparisons, and integer accesses are represented as signed 32-bit operations. Encoded destinations are 8 bits. These widths describe the supplied representation; independent ABI confirmation is unavailable.

### Addressing and storage

For an accepted write:

- **Index** is the signed integer read from Context + `0x24C`.
- **Selected base** is Context + `0x944` + Index × `0x24`.
- The destination is Selected base plus the field offset specified below.

The Context + `0x944` base expression is evaluated before category dispatch. This is address arithmetic, not a shown memory read.

The `0x24` stride is not a proven record size. Literal destination offsets extend through `0x42`, and selector-derived offsets can be outside the literal ranges.

### Externally backed inputs

The names below identify backing locations, not immutable constants or additional explicit arguments.

| Location | Uses |
|---|---|
| R1 | Top-level selector boundary; two translation comparisons. |
| R2 | Direct-store selector equality; subsequent selector ordering. |
| R3 | Conditional translation comparisons. |
| R4 | Cached derived boundary; later independent comparisons. |
| R5 | Two translation comparisons. |
| R6 | Upper-partition equality; subsequent ordering. |

Each reference described as a **fresh read** obtains a separate source-visible observation. A **cached** quantity retains the value already obtained and is not refreshed.

All arithmetic descriptions below concern executions in which the represented operations are valid without overflow. They do not establish wrapping, trapping, or machine-specific overflow behavior.

## Behavior

### Read and dispatch rules

Rules are ordered. A successful earlier match prevents later comparisons and their associated reads.

A failed match proceeds only as described. Unless a rule specifies another result, rejection invokes F-375c8199e7c2 without a local destination write.

Read ordering here is the source-visible evaluation structure. It does not claim an independently verified instruction-level memory trace.

### Categories other than `0x2300`

Category `0x8861` with Selector `0x8862` writes one byte at offset `0x42`:

- Value zero produces `0`.
- Any nonzero Value, including a negative value, produces `1`.

Every other combination outside Category `0x2300` is rejected. These paths do not read R1–R6.

### Category `0x2300`: initial partition

The function first freshly reads R1, adds `0x11`, and compares Selector for equality. Equality selects translation family A.

Only if equality fails does it freshly read R1 again, add `0x11`, and test whether Selector is less than this second boundary:

- A true comparison selects the lower partition.
- A false comparison selects the upper partition.

The two boundary observations are independent. The upper partition means Selector is greater than or equal to the **second** boundary and unequal to the **first** boundary. It does not unconditionally mean Selector is greater than one stable boundary.

### Lower partition

R2 is freshly read for equality with Selector.

- Equality writes Value unchanged as a full integer at offset `0x34`.
- On inequality, R2 is freshly read again for the comparison “R2 is less than Selector.”

If that second comparison is true, only Selectors `0x8580`, `0x8581`, and `0x8582` are accepted, through family B. All other selectors are rejected.

If that comparison is false, the residual lower branch handles:

| Selector | Operation |
|---|---|
| `0x0D1C` | Store Value unchanged as a full integer at offset `0x38`. |
| `0x2200` | Mode translation at offset `0x25`. |
| Other | Read R4 and cache its value plus `0x8D`, then examine the extended selectors. |

With changing R2 observations, the residual branch establishes Selector ≤ the second R2 observation, not necessarily strict inequality. Equality failed against the first observation only.

#### Mode translation

For Selector `0x2200`, the ordered rules are:

| Condition | Stored byte at `0x25` |
|---|---:|
| Value equals `0x2100` | 4 |
| Otherwise, Value < `0x2101` and equals `0x0104` | 3 |
| Otherwise, within that lower range, Value equals `0x0BE2` | 2 |
| Otherwise, within that lower range, Value equals a fresh R3 read | 1 |
| Value equals `0x2101` | 0 |
| Value > `0x2101` and equals `0x8570` | 5 |

Other values are rejected. R3 is read only after both lower-range literal matches fail. No R3 read occurs for the boundary or upper-range cases.

#### Extended-selector preparation

After the residual branch excludes `0x0D1C` and `0x2200`, it freshly reads R4 and caches:

**Derived boundary = that R4 observation + `0x8D`.**

This read and addition occur **before** determining whether Selector is `0x8571` or `0x8572`. Unsupported selectors reaching this point therefore also perform this R4 read before rejection.

The derived boundary remains cached throughout the selected extended translation.

#### Selector `0x8571`

The destination is byte offset `0x26`.

1. Value equal to the cached derived boundary stores `3`.
2. Value greater than that boundary is checked against literals in order:
   - `0x8575` stores `4`.
   - `0x86AE` stores `6`.
   - `0x86AF` stores `7`.
   - Anything else is rejected.
3. Value less than that boundary is checked in order:
   - `0x0104` stores `2`.
   - Equality with a fresh R3 read stores `0`.
   - `0x2100` stores `1`.
   - Equality with a **new R4 read** stores `5`.
   - Anything else is rejected.

The final R4 observation need not equal the observation used to construct the cached boundary. It is read only if the preceding lower-branch comparisons fail.

#### Selector `0x8572`

The destination is byte offset `0x27`. After the earlier derived-boundary read:

1. R4 is freshly read for equality with Value. Equality stores `5`.
2. If equality fails, R4 is freshly read **again** for the ordering comparison “R4 is less than Value.”
3. If that comparison is true:
   - Value equal to the previously cached derived boundary stores `3`.
   - Otherwise, `0x8575` stores `4`.
   - Otherwise reject.
4. If that comparison is false:
   - `0x0104` stores `2`.
   - Otherwise, equality with a fresh R3 read stores `0`.
   - Otherwise, `0x2100` stores `1`.
   - Otherwise reject.

This path can observe R4 three times: derived-boundary construction, equality, and ordering. The final branch establishes Value ≤ the ordering observation; it does not establish Value is strictly below the earlier equality observation.

There are no independent `0x86AE` or `0x86AF` acceptance rules here. Either literal could nevertheless match a separately specified dynamic condition, such as equality with the cached derived boundary.

### Upper partition

R6 is freshly read for equality with Selector. Equality selects family C.

On inequality, R6 is freshly read again for “R6 ≤ Selector.”

- If true, only Selectors `0x8598`, `0x8599`, and `0x859A` are accepted, through family D.
- If false:
  - Selectors `0x8589` and `0x858A` select family A.
  - Otherwise, Selectors `0x8590` and `0x8591` select family C.
  - Other selectors are rejected.

These literal groups remain subject to all preceding dispatch conditions. Failure of equality against the first R6 observation does not establish inequality against its later observation.

### Translation families

Every successful family writes one byte.

| Family | Destination offset | Literal selector offsets |
|---|---|---|
| A | Selector − `0x855D` | `0x8589` → `0x2C`; `0x858A` → `0x2D` |
| B | Selector − `0x8558` | `0x8580` → `0x28`; `0x8581` → `0x29`; `0x8582` → `0x2A` |
| C | Selector − `0x8562` | `0x8590` → `0x2E`; `0x8591` → `0x2F` |
| D | Selector − `0x8567` | `0x8598` → `0x31`; `0x8599` → `0x32`; `0x859A` → `0x33` |

Offsets for the dynamic selector-equality entries into A and C remain selector-derived. No canonical adjacent selector value is assumed.

#### Families A and B

Apply the following ordered comparisons:

| Match | Stored byte |
|---|---:|
| Value equals `0x1702` | 0 |
| Otherwise, Value equals a fresh R1 read minus 1 | 1 |
| Otherwise, Value equals another fresh R1 read | 2 |
| Otherwise, Value equals `0x8578` | 3 |

No match means rejection.

The literal `0x1702` match avoids both translation-time R1 reads. A successful second comparison avoids the third comparison’s read. Translation-time observations are independent of any R1 observations used during initial dispatch.

#### Family C

| Ordered match | Stored byte |
|---|---:|
| Value equals `0x0300` | 0 |
| Otherwise, Value equals `0x0301` | 1 |
| Otherwise, Value equals a fresh R5 read minus 1 | 2 |
| Otherwise, Value equals another fresh R5 read | 3 |

No match means rejection. Neither R5 read occurs when either literal matches.

#### Family D

Value is first compared with a fresh R5 read minus 1; equality stores `2`. Only after failure is Value compared with another fresh R5 read; equality stores `3`. Otherwise the request is rejected.

### Stable-backing interpretation

If every read of each backing location returns the same value throughout the invocation, the ordered comparisons can be summarized using one stable value per location. Under that **additional assumption**:

- Lower residual dispatch is strictly below the stable R2 value.
- Upper-partition literal groups below R6 are strictly below its stable value.
- The second extended translation’s final branch is strictly below stable R4.
- The family mappings resemble fixed enumeration tables.

These simplifications are conditional, not unconditional behavior.

## Side effects

### Accepted paths

An accepted path performs any applicable backing reads described above, then reads Index once from Context + `0x24C`, computes the destination, and performs one byte or full-integer write.

No destination read or read-modify-write is shown. Direct integer stores preserve the supplied integer payload. No accepted path calls F-375c8199e7c2.

The source shows no allocation, release, ownership transfer, synchronization, or write to R1–R6. This does not establish that their backing storage is immutable or inaccessible to other actors.

### Rejected paths

Rejection performs no shown context-index read or destination write before the dependency call. It can nevertheless perform backing reads, including the unconditional extended-preparation R4 read for unsupported selectors reaching that branch.

The dependency’s effects are unknown. Rejection must not be described as leaving all state unchanged.

### Memory and arithmetic preconditions

Accepted paths require readable index storage and writable destination storage of the required width. Executed backing reads require readable backing locations.

No null, alignment, index-range, or destination-bounds checks are shown. Negative and excessive indices are not rejected. Caller/environment validity requirements include all bytes of a full-integer destination.

The simplified address equations do not define overflow behavior for the represented intermediate additions, multiplications, or selector subtractions. Endianness, atomicity, concurrent access semantics, and fault handling are unestablished.

## Dependencies

### F-375c8199e7c2

This is the only shown function dependency.

- Invoked when dispatch or translation rejects the request.
- No explicit argument is shown.
- No return value is consumed.
- May use incoming machine state; its actual interface is unknown.
- Reads, writes, global-state preservation, ownership behavior, and nonlocal control transfer are unknown.
- The caller reaches its ordinary return only if this dependency returns normally.

No assumption about preservation or memory release is part of this contract.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Successful examples assume valid memory and non-overflowing executed arithmetic. “Reaches” a branch includes satisfaction of all preceding dispatch conditions.

1. **Boolean normalization.** Category `0x8861`, Selector `0x8862`, and Value `−1` write byte `1` at offset `0x42`. Value zero writes `0`. No R1–R6 read is shown.

2. **Unsupported category.** Category zero invokes F-375c8199e7c2 without a shown backing read, context-index read, or destination write.

3. **Direct integer store.** Once the lower partition is reached, equality between Selector and its first R2 observation stores Value unchanged at offset `0x34`. There is no second R2 read.

4. **Changing R2 changes dispatch.** Suppose Selector is `0x0D1C` and reaches the lower partition. An equality observation of R2 = `0x0D1D` fails; an ordering observation of R2 = `0x0D1B` selects the greater-selector branch. The selector then rejects rather than performing the literal direct store.

5. **Unsupported selector still reads R4.** A selector reaching the residual lower branch that is none of `0x0D1C`, `0x2200`, `0x8571`, or `0x8572` reads R4 and computes the derived boundary before rejection.

6. **Literal translation avoids reads.** A request reaching family A or B with Value `0x1702` stores `0` without a translation-time R1 read, regardless of earlier dispatch observations.

7. **Separate R1 translation observations.** In family B, Value `0x4000`, a first translation-time R1 observation of `0x5000`, and a second observation of `0x4000` produce byte `2`. The failed minus-one comparison does not pin the subsequent observation.

8. **Mode boundary.** A request reaching mode translation stores `4` for Value `0x2100` and `0` for Value `0x2101`, without reading R3.

9. **Cached boundary with later R4 changes.** For Selector `0x8572`, let the preparation read return `0x2073`, making the cached boundary `0x2100`. With Value `0x2100`, a later equality read of `0x2200` fails and an ordering read of `0x2000` selects the greater branch. Equality with the retained boundary stores `3`.

10. **Extended upper-range literal.** Selector `0x8571` with Value `0x86AE` stores `6` when Value exceeds the cached boundary. Equality with that boundary instead stores `3`.

11. **Family C literal precedence.** Once family C is reached, Value `0x0301` stores `1` without reading R5, even if its backing value would also match.

12. **Family D has no independent literal acceptance.** Value `0x0300` succeeds in family D only if one of its two R5-dependent comparisons matches; the literal itself grants no acceptance.

## Uncertainties

The following remain **documented_only**:

- Values, mutability, storage class, and permitted changes of R1–R6.
- Whether distinct reads can observe changes in the actual execution environment.
- Exact ABI, physical input locations, pointer width, and independent width confirmation.
- Overflow semantics and validity of intermediate address arithmetic.
- Layout interpretation, valid index range, and alignment requirements.
- Instruction-level load behavior, memory ordering, atomicity, and concurrent mutation rules.
- Dependency effects, error policy, ownership, and return behavior.
- Application meanings of categories, selectors, and encoded bytes.

Compact encodings and recurring selector groups suggest an enumeration-setting interface. That is inferred intent only; it supplies neither missing backing values nor an immutability guarantee.

## Evidence

Established static evidence is the supplied function representation: ordered comparisons, distinct conditional backing-read expressions, retained derived values, context-relative index reads, byte and integer stores, and one dependency call.

The contract preserves the visible read structure, including repeated R1, R2, R4, R5, and R6 observations and the R4 read preceding extended-selector validation. It distinguishes that structure from unknown machine-level ordering and from the optional stable-backing interpretation.

No implementation access or runtime tests have been performed.
