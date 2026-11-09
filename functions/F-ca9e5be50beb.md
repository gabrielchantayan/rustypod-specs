<!-- Trusted provenance: F-ca9e5be50beb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca9e5be50beb

## Purpose

F-ca9e5be50beb updates an existing object from inherited execution context and conditionally completes or abandons an initialization operation.

The established behavior has two outcomes:

- **Both dependency checks return nonzero:** populate additional object fields, perform three arithmetic-helper calls, obtain another dependency result, and finally write the literal byte value `1` to an object status field.
- **Either dependency check returns zero:** write a supplied low-order byte to that status field, conditionally invoke several dependencies, conditionally replace a resource-related field, and dispatch an indirect callback.

Both outcomes occur after two unconditional 16-bit object writes. The failure outcome therefore does not leave the object untouched.

“Initialization,” “status,” and “resource-related” describe inferred roles. The available evidence establishes offsets, values, calls, and ordering, but not the object’s application-level type.

## Interface

### Calling context and result

The supplied representation declares no explicit parameters and no return value. Nevertheless, the function consumes an object address, several inherited register values, and numerous values from an existing frame. It must not be treated as an independently callable, parameterless API without establishing that surrounding calling context.

There is no explicit success code or returned object. Completion is observable through object writes and dependency calls. Normal return is shown after either branch, provided every invoked dependency and indirect callback returns normally.

### Required inherited values

The following names are descriptive contract terms, not recovered program identifiers:

| Input | Width or interpretation | Use |
|---|---|---|
| Object address | Target address | Base for all documented object accesses |
| Replacement value | 32-bit inherited integer | Low byte supplies two byte-field writes; full value may replace the field at `+0x74` |
| First cleanup indicator | 32-bit inherited integer | Enables the first conditional no-explicit-argument dependency call |
| Second cleanup indicator | 32-bit inherited integer | Enables the second such call |
| Third cleanup indicator | 32-bit inherited integer | Enables the third such call |

The cleanup indicators are tested only for zero versus nonzero. Their values are not explicitly passed as arguments in the supplied representation.

### Frame-supplied data

Offsets below identify slots in the supplied frame representation. They are not a verified public stack ABI.

| Frame offset | Width or interpretation | Use |
|---|---|---|
| `+0x0C` | 16-bit value | Input to a success-only call to F-4a5c45c1e447 |
| `+0x0E` | 16-bit value | Input to another success-only call to F-4a5c45c1e447; its return is discarded |
| `+0x10` | 32-bit value | Input to a success-only call to F-10cfc86a3233 |
| `+0x14` | Signed 32-bit integer | One term in the value stored at object `+0x68` |
| `+0x24` | Four-byte storage value | Storage byte at offset 2 supplies a zero-extended 16-bit field |
| `+0x40` | Four-byte storage value | Storage byte at offset 1 supplies a scale and a 32-bit field; byte at offset 2 supplies a byte field |
| `+0x48` | Signed 16-bit integer | First arithmetic input |
| `+0x4A` | Signed 16-bit integer | Second arithmetic input, negated after promotion |
| `+0x4C` | Signed 16-bit integer | Third arithmetic input |
| `+0x68` | 16-bit value | Unconditionally copied to object `+0x6C` |
| `+0x6C` | 32-bit value | Input to the unconditional call to F-10cfc86a3233 |
| `+0xB8` | Unsigned 32-bit integer | Its upper 16 bits become the arithmetic-helper denominator |
| `+0xD4` | Unsigned 16-bit integer | Low three bits supply a flag byte |
| `+0x100` | Signed 32-bit integer | Another term in the value stored at object `+0x68` |

Storage-byte offsets are stated directly rather than assuming a numerical byte significance for the four-byte containers. Target byte order is not independently established.

### Object layout touched by this function

All offsets are byte offsets from the object address.

| Offset | Access width | Meaning established by use |
|---|---:|---|
| `+0x00` | Address-sized entry, represented as 32 bits | Supplies the base used to find the indirect callback |
| `+0x48` | 32 bits | Receives zero-extended storage byte 1 from frame `+0x40` |
| `+0x4C` | 8 bits | Receives input flag bits 0–2 |
| `+0x4D` | 8 bits | Receives storage byte 2 from frame `+0x40` |
| `+0x4E` | 8 bits | Receives the replacement value’s low byte |
| `+0x50` | 16 bits | Receives zero-extended storage byte 2 from frame `+0x24` |
| `+0x52` | 16 bits | Receives first arithmetic-helper result |
| `+0x54` | 16 bits | Receives second arithmetic-helper result |
| `+0x56` | 16 bits | Receives third arithmetic-helper result |
| `+0x64` | 16 bits | Receives a conversion-like dependency result |
| `+0x68` | 32 bits | Receives a computed sum |
| `+0x6C` | 16 bits | Receives frame `+0x68` |
| `+0x6E` | 16 bits | Receives the first dependency result |
| `+0x70` | 32 bits | Receives a success-only dependency result |
| `+0x74` | 32 bits | Conditionally tested, passed to a dependency, and replaced |
| `+0x89` | 8 bits | Receives either the replacement value’s low byte or literal `1` |
| `+0x9C` | 32 bits | Receives the result of F-d2e51f07aabd |

The caller must provide valid storage and a valid surrounding context for every access on the selected path. No null, bounds, alignment, or callback-validity checks are shown.

## Behavior

### Unconditional prefix

The function first logically shifts the unsigned frame value at `+0xB8` right by 16 bits. Call the resulting value **D**. Its range is `0` through `65535`.

The following operations then occur in order:

1. Call F-4a5c45c1e447 with no explicit arguments shown; retain a 16-bit result.
2. Call F-10cfc86a3233 with the 32-bit frame value at `+0x6C`; retain a 32-bit result for possible later arithmetic.
3. Write the 16-bit frame value at `+0x68` to object `+0x6C`.
4. Write the retained 16-bit result to object `+0x6E`.
5. Call F-0fd4363fa1f0.
6. If that result is nonzero, call F-a39fb6744487.

The second check is short-circuited: F-a39fb6744487 is not called when F-0fd4363fa1f0 returns zero.

Both results must be nonzero to select the success path. No distinction is made among different nonzero values.

### Failure path

If either check returns zero:

1. Write the replacement value’s low byte to object `+0x89`.
2. If the first cleanup indicator is nonzero, call F-8349c252c168 with no explicit arguments shown.
3. If the second cleanup indicator is nonzero, call the same dependency, again with no explicit arguments shown.
4. If the third cleanup indicator is nonzero, call it a third time in the same form.
5. Read the 32-bit field at object `+0x74`.
6. If that field is nonzero:
   - Call F-8349c252c168 with the field value and the integer constant `0x2E`.
   - After that call returns, write the full 32-bit replacement value to object `+0x74`.
7. Read the dispatch base from the object’s first word, obtain the callback at dispatch-base offset `+0x30`, and invoke it.

The callback has no explicit arguments or consumed return value in the supplied representation.

The function does not itself perform the success-path field writes on this branch. Dependencies and the callback may nevertheless modify those fields; their preservation is not established.

In particular, the replacement value is not proven to be zero. Consequently, neither the status write nor the resource-field replacement may be specified as clearing its destination.

### Success path

When both checks return nonzero, the following calls precede the next object writes:

1. Call F-4a5c45c1e447 with the 16-bit frame value at `+0x0C`; retain its 16-bit result.
2. Call F-4a5c45c1e447 with the 16-bit frame value at `+0x0E`; discard the result.
3. Call F-10cfc86a3233 with the 32-bit frame value at `+0x10`; retain its 32-bit result.

Discarding a result does not make the corresponding call optional.

The function then writes, in order:

1. The retained 16-bit result to object `+0x64`.
2. A 32-bit sum to object `+0x68`: frame `+0x14`, plus frame `+0x100`, plus the unconditional F-10cfc86a3233 result, plus `8`.
3. The success-only F-10cfc86a3233 result to object `+0x70`.
4. Zero-extended storage byte 1 of frame `+0x40` to object `+0x48`.
5. The low three bits of frame `+0xD4` to byte field `+0x4C`.
6. Storage byte 2 of frame `+0x40` to byte field `+0x4D`.
7. The replacement value’s low byte to `+0x4E`.
8. Zero-extended storage byte 2 of frame `+0x24` to the 16-bit field at `+0x50`.

The 32-bit sum uses recovered signed-integer arithmetic. Its behavior on signed overflow is not independently established; no saturation, validation, or wider intermediate is shown.

### Arithmetic-helper sequence

Let **S** be storage byte 1 of frame `+0x40`, interpreted as an unsigned value from `0` through `255`. Let **H** be the unsigned integer quotient of D divided by two.

F-870c4fcf7295 is called three times, each time with D as its second argument:

| Call | First argument, interpreted as a 32-bit unsigned bit pattern | Destination |
|---|---|---|
| First | Signed frame value `+0x48` multiplied by S, then H added modulo \(2^{32}\) | Object `+0x52`, 16 bits |
| Second | S multiplied by the negation of signed frame value `+0x4A`, then H added modulo \(2^{32}\) | Object `+0x54`, 16 bits |
| Third | Signed frame value `+0x4C` multiplied by S, then H added modulo \(2^{32}\) | Object `+0x56`, 16 bits |

Each result is written before the next helper call.

The signed 16-bit inputs are promoted before multiplication or negation. Negating `-32768` therefore produces `32768`; it is not a 16-bit negation. The individual products fit in signed 32-bit range. Unsigned addition or multiplication in these expressions determines the final 32-bit argument bit pattern, including when a mathematical result is negative.

The helper’s actual operation is unknown. Division, rounding, clamping, and signed interpretation of its first argument must not be assumed merely from this call pattern.

There is no local guard against D being zero. Even when D is zero, all three calls are reached if the success path otherwise proceeds and earlier calls return normally.

Finally:

1. Call F-d2e51f07aabd.
2. Store its 32-bit result at object `+0x9C`.
3. Write literal byte `1` to object `+0x89`.

The status write is the last explicit object write on the success path.

## Side effects

The function mutates the supplied object in place. It does not explicitly allocate storage, return ownership, or establish an ownership transfer.

The failure-path use of F-8349c252c168 is consistent with cleanup intent, but release semantics are not established. In particular:

- The three indicator-controlled calls expose no explicit pointer arguments.
- The call involving object `+0x74` does expose that field and `0x2E`.
- None of these calls is proven to release memory, retain memory, preserve the object, or preserve global state.
- The subsequent callback may perform additional mutations or resource operations.

All ordering in this contract concerns the function’s explicit accesses and calls. It is not a concurrency guarantee or a promise about accesses internal to dependencies.

There is no explicit rollback, exception handling, retry, or error reporting beyond branch selection and field updates. If a dependency does not return normally, later documented operations need not occur.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-4a5c45c1e447 | One unconditional call producing 16 bits; two success-only calls with 16-bit inputs, the second result discarded |
| F-10cfc86a3233 | One unconditional and one success-only call, each with a 32-bit input and a retained 32-bit result |
| F-0fd4363fa1f0 | First zero/nonzero gate |
| F-a39fb6744487 | Second zero/nonzero gate, called only after a nonzero first result |
| F-8349c252c168 | Up to four failure-path calls; only the final conditional form has explicit arguments |
| F-870c4fcf7295 | Three ordered success-path calls with the arithmetic arguments above; 16-bit results stored |
| F-d2e51f07aabd | Final success-path dependency; 32-bit result stored before status becomes `1` |

A further indirect callback is selected through object offset `+0x00` and dispatch offset `+0x30`. Its identity is unavailable, so no opaque function ID or semantic contract can be assigned to it.

## Acceptance examples

These are conditional contract examples, not executed tests. Dependency results and continued execution are stipulated where necessary.

### First gate rejects

If F-0fd4363fa1f0 returns zero:

- F-a39fb6744487 is not called.
- The unconditional writes at `+0x6C` and `+0x6E` have already occurred.
- The function takes the failure path, beginning with the byte write at `+0x89`.
- The function does not itself invoke the three arithmetic-helper calls.

### Second gate rejects

If the first gate returns `7` and the second returns zero, the failure path is selected. The first gate’s particular nonzero value has no additional established effect.

### Conditional cleanup order

Suppose the first and third cleanup indicators are nonzero, the second is zero, and object `+0x74` is nonzero when examined. Assuming calls return normally, F-8349c252c168 is invoked in this order:

1. First-indicator call.
2. Third-indicator call.
3. Call with the observed resource-related field and `0x2E`.

The replacement-field write follows the third call. The indirect callback follows that write.

### Flag and byte conversion

On success, if frame `+0xD4` is `0xFFF5`, object byte `+0x4C` receives `0x05`.

If storage byte 2 of frame `+0x24` is `0xAB`, the 16-bit field at object `+0x50` receives numerical value `171`, not a sign-extended negative value.

### Arithmetic arguments

Suppose D is `10`, S is `3`, and the signed frame values at `+0x48`, `+0x4A`, and `+0x4C` are `2`, `-4`, and `-2`.

The three first-argument bit patterns passed to F-870c4fcf7295 are respectively:

- `11`
- `17`
- `0xFFFFFFFF`

Each call has second argument `10`. Output fields depend on the helper results; this evidence does not establish their numerical values.

### Zero denominator and final status

If D is zero, the function provides zero as the second argument of each reached arithmetic-helper call. The resulting behavior is dependency-defined.

If all success-path calls return normally, the final explicit status write is `1`, regardless of the replacement value’s low byte.

## Uncertainties

The following remain **documented_only**:

- The complete entry ABI and origin of inherited frame/register values.
- Object type, field names, alignment requirements, and semantic status meanings.
- Actual arguments omitted from the displayed calls.
- Dependency effects on globals, object storage, memory lifetime, and control flow.
- Whether the indirect callback receives an implicit object or other arguments.
- Arithmetic-helper semantics, including zero-denominator handling.
- Signed-overflow behavior of the sum stored at `+0x68`.
- Target byte order and external serialization format.
- Whether the replacement value is normally constrained to zero.
- Whether any dependency invalidates storage accessed later.

No ownership or preservation guarantee resolves these unknowns by assumption.

## Evidence

Established static evidence consists of the supplied function representation: explicit field widths and offsets, branch structure, dependency-call order, integer conversions, arithmetic expressions, and final writes.

The description of a conditionally completed initialization and a cleanup-oriented failure path is inferred intent, not an established domain contract.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. The acceptance examples are derived conditions for evaluating the stated behavior, not reports of observed execution.
