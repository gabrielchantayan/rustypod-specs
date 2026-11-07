<!-- Trusted provenance: F-8bc6b3aca8ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8bc6b3aca8ef

## Purpose

F-8bc6b3aca8ef prepares a rectangular operation from a center position and dimensions, updates a byte in a selected record, and invokes F-a05963f65233 exactly once on every normally completing path.

The geometry uses horizontal and vertical boundaries of 320 and 240, respectively. It adjusts rectangles crossing the left or top boundary but does not itself trim rectangles crossing the right or bottom boundary. A record byte and a parent-level comparison determine whether the dependency receives the calculated geometry or a fixed fallback geometry.

The fallback consists of horizontal position 320, vertical position 240, width 1, and height 1. Selecting the fallback does **not** necessarily clear the record byte. Conversely, the left and top adjustment amounts survive fallback selection.

The interpretation as rectangle clipping is inferred intent. The geometry calculations, memory accesses, branch conditions, and dependency arguments described below are established by the supplied static evidence. Whether the dependency draws, copies, queues, or otherwise processes the rectangle is unknown.

## Interface

The function accepts seven arguments and has no explicit return value.

| Argument | Representation | Observable use |
|---|---|---|
| Parent base | 32-bit address-bearing value | Locates record storage and a parent-level comparison word; forwarded to the dependency. |
| Opaque value | 32 bits, uninterpreted | Forwarded unchanged in two dependency argument positions. |
| Record offset | Unsigned 32-bit integer | Added to the parent base to locate record fields; compared with a parent-level word; forwarded unchanged. |
| Horizontal center | Signed 32-bit integer | Used for horizontal geometry and forwarded unchanged as the dependency’s final argument. |
| Vertical center | Signed 32-bit integer | Used for vertical geometry; not separately forwarded unchanged. |
| Width | Signed 32-bit integer | Used to calculate horizontal geometry; may be reduced or replaced before forwarding. |
| Height | Signed 32-bit integer | Used to calculate vertical geometry; may be reduced or replaced before forwarding. |

The record offset is a byte displacement, not an established array index. No multiplication by a record size occurs.

The following memory locations are relevant:

| Location | Width | Access |
|---|---:|---|
| Parent base + record offset + 136 bytes | 1 byte | Written initially; may be cleared; conditionally read later. |
| Parent base + record offset + 141 bytes | 1 byte | Conditionally read as an unsigned value. |
| Parent base + 240 bytes | 4 bytes | Conditionally read as an unsigned comparison value. |

For this contract, these locations are called the **status byte**, **gate byte**, and **parent comparison word**, respectively. These names describe their observable roles, not established domain meanings.

There is no null check, bounds check, allocation check, or recoverable error result. The caller must provide storage permitting every access reached by the selected path. The dependency may impose additional requirements that are not available here.

The contract below describes arithmetic when the signed intermediates are representable and the address calculations resolve to valid storage. Overflow behavior and address-space boundary behavior are documented under Uncertainties.

## Behavior

### Initial state and division

The function derives a horizontal half-extent from width divided by 2 and a vertical half-extent from height divided by 2. Signed division truncates toward zero. Thus, positive odd dimensions lose their fractional half; negative odd dimensions are not rounded toward negative infinity.

The function writes the value 1 to the status byte before making any geometry-dependent decision.

Its working left-adjustment amount, top-adjustment amount, and vertical position initially equal zero. The original horizontal center remains available unchanged for the eventual dependency call.

### Horizontal geometry

The horizontal end coordinate is the horizontal center plus the horizontal half-extent.

A left-boundary adjustment occurs only when both of these conditions hold:

- The horizontal center is less than the horizontal half-extent.
- The horizontal end coordinate is strictly greater than zero.

In that case:

- The left-adjustment amount becomes the horizontal half-extent minus the horizontal center.
- The working width becomes the original width minus that adjustment.
- The horizontal position becomes zero.
- This branch does not clear the status byte.

Otherwise, the horizontal position becomes the horizontal center minus the horizontal half-extent. The function then checks whether that position is greater than 319 or the horizontal end coordinate is less than 1.

If either condition holds:

- The status byte is written as zero.
- The horizontal position is temporarily set to zero.

If neither condition holds, the calculated horizontal position and original width remain in use, with a zero left-adjustment amount.

These conditions use the center and truncated half-extent, not a separately calculated start-plus-width interval. Their edge behavior must therefore not be replaced by a conventional rectangle-intersection formula.

There is no horizontal right-edge reduction. For example, a positive-width rectangle beginning at horizontal position 318 can retain a width extending beyond 320.

### Vertical geometry

The vertical end coordinate is the vertical center plus the vertical half-extent.

A top-boundary adjustment occurs only when both of these conditions hold:

- The vertical center is less than the vertical half-extent.
- The vertical end coordinate is strictly greater than zero.

In that case:

- The top-adjustment amount becomes the vertical half-extent minus the vertical center.
- The working height becomes the original height minus that adjustment.
- The vertical position remains zero.
- Execution proceeds to the record eligibility checks.

Otherwise, the vertical position becomes the vertical center minus the vertical half-extent.

If that position is less than 240 and the vertical end coordinate is strictly greater than zero, execution proceeds to the record eligibility checks.

If either condition fails:

- The status byte is written as zero.
- Record eligibility checks are skipped.
- Fallback geometry is selected.

There is no vertical bottom-edge reduction.

A horizontal rejection does not bypass vertical processing. In particular, vertical processing can still calculate a top-adjustment amount after the horizontal status write has cleared the status byte.

### Record eligibility checks

These checks occur only after a vertically accepted path.

First, the function reads the status byte and interprets any nonzero byte as true. Under ordinary noninterfering execution, that read observes the initial value 1 or a horizontal rejection’s value 0.

If the status byte is zero:

- The gate byte is not read.
- The parent comparison word is not read.
- Fallback geometry is selected.

If the status byte is nonzero, the function reads the gate byte as an unsigned 8-bit value.

If the gate byte is zero:

- The parent comparison word is not read.
- Fallback geometry is selected.

If both bytes are nonzero, the function reads the parent comparison word and compares its full unsigned 32-bit value with the record offset.

Calculated geometry is retained only when the parent comparison word differs from the record offset. Equality selects fallback geometry.

Thus, retaining calculated geometry requires all of the following:

- Vertical processing reaches the eligibility checks.
- The status-byte read is nonzero.
- The gate-byte read is nonzero.
- The parent comparison word differs from the record offset.

The gate does not check for a particular nonzero gate-byte value.

### Fallback selection

Fallback selection replaces exactly four working values:

| Value | Fallback |
|---|---:|
| Horizontal position | 320 |
| Vertical position | 240 |
| Width | 1 |
| Height | 1 |

It does not reset the left-adjustment or top-adjustment amount.

It also does not independently write the status byte. A fallback selected because the gate byte is zero or the parent comparison word equals the record offset can leave the status byte equal to 1.

The fixed fallback may be intended to describe an off-surface operation, but whether it is ignored or has some other effect depends on F-a05963f65233 and is not established here.

### Dependency invocation and completion

After choosing calculated or fallback geometry, the function calls F-a05963f65233 with thirteen arguments in this exact order:

| Position | Value |
|---:|---|
| 1 | Original opaque 32-bit value |
| 2 | Selected horizontal position |
| 3 | Selected vertical position |
| 4 | Top-adjustment amount |
| 5 | Left-adjustment amount |
| 6 | Selected width |
| 7 | Selected height |
| 8 | 32-bit bit pattern `0xFFFFFFFF` |
| 9 | 32-bit bit pattern `0xFFFFFFFF` |
| 10 | Original parent base |
| 11 | Original opaque 32-bit value, repeated |
| 12 | Original unsigned record offset |
| 13 | Original horizontal center |

The two all-one arguments have the signed interpretation −1 and the unsigned interpretation 4,294,967,295. Their semantic types and meanings are unknown.

There is no early normal return suppressing the call. Even horizontally or vertically rejected geometry reaches the dependency with fallback geometry.

The function does not inspect or forward a dependency result. If the dependency returns normally, this function returns without an explicit value.

## Side effects

Before entering F-a05963f65233, the only explicit memory writes are to the status byte:

1. An unconditional write of 1.
2. A possible write of 0 during horizontal rejection.
3. A possible write of 0 during vertical rejection.

Both rejection writes can occur in one invocation. A specification of observable access order must preserve those separate writes rather than describe only the final byte value.

On vertically accepted paths, the explicit reads follow geometry processing in this order:

1. Status byte.
2. Gate byte, only if the status-byte read is nonzero.
3. Parent comparison word, only if both preceding byte reads are nonzero.

No status read occurs on the vertically rejected path.

The evidence does not require the selected record fields to be disjoint from the parent comparison word. Consequently, the initial or rejection writes can affect the later comparison-word read when their storage overlaps. For example, a record offset of 104 makes the status-byte address coincide with the beginning of the parent comparison word. The resulting word value depends on the other bytes and the target byte order.

The function itself shows no allocation, deallocation, ownership transfer, or memory copy. Passing the parent base or opaque value does not establish that the dependency borrows, retains, owns, or releases anything.

Any additional memory changes, global changes, resource operations, or retained references performed by F-a05963f65233 are unknown. The pre-call status value is therefore not a guaranteed post-return value.

No error is explicitly reported for rejected geometry. It is converted into fallback arguments rather than an error return.

## Dependencies

**F-a05963f65233**

- Called exactly once on every path that reaches normal call execution.
- Receives the thirteen ordered values specified above.
- Receives original context values alongside calculated or fallback geometry.
- Its return value is not used.
- Its argument types, ownership rules, side effects, validation behavior, and failure behavior are unavailable.

The contract must not assume that this dependency preserves globals, leaves the status byte unchanged, treats the fallback as a no-op, or accepts invalid addresses and unusual dimensions.

## Acceptance examples

These are conditional static acceptance examples, not observed runtime results. Unless stated otherwise, assume representable arithmetic, valid storage, no concurrent interference, no overlap between the status byte and the parent comparison word, a nonzero gate byte, and a parent comparison word unequal to the record offset. Every example still forwards the original context arguments and both all-one arguments.

| Inputs or condition | Required pre-call result |
|---|---|
| Horizontal center 100, vertical center 80, width 20, height 10 | Position (90, 75), adjustments top 0 and left 0, dimensions (20, 10), status byte 1. |
| Horizontal center 3, vertical center 80, width 10, height 10 | Position (0, 75), top adjustment 0, left adjustment 2, dimensions (8, 10). |
| Horizontal center 100, vertical center 2, width 20, height 10 | Position (90, 0), top adjustment 3, left adjustment 0, dimensions (20, 7). |
| Horizontal center 319, vertical center 239, width 2, height 2 | Position (318, 238), dimensions (2, 2); no right or bottom reduction. |
| Horizontal center 320, vertical center 80, width 2, height 10 | Position (319, 75) is retained. A center at 320 alone does not cause rejection. |
| Horizontal center 321, vertical center 80, width 2, height 10 | Horizontal rejection clears status; fallback position (320, 240), dimensions (1, 1). The status byte is read, but neither gate byte nor parent comparison word is read. |
| Horizontal center −5, vertical center 80, width 10, height 10 | Horizontal end equals zero, so horizontal rejection occurs; fallback is selected. |
| Horizontal center 100, vertical center 241, width 20, height 2 | Vertical position equals 240, causing vertical rejection. Status is cleared and all eligibility reads are skipped. |
| Otherwise accepted geometry, but gate byte zero | Fallback is selected; status remains 1 before the dependency; parent comparison word is not read. |
| Otherwise accepted geometry, but parent comparison word equals the record offset | Fallback is selected; status remains 1 before the dependency. |
| Horizontal center 3, vertical center 2, width 10, height 10, but gate byte zero | Fallback position and dimensions are used, while top adjustment 3 and left adjustment 2 are preserved. |
| Horizontal center 10, vertical center 10, width 0, height 0 | Calculated position (10, 10) and zero dimensions can be retained. There is no explicit positive-dimension requirement. |
| Horizontal center 10, vertical center 10, width −3, height −3 | Truncated half-extents are −1; calculated position (11, 11) and dimensions (−3, −3) can be retained. There is no explicit negative-dimension rejection. |
| Horizontal center 10, vertical center 10, width 5, height 5 | Position (8, 8), dimensions (5, 5), demonstrating truncation of positive odd half-extents. |

## Uncertainties

The following are **documented_only**, not behavior to invent:

- **Domain meaning:** The roles of the status byte, gate byte, parent comparison word, opaque value, and all-one arguments are unknown beyond their observable uses.
- **Dependency behavior:** No implementation or behavioral contract for F-a05963f65233 is supplied.
- **Overflow:** Signed subtraction and addition can exceed the signed 32-bit range for extreme inputs. The supplied representation does not independently establish a portable overflow contract. Saturation, rejection, or arbitrary-precision arithmetic must not be assumed.
- **Address arithmetic:** Boundary wrapping, pointer validity, alignment requirements, and fault behavior are not established.
- **Byte order:** Relevant when record writes overlap the parent comparison word; not specified independently.
- **Concurrency:** No synchronization or atomicity guarantee is shown. Intermediate status writes may matter to an external observer, but a supported concurrent-access model is not established.
- **Ownership:** No lifetime, retention, or release guarantees can be inferred from forwarding an address-bearing or opaque value.
- **Failure propagation:** The function has no explicit error channel. Faults, exceptions, nonlocal exits, or nonreturning behavior in the dependency are unspecified.

## Evidence

The supplied static function body establishes:

- Seven incoming arguments and no explicit return value.
- Signed half-extent calculations and the exact boundary comparisons.
- An unconditional status write followed by conditional clearing writes.
- Left and top adjustments without corresponding right and bottom reductions.
- Conditional status, gate-byte, and comparison-word reads.
- Selection between calculated geometry and the fixed fallback.
- One thirteen-argument call to F-a05963f65233, including repeated opaque context, two all-one values, and the original horizontal center.

The supplied dependency mapping establishes the opaque dependency identity used here. No independent dependency implementation, runtime trace, memory-layout documentation, or target arithmetic specification was provided. No runtime tests have been performed.
