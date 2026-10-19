<!-- Trusted provenance: F-fb6e215aa4fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fb6e215aa4fd

## Purpose

F-fb6e215aa4fd returns an unsigned scalar selected from fields located at byte offset 8 relative to two supplied address bases.

For valid, stable fields, the result is their unsigned maximum. This interpretation depends on stability: the supplied representation reads the first field more than once and conditionally reads the second field again. If the execution environment permits those reads to observe different values, the result need not be the maximum of the values used in the comparison.

The established operation is field selection. The evidence does not establish the fields’ application-level meaning, the containing objects’ types, or a broader ordering purpose.

## Interface

### Inputs

The function accepts two integer-valued address bases, represented by a signed integer type.

Each base participates in an integer addition of 8, after which the result is used as the address of an unsigned scalar field. The arguments are therefore not the scalar values being compared. An argument numerically equal to 12 identifies a field through the calculation 12 + 8; it does not request comparison against the value 12.

Only the computed field locations are accessed. The function does not first read memory at either unadjusted base.

A base value of zero receives no special handling. It is not independently rejected or independently disqualified by the visible operation. Its validity depends on whether adding 8 and converting the result to an address are valid under the execution model, and whether the resulting field access is valid and readable.

### Output

The function returns one unsigned scalar value of the same represented type as the accessed fields.

There is no separate status result, output parameter, or visible error sentinel. Zero is an ordinary possible field value and result, not an established indication of failure.

### Widths and layout

The field displacement is exactly 8 bytes. No other field is accessed.

The supplied representation does not include definitions of the signed argument type or unsigned field type, nor an independently established target ABI. Consequently, exact argument, address, field, and return widths remain unresolved.

Under a 32-bit unsigned-field interpretation:

- Each field occupies four bytes beginning at its computed address.
- Field values range from 0 through 4,294,967,295.
- Comparison uses unsigned ordering throughout that range.
- The selected field value is returned without conversion to signed ordering.

That interpretation is conditional, not a universal width requirement.

Field decoding uses the execution environment’s representation for the unsigned type. Byte order, alignment requirements, and the containing regions’ total sizes are not established. A displacement of 8 does not establish the existence or readability of a complete object beginning at the base.

### Preconditions for an ordinary defined result

Each address calculation, address conversion, and field read must be valid under the applicable execution model. The field storage must remain readable and alive for every access actually made.

The visible operation supplies no base-value check, bounds check, alignment check, overflow check, or access-failure recovery.

These requirements concern the computed field addresses rather than categorical acceptance or rejection of particular base values. In particular:

- A zero base is not automatically invalid.
- A nonzero base is not automatically valid.
- Readability of memory at the base itself is not sufficient to establish readability of the field.
- Readability of memory at the base itself is not a separate requirement shown by this function.

Stable field contents are additionally required for the simple unsigned-maximum guarantee.

## Behavior

### Stable-field selection

For stable readable fields, call the first field’s value A and the second field’s value B.

- If A is greater than B in unsigned order, return A.
- If A is less than B in unsigned order, return B.
- If A equals B, take the second-field selection path and return that common value.

Equality therefore affects the represented read path but not the numeric result for stable fields.

There is no special handling for zero, the unsigned maximum, or values whose most significant bit is set.

### Read structure and ordering

The supplied representation establishes these logical accesses:

1. Read the first field and retain its value as the initial return candidate.
2. Read the first field again and read the second field for an unsigned less-than-or-equal comparison.
3. If that comparison is true, read the second field again and replace the retained candidate with that value.
4. Return the retained candidate.

The initial first-field read precedes the comparison stage. The conditional second-field read follows the comparison when selected.

The relative evaluation order of the two operand reads within the comparison is not established. The contract therefore does not require the comparison’s first-field read to occur before its second-field read, or the reverse.

The represented false-comparison path contains two first-field reads and one second-field read. The represented true-comparison path contains two reads of each field.

These counts describe logical accesses in the supplied representation. They do not establish a corresponding count of physical memory transactions. No volatile or atomic access qualification is shown, and no executable instruction trace is supplied.

### Changing field contents

The initial first-field value and the first-field comparison value come from separate represented reads.

If the comparison is false, the function returns the initial candidate. It does not replace that candidate with the first-field value used in the comparison.

If the comparison is true, the function returns the subsequent second-field value. It does not necessarily return the second-field value used in the comparison.

Where the execution environment permits changing observations, the result may therefore differ from:

- Either comparison operand.
- The maximum of the comparison operands.
- The maximum of the two fields at return time.

No snapshot, lock, retry, consistency check, or synchronization mechanism is visible.

This description does not establish that unsynchronized concurrent modification is valid. Whether changing observations are permitted, and which sequences are possible, depends on the memory model and access semantics.

### Aliasing

No distinctness check or distinctness requirement is visible.

If both bases designate the same stable readable field, both comparison values are equal, the second-field path is selected, and the field’s value is returned.

For overlapping field storage that does not identify exactly the same scalar location, the evidence supplies no independent layout or access-validity guarantee. The ordinary requirements for valid reads still apply.

### Arithmetic

The field values undergo comparison and selection only. There is no addition, subtraction, scaling, or increment of those values. Selecting a large unsigned value therefore introduces no field-value arithmetic overflow.

Address formation is separate. The addition of 8 is expressed on signed integer-valued arguments before address use; it is not established as unsigned wrapping or as typed object-pointer arithmetic.

The evidence does not define an ordinary result for signed overflow, an invalid integer-to-address conversion, or an inaccessible resulting address. Such cases must not be assigned a wrapping-address rule merely because a particular machine might implement one.

## Side effects

No explicit memory write or direct function call appears in the supplied body.

The visible operation does not allocate memory, release memory, store an input for later use, update an explicit global variable, or emit a diagnostic.

Neither input transfers ownership. The function does not visibly acquire ownership of the field storage or extend its lifetime. The caller or surrounding execution environment must provide valid storage for the reads.

Reads can themselves matter in environments with externally observable read effects. However, support for device-backed or read-sensitive storage is not established: the representation does not specify volatile access, device-memory ordering, or a supported repeated-read protocol.

An invalid access may produce a platform-dependent failure. No particular exception, signal, termination, recovery behavior, or fallback return value is established.

## Dependencies

No direct function dependencies are visible. There are no additional opaque function IDs to enumerate.

The behavior depends on environmental definitions and conditions:

- Signed argument width and arithmetic rules.
- Unsigned field width and representation.
- Address width and integer-to-address conversion rules.
- Argument and return calling conventions.
- Field readability, alignment, and lifetime.
- Native byte order.
- Optimization and memory-model rules governing repeated reads.
- Platform treatment of invalid address calculations and accesses.

These are environmental dependencies, not evidence of hidden calls. No assumptions about an unknown callee’s ownership behavior or preservation of global state are needed because no call is shown.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not executed test results.

Unless an example states otherwise, both computed field locations are valid, readable, and stable, and their values are representable in the unsigned field type.

| First field | Second field | Required result | Represented selection |
|---|---|---|---|
| 3 | 9 | 9 | Second field |
| 9 | 3 | 9 | Initial first-field candidate |
| 7 | 7 | 7 | Second field on equality |
| 0 | 0 | 0 | Second field on equality |
| 0 | 1 | 1 | Second field |
| 1 | 0 | 1 | Initial first-field candidate |

Under the additional assumption of 32-bit unsigned fields:

| First field | Second field | Required result |
|---|---|---|
| 2,147,483,648 | 2,147,483,647 | 2,147,483,648 |
| 4,294,967,295 | 0 | 4,294,967,295 |
| 0 | 4,294,967,295 | 4,294,967,295 |
| 4,294,967,295 | 4,294,967,295 | 4,294,967,295 |

These examples require unsigned rather than signed ordering.

**Same field through both inputs.** If both bases identify the same stable readable field containing 42, the required result is 42. Separate allocations are not required.

**Zero base with a valid computed field.** Suppose the first base is zero, its addition and address conversion are valid, and the field at the resulting address 8 is valid, readable, and stably contains 3. If the second computed field stably contains 9, the required result is 9. There is no zero-base rejection path.

**Zero base with an invalid computed field.** If that same base instead produces an invalid field access, no ordinary return value is specified. Zero does not select a fallback result or bypass the access.

**Nonzero base with an invalid computed field.** A nonzero input whose computed field address is unreadable likewise has no specified ordinary result. The numeric nonzeroness of the base provides no validity guarantee.

For an environment that permits the represented reads to observe changing values:

- If the initial first-field read yields 100, and the comparison observes first-field value 5 and second-field value 4, the comparison is false and the returned candidate is 100.
- If the comparison observes first-field value 5 and second-field value 6, and the subsequent second-field read yields 2, the returned value is 2.

These changing-value examples describe selection from the represented reads, not portable concurrency guarantees.

## Uncertainties

**documented_only — Exact widths and ABI.** Argument width, unsigned field width, address width, and calling convention are unavailable. The 32-bit examples remain conditional.

**documented_only — Executable memory transactions.** Logical repeated reads are visible. Their correspondence to machine loads and external observations is not established.

**documented_only — Address validity.** The execution model determines whether a computed address is usable. In particular, the evidence neither guarantees nor rules out a readable field at address 8 when a base is zero.

**documented_only — Object meaning and layout.** Only the field displacement and unsigned access type are established. Object identity, allocation size, other fields, and semantic purpose remain unknown.

**documented_only — Concurrency and read-sensitive storage.** Atomicity, synchronization, volatile semantics, and supported changing-memory protocols are not shown.

**documented_only — Invalid-input outcomes.** Overflow, conversion failures, invalid alignment, inaccessible storage, and expired lifetimes have no established ordinary-result or recovery contract.

## Evidence

Established static evidence supports integer-based address formation using an addition of 8, unsigned field reads, unsigned less-than-or-equal comparison, retention of an initial first-field value, conditional replacement by a subsequent second-field read, and return of the selected scalar.

The supplied body also supports the absence of explicit writes, direct calls, and special handling for a zero base. It accesses the computed field addresses rather than the unadjusted bases.

The unsigned-maximum description is a consequence of valid, stable field contents. Any application-level meaning for that maximum is inferred intent, not established behavior.

No runtime tests, executable inspection, or independent ABI validation have been performed.
