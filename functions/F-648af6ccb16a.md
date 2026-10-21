<!-- Trusted provenance: F-648af6ccb16a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-648af6ccb16a

## Purpose

F-648af6ccb16a returns one more than the number of consecutive nonzero character units encountered within a signed scan budget.

The result includes an unconditional additional unit. That unit does not establish that a zero terminator exists, was read, or fits within the supplied budget. A null address also produces a result of one.

The established behavior is a bounded scan followed by an incremented result. Interpreting the result as a storage requirement, terminator-inclusive length, or allocation size is inferred intent, not an established interface guarantee.

## Interface

The function accepts two inputs:

- **Start address:** a nullable pointer to character-addressable memory.
- **Scan budget:** a signed integer limiting the number of nonzero character units that may be counted.

It returns a signed integer.

There is no output buffer, explicit status output, or ownership-bearing return value.

### Widths and representation

The supplied representation uses character units for memory access and pointer advancement. Each successful scan step advances by one such unit. The function does not interpret multibyte encodings, combine code units, or count user-perceived characters.

Zero is the only distinguished character value. Every nonzero value contributes equally to the count, regardless of whether a target represents ordinary characters as signed or unsigned.

The budget, internal count, and return value are represented as the same signed integer type. Its exact width is not established by the supplied evidence. In particular, a 32-bit signed range must not be assumed solely from the displayed type.

Let **M** denote the maximum representable value of that signed integer type. Results are unambiguous when the number of counted units is less than M. The boundary at M is addressed under **Uncertainties**.

### Preconditions

For a null start address, no readable character storage is required.

For a nonnull start address with a zero or negative budget, no character contents are read. The address must nevertheless be representable under the applicable calling convention.

For a nonnull start address with a positive budget:

- Each character location reached before stopping must be readable.
- Address advancement must remain valid under the applicable execution model.
- If a zero is encountered, no readable storage beyond that zero is required by this function’s scan.
- If no zero occurs within the budget, the budgeted character region must be readable.

There is no requirement that a zero occur within the budgeted region.

## Behavior

### Null address

A null start address produces one, independently of the budget.

The function does not inspect character memory on this path. A positive budget does not override the null-address case.

### Zero or negative budget

For a nonnull address and a budget at or below zero, the function produces one without reading the addressed character.

A negative budget is not converted into a large unsigned limit. It disables scanning in the same way as a zero budget.

### Positive-budget scan

For a nonnull address and positive budget, scanning begins at the supplied address.

At each scan position, the budget condition is evaluated before the character is inspected. If no budget remains, the function stops without reading that position.

When budget remains:

- A zero character ends the scan without contributing to the count.
- A nonzero character contributes one to the count, consumes one budget unit, and advances the scan address by one character unit.

The function returns the count plus one, subject to the signed-integer boundary described below.

Consequently, when the arithmetic is representable:

- An initially empty sequence produces one.
- A zero encountered after k nonzero units produces k + 1.
- Exhausting a positive budget of b without encountering zero produces b + 1.

The counted region excludes any encountered zero. The returned value nevertheless includes the additional unit even when the scan stops because its budget is exhausted.

### Boundary reads

The scan does not inspect the first character beyond the budgeted region.

For example, with a positive budget of three and three initial nonzero units, the result is four. Whether a zero, another nonzero unit, or inaccessible storage follows those three units does not affect the established scan result, provided the required reads and address advancement are valid.

By contrast, a zero within the budget is inspected to establish termination. If the first character is zero and the budget is positive, that character is read even though no character contributes to the count.

These are source-level access rules. They do not establish the number or granularity of physical loads performed by a compiled implementation.

### Arithmetic

The count starts at zero and increases only after a successful nonzero-character check. The remaining budget decreases only on that same successful step.

For an initial positive budget b:

- The count cannot exceed b.
- The remaining budget does not become negative during scanning.
- The final addition can produce a mathematical value as large as b + 1.

Thus, a budget of M can allow the count to reach M, after which the requested return expression is M + 1. That value is outside the signed return type’s representable range.

The evidence does not support a portable numeric result for this boundary. In particular, neither saturation at M nor wrapping to a negative value is an established contract.

## Side effects

No writes through the supplied address are present in the supplied representation. The function does not append a zero, modify scanned characters, or store the returned length in caller memory.

The changing address, remaining budget, and accumulated count are internal scan state. Their updates do not modify the caller’s argument variables.

No allocation, release, ownership transfer, callback, logging operation, or explicit global-state access appears in the supplied representation. The caller retains responsibility for any supplied storage.

The observable memory interaction is inspection of the character sequence, constrained by the null check, budget, and first encountered zero. No special synchronization or volatile-access behavior is established.

Invalid memory and invalid address advancement are not handled through an explicit error result. The function contains no recovery path for them. Any fault, trap, or other outcome depends on the execution environment.

## Dependencies

No explicit function dependencies appear in the supplied representation of F-648af6ccb16a.

Its behavior relies on the execution environment’s support for:

- Null-address comparison.
- Character-unit reads and address advancement.
- Signed integer comparisons and arithmetic.
- Returning a signed integer through the applicable calling convention.

No external callee is available whose behavior could establish additional validation, error reporting, ownership management, or global-state preservation.

## Acceptance examples

The following examples are conditional specifications derived from static evidence. They are not results of runtime tests. All readable-memory examples assume valid address advancement and representable result arithmetic unless stated otherwise.

| Start address and contents | Budget | Required result and access behavior |
|---|---:|---|
| Null address | Any representable signed value | Return one; inspect no character memory. |
| Nonnull address | Zero | Return one; inspect no character memory. |
| Nonnull address | Negative | Return one; inspect no character memory. |
| First unit is zero | Positive | Inspect the first unit and return one. |
| Two nonzero units followed by zero | Five | Count two, inspect the following zero, and return three. |
| Two initial nonzero units | Two | Inspect those two units and return three; do not inspect the next unit. |
| One nonzero unit followed by zero | One | Return two without inspecting the zero. |
| One nonzero unit followed by zero | Two | Inspect both units and return two. |
| Nonzero units with differing high-bit patterns | Budget covers them | Count each nonzero unit once; do not interpret an encoding. |

Additional boundary conditions:

- **Inaccessible contents with zero budget:** No character read is required. The supplied nonnull address must still satisfy the applicable representation and calling requirements.
- **Inaccessible next character after budget exhaustion:** No read of that character is required. Address advancement itself must remain valid.
- **Inaccessible next character while budget remains:** Successful completion is not guaranteed; there is no explicit invalid-memory return value.
- **Count reaches M:** The mathematical result is M + 1. A particular returned integer is documented_only rather than an acceptance requirement.
- **Budget is M, but the first character is zero:** Return one. The extreme budget alone does not cause the final-addition boundary.

## Uncertainties

### Documented_only

The supplied evidence does not establish:

- The target’s signed integer width or character-unit width.
- The calling convention or binary interface layout.
- The machine-level outcome when the count reaches M.
- The environment’s response to unreadable memory or invalid address advancement.
- Behavior under concurrent modification of the scanned storage.
- Exact physical load counts, speculative accesses, or compiler transformations.

If the representation is treated as literal C, a signed result outside the type’s range is not a defined wrapping operation. The supplied representation alone is insufficient to determine a corresponding machine-level result.

### Inferred intent

The extra returned unit is consistent with reserving space for a terminator. That interpretation does not imply that a terminator was found, that any terminator was written, or that the returned value lies within the supplied budget.

The function therefore must not be described as returning the length of a proven terminated sequence in every case.

## Evidence

Established behavior comes solely from the supplied static representation of F-648af6ccb16a:

- A null-address branch returns one.
- Scanning requires both a positive remaining budget and a nonzero current character.
- The budget condition precedes the character check.
- Each successful scan step increments the count, decrements the budget, and advances by one character unit.
- The nonnull path returns the count plus one.
- No explicit memory writes or function calls are present.

No implementation execution, runtime tests, target-width inspection, or machine-code validation has been performed. The acceptance examples express consequences of the supplied representation, not observed executions.
