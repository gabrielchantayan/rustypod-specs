<!-- Trusted provenance: F-efc2c252d6ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efc2c252d6ce

## Purpose

F-efc2c252d6ce conditionally processes an addressed object under a signed size-limit comparison. On the admissible path, it increases an associated accounting field, saves and clears a context state byte, invokes a processing dependency, and then performs further accounting and flag updates using post-call state.

Three paths are established:

- A zero object address returns zero without accessing the context.
- An admissible size invokes F-f59a6024d361, performs the subsequent state updates if that call returns normally, and returns the final value of object flag bit 0x0008 normalized to zero or one.
- An excessive size invokes F-2022b0f73f3b and returns one if that call returns normally.

“Processing,” “accounting,” and “size limit” describe inferred roles. The static representation establishes accesses, arithmetic, calls, and branches, but not the surrounding application semantics.

The result is not an unambiguous success indicator: one can mean either the excessive-size path returned normally or the admissible path ended with flag bit 0x0008 set.

## Interface

### Inputs and result

The function receives:

1. **Context address:** an address used to access an associated-record selector, a signed comparison field, and a state byte.
2. **Object address:** a 32-bit address-like input. Zero selects the immediate-return path; nonzero selects object access and processing.

The result is an unsigned 32-bit value. Every explicit normal return produces zero or one.

The supplied representation uses 32-bit integer and address-like operations. It does not establish portability to another width or execution model.

### Required layouts

All offsets below are byte offsets.

**Context**

| Offset | Width | Observed use |
|---|---:|---|
| 0x00 | 32 bits, pointer-like | Selects an associated record before the limit check and again after processing. |
| 0x10 | 32 bits, signed | Compared with zero after the accounting decrement. |
| 0x15 | 8 bits | Saved, cleared before processing, examined afterward, and conditionally written as one. |

**Associated record**

| Offset | Width | Observed use |
|---|---:|---|
| 0x00 | 32 bits, pointer-like | Selects the limit-bearing record for the initial comparison. |
| 0x1A4 | 32 bits, signed arithmetic | Supplies the accounting value and receives the increment or decrement. |

**Limit-bearing record**

| Offset | Width | Observed use |
|---|---:|---|
| 0x5C | 32 bits, signed | Supplies the bound for the initial comparison. |

**Object**

| Offset | Width | Observed use |
|---|---:|---|
| 0x02 | 16 bits, unsigned | Flags; bits 0x0008 and 0x0002 may be set. |
| 0x40 | 32 bits, signed arithmetic | Supplies the size-like quantity for the comparison, increment, and decrement. |

The context state byte is tested only for zero versus nonzero. Its signedness does not affect those tests.

### Global-sourced processing argument

The second argument to F-f59a6024d361 comes from a global data operand. It is not established to be an immutable value, a constant, or a literal address argument.

Its argument value is evaluated after the accounting increment and context-state clear, and before entry into F-f59a6024d361. The supplied evidence does not establish its type, width, value, mutability, or relationship to other storage. In particular, possible aliasing with earlier writes is unresolved.

### Preconditions and ownership

For a nonzero object address, every storage location actually accessed must be readable at the required width, and every location written must be writable. The context, its selected associated record, and the limit-bearing record are dereferenced without explicit null checks.

After F-f59a6024d361 returns, the context and object must still support the subsequent accesses. A replacement associated record selected by the context must support the accounting decrement.

No explicit validation establishes alignment, extent, lifetime, non-aliasing, or arithmetic range. No ownership transfer, retention rule, allocation, or release is established by the caller’s operations. Dependency ownership behavior remains unknown.

## Behavior

### Null-object path

The function tests the object address before accessing the context.

If the object address is zero, it returns zero without reading or writing the context and without invoking either dependency. The context therefore need not be valid on this path.

### Initial limit comparison

For a nonzero object address, the function reads the context’s associated-record selector and retains that selected address for the initial comparison and admissible-path increment.

It compares the sum of:

- The signed 32-bit object quantity at offset 0x40.
- The signed 32-bit accounting value at associated-record offset 0x1A4.

The bound is the signed 32-bit value at offset 0x5C in the limit-bearing record selected through the associated record’s offset-0x00 field.

The admissible condition is **sum less than or equal to bound**. Equality is accepted. Negative quantities, accounting values, and bounds receive no independent rejection.

The exact relative order of independent operand reads within the comparison is not established. The reads supplying the decision occur before entry into the selected branch.

The addition uses signed 32-bit operands without an explicit widening operation or overflow check. This contract does not assume defined wraparound on overflow. Numerical examples below require representable arithmetic.

### Excessive-size path

If the comparison sum is greater than the bound, the function invokes F-2022b0f73f3b with:

1. The initially selected associated-record address.
2. A diagnostic-message address associated with an expression-tree-size diagnostic.

If the dependency returns normally, the function returns one.

This path performs no explicit accounting update, context-state clear, or object-flag update before the call. The dependency may nevertheless modify state or fail to return; neither possibility is resolved by the supplied representation.

### Admissible-path preparation and call

If the comparison accepts, the function:

1. Increases the initially selected associated record’s accounting field by the object quantity read for that update.
2. Saves the context state byte.
3. Writes zero to the context state byte.
4. Evaluates the global-sourced second argument and invokes F-f59a6024d361.

The dependency arguments, in positional order, are:

- The object address.
- The value supplied by the global data operand.
- The context address.

The accounting increment is separate from the comparison and reads the object quantity for the update. A single quantity snapshot is not established across these operations.

The saved state byte is read after the accounting increment. If storage overlaps, that increment could affect the value subsequently saved.

The global-sourced argument is evaluated after both the increment and the zero store. Consequently, its supplied value cannot be specified independently of those writes when aliasing is possible. No particular ordering among the argument evaluations is established beyond their occurrence before dependency entry.

No return value from F-f59a6024d361 is used.

### Post-call accounting decrement

If F-f59a6024d361 returns normally, the function subtracts the object’s current quantity from the accounting field selected through the context’s current associated-record selector.

This operation uses post-call storage. It is not guaranteed to undo the earlier increment:

- The context may select a different associated record.
- The object quantity may have changed.
- The selected accounting field may have changed.
- Overlapping storage may couple otherwise distinct fields.

The supplied representation does not establish an atomic snapshot of the post-call operands or a synchronization guarantee.

No recovery mechanism is shown for a dependency that does not return normally. The caller does not establish automatic reversal of the increment or restoration of the cleared byte in that case.

### Flag and state updates

After the decrement, the function examines the signed 32-bit context field at offset 0x10.

If that field is strictly positive, it sets object flag bit 0x0008 through a 16-bit read-modify-write. Zero and negative values do not trigger that write.

The function then reads the current context state byte:

- **Current byte is zero, saved byte is nonzero:** write one to the context state byte.
- **Current byte is zero, saved byte is zero:** do not write the byte.
- **Current byte is nonzero:** set object flag bit 0x0002 through a 16-bit read-modify-write, without rewriting the state byte.

Restoring a previously nonzero byte is Boolean normalization, not restoration of its original numeric value. For example, a saved value of 0x7F is restored as one when the current byte is zero.

The restoration branch does not itself set flag bit 0x0002. That flag depends on the current byte being nonzero when inspected, before any restoration.

Each explicit flag update preserves the other bits in the 16-bit value read for that update. Neither identified flag bit is explicitly cleared.

### Admissible-path return

The function finally reads the object’s 16-bit flags and returns bit 0x0008 normalized to zero or one.

The result depends on this final read, not solely on the signed context comparison field. A preexisting bit 0x0008 can produce one even when the context field is nonpositive.

## Side effects

On the admissible path, the function directly performs these writes in order:

1. Accounting increment in the initially selected associated record.
2. Zero store to the context state byte.
3. After normal dependency return, accounting decrement through the current context selector.
4. Conditional set of object flag bit 0x0008.
5. Either a conditional state-byte store of one, a conditional set of flag bit 0x0002, or neither.

The global-sourced processing argument is evaluated between the second write and dependency entry. Although this evaluation is not itself an established write, it may depend on storage affected by earlier writes if aliasing exists.

The increment and decrement are distinct observable updates separated by a dependency call. They must not be treated as a guaranteed net-zero effect.

The function directly performs no allocation or release. This does not constrain either dependency’s memory-management behavior.

Ordering here describes the function’s explicit operations. No locking, atomicity, concurrency safety, or inter-thread visibility guarantee is established.

## Dependencies

### F-f59a6024d361

Invoked once on the admissible path, after the accounting increment and context-state clear.

Its positional arguments are the object address, a global-sourced operand value, and the context address. The second argument is evaluated after those preceding writes and before dependency entry. Neither an immutable value nor a literal address argument is established.

The second argument’s type, width, value, mutability, and possible aliasing with earlier writes are **documented_only**.

The dependency’s implementation, return behavior, ownership rules, and preservation guarantees are also **documented_only**. No assumption is made that it preserves globals, the context selector, object quantity, accounting values, flags, state bytes, or storage lifetimes.

### F-2022b0f73f3b

Invoked once on the excessive-size path, with the initially selected associated record and a diagnostic-message address.

Error-reporting intent is inferred. Output destination, complete message contents, state changes, ownership behavior, and whether the call returns are **documented_only**.

## Acceptance examples

These examples are conditional deductions, not runtime observations. Unless stated otherwise, assume valid non-overlapping storage, representable signed arithmetic, normal dependency return, and no dependency changes beyond those described. Non-overlapping storage includes the global operand’s storage where applicable.

1. **Null object:** With object address zero, return zero without accessing the context or invoking a dependency.

2. **Exact-bound acceptance:** Accounting 40, quantity 60, and bound 100 select F-f59a6024d361. With unchanged selector, quantity, and accounting during the call, accounting becomes 100 before entry and returns to 40 afterward.

3. **Over-bound path:** Accounting 41, quantity 60, and bound 100 select F-2022b0f73f3b. There are no direct accounting or state-byte writes on this path. Return one if the dependency returns normally.

4. **Saved nonzero state:** With saved byte 0x7F and current byte zero after processing, write one to the byte. If flags were zero and the signed context field is nonpositive, neither identified flag is set and the result is zero.

5. **Processing-produced state:** If processing leaves the byte at 0x80, set flag bit 0x0002 and leave the byte at 0x80. If bit 0x0008 remains clear, return zero.

6. **Positive context field:** A post-decrement context field of one causes bit 0x0008 to be set and, absent later aliasing effects, the result to be one. Values zero and negative one do not trigger that flag update.

7. **Preexisting return flag:** With bit 0x0008 already set and a nonpositive context field, return one if no dependency or overlapping write changes that flag.

8. **Changed quantity:** Accounting 20 and initial quantity 5 produce an increment to 25. If processing changes only the quantity to 7, the decrement produces 18.

9. **Changed associated record:** If processing redirects the context to another valid record, the initial record receives the increment and the replacement receives the decrement. No balancing update to the initial record is established.

10. **Negative quantity:** Accounting 10, quantity −3, and bound 10 are admissible. With unchanged relevant state, accounting becomes 7 before processing and returns to 10 afterward.

11. **Global argument timing:** If the global operand’s storage overlaps storage modified by the accounting increment or state-byte clear, the second argument is evaluated after those writes. Its concrete value cannot be predicted without the operand’s type, width, layout, and overlap details.

## Uncertainties

**Established static evidence:** field offsets and access widths listed above; signed comparisons; explicit write order; dependency argument positions; global sourcing and evaluation timing of the processing argument; branch conditions; and explicit return values.

**Inferred intent:** temporary size reservation, object processing, expression-tree-size error reporting, and accumulation of Boolean-like context state.

**Documented_only:**

- The global-sourced second argument’s type, width, value, mutability, and aliasing relationships. It is not established as constant or as an address literal.
- Dependency effects, global preservation, ownership, retention, release, and nonlocal control transfer.
- Exact diagnostic contents and output destination.
- Arithmetic-overflow behavior, invalid-address behavior, unsupported alignment, and concurrent mutation.
- Concrete effects of unusual storage overlap.

No explicit check prohibits aliasing. Earlier writes may change fields or global-sourced data consumed later. The examples do not establish independent-record behavior when their non-overlap assumptions are removed.

## Evidence

This contract is derived from the supplied static function representation and opaque dependency mapping. The prior document and reviewer feedback are not treated as independent behavioral evidence.

The representation supports the stated accesses, branches, argument sources, ordering boundaries, calls, and return expressions. It does not supply dependency implementations, the global operand’s declaration or contents, or a complete execution-environment specification.

No runtime tests have been performed. All acceptance examples are conditional static deductions.
