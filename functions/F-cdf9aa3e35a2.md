<!-- Trusted provenance: F-cdf9aa3e35a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdf9aa3e35a2

## Purpose

F-cdf9aa3e35a2 adjusts three signed-integer fields in caller-supplied storage according to a signed threshold. The fields have different rules: one is threshold-adjusted without exceptions, one preserves two negative sentinel values, and one replaces a particular sentinel using an indirect integer read.

The contract concerns field values, conditional writes, and their ordering. It does not establish what the record represents or why its fields need adjustment.

**Inferred intent:** The operation could maintain positions or indices after a change elsewhere. Neither that interpretation nor any application-specific meaning of the negative values is established by the supplied evidence.

## Interface

### Arguments and result

The function accepts:

- A pointer to mutable, integer-addressed record storage.
- A signed-integer threshold.

There is no explicit return value, status result, or returned pointer. Results are communicated through mutations of the record.

### Record layout and widths

The record contains at least four consecutive integer slots:

| Slot position | Contract name | Use |
|---|---|---|
| First | Reference field | Integer value used to form an address when indirect replacement is selected |
| Second | Ordinary field | Threshold-adjusted without sentinel exemptions |
| Third | Protected field | Threshold-adjusted except for −4 and −3 |
| Fourth | Replacement field | Threshold-adjusted, except that −3 selects indirect replacement |

The supplied representation uses the same signed-integer type for the threshold, the four record slots, and the indirectly read value. Its exact width is not explicitly established.

If that type occupies four bytes, the respective byte offsets are 0, 4, 8, and 12, and the record needs at least 16 bytes for these slots. Those offsets are conditional on the four-byte integer model; the literal addition of four used for the indirect address does not, by itself, prove the slot width.

The reference field is integer-valued storage, not an established native-pointer-width field. In particular, the evidence does not justify enlarging it to the pointer width of a different execution environment.

### Conditional indirect location

When the replacement field contains −3, the function obtains an indirect location by:

- Reading the reference field as an integer.
- Adding the integer value four.
- Interpreting the result as an address suitable for reading one integer.

The addition occurs in the integer domain before address interpretation. It is not an advance by four integer elements. Under the conventional interpretation in which integer address values identify byte addresses, it denotes a four-byte displacement.

The integer read from that location is called the **referenced value** in this contract.

### Preconditions

Normal completion requires:

- Readable storage for the ordinary, protected, and replacement fields.
- Writable storage for every field whose selected rule requires an assignment.
- When indirect replacement is selected, a readable reference field and a valid indirect integer read.
- Alignment compatible with the target’s integer-access requirements.
- Defined arithmetic and address interpretation for all operations actually selected.

An ordinary field below the threshold does not require a write to that field. The analogous conditional-write distinction also applies to the other fields.

If the replacement field is not −3, the reference field’s value is not consulted. An unusable reference value alone therefore does not prevent completion on that path.

No concurrency, synchronization, or device-memory guarantees are established.

## Behavior

### Value contract

For ordinary accessible memory without external mutation, the final values satisfy the following rules. “Incoming value” means the value read when the relevant field is processed; the indirect read occurs later than the ordinary-field adjustment.

| Field | Condition on its incoming value | Required result |
|---|---|---|
| Ordinary | Below the threshold | Unchanged; no assignment |
| Ordinary | Equal to or above the threshold | Increased by one |
| Protected | −4 or −3 | Unchanged; no assignment |
| Protected | Any other value below the threshold | Unchanged; no assignment |
| Protected | Any other value equal to or above the threshold | Increased by one |
| Replacement | −3 | Replaced with the referenced value minus one |
| Replacement | Any other value below the threshold | Unchanged; no assignment |
| Replacement | Any other value equal to or above the threshold | Increased by one |

All threshold comparisons are signed. Equality belongs to the incrementing case unless a field-specific sentinel rule overrides it.

The ordinary field has no sentinel exemptions. For example, an ordinary-field value of −3 becomes −2 when the threshold is −3.

The protected field preserves both −4 and −3 regardless of the threshold. The replacement field preserves neither as a general rule: −3 requests indirect replacement, while −4 follows the ordinary threshold rule.

### Ordering constraints

The observable logical order is:

1. Any ordinary-field assignment precedes processing of the replacement field.
2. Replacement-field processing, including its conditional indirect read and assignment, precedes processing of the protected field.
3. Any protected-field assignment is last.

These ordering constraints matter when the indirect location aliases the record and when an access fails after earlier effects.

The replacement-field sentinel rule is independent of threshold ordering. A value of −3 requests indirect replacement whether it is below, equal to, or above the threshold.

Each field is considered once. A value produced by an assignment is not fed back through that field’s rules during the same invocation. Consequently:

- A replacement result of −3 does not trigger a second indirect read.
- Incrementing a replacement-field value of −4 to −3 does not trigger indirect replacement.
- Incrementing an ordinary-field value to a sentinel value has no additional effect.

### Arithmetic

Every selected increment adds one. Indirect replacement subtracts one from the referenced value. Address formation separately adds four to the reference-field integer.

No overflow detection, saturation, range validation, or arithmetic-error result is present.

For an integer type with maximum value **maximum** and minimum value **minimum**:

- A selected increment of **maximum** exceeds the signed range.
- Indirect replacement using **minimum** exceeds the signed range.
- Adding four to a reference value greater than **maximum − 4** exceeds the signed range before address interpretation.

Under a signed 32-bit model, the endpoints are −2,147,483,648 and 2,147,483,647. The evidence does not establish wrapping behavior at either endpoint.

Avoiding integer overflow is not sufficient to validate indirect addressing. Conversion of the resulting integer to an address, the destination’s accessibility, and its alignment remain separate requirements.

### No-update paths

Leaving a field unchanged is not equivalent to assigning its previous value back.

In particular, an ordinary replacement-field value below the threshold has no assignment. By contrast, a replacement-field value of −3 selects an assignment even when the referenced value minus one is also −3.

The protected field’s preserved values do not cancel earlier work. An invocation may update the ordinary and replacement fields while leaving the protected field untouched.

## Side effects

### Direct writes

The only direct write destinations shown are the ordinary, replacement, and protected fields. Their conditional writes occur in that order.

The reference field is not directly modified. The indirect location is read, not explicitly written through the derived address.

These statements describe the supplied representation’s logical accesses. They do not establish an exact instruction trace, atomic access widths, or guarantees about separate physical loads for repeated ordinary-memory expressions.

### Aliasing

No evidence requires the indirect location to be disjoint from the record.

If that location identifies:

- **The ordinary field:** the indirect read observes its value after any ordinary-field increment.
- **The protected field:** the indirect read observes its value before any protected-field increment.
- **The replacement field:** the indirect read observes −3, and the replacement assignment stores −4.

These consequences assume the address can validly be formed and used, and that no external agent changes the memory.

The referenced value must therefore not be modeled as an immutable input captured at function entry.

### Ownership and lifetime

The function contains no allocation, release, resource acquisition, or ownership-transfer operation. It uses storage supplied through the record and, conditionally, through the integer-derived address.

The evidence does not establish who owns either region or how long either remains valid outside the invocation. No ownership conclusion is needed to describe the function’s accesses.

### Errors and partial effects

There is no explicit null check, bounds check, indirect-address validation, overflow report, or recoverable error channel.

An earlier assignment can precede a later invalid access. For example, the ordinary field can be incremented before an invalid indirect read is attempted. There is no explicit rollback mechanism.

The exact outcome of invalid storage, misalignment, unrepresentable arithmetic, or an unusable integer-to-address conversion is **documented_only**. No particular fault, exception, termination mode, or recoverable final state is promised.

## Dependencies

No calls to other functions appear in the supplied body. There are therefore no additional opaque function dependency IDs.

The operation depends on:

- The caller-supplied record storage.
- One conditional indirect integer read.
- The target’s signed-integer representation and arithmetic semantics.
- Integer-to-address conversion, address validity, and alignment rules.

No named global state is directly accessed. Caller-supplied or integer-derived addresses could nevertheless identify globally allocated storage; the evidence does not constrain their allocation origin.

Because there are no callees, the contract makes no assumptions about callee preservation of globals or callee ownership of memory.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Unless otherwise stated, assume ordinary accessible memory, no external interference, valid address formation where needed, and representable arithmetic.

Triples list **ordinary, protected, replacement** fields.

| Initial triple | Threshold | Referenced value, if required | Expected final triple |
|---|---:|---:|---|
| (4, 4, 4) | 4 | Not read | (5, 5, 5) |
| (3, 3, 3) | 4 | Not read | (3, 3, 3) |
| (7, −4, 2) | 5 | Not read | (8, −4, 2) |
| (−3, −3, 0) | −3 | Not read | (−2, −3, 1) |
| (0, 0, −4) | −4 | Not read | (1, 1, −3) |
| (0, 0, −3) | 100 | 10 | (0, 0, 9) |
| (0, −3, −3) | −10 | 0 | (1, −3, −1) |

Additional acceptance conditions capture effects not established by final-value comparisons alone:

- **Unused reference:** With a replacement-field value other than −3, no reference-field read or derived indirect read is required, even if the reference value would be unusable.
- **Ordinary-field alias:** Starting from (5, 0, −3) with threshold 5, let the indirect location identify the ordinary field. The referenced value is 6, giving final fields (6, 0, 5).
- **Protected-field alias:** Starting from (0, 8, −3) with threshold 5, let the indirect location identify the protected field. The referenced value is 8, giving final fields (0, 9, 7).
- **Replacement-field alias:** Starting from (0, 0, −3) with threshold 5, let the indirect location identify the replacement field. The final fields are (0, 0, −4).
- **Assignment without value change:** With replacement value −3 and referenced value −2, the replacement assignment stores −3. It still constitutes an assignment, and no second indirect read follows.
- **Sentinel produced by increment:** With replacement value −4 and threshold −4, the result is −3 without any indirect read.
- **Protected negative sentinels:** Both −4 and −3 remain unchanged in the protected field even when the threshold is lower than either.
- **Arithmetic boundary:** A selected increment of the signed maximum has no established acceptance result. The same restriction applies to subtracting one from the signed minimum and to overflowing the reference integer while adding four.

## Uncertainties

The following remain **documented_only**:

- Exact signed-integer width, record byte size, and target ABI.
- Pointer width and the mapping from integer values to usable addresses.
- Address interpretation outside conventional byte-addressed environments.
- Overflow behavior for increments, indirect-value subtraction, and reference-value addition.
- Failure behavior for null, inaccessible, or misaligned storage.
- Atomicity, concurrent mutation, synchronization, and memory-mapped-device behavior.
- Application meanings of the fields, threshold, and sentinel values.
- Caller invariants that may rule out aliasing, negative values, or arithmetic boundaries.

The four-byte displacement under conventional byte-address interpretation must not be confused with proof of four-byte record slots. Likewise, a plausible position-adjustment purpose does not establish application-level validity constraints.

## Evidence

**Established static evidence:** The supplied body contains three conditional field adjustments with ordinary-field effects before replacement-field effects and protected-field effects last. It establishes signed comparisons, the distinct sentinel policies, conditional integer-derived addressing using an addition of four, an indirect integer read followed by subtraction of one, and the absence of explicit calls, allocation, release, validation, or rollback.

**Inferred intent:** Threshold-relative position or index maintenance. This interpretation is not required for any specified result.

**Unknowns:** Target-dependent widths, arithmetic and address semantics, application invariants, and invalid-access outcomes remain as listed above.

No runtime tests have been performed. The acceptance examples specify consequences of the supplied static evidence under their stated preconditions; they are not observations of execution.
