<!-- Trusted provenance: F-4dff4d3e7c26; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4dff4d3e7c26

## Purpose

F-4dff4d3e7c26 updates a destination record using geometry, flags, measurements, and optional byte-table data obtained through a context record. It returns a 32-bit zero on every normally completing path.

The established behavior is a sequence of reads, arithmetic, dependency calls, and destination writes. Interpreting the resulting fields as placement, extents, or device-space measurements is inferred intent, not an established external interface.

The routine does not validate its pointers or report malformed data. Successful completion requires every memory access actually reached, including accesses made by dependencies, to be valid.

## Interface

### Arguments and result

| Item | Contract |
|---|---|
| First argument | Pointer to a context record containing 32-bit addresses, flags, and measurements at the offsets below. |
| Second argument | A 32-bit value used in optional table-column selection. Its addition and comparison in that path use unsigned 32-bit arithmetic. |
| Result | A 32-bit zero on normal completion. No alternative result or explicit error result is present. |

The supplied representation uses 32-bit addresses and 32-bit integer fields. Offsets below are byte offsets. Signedness is specified where it affects behavior; a stored 32-bit quantity otherwise denotes its raw bit pattern.

### Context record

| Offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Address of the primary state record. |
| `0x04` | 32 bits | Address of the secondary record. |
| `0x08` | 32 bits | Address of the destination record. |
| `0x10` | 32 bits | Flags; bits `0x1`, `0x2`, and `0x200` affect separate decisions. |
| `0x24`, `0x28`, `0x2C`, `0x30` | 32 bits each | Four rectangle quantities, used directly on one rectangle-selection path. |
| `0x3C` | 32 bits | Default value for destination offset `0x38`. |
| `0x44`, `0x4C` | 32 bits each | Operands for the initial destination value at `0x28`. |
| `0xAC`, `0xB4` | 32 bits each | Measurements used by one alignment branch. |

For this contract, call the four selected rectangle quantities **R0**, **R1**, **R2**, and **R3**, in that order. These names identify values, not their geometric interpretation.

### Primary state record

| Offset | Width | Use |
|---|---:|---|
| `0xDC`, `0xDE` | Signed 16 bits each | One pair of span operands. |
| `0xE2` | Unsigned 16 bits | Alternate value for destination offset `0x38`. |
| `0x124` | 8 bits | Enables a further alignment-state read when nonzero. |
| `0x16C` | Signed 16 bits | Selects which pair of span operands is used. |
| `0x1B2`, `0x1B4` | Signed 16 bits each | Alternate span operands. |
| `0x1DC` | 32 bits | Nonzero-state gate, read at two different stages. |
| `0x2BC` | 32 bits | Address of table storage. |
| `0x2C4` | Unsigned 32 bits | Number of entries in the key list. |
| `0x2C8` | Unsigned 32 bits | Table row stride and column bound. |
| `0x2CC` | 32 bits | Address of the byte key list. |

A signed 16-bit alignment value may also be read at an address formed by adding a global offset to the primary state address. The global offset’s value and provenance are not supplied.

### Secondary record

| Offset | Width | Use |
|---|---:|---|
| `0x0C` | Unsigned 16 bits | Key sought in the byte key list. |
| `0x14` | 32 bits | Conversion parameter when context flag `0x1` is clear. |

### Destination record

The routine explicitly writes 32-bit fields at offsets `0x18` through `0x3C`, as detailed below. It also reads a 32-bit selector at `0x48` and may pass the address of its region at `0x6C` to a dependency.

## Behavior

### 1. Capture record addresses and choose the conversion parameter

The routine first reads the destination address from context offset `0x08`, then the primary state address from `0x00`, then the secondary address from `0x04`. Those three addresses are retained for subsequent work; the routine does not reload them from the context.

The conversion parameter starts as the 32-bit value `0x00010000`. If context flag `0x1` is clear, it is replaced by the 32-bit value read from secondary offset `0x14`.

This selected parameter is retained for all later conversion calls. Later changes to the secondary parameter do not change that captured value.

### 2. Obtain the rectangle quantities

The 32-bit value at destination offset `0x48` is compared with a global selector value.

- If equal, R0 through R3 are read from context offsets `0x24`, `0x28`, `0x2C`, and `0x30`, respectively.
- Otherwise, F-960c2eabcfd8 is called with the address of destination offset `0x6C` and the address of temporary storage for four 32-bit rectangle quantities.

The rectangle dependency’s return value is not inspected. The routine subsequently uses all four temporary quantities. Consequently, the nonmatching-selector path requires that dependency to establish usable values for all four quantities. No initialization fallback is visible.

### 3. Write the first destination fields

The candidate for destination offset `0x38` is read from context offset `0x3C`.

If primary state offset `0x1DC` is nonzero and context flag `0x200` is clear, the candidate is replaced with the unsigned 16-bit value at primary offset `0xE2`, zero-extended to 32 bits.

The following writes then occur in order:

1. Destination `0x38`: the selected candidate.
2. Destination `0x20`: R0.
3. Destination `0x24`: R3.
4. Destination `0x28`: context value at `0x4C` minus context value at `0x44`, as a 32-bit result.

The last of these may be overwritten by the optional table path.

### 4. Select and calculate alignment values

The byte at primary offset `0x124` is read.

- If zero, no indirect alignment value is read, and the centered branch below is selected.
- If nonzero, a signed 16-bit value is read using the global offset described in the interface. A zero value selects the centered branch; any nonzero value selects the alternate branch.

Only zero versus nonzero matters for this indirect value. Its magnitude and sign are not otherwise used.

#### Centered branch

F-73d099a86adb is called with the 32-bit difference R3 − R1 and the captured conversion parameter. Only the low 16 bits of its result are retained, interpreted as a signed 16-bit quantity.

After that call, the routine reads primary offset `0x16C`:

- If it is signed `−1`, the span is the signed 16-bit value at `0xDC` minus the signed 16-bit value at `0xDE`.
- Otherwise, the span is the signed 16-bit value at `0x1B2` minus the signed 16-bit value at `0x1B4`.

Both operands are sign-extended before subtraction. The span therefore lies between `−65535` and `65535`.

The alignment offset is the signed result of subtracting the retained signed 16-bit conversion result from the span, then dividing by two with truncation toward zero. The associated extent is the span’s 32-bit representation.

These span reads occur after F-73d099a86adb. They must not be treated as a pre-call snapshot of primary state.

#### Alternate branch

F-73d099a86adb is called with the 32-bit difference between context offset `0xAC` and R3, plus the captured conversion parameter. Its low 16 bits are interpreted as signed and sign-extended to produce the alignment offset.

The context values at `0xB4` and `0xAC` are then compared as signed 32-bit integers:

- If the value at `0xB4` is less than the value at `0xAC`, F-73d099a86adb is called again, using the 32-bit difference `0xAC − 0xB4` and the same conversion parameter. The associated extent is the low 16 bits of this result, **zero-extended** to 32 bits.
- Otherwise, the associated extent is zero, and this second call is omitted.

The signed treatment of the first conversion result and unsigned treatment of the second are distinct requirements.

### 5. Preserve the extent, optionally convert, and write placement fields

The associated extent is written to destination offset `0x3C` before any calls to F-74efb2674226.

Context flag `0x1` is read again. If clear:

1. F-74efb2674226 is called with the alignment offset and captured conversion parameter; its 32-bit result replaces the offset.
2. F-74efb2674226 is called with the associated extent and captured conversion parameter; its 32-bit result replaces the extent.

If the flag is set at this later decision, both calls are omitted.

The following destination writes then occur in order:

1. `0x2C`: the signed 32-bit difference R0 − R2, divided by two with truncation toward zero.
2. `0x30`: the resulting alignment offset.
3. `0x34`: the resulting associated extent.

Thus `0x3C` preserves the extent before this optional conversion, while `0x34` receives its post-conversion value.

### 6. Optionally overwrite destination `0x28` from a byte table

Primary offset `0x1DC` is read again. This is a separate observation from the earlier read.

The table path is entered only if that value is zero and context flag `0x2` is clear. When the primary value is nonzero, the flag test is not needed for this decision.

The routine scans key-list indices from zero upward, comparing each unsigned byte with the unsigned 16-bit key at secondary offset `0x0C`. The first match ends the scan. A key greater than 255 cannot match any byte.

At the first match:

- The second argument plus two is calculated modulo \(2^{32}\).
- That unsigned result must be strictly less than the unsigned row stride at primary offset `0x2C8`.
- If the bound passes, a candidate address is formed from the table base, eight bytes of additional displacement, the matched index multiplied by the stride, and the wrapped column value.
- Regardless of whether the bound passes, no later matching key is considered.

If the resulting candidate address is nonzero, one unsigned byte is read there. Destination offset `0x28` is overwritten with that byte multiplied by 64, represented as a 32-bit value.

No overwrite occurs if there is no match, the bound fails, or the calculated address is zero. A nonzero but invalid address is not protected against.

The visible loop rereads its count condition and relevant record fields; it is not specified as operating on a separately captured table snapshot.

### 7. Write final dimensions and return

The final explicit destination writes are:

1. `0x18`: R2 − R0, as a 32-bit result.
2. `0x1C`: R3 − R1, as a 32-bit result.

The function then returns zero.

## Side effects

The explicit destination-write order is:

`0x38`, `0x20`, `0x24`, `0x28`, `0x3C`, `0x2C`, `0x30`, `0x34`, optionally `0x28` again, then `0x18`, `0x1C`.

There is no transaction, rollback, or deferred commit. A failure during a dependency call or later memory access can leave earlier writes visible.

The routine itself contains no explicit allocation, release, ownership transfer, or retention of the supplied record addresses. F-960c2eabcfd8 receives an address within the destination and an address to temporary rectangle storage. Its ownership and retention behavior are unknown.

No dependency may be assumed to preserve globals or other reachable memory. This matters particularly for:

- The later reread of context flag `0x1`.
- Span operands read after a conversion call.
- The second observation of primary offset `0x1DC`.
- Table metadata read after the conversion calls.

Aliasing is not rejected. If the context, destination, primary state, secondary record, or table storage overlap, earlier writes can affect later reads. The operation order above therefore matters; a snapshot-based reformulation is not generally equivalent.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-960c2eabcfd8 | Called on selector mismatch with a destination-region address and writable storage for four 32-bit quantities. | Rectangle semantics, errors, global effects, ownership, or whether all four values are always written. |
| F-73d099a86adb | Receives a 32-bit arithmetic input and the captured conversion parameter. Its result is consumed through the low 16 bits. | Mathematical transformation, rounding, overflow handling, and side effects. |
| F-74efb2674226 | Conditionally called first for the offset, then for the extent, using the captured parameter. Full 32-bit results are used. | Mathematical transformation, signedness of its internal interpretation, errors, and side effects. |

The constant `0x00010000` and paired conversion calls suggest a scale representation. That interpretation is inferred intent only; no identity-at-that-constant rule is established.

## Acceptance examples

These examples are conditional on valid memory, normal dependency completion, and absence of unspecified mutation or aliasing that would change later inputs.

1. **Direct rectangle selection.**  
   If the destination selector equals the global selector and the context rectangle is `(10, 20, 30, 60)`, F-960c2eabcfd8 is not called. Destination `0x20` becomes 10, `0x24` becomes 60, `0x2C` becomes −10, `0x18` becomes 20, and `0x1C` becomes 40.

2. **Centered negative odd result.**  
   If the selected span is 10 and the retained signed conversion result is 13, the alignment offset before optional conversion is −1, not −2. Destination `0x3C` receives 10.

3. **Distinct signed and unsigned low-half handling.**  
   In the alternate branch, a first conversion result whose low 16 bits are `0xFFFF` produces an alignment offset of −1. A second conversion result with the same low bits produces an associated extent of 65535.

4. **Primary-state override.**  
   If primary `0x1DC` is nonzero, flag `0x200` is clear, and primary `0xE2` is 65535, destination `0x38` receives 65535 rather than the context default.

5. **Byte-table overwrite.**  
   If the table path is enabled, the first matching key passes the column bound, the calculated address is nonzero, and its byte is 7, destination `0x28` is overwritten with 448.

6. **First match blocks later alternatives.**  
   If the first matching key fails the column bound, a later duplicate key is not examined. The earlier subtraction-based value at destination `0x28` remains.

7. **Unsigned column wrap.**  
   For a second argument with bit pattern `0xFFFFFFFF`, adding two produces column 1. A stride of 2 passes the strict bound. The source establishes no rejection merely because the argument could be interpreted as negative.

8. **Conversion flag is observed twice.**  
   If flag `0x1` is initially clear, the secondary parameter is captured. If it becomes set before the later flag test, both F-74efb2674226 calls are skipped; the parameter is not recaptured or replaced.

## Uncertainties

- **documented_only — external meaning:** Units, axis conventions, selector meaning, and field names are unavailable.
- **documented_only — dependency semantics:** Conversion formulas, rounding, failures, memory effects, and ownership behavior are unknown.
- **documented_only — arithmetic extremes:** The supplied representation establishes 32-bit operations and unsigned wrapping for table-column arithmetic. It does not establish a higher-level checked-overflow policy for signed geometry expressions. No saturation or validation is visible.
- **documented_only — address overflow:** Table multiplication and address additions have 32-bit width in the supplied representation. Wrapped nonzero addresses are not independently validated.
- **documented_only — concurrency:** No synchronization is shown. Repeated reads do not establish a supported concurrent-mutation contract.
- **documented_only — malformed inputs:** Null pointers, undersized records, invalid global offsets, and incomplete dependency output have no explicit recovery path.

## Evidence

This contract is based solely on static examination of the supplied function representation and dependency mapping. Established claims cover visible field widths, branches, call order, result truncation, signed comparisons, explicit stores, and the normal return value.

Geometric and scaling interpretations are labeled as inferred intent. Missing callee behavior and environmental guarantees remain documented unknowns.

No runtime tests have been performed.
