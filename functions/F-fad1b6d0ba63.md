<!-- Trusted provenance: F-fad1b6d0ba63; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fad1b6d0ba63

## Purpose

F-fad1b6d0ba63 selects one active point from a sequence of point groups, finds its nearest active predecessor and successor within that group’s circular ordering, and compares two results returned by dependency F-39d3b95c505b.

The selected point is the first strictly lowest first-coordinate value found across eligible groups, subject to an initial selection ceiling of 32768. A group is eligible only if it contains at least three active points.

The function returns 1 when the dependency result for the successor displacement is strictly less than the dependency result for the predecessor displacement. It returns 0 otherwise, including when processing is disabled or no point qualifies.

The interpretation of these dependency results as distances, costs, or another geometric measure is not established. “Point,” “coordinate,” and “group” describe the observed data organization, not a verified application-level purpose.

## Interface

### Input

The function accepts one nullable pointer to a descriptor. The supplied representation uses the following layout:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 16 bits | Signed number of group-end entries |
| 2 | 16 bits | Signed processing gate |
| 4 | 32 bits | Pointer to the point-record array |
| 8 | 32 bits | Pointer to the activity-byte array |
| 12 | 32 bits | Pointer to the group-end array |

This layout reflects the supplied 32-bit representation. It is not a portable layout for an environment with wider pointers.

Each point record occupies eight bytes:

| Record-relative byte offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Signed first coordinate |
| 4 | 32 bits | Signed second coordinate |

Each activity entry occupies one byte. Bit 0 determines whether the corresponding point is active. Other bits do not affect this function’s decisions.

Each group-end entry is a signed 16-bit point index. It identifies the inclusive end of a group. The first group starts at point index 0. Every later group starts immediately after the preceding group’s endpoint.

The processing gate is not a group count, point count, or coordinate bound. Any strictly positive signed 16-bit value enables the same processing path.

### Output

The return value is a 32-bit value restricted by the visible implementation to:

- **1:** a qualifying point was selected, and the successor dependency result is strictly smaller than the predecessor dependency result;
- **0:** every other normal-return case.

No separate error code is exposed.

### Preconditions for a defined, terminating interpretation

When the descriptor is non-null and its processing gate is positive:

- The descriptor fields that are reached must be readable.
- A positive group count requires a readable group-end array with that many entries.
- Endpoint indices, point-array extents, and activity-array extents must support every address formed and every access performed.
- For ordinary nonoverlapping groups, endpoints must be nonnegative and strictly increasing.
- Relevant pointer arithmetic and comparisons must be meaningful in the target execution environment.
- Both coordinate-subtraction pairs passed to the dependency must have defined arithmetic behavior, or the target’s overflow behavior must be specified separately.
- Data must remain sufficiently stable for the activity searches to find the active neighbors established by the earlier scan.

The function does not validate these conditions. Malformed inputs are not converted into a documented failure return.

## Behavior

### Disabled processing

A null descriptor returns 0 without dereferencing the descriptor or calling the dependency.

For a non-null descriptor, the processing gate at byte offset 2 is read as a signed 16-bit value. A zero or negative gate returns 0 without traversing the arrays or calling the dependency.

Only a strictly positive gate enables group processing. Its magnitude has no further effect.

### Group traversal and eligibility

For enabled processing, the group-end and point-array pointers are obtained before group traversal. The group count determines the group-end-array traversal extent.

Groups are considered in group-end-array order. Within a normally formed group, point records are considered in ascending index order, including both endpoints.

A group whose endpoint precedes its third record is not scanned for activity or coordinate selection. For normally formed groups, this means that groups with fewer than three records are skipped.

For each scanned record:

1. The corresponding activity byte is obtained through the activity-array pointer held in the descriptor.
2. Bit 0 contributes either zero or one to the group’s active-point count.
3. The first coordinate is considered for the group minimum.
4. That record becomes the group’s candidate only when it is active and its first coordinate is strictly smaller than the group’s current minimum.

The first-coordinate comparison is present even for inactive records. Inactive point records therefore cannot be assumed unread merely because their activity bit is clear.

Every group starts with a minimum threshold of 32768. Consequently:

- An active point with first coordinate 32767 or less can become a candidate.
- A first coordinate of exactly 32768 does not qualify.
- A group whose active points all have first coordinates of 32768 or more has no qualifying candidate.
- Negative first coordinates are compared as signed 32-bit values.

A group can update the overall selection only after its scan establishes that it has at least three active points.

### Selection across groups

The overall minimum also starts at 32768.

An eligible group replaces the current selection only if its candidate’s first coordinate is strictly smaller than the current overall minimum. Replacement records the candidate and that group’s inclusive boundaries.

These strict comparisons establish the following tie rules for stable input:

- Within a group, the earliest active point attaining the qualifying minimum wins.
- Between eligible groups with the same qualifying minimum, the earlier group wins.
- A later equal coordinate never displaces an earlier selection.
- A group with fewer than three active points never contributes a selection, even if it contains a lower first coordinate.

All groups are processed before neighbor discovery begins. The function does not stop after finding its first eligible group.

If no candidate survives these rules, the function returns 0 without calling F-39d3b95c505b.

### Circular active-neighbor discovery

After selection, the function searches only within the selected group.

The predecessor search begins one record before the selected point. If the selected point is the group’s first record, the search begins at the group’s last record instead.

It then moves backward, wrapping from the group’s first record to its last record, until it encounters a record whose activity bit 0 is set.

The successor search begins one record after the selected point. If the selected point is the group’s last record, the search begins at the group’s first record instead.

It then moves forward, wrapping from the group’s last record to its first record, until it encounters an active record.

The predecessor search completes before the successor search starts. Inactive records are skipped based on activity bytes; their coordinates are not used for the subsequent displacement calls.

For stable, well-formed input, the requirement for at least three active points ensures that the selected point has distinct active predecessor and successor records. No explicit iteration limit protects either search if activity data changes or the group boundaries are invalid.

### Dependency calls and final comparison

Exactly two dependency calls occur after successful selection and neighbor discovery.

The first call receives the predecessor’s displacement from the selected point:

- predecessor first coordinate minus selected first coordinate;
- predecessor second coordinate minus selected second coordinate.

The second call receives the successor’s displacement from the selected point in the same coordinate order.

Both displacement components are formed using signed 32-bit coordinate operands. Each dependency result is subsequently treated as a signed 32-bit integer.

The final return is:

- 1 if the second result is strictly less than the first;
- 0 if the second result is equal to or greater than the first.

The function does not compare raw coordinate differences directly and does not establish any mathematical properties of the dependency.

## Side effects

### Direct memory effects

The visible function performs no explicit stores into the descriptor, point records, activity bytes, or group-end entries.

Its observable input reads are not limited to the ultimately selected group:

- The enabled path obtains descriptor array information before traversal.
- Group-end entries are read in traversal order.
- Scanned point records are visited in ascending order within each group.
- Activity bytes are read before the associated minimum-selection decision.
- First coordinates are considered during selection, including for inactive records.
- Activity bytes in the selected group are read again during predecessor and successor discovery.
- Coordinate values used for displacement arguments are read after neighbor discovery.

The activity-array pointer is obtained from the descriptor at the activity-access sites rather than captured once for the whole operation. The function therefore does not establish an immutable activity-array snapshot.

The supplied representation establishes the predecessor search before the successor search, and the predecessor dependency call before the successor dependency call. It does not establish a portable evaluation order among individual reads used to construct a single call’s arguments.

### Ownership and lifetime

The visible function does not allocate, release, or transfer memory ownership.

The caller must provide valid storage for all reached accesses. A successful initial scan does not establish that later reads use unchanged data.

No ownership or lifetime guarantee is established for F-39d3b95c505b. In particular, the absence of pointer arguments is not evidence that the dependency cannot modify global state, affect shared storage, or release memory through other references.

Because the successor displacement is formed after the predecessor call returns, dependency side effects can affect the second call’s inputs or their validity.

### Failure behavior

There is no visible null-array check, bounds check, arithmetic-error check, recovery path, or dependency-error translation.

Unreadable memory, invalid pointer arithmetic, nonterminating neighbor searches, or abnormal dependency behavior are outside the established normal-return contract. They must not be described as reliably returning 0.

## Dependencies

### F-39d3b95c505b

**Observed interface:** two signed 32-bit displacement components supplied as scalar arguments; a result used as a signed 32-bit integer.

**Observed call order:**

1. Active predecessor displacement.
2. Active successor displacement.

**Observed call count:** zero if processing is disabled or selection fails; otherwise two, provided execution reaches both calls normally.

**Unknown behavior:** its mathematical operation, accepted input range, overflow handling, global effects, memory effects, and abnormal-return behavior are not supplied.

The caller uses only a strict signed comparison between the two results. It does not test either result for a sentinel, error indication, or special range.

## Acceptance examples

These examples are conditional static expectations, not runtime observations. Except where stated otherwise, assume valid storage, stable input, defined arithmetic, and normally returning dependency calls.

### Null or disabled descriptor

- A null descriptor returns 0 with no dependency calls.
- A readable descriptor with gate 0 returns 0 without array traversal.
- A descriptor with gate −1 also returns 0.
- Changing a positive gate from 1 to 7 does not, by itself, change traversal or selection.

### Too few records or active points

A group containing two records is skipped.

A group containing five records but only two active points cannot supply the selected point, even if one active first coordinate is extremely low.

If every group is ineligible, the result is 0 and the dependency is not called.

### Selection ceiling

A group has three active records with first coordinates 32768, 40000, and 50000. No point in that group qualifies for selection.

Changing the first coordinate of the first record to 32767 makes it a qualifying candidate, provided no other eligible group supplies a lower value.

### Tie handling

A group’s active first coordinates, in record order, are 4, −3, −3, and 9. The second record wins the group selection.

If a later eligible group also has minimum −3, it does not replace that selection. If its minimum is −4, it does replace it.

### Activity-byte interpretation

Activity bytes 1 and 129 both mark records active. Activity bytes 0 and 128 both mark records inactive.

Changing only bits other than bit 0 does not change the function’s activity decisions.

### Circular neighbors and inactive gaps

A five-record group has activity pattern active, inactive, active, inactive, active. If its first record is selected:

- Its predecessor is the fifth record.
- Its successor is the third record.
- The predecessor dependency call occurs first.

This demonstrates both wraparound and skipping of inactive records.

### Displacement direction and result comparison

Suppose the selected point is at coordinates (10, 20), its active predecessor at (7, 24), and its active successor at (12, 19).

The first dependency call receives (−3, 4). The second receives (2, −1).

Conditional on dependency returns of 9 and 4 respectively, the function returns 1. Conditional on returns of 4 and 4, it returns 0. Conditional on returns of −5 and −2, it returns 0 because −2 is not less than −5.

These cases specify comparison behavior without assuming what the dependency computes.

## Uncertainties

- **documented_only — dependency semantics:** No evidence establishes a distance, norm, angle, or other mathematical interpretation.
- **documented_only — malformed descriptors:** Negative group counts, negative or decreasing endpoints, overlapping groups, and out-of-range indices have no validated-input behavior. The visible traversal is not an input validator.
- **documented_only — arithmetic overflow:** Subtracting two signed 32-bit coordinates can exceed the signed 32-bit range. Defined wrapping, trapping, or another result is not established.
- **documented_only — concurrent mutation:** The function uses repeated reads rather than a snapshot. Behavior under concurrent changes is not specified.
- **documented_only — dependency side effects:** The dependency may affect state used by the second call. Preservation of globals and input storage is not established.
- **documented_only — machine-level read details:** The supplied representation supports logical read dependencies and operation ordering, not an exhaustive instruction-level memory trace.
- **Inferred intent:** Selecting a low-first-coordinate active point and comparing its neighboring displacements suggests geometric processing. This interpretation is not required for the behavioral contract.

## Evidence

The established behavior comes from static inspection of the supplied function representation: descriptor offsets, signed field widths, eight-byte record spacing, activity-bit tests, strict minimum comparisons against 32768, active-count eligibility, circular neighbor searches, ordered dependency calls, and the final signed comparison.

The dependency’s implementation was not supplied. Its name and apparent use do not establish its semantics.

No implementation access, execution, runtime tests, or dynamic memory observations were used. The acceptance examples are conditional consequences of the visible control flow and arithmetic, not reports of executed verification.
