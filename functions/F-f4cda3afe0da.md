<!-- Trusted provenance: F-f4cda3afe0da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4cda3afe0da

## Purpose

F-f4cda3afe0da conditionally replaces a pointer-sized value stored in a caller-supplied record. Only selector value 4 activates replacement. All other selector values leave the record untouched by this function.

For the active selector, the function conditionally invokes an indirect callback associated with the record’s existing value, clears the record’s value after that callback returns, invokes F-61e23c4aaa8d with the record address, and stores the dependency’s result in the record.

The established behavior is replacement-oriented. Interpreting the callback as destruction or release, or interpreting F-61e23c4aaa8d as allocation or construction, is inferred intent rather than a demonstrated ownership contract.

## Interface

### Inputs

The function takes two values represented as 32-bit integers in the supplied source model:

| Input | Interpretation |
|---|---|
| Record address | Base address used to access a four-byte field at byte offset 4. Passed unchanged to F-61e23c4aaa8d on the active path. |
| Selector | Compared with the integer value 4. No other selector value has distinct behavior. |

The record address is used as an address even though its recovered parameter type is integer-valued. This contract describes the supplied 32-bit model; it does not establish compatibility with a wider-pointer ABI.

### Return value

The returned value occupies one byte.

| Condition | Returned byte |
|---|---:|
| Selector is not 4 | 1 |
| Selector is 4 and F-61e23c4aaa8d returns a nonzero value | 0 |
| Selector is 4 and F-61e23c4aaa8d returns zero | 15 |

The function does not forward a nonzero dependency result as its return value. That result is stored in the record; the function itself returns zero in that case.

No signed interpretation of the returned byte is needed to distinguish these outcomes.

### Relevant memory layout

Only the following portions of the reachable structures are established:

| Location | Size in the supplied model | Use |
|---|---:|---|
| Record base + 4 | 4 bytes | Existing value, subsequently replaced with the dependency result |
| Address designated by a nonzero existing value, offset 0 | 4 bytes | Address of a dispatch structure |
| Dispatch structure + 4 | 4 bytes | Indirect callback target |

The total record size, the contents at record offset 0, and all other object and dispatch-structure fields are unknown.

### Preconditions

For selector 4, defined completion requires:

- A readable four-byte field at record offset 4.
- A writable four-byte field at that same location when each store occurs.
- If the initial field read is nonzero, valid memory for the subsequent dispatch lookup and a callable target.
- Successful completion of any invoked callback and of F-61e23c4aaa8d.
- Continued validity of the record field after those calls return.

These are requirements on the execution environment, not validations performed by this function. There is no explicit check that the record address itself is nonzero or that any dereferenced address is valid.

For selectors other than 4, this function does not dereference the record address.

## Behavior

### Inactive selector

When the selector differs from 4:

- The function returns the byte value 1.
- It does not read or write the record field.
- It does not perform the indirect callback.
- It does not invoke F-61e23c4aaa8d.

This includes zero, negative values under the displayed integer interpretation, and every other representable selector except 4.

### Active selector: existing value is zero

When the selector is 4, the function first reads the four-byte field at record offset 4.

If that read produces zero:

1. No indirect callback is invoked.
2. No preliminary zero store is performed by this function.
3. F-61e23c4aaa8d is invoked with the original record-address value.
4. Its 32-bit result is retained for the final store.
5. A zero result selects return byte 15; a nonzero result selects return byte 0.
6. The dependency result is stored into the four-byte record field.
7. The selected byte is returned.

The absence of a preliminary store matters even though the field was observed to contain zero. The function does not perform a redundant clear on this branch.

### Active selector: existing value is nonzero

If the initial record-field read is nonzero, the function performs an indirect dispatch before invoking F-61e23c4aaa8d.

The source expression obtains the callback through the record field, the first four-byte word at the designated object address, and the four-byte entry at offset 4 in that dispatch structure. The callback is invoked with no explicit arguments shown in the supplied source representation. An implicit receiver or other calling-convention information is not established.

After the callback returns normally:

1. The function writes zero to the four-byte record field.
2. It invokes F-61e23c4aaa8d with the original record-address value.
3. It classifies the dependency result as zero or nonzero.
4. It stores that result into the record field.
5. It returns 15 for a zero result or 0 for a nonzero result.

The zero store occurs **after**, not before, the indirect callback. Consequently, this function does not itself clear the old value before the callback executes.

The preliminary clear is also unconditional once the callback has returned normally. If the callback changes the record field, the subsequent zero store overwrites that change, provided the field remains writable.

### Read and write ordering

The relevant ordering established by the source is:

| Stage | Observable operation |
|---|---|
| 1 | Read the selector for comparison with 4. |
| 2, active path only | Read the record field to decide whether indirect dispatch is required. |
| 3, nonzero-field branch only | Obtain the indirect callback through the record field and dispatch structure, then invoke it. |
| 4, after normal callback return only | Store zero into the record field. |
| 5, active path only | Invoke F-61e23c4aaa8d. |
| 6, after normal dependency return | Store the dependency result into the record field. |
| 7 | Return the selected byte. |

The callback-selection expression accesses the record field again rather than explicitly retaining the value used by the initial nonzero test. The source therefore does not establish a single captured object pointer shared by the test and dispatch.

This is source-level ordering, not a guarantee about machine-instruction count, atomicity, volatile accesses, or synchronization. Concurrent or asynchronous changes to the field have no established supported semantics.

### Integer and address handling

The function uses exact equality against selector value 4 and exact zero comparisons for the record field and dependency result.

No range check or sign check is applied to a nonzero dependency result. Every nonzero 32-bit result takes the same return-value path and is stored in the record field.

The address expressions use fixed byte offsets of 4. No overflow handling is shown. Supported behavior therefore requires address calculations and dereferences to remain valid in the execution model; the source does not establish wraparound addressing as an accepted input case.

## Side effects

### Direct memory writes

On normal completion with selector 4, the function writes the result of F-61e23c4aaa8d to the four-byte field at record offset 4.

When the initial field value is nonzero and the callback returns normally, the function additionally writes zero to that field before invoking F-61e23c4aaa8d.

When the initial field value is zero, there is no preliminary clear.

No direct writes to other record fields are visible. This does not constrain the memory effects of either invoked operation.

### Dependency effects

Both the indirect callback and F-61e23c4aaa8d may have effects not described by the supplied body. In particular, this contract does not assume that either operation:

- Preserves global state.
- Leaves the record unchanged.
- Preserves the existing object.
- Allocates, releases, or transfers ownership of memory.
- Returns normally.
- Is safe to invoke reentrantly.

On normal return from F-61e23c4aaa8d, this function’s final store overwrites any value that dependency may have placed in the record field.

### Ownership and lifetime

No explicit allocation or deallocation operation is established in this function.

The callback’s position before clearing the old value is consistent with a cleanup operation, but does not prove one. Likewise, storing F-61e23c4aaa8d’s result does not establish ownership of that result.

If either invoked operation invalidates the record, subsequent stores may be invalid. The function contains no lifetime safeguard or recovery path.

### Errors and interrupted execution

The only explicit failure-like result is return byte 15 when F-61e23c4aaa8d returns zero.

A zero dependency result is still stored in the record field. There is no rollback to the prior value.

No explicit error handling covers invalid addresses, invalid callback targets, or failures that prevent a dependency from returning normally. If the callback does not return normally, the subsequent clear and dependency invocation are not reached. If F-61e23c4aaa8d does not return normally, the final result store and ordinary return are not reached.

## Dependencies

### F-61e23c4aaa8d

Established interface at this call site:

- Invoked exactly once on each normally progressing selector-4 path.
- Not invoked for other selectors.
- Receives the original record-address value as its sole explicit argument.
- Produces a 32-bit result.
- Has that result stored at record offset 4.
- Determines whether this function returns 0 or 15.

Its implementation, memory effects, ownership rules, and interpretation of zero are unknown beyond this caller’s handling.

### Indirect callback

No opaque function ID is available for the dynamically selected target.

Established behavior at this call site:

- Invoked only when selector 4 is active and the initial record-field read is nonzero.
- Selected through the object’s first word and dispatch entry at byte offset 4.
- Invoked before this function clears the record field.
- Has no explicit arguments shown.
- Has no return value consumed by this function.

The target’s identity and calling convention are documented_only unknowns.

## Acceptance examples

These examples are conditional expectations derived from the supplied source, not executed tests.

| Conditions | Required behavior on normal completion |
|---|---|
| Selector is 3; record address is otherwise unusable | Return 1 without dereferencing that address or invoking either dependency. |
| Selector is 4; field initially zero; F-61e23c4aaa8d returns nonzero value R | Skip the callback and preliminary clear; store R; return 0. |
| Selector is 4; field initially zero; F-61e23c4aaa8d returns zero | Skip the callback and preliminary clear; store zero; return 15. |
| Selector is 4; field initially designates a valid object; callback returns normally; F-61e23c4aaa8d returns R | Invoke the callback, clear the field, invoke F-61e23c4aaa8d, store R, and return according to whether R is zero. |
| Callback changes the field to another nonzero value and returns normally, leaving the record writable | Overwrite that change with zero before invoking F-61e23c4aaa8d. |
| F-61e23c4aaa8d writes one value into the field but returns a different value R | The final field value is R, because this function stores the returned result after the dependency returns. |
| F-61e23c4aaa8d returns a 32-bit value whose high bit is set | Store that nonzero value and return 0; no sign-based rejection occurs. |

A nonzero initial field does not by itself satisfy the active-path preconditions. The reachable dispatch structure and callback must also be valid.

## Uncertainties

### Established static evidence

The selector gate, field offset, indirect-dispatch structure, call order, stores, and three return values are directly supported by the supplied function body.

### Inferred intent

“Replacement,” “cleanup,” and “creation” are useful conceptual descriptions, but only replacement is directly reflected by the stores. Cleanup and creation remain unverified interpretations of the dependencies.

### documented_only unknowns

The following are not resolved by the supplied evidence:

- The semantic meaning of selector 4.
- The record’s complete type and size.
- The indirect callback’s identity, ABI, and implicit arguments.
- Ownership of the old and new values.
- Effects of either dependency on globals, other fields, or memory lifetime.
- Reentrancy and concurrency requirements.
- Exception, fault, or nonlocal-exit behavior.
- Exact machine-level load count and atomicity.
- Portability beyond the displayed 32-bit representation.

## Evidence

This contract is based solely on static inspection of the supplied body for F-f4cda3afe0da and the supplied opaque identity for F-61e23c4aaa8d.

No dependency implementation was available. No runtime tests have been performed. Acceptance examples express conditional consequences of the visible behavior, not observed executions.
