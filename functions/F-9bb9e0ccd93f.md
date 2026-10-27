<!-- Trusted provenance: F-9bb9e0ccd93f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9bb9e0ccd93f

## Purpose

F-9bb9e0ccd93f performs a selector-controlled update to a caller-supplied 32-bit word and conditionally invokes F-a48bd86e5ff0. Its ordinary return value is either 3 or 13.

Only selector value 2 activates the update and conditional dependency call. All other selector values return 3 without any explicit memory access through the supplied object address or word pointer.

The operational behavior is established by the supplied static representation. The application-level meanings of the selector, modified bit, examined object field, and return values are unknown. In particular, 3 and 13 must not be labeled success, failure, or retry without additional evidence.

## Interface

The interface has four inputs and one 32-bit result:

| Input | Represented width and role |
|---|---|
| Object address | A 32-bit integer-form address used to locate an object field at byte offset 40. It is also passed unchanged to F-a48bd86e5ff0 if that dependency is invoked. |
| Selector | A represented 32-bit integer compared with the exact value 2. No other selector value activates the operation. |
| Word pointer | A pointer to a readable and writable unsigned 32-bit word. The same pointer, not a copy of the pointed-to value, is passed to F-a48bd86e5ff0. |
| Opaque argument | A 32-bit value forwarded unchanged to F-a48bd86e5ff0 when it is called. This function does not otherwise interpret it. |

The result is a 32-bit value with the exact numeric value 3 or 13 on ordinary completion.

For an active invocation, the directly accessed object layout is:

| Location | Access width | Direct use |
|---|---:|---|
| Object address plus 40 bytes | 32 bits | Supplies a value for the conditional-call decision. |

No other object field is directly accessed in the supplied representation. The complete object size and layout are unknown.

The word pointer identifies one 32-bit storage location. This function sets the bit with mask 0x00000002 in that word. Bit positions in this contract are numbered from zero, so this is bit 1.

For selector 2, ordinary execution requires that the word pointer support a 32-bit read followed by a 32-bit write and that the object field support the required 32-bit reads. Any further requirements imposed by F-a48bd86e5ff0 are unknown.

For selectors other than 2, neither address is dereferenced by this function. Consequently, this function’s explicit behavior on that path does not require those inputs to designate accessible storage. This does not establish requirements imposed by callers, instrumentation, or the surrounding execution environment.

## Behavior

### Inactive selector

The selector comparison occurs before the pointer-based accesses.

If the selector is not 2, the function:

- Returns 3.
- Does not explicitly read or write the supplied word.
- Does not explicitly read the object field.
- Does not invoke F-a48bd86e5ff0.
- Does not interpret the opaque argument.

There is no separate handling for zero, negative selector values, or other positive values.

### Active selector: word update

If the selector is 2, the function first reads the unsigned 32-bit word designated by the word pointer. It combines that value with mask 0x00000002 using bitwise inclusive OR, then writes the resulting 32-bit value back to the same location.

At this write:

- Bit 1 is set.
- Every other bit retains its value from the preceding read.
- A write is represented even if bit 1 was already set.
- No smaller-width store is represented.
- No atomic read-modify-write guarantee is established.

This update precedes every explicit object-field read and precedes any dependency invocation. Therefore, F-a48bd86e5ff0, if called, is invoked after the update has occurred. That statement does not imply that the word remains unchanged until the dependency reads it in a concurrent environment.

### Active selector: conditional object reads

After updating the word, the function reads a 32-bit value at byte offset 40 from the object address.

The first condition examines only bits 4 through 15 of that read. It is satisfied exactly when all twelve of those bits are zero.

Equivalently, the first condition masks the value to its low 16 bits, shifts that unsigned masked value right by four positions, and compares the result with zero. Important boundaries follow:

- Bits 0 through 3 do not affect the first condition.
- Bits 16 through 31 do not affect the first condition.
- Bit 15 does affect the first condition.
- The condition does not require the entire low 16-bit portion to be zero.

If the first condition is satisfied, the function does not need the second object-field read and proceeds to invoke F-a48bd86e5ff0.

If the first condition is not satisfied, the function performs a second 32-bit read from the same object-field address. It tests mask 0x00000200, which is bit 9, in that second value.

- If bit 9 is set in the second read, the dependency is invoked.
- If bit 9 is clear in the second read, the dependency is not invoked, and the function returns 3.

The two reads must not be collapsed into a promised single snapshot. If the location changes between reads, the first condition uses the first value and the bit-9 condition uses the second value.

When the field remains stable, the dependency-call condition can be summarized as: bits 4 through 15 are all zero, or bit 9 is set. That summary describes stable storage only; it does not replace the ordered two-read behavior.

### Dependency invocation and result selection

When the object-field conditions permit the call, F-a48bd86e5ff0 is invoked exactly once with these three arguments, in order:

1. The original object address.
2. The original word pointer.
3. The original opaque argument.

The selector is not forwarded.

The dependency’s represented 32-bit integer result is compared with zero:

- A zero result causes F-9bb9e0ccd93f to return 3.
- Any nonzero result causes F-9bb9e0ccd93f to return 13.

The comparison is not a positivity test. A negative represented integer result is nonzero and therefore selects 13.

The dependency’s numeric result is not returned directly. On ordinary completion, this function maps it into one of the two fixed result values. If the dependency is skipped, the result is 3.

## Side effects

The only direct write shown is the 32-bit update through the word pointer. It occurs only for selector 2 and always precedes the call-decision reads.

No direct object-field write, allocation, deallocation, ownership transfer, global-state access, or external I/O is shown. These observations apply to F-9bb9e0ccd93f’s explicit operations, not to the transitive behavior of F-a48bd86e5ff0.

Aliasing is not excluded by the supplied representation. If the word pointer identifies the same storage as the examined object field, the bit-setting write happens before that field is read. More general overlap, alignment, and memory-model constraints are not specified.

The function does not directly restore the word’s previous value when the dependency is skipped or returns zero. Thus, return value 3 does not imply that no state change occurred.

Once the dependency is invoked, the final contents of the word, the object, and any reachable or global state are unknown without its contract. The dependency receives the actual word pointer and may affect the same storage. No assumption that it preserves the newly set bit is justified.

The normal return rules assume the represented accesses and any dependency call complete normally. No validation, recovery path, or special error result is shown for inaccessible memory, prohibited alignment, or dependency failure.

## Dependencies

### F-a48bd86e5ff0

This is the only explicit function dependency.

Its established interaction contract here is limited to:

- Invocation only for selector 2, after the word update and a satisfied object-field condition.
- At most one invocation per call to F-9bb9e0ccd93f.
- Receipt of the object address, word pointer, and opaque argument unchanged.
- Production of a represented 32-bit integer result tested only for zero versus nonzero.

Its purpose, memory accesses, global-state effects, ownership rules, termination behavior, and error behavior are documented_only unknowns.

In particular, this contract does not assume that the dependency preserves global state, preserves the object or supplied word, allocates nothing, or releases no memory.

## Acceptance examples

These examples are conditional static expectations, not reports of executed tests. Unless stated otherwise, they assume valid accesses, normally completing execution, a stable object field, and no aliasing or external interference.

| Conditions | Expected behavior |
|---|---|
| Selector is 0 | Return 3; no explicit word or object-field access; no dependency call. |
| Selector is −1 | Same inactive-path behavior: return 3 without the pointer-based accesses or dependency call. |
| Selector is 2; initial word is 0x00000000; field is 0x00000000; dependency returns zero | Write 0x00000002; first field condition succeeds; invoke the dependency once; return 3. |
| Selector is 2; initial word is 0x80000000; field is 0x0000000F; dependency returns 7 | Write 0x80000002; low four field bits do not prevent the first condition from succeeding; invoke once; return 13. |
| Selector is 2; initial word is 0x00000002; field is 0x00000010 | Write 0x00000002 again; first condition fails; second read has bit 9 clear; skip the dependency; return 3. |
| Selector is 2; field is 0x00000200; dependency returns −1 | First condition fails because bit 9 lies within bits 4–15; second condition succeeds; invoke once; return 13. |
| Selector is 2; field is 0xFFFF0000; dependency returns zero | Upper 16 bits do not affect the first condition; invoke once; return 3. |
| Selector is 2; field is 0x00008000 | Bit 15 makes the first condition fail; bit 9 is clear on the second read; skip the dependency; return 3. |

Two changing-field cases establish the read-order distinction:

- If the first field read is 0x00000010 and the second is 0x00000200, the first condition fails but the second succeeds. The dependency is invoked.
- If the first field read is 0x00000200 and the second is 0x00000000, the first condition fails and the second also fails. The dependency is skipped, despite bit 9 having been set in the first read.

For any example involving a dependency call, the stated word value describes this function’s direct pre-call write, not a guaranteed final value after the dependency returns.

## Uncertainties

**Established static evidence:** selector gating, access widths, byte offset 40, the bit-setting update, short-circuit read order, argument forwarding, and zero/nonzero return mapping are visible in the supplied representation.

**Inferred intent:** the writable word may be a flag word, and the examined object field may encode state or options. These interpretations are not required by the behavior and are not normative names.

**Documented_only unknowns:**

- Application meanings of selector 2, masks 0x00000002 and 0x00000200, and results 3 and 13.
- Full object layout, byte order, alignment requirements, and address-overflow behavior.
- Whether the object field can change asynchronously or denotes device-backed storage.
- Compiler and hardware guarantees concerning physical memory transactions, atomicity, and concurrency.
- All dependency behavior beyond its visible invocation and result comparison.
- Ownership and lifetime requirements beyond the storage access needed for ordinary execution.

The address operation identifies a field 40 bytes from the supplied base. The static representation does not establish a checked address calculation or a defined outcome when that location cannot be represented or accessed.

## Evidence

This contract is based solely on the supplied static function representation and the supplied opaque dependency mapping.

The representation shows one selector comparison, one conditional 32-bit read-modify-write, one required object-field read on the active path, one short-circuited second read, one conditional dependency call, and fixed ordinary return values of 3 and 13.

No runtime tests, execution traces, dependency inspection, or independent implementation verification have been performed. Acceptance examples are deductions from the represented behavior under their stated conditions.
