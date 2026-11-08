<!-- Trusted provenance: F-04cc69fad3ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-04cc69fad3ed

## Purpose

F-04cc69fad3ed assigns a mapped coordinate to each selected record using two anchor records.

Each record supplies a signed coordinate at byte offset `0x1c`; the function writes its result at byte offset `0x18`. The two anchors define:

- A constant coordinate displacement on either exterior side of the anchor interval.
- A dependency-mediated mapping strictly inside the interval.
- A threshold between two displacements when the anchor coordinates coincide.

The apparent intent is piecewise coordinate remapping with interpolation between anchors and displacement-preserving extension outside them. The exterior behavior is established by the supplied static evidence. The interpolation formula, including rounding and overflow behavior, is not established because the implementation of F-8f41f510c86c is unavailable.

## Interface

The function accepts four arguments and returns no value:

| Argument | Role |
|---|---|
| First record address | Unsigned address of the first candidate record to update. |
| Inclusive address limit | Unsigned bound against which each candidate record address is compared. It need not identify an actual record. |
| First anchor address | Address of the first anchor record. |
| Second anchor address | Address of the second anchor record. |

The recovered interface uses unsigned address arguments for the record traversal and signed integer arguments holding the anchor addresses. The latter are used as addresses, not as signed coordinate values.

The intended representation is a 32-bit address and integer model: four-byte coordinate fields, signed 32-bit coordinate comparisons, and unsigned 32-bit traversal comparisons. The supplied text does not include the underlying integer type definitions or target-language rules; those limitations matter for overflow and address wraparound.

### Record layout

Only two fields are relevant:

| Byte offset | Width | Meaning |
|---|---:|---|
| `0x18` | 4 bytes | Mapped coordinate: read from anchors and written to selected records. |
| `0x1c` | 4 bytes | Source coordinate: read from anchors and selected records. |

Candidate record addresses advance by `0x28` bytes. This establishes a 40-byte traversal stride, not a complete record schema. Other bytes are not explicitly accessed by this function.

### Notation

For the behavior below:

- \(a\) is the first anchor’s source coordinate, captured on entry.
- \(b\) is the second anchor’s source coordinate, captured on entry.
- \(A_0\) and \(B_0\) are the mapped coordinates read from the first and second anchors during initialization.
- \(d_A=A_0-a\) and \(d_B=B_0-b\) are the cached exterior displacements.
- \(x\) is a selected record’s source coordinate.
- \(A_{\mathrm{pre}}\) and \(B_{\mathrm{pre}}\) are mapped-coordinate values read from the anchors while preparing an interior dependency call.
- \(A_{\mathrm{post}}\) is the first anchor’s mapped coordinate read after that call returns.

These are mathematical names for contract concepts, not additional stored fields.

### Preconditions

Normal completion requires:

- Both anchor fields to be readable during initialization, even when no records will be selected.
- Each visited record’s source field to be readable and mapped field to be writable.
- Any additional anchor reads on interior paths to be valid.
- Arithmetic and address calculations to have defined behavior in the execution environment.
- Any invoked dependency to return normally if continued traversal is expected.

There is no explicit validation of pointers, alignment, region size, record count, arithmetic range, or dependency results.

## Behavior

### 1. Initialize from both anchors

Before deciding whether the traversal range is empty, the function performs these logical reads and calculations:

1. Read the first anchor’s source coordinate.
2. Read the second anchor’s source coordinate.
3. Read the first anchor’s mapped coordinate and calculate \(d_A\).
4. Read the second anchor’s mapped coordinate and calculate \(d_B\).

Thus an empty traversal is not a no-access operation.

The captured source coordinates and exterior displacements remain fixed throughout the invocation. Later changes to anchor memory do not update those cached quantities.

### 2. Select records by unsigned address comparison

If the first record address exceeds the inclusive limit under unsigned comparison, the function performs no record updates and returns.

Otherwise it processes the current candidate address, adds `0x28`, and repeats while the resulting candidate address is less than or equal to the limit under unsigned comparison.

Consequences:

- Equal first and limit addresses select one record, provided the next increment does not wrap back into the permitted range.
- The limit need not be stride-aligned.
- A candidate is selected by its starting address alone. The function does not check that its accessed fields fit below the limit.
- Records are handled in traversal order, with each result written before advancing.

For a nonwrapping traversal, the number of selected records is:

\[
1+\left\lfloor\frac{\text{limit}-\text{first}}{40}\right\rfloor.
\]

This count does not describe executions in which the traversal address wraps.

### 3. Distinct anchors in ascending source-coordinate order

When \(a<b\), each selected record is handled as follows:

| Source-coordinate region | Result |
|---|---|
| \(x\le a\) | \(x+d_A\) |
| \(a<x<b\) | Interior dependency path |
| \(x\ge b\) | \(x+d_B\) |

The anchor boundaries use the exterior paths, not the dependency.

For the interior path, the function invokes F-8f41f510c86c with three signed integer arguments:

\[
\left(x-a,\;B_{\mathrm{pre}}-A_{\mathrm{pre}},\;b-a\right).
\]

After the dependency returns, the function rereads the first anchor’s mapped coordinate and adds it to the returned integer:

\[
\text{result}
=
\text{dependency result}+A_{\mathrm{post}}.
\]

It then writes the result into the current record’s mapped field.

The mapped-coordinate difference passed to the dependency is therefore not necessarily \(B_0-A_0\), and the final addition is not necessarily based on the same first-anchor value used to construct that difference.

### 4. Distinct anchors in descending source-coordinate order

When \(a>b\), the geometric regions are reversed, but the original anchor identities remain significant:

| Source-coordinate region | Result |
|---|---|
| \(x\le b\) | \(x+d_B\) |
| \(b<x<a\) | Interior dependency path |
| \(x\ge a\) | \(x+d_A\) |

The interior call has exactly the same argument relationships as in the ascending case:

\[
\left(x-a,\;B_{\mathrm{pre}}-A_{\mathrm{pre}},\;b-a\right).
\]

The first and third arguments are negative for a mathematically ordinary descending interior point. The function does not exchange the anchors, take absolute values, or normalize the signs before calling the dependency.

After the call, it rereads the first anchor’s mapped coordinate and adds that value to the dependency result before storing.

### 5. Coincident source coordinates

When \(a=b\), no dependency call occurs. The common source coordinate is a threshold:

| Source-coordinate region | Result |
|---|---|
| \(x\le a\) | \(x+d_A\) |
| \(x>a\) | \(x+d_B\) |

Equality favors the first anchor’s displacement. Different mapped coordinates at the same anchor source coordinate are not rejected or averaged.

The supplied representation reads the current source field to choose the displacement and reads it again for the final addition. Under stable ordinary memory, both reads yield the same \(x\). The source alone does not establish a contract for externally changing or device-backed memory.

### 6. Store and continue

Each selected record receives one explicit four-byte store to offset `0x18`.

The function does not explicitly write the source field or any other record field. It does not read the old destination merely to preserve or combine it; any read of the same memory would arise through aliasing with an anchor or another accessed field.

There is no success flag, updated-record count, or explicit error result.

## Side effects

### Direct memory effects

The function directly:

- Reads two fields from each anchor during initialization.
- Reads the source field of each visited record.
- Performs additional anchor mapped-field reads for interior records.
- Writes each visited record’s mapped field in traversal order.

The two anchor mapped-field reads used to form a dependency argument occur before the dependency call. Their relative evaluation order within that subtraction is not established by the supplied expression.

The first anchor’s mapped-field read used for the final addition occurs after the dependency returns.

### Aliasing and mutable state

The function does not enforce separation between anchors, records, or partially overlapping storage.

Consequently:

- Earlier record writes may affect later reads if the accessed storage overlaps.
- Cached anchor source coordinates and cached exterior displacements do not change after initialization.
- Interior calls can observe changed anchor mapped coordinates through their fresh reads.
- A dependency-induced change to the first anchor’s mapped field can affect the result of the very record whose dependency call caused that change.

The function cannot be treated as operating on an immutable snapshot of both anchors.

No direct allocation, deallocation, ownership transfer, or global-state access is visible. Nothing in the evidence establishes that the dependency preserves globals, leaves passed-through state unchanged, or refrains from allocating or releasing memory.

### Failure behavior

There are no explicit error branches.

Invalid memory access, unsupported alignment, arithmetic overflow, dependency failure, or nontermination is not converted into a reported status. The behavior of such executions is not established here.

The function has no visible rollback mechanism. If execution stops after earlier record stores, those stores have already occurred.

## Dependencies

### F-8f41f510c86c

This is the only visible function dependency.

It is called once for each visited record whose source coordinate lies strictly between the two distinct cached anchor source coordinates. It is not called for exterior records, boundary records, coincident anchors, or an empty traversal.

Its visible interface is:

- Three signed integer arguments.
- One signed integer return value, which is used without validation.

The arguments represent:

1. Displacement from the first anchor’s cached source coordinate.
2. A freshly read difference between the anchors’ mapped coordinates.
3. The cached second-to-first source-coordinate difference.

**Inferred intent:** the dependency may implement a multiply/divide-style interpolation operation.

**Not established:** its formula, rounding, intermediate precision, saturation, signed-overflow behavior, error behavior, memory effects, ownership behavior, and global-state effects.

Distinct anchors make the third argument mathematically nonzero. That observation does not establish that the subtraction is representable in a signed 32-bit integer or that all dependency inputs are valid.

## Acceptance examples

These examples assume valid stable memory, representable arithmetic, no traversal wraparound, and no dependency side effects unless explicitly stated otherwise.

### Ascending anchors: exterior and boundary behavior

Let the anchors contain:

- First anchor: source `10`, mapped `40`.
- Second anchor: source `30`, mapped `100`.

The cached displacements are `30` and `70`.

| Record source | Required mapped result | Dependency call |
|---:|---:|---|
| `5` | `35` | None |
| `10` | `40` | None |
| `30` | `100` | None |
| `35` | `105` | None |

For source `20`, the dependency arguments must be `(10, 60, 20)`. If the dependency returns `30` and the first anchor’s mapped value remains `40`, the stored result must be `70`. The return value `30` is a condition of this example, not a proven dependency behavior.

### Descending anchors preserve argument signs

Let the first anchor be source `30`, mapped `100`, and the second be source `10`, mapped `40`.

- Source `5` produces `35`.
- Source `10` produces `40`.
- Source `30` produces `100`.
- Source `35` produces `105`.

Source `20` invokes the dependency with `(-10, -60, -20)`. If it returns `-30` and the first anchor’s mapped value remains `100`, the result is `70`.

An implementation that silently reorders anchors and passes only positive distances would not reproduce the evidenced dependency-call contract.

### Coincident anchors select a displacement at a threshold

Let both source coordinates be `10`, with first mapped coordinate `40` and second mapped coordinate `100`.

- Source `9` produces `39`.
- Source `10` produces `40`.
- Source `11` produces `101`.

No dependency calls occur. The discontinuity is permitted by the evidenced behavior.

### Empty traversal still requires anchor access

If the unsigned first address exceeds the limit, no candidate record is updated and no dependency is invoked. Both anchor source fields and both anchor mapped fields are nevertheless read during initialization.

An implementation that skips all anchor accesses before checking the range would change this access behavior.

### Inclusive, nonaligned limit

Suppose candidate addresses are \(P\), \(P+40\), and \(P+80\), without wraparound. A limit of \(P+79\) selects the first two candidates, not the third.

The limit does not constrain where the fields of the second selected record end.

### Dependency changes the final base value

Using ascending anchors from the first example, consider source `20`.

If the dependency receives `(10, 60, 20)`, returns `30`, and changes the first anchor’s mapped field to `50` before returning, the stored result is `80`.

The exterior displacement for subsequent records remains the cached `30`, unless their own accessed memory has changed.

## Uncertainties

The following are **documented_only**, not silently resolved:

- Exact target type definitions and language-level arithmetic semantics.
- Signed overflow in displacement calculations, dependency arguments, or final additions.
- Whether unsigned traversal wraparound leads to additional visits, repeated addresses, or nontermination for a particular range.
- Exact dependency semantics and side effects.
- Alignment and mapped-memory requirements of the execution environment.
- Behavior under concurrent modification, device-backed memory, or invalid aliasing.
- Guaranteed termination or failure signaling from the dependency.
- Any complete record schema beyond the two accessed fields and traversal stride.

A faithful contract must not substitute a conventional interpolation formula, introduce clamping, reject coincident anchors, add implicit allocation, or assume dependency purity.

## Evidence

This specification is based solely on the supplied static function representation and the supplied mapping of its single visible dependency to F-8f41f510c86c.

Established evidence includes the field offsets, traversal stride, comparison directions, initialization before the empty-range decision, cached versus refreshed anchor values, dependency argument relationships, boundary handling, and destination stores.

Piecewise coordinate remapping is inferred intent. The dependency’s numerical operation and execution-environment details remain unknown.

No runtime tests have been performed.
