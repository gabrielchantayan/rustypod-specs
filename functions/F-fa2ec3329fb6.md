<!-- Trusted provenance: F-fa2ec3329fb6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa2ec3329fb6

## Purpose

F-fa2ec3329fb6 obtains a 120-byte block through a caller-supplied interface, recognizes either the four-byte signature `Xing` or `Info`, and conditionally updates a mutable state object.

Recognition sets two markers and saves the current base-position field. Optional processing can decode two unsigned 32-bit values, calculate an input to an arithmetic dependency, and pass part of the temporary block to another dependency.

The structure resembles audio metadata parsing. That is **inferred intent**, not a requirement that the function validate a complete audio format.

The three normal return values are:

| 32-bit result | Meaning |
|---|---|
| 0 | The read reported exactly 120 bytes, but the signature was rejected. |
| 1 | The signature was accepted and all selected processing returned normally. |
| 2 | Positioning returned nonzero, or the read result was not exactly 120. |

A return value does not establish that dependencies preserved state, temporary storage, globals, ownership, or external position.

## Interface

### State object

The explicit input is an address identifying mutable state. The supplied representation uses 32-bit addresses and dispatch entries.

All offsets below are byte offsets from the input address. Descriptive roles do not establish application-level units or meanings.

| Offset | Width | Use |
|---|---:|---|
| `0x04` | 32 bits | Attached interface-object address |
| `0x08` | 32 bits | Arithmetic dependency’s result destination |
| `0x10` | 32 bits | Selector compared with 2 |
| `0x18` | 32 bits | Unsigned threshold input; later passed to the arithmetic dependency |
| `0x20` | 32 bits | Base-position input and recognition snapshot source |
| `0x24` | 32 bits | Unsigned multiplication input |
| `0x48` | 64 bits | Multiplication input represented as signed 64-bit |
| `0x58` | 32 bits | First decoded optional value |
| `0x5C` | 8 bits | Recognition marker |
| `0x5D` | 8 bits | Marker cleared before delegation |
| `0x5E` | Unknown extent | Embedded destination passed to a dependency |
| `0x600` | 32 bits | Second decoded optional value |
| `0x614` | 8 bits | Additional recognition marker |
| `0x618` | 32 bits | Saved base-position value |

The interface object contains a dispatch-table address. Entries at table byte offsets `0x14` and `0x10` supply the positioning and read dependencies, respectively.

No explicit null, bounds, alignment, lifetime, or interface-validity checks are visible. Valid readable and writable storage is a precondition for the selected path. Destination-capacity requirements for the delegated operation are unknown.

### Temporary block

The read requests 120 bytes into temporary storage. The parsing layout is:

| Block offset | Use |
|---|---|
| `0–3` | Signature |
| `4–6` | Not directly examined |
| `7` | Flag byte, observed at separate processing stages |
| `8–11` | First optional value when the initial bit-0 test succeeds |
| `8–11` or `12–15` | Second optional value, selected by the saved parsing cursor |
| `8`, `12`, or `16` onward | Potential delegated source |

Optional integers are unsigned, big-endian 32-bit values. A full 32-bit flag word is not decoded.

The requested length does not independently prove that the displayed temporary allocation has a valid 120-byte extent. That allocation question remains unresolved.

### Ownership and lifetime

No direct allocation, release, or ownership transfer is visible.

The read dependency receives the temporary destination address. The final dependency can receive an interior pointer into the same temporary storage. Whether either dependency retains a pointer, copies data, allocates memory, releases memory, or accesses storage after return is unknown.

In particular, the read dependency might retain its destination pointer and make that storage accessible to a later dependency. The contract therefore does not assume that temporary bytes remain unchanged across the arithmetic call.

## Behavior

### 1. Select and request a position

The function compares the unsigned 32-bit field at `0x18` with 32000 and the 32-bit selector at `0x10` with 2.

| Field at `0x18` | Selector | Displacement |
|---|---|---:|
| Below 32000 | Equal to 2 | 21 |
| Below 32000 | Not equal to 2 | 13 |
| At least 32000 | Equal to 2 | 36 |
| At least 32000 | Not equal to 2 | 21 |

It adds the selected displacement to the 32-bit base-position field at `0x20`. No overflow check is visible. The representation treats this addition as signed-width arithmetic; behavior outside the signed 32-bit range is not independently resolved.

F-000000000001 receives these arguments in the displayed order:

1. Attached interface-object address.
2. The selected positioning entry’s own function-address value.
3. Base position plus displacement.
4. Zero.
5. Zero.

The second argument’s semantic purpose is unknown.

A nonzero result immediately produces status 2. The read dependency is not called on that path.

### 2. Obtain the block

After positioning returns zero, the function obtains the attached interface and read entry again. It does not rely on the earlier interface selection remaining unchanged.

F-000000000002 receives:

1. The then-current interface-object address.
2. The temporary destination address.
3. Requested length 120.
4. Constant 2.

Only a reported result of exactly 120 permits parsing. Every other result, including a negative value, zero, 119, or 121, produces status 2. There is no visible retry or continuation of a partial read.

Deterministic parsing requires that the dependency initialize the bytes subsequently observed. Reporting 120 alone does not independently prove initialization.

### 3. Recognize the signature

The accepted signatures are exactly:

- `Xing`: hexadecimal bytes `58 69 6E 67`.
- `Info`: hexadecimal bytes `49 6E 66 6F`.

Comparisons are case-sensitive and require no terminator.

Recognition includes temporary writes:

- If byte 0 is `X` and byte 1 is `i`, byte 1 is replaced with byte 2 before the remaining `Xing` checks.
- If byte 0 is `I`, byte 0 is replaced with byte 1 before the remaining `Info` checks.

These writes can occur even when the eventual signature is rejected. They affect temporary storage, not a direct state-field destination. A dependency retaining access to that storage is not guaranteed to observe an unmodified signature.

A rejected signature returns 0. No direct state-object write precedes signature acceptance.

### 4. Record recognition

For either accepted signature, direct state writes occur in this order:

1. Write byte value 1 at `0x5C`.
2. Write byte value 1 at `0x614`.
3. Read the then-current 32-bit field at `0x20` and copy it to `0x618`.

The saved value is the base field itself, not base plus displacement. Because it is read after both interface calls, it may differ from the value used for positioning.

The parsing cursor is initialized to block offset 8 before optional processing.

### 5. Observe bit 0 and process the first value

The function tests bit 0 of temporary byte 7 **before** invoking F-1e0edc6e0bc9.

If this test is clear:

- The first optional value is not decoded.
- Neither `0x58` nor `0x08` is directly written.
- The parsing cursor remains 8.
- F-1e0edc6e0bc9 is not called.

If this test is set:

1. Decode block bytes `8–11` as an unsigned, big-endian 32-bit value.
2. Store the decoded value at `0x58`.
3. Form the 64-bit product described below.
4. Invoke F-1e0edc6e0bc9.
5. Store its 32-bit result at `0x08`.
6. Set the parsing cursor to 12.

The cursor becomes 12 after the helper returns regardless of any changes that helper caused to byte 7. It records the earlier branch decision; it is not recomputed from the current bit 0.

#### Arithmetic width

The product uses:

- The unsigned 32-bit field at `0x24`, widened to unsigned 64-bit.
- The decoded unsigned 32-bit value, widened to unsigned 64-bit.
- The 64-bit field at `0x48`.

The explicit unsigned 64-bit operands make the multiplication unsigned at that width. The signed-represented third operand is converted to unsigned 64-bit for multiplication. The resulting product is reduced modulo \(2^{64}\); no saturation or overflow rejection is visible.

F-1e0edc6e0bc9 receives four 32-bit arguments:

1. Low 32 bits of the product.
2. High 32 bits of the product.
3. The then-current field at `0x18`.
4. Zero.

Its mathematical operation, units, rounding, exceptional inputs, and side effects are unknown. Its returned bit pattern is stored at `0x08` without a visible error check.

### 6. Observe bit 1 and process the second value

After the first optional path completes or is skipped, the function tests bit 1 from temporary byte 7.

If the first path called F-1e0edc6e0bc9, this is a flag observation **after that call**, not a reuse of the flag value observed before it.

If bit 1 is set:

- Decode four bytes starting at the saved parsing cursor.
- Store the unsigned, big-endian result at `0x600`.
- Advance the delegated-source cursor by four bytes.

If bit 1 is clear:

- Do not directly write `0x600`.
- Use the saved parsing cursor unchanged as the delegated-source cursor.

The second value’s offset depends on the **earlier bit-0 decision**:

| Earlier bit-0 decision | Second-value offset, if the later bit-1 test succeeds |
|---|---:|
| Clear | 8 |
| Set | 12 |

Its bytes are read during this stage. They are not a snapshot taken before the arithmetic call.

### 7. Observe bit 2 and optionally delegate

The function next tests bit 2 from temporary byte 7, after the bit-1 stage.

If set:

1. Write byte value 0 at `0x5D`.
2. Call F-000000000003 with the address at state offset `0x5E` and the temporary-block address at the delegated-source cursor.

There is no intervening dependency call between the bit-1 and bit-2 tests, but they are separate observations rather than a single saved flag snapshot.

The delegated-source offset is:

| Earlier bit-0 decision | Later bit-1 decision | Source offset |
|---|---|---:|
| Clear | Clear | 8 |
| Set | Clear | 12 |
| Clear | Set | 12 |
| Set | Set | 16 |

The current bit 0 at delegation time does not determine this offset.

No length is passed. The dependency’s result is not visibly inspected. If bit 2 is clear, neither the marker clear nor this call occurs.

Bits 3–7 have no direct control-flow effect. Normal completion returns 1.

## Side effects

Direct state writes are conditional and nontransactional. Recognition markers and the base snapshot precede optional processing. The first decoded value is stored before the arithmetic call; the helper result is stored after it. The second value follows that path, and the delegation marker is cleared immediately before delegation.

No rollback is visible. If a dependency faults, exits nonlocally, or does not return, preceding effects may remain.

Absent options leave their corresponding destinations untouched **by this function**. They do not reset stale values, nor do they prevent dependencies from changing those fields.

Positioning and reading occur before signature validation. Thus:

- Status 0 can follow external positioning and read effects.
- Status 2 does not imply unchanged external position.
- No direct state writes on an early-return path does not establish dependency preservation.

Temporary-storage mutation is also observable in principle. Signature checking modifies some signature bytes. Later dependencies may have retained access to the read destination; preservation, retention, and mutation behavior are unknown.

No exact ordering is established among operand loads within a compound arithmetic or byte-assembly expression. The stage ordering and dependency boundaries above are established.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-000000000001 | Indirect entry at dispatch offset `0x14`; five displayed arguments; only zero allows reading |
| F-000000000002 | Indirect entry at dispatch offset `0x10`; interface, temporary destination, 120, and 2; only result 120 allows parsing |
| F-1e0edc6e0bc9 | Four 32-bit inputs; 32-bit result stored at `0x08`; called only when the initial bit-0 test succeeds |
| F-000000000003 | Embedded destination and temporary source addresses; no explicit length; result not inspected |

The three numbered document-local IDs identify dependency roles, not independently established global identities.

No dependency is assumed to preserve globals, object fields, interface selection, temporary bytes, or ownership. In particular, nothing establishes that F-1e0edc6e0bc9 cannot reach the temporary block through access retained by F-000000000002.

## Acceptance examples

These are conditional static examples, not executed tests. Except where mutation is specified, dependencies return normally and preserve relevant state and temporary bytes.

1. **Threshold boundary.** With selector 2, threshold 31999 selects displacement 21 and threshold 32000 selects 36. With another selector, those thresholds select 13 and 21.

2. **Positioning failure.** A nonzero F-000000000001 result produces status 2 without calling F-000000000002. There are no direct state writes.

3. **Exact read gate.** Read results 119 and 121 both produce status 2. Neither permits signature processing.

4. **Rejected signature.** A complete block beginning `Xinf` produces status 0. Temporary byte 1 becomes `n` during checking. Existing state recognition markers are not directly cleared.

5. **Recognition without options.** An `Info` block whose relevant flag observations are all zero produces status 1, sets both recognition markers, and saves the current base field. Optional destinations are not directly changed.

6. **First value and helper arguments.** With the initial bit-0 test set, bytes `00 00 01 02` at offsets `8–11` store 258 at `0x58`. If the field at `0x24` is 3 and the field at `0x48` is 2, the helper receives product halves 1548 and 0, followed by the current `0x18` value and zero. Its return value is stored at `0x08`.

7. **Unsigned product reduction.** With decoded value 2, multiplicative field `0x24` equal to 1, and the signed-represented field `0x48` equal to −1, the product bit pattern is `FFFFFFFFFFFFFFFE` hexadecimal. The helper’s low and high halves are `FFFFFFFE` and `FFFFFFFF`.

8. **Second option alone, unchanged block.** A stable flag byte of `02` hexadecimal causes bytes `8–11` to be decoded into `0x600`. The arithmetic dependency is not called.

9. **All options, unchanged block.** A stable flag byte of `07` hexadecimal places the first value at `8–11`, the second at `12–15`, and the delegated source at 16.

10. **Flag mutation from `01` to `02`.** Suppose the initial bit-0 test succeeds, and the arithmetic dependency changes byte 7 from `01` to `02` through retained access. The first path still completes and sets the cursor to 12. The later bit-1 test succeeds, so the second value comes from `12–15`, despite current bit 0 being clear. Bit 2 is clear, so there is no delegation.

11. **Flag mutation from `01` to `04`.** Under the same access condition, changing byte 7 to `04` leaves the cursor at 12, skips the second value, clears `0x5D`, and delegates from offset 12.

12. **Flag and payload mutation.** If the helper changes byte 7 from `01` to `06` and replaces bytes `12–15`, the second stage decodes those replacement bytes and delegation starts at 16. The previously stored first value is not decoded again.

13. **Upper flag bits.** For otherwise identical, unchanged blocks with flag bytes `00` and `80`, direct optional-processing choices are identical.

## Uncertainties

The following remain **documented_only**:

- Application-level field meanings and units.
- The positioning call’s unusual function-address argument.
- Out-of-range behavior of the signed-represented base-position addition.
- The complete size and validity of temporary storage.
- Dependency side effects, global preservation, and external-position semantics.
- Whether the read destination is retained, subsequently reachable, or mutated.
- The arithmetic dependency’s operation and exceptional-input behavior.
- The delegated format, read extent, destination capacity, ownership, and retention rules.
- Fault, exception, concurrency, and nonlocal-exit behavior.

Immutable-block descriptions apply only when dependencies and other permitted actors leave the relevant bytes unchanged. They are not unconditional guarantees.

## Evidence

Established static evidence includes the displacement selection, dispatch offsets, displayed argument lists, exact read-count gate, signature checks and temporary writes, recognition-write order, unsigned 64-bit multiplication, helper arguments, and three normal results.

The optional-processing evidence establishes a temporal distinction: bit 0 is tested before the arithmetic call; a successful first path sets the cursor to 12 after that call; bits 1 and 2 are subsequently tested from temporary storage. Later offsets depend on saved decisions, not a reconstructed interpretation of one final flag byte.

Metadata parsing and field meanings beyond their demonstrated uses are inferred intent. Dependency internals and the unresolved behaviors listed above are not established.

No runtime tests, dependency executions, or dynamic memory observations have been performed.
