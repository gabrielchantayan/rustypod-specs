<!-- Trusted provenance: F-d52e9ba0fbc2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d52e9ba0fbc2

## Purpose

F-d52e9ba0fbc2 performs a selector-based, conditionally gated dispatch and publishes a separate 32-bit status value before returning a normalized success or failure result.

The established behavior has three stages:

1. Obtain a selector, even when the second input is zero.
2. Reject a zero second input, or use the selector to run a prerequisite check and, if permitted, invoke a table-selected callback.
3. Obtain a status destination and write the selected status into it.

The return value and the published status are distinct outputs. Normal success returns the 32-bit value zero and publishes zero. Normal failure returns the 32-bit pattern `0xFFFFFFFF`; the published status identifies the failure branch or contains the nonzero result of a dependency.

The semantic meaning of the selector, the second input, status value 26, and the callback table is not established. Descriptions such as “selector,” “prerequisite check,” and “status destination” identify observable roles, not a known application-level protocol.

## Interface

### Inputs

The function has two input positions.

| Input | Established representation | Established use |
|---|---|---|
| First input | A 32-bit value | Not inspected, forwarded, or otherwise used by the visible function body. |
| Second input | A 32-bit integer | Compared with zero. If nonzero and the prerequisite check succeeds, passed unchanged as the sole explicit callback argument. |

For the second input, the only visible validity test is equality with zero. Negative values and nonzero values with the high bit set are not rejected by this function merely because of their numeric representation.

The second input is not visibly dereferenced by F-d52e9ba0fbc2. Whether it represents a number, pointer, handle, or other token is unknown. Any interpretation by the callback belongs to that callback’s contract.

### Return value

The function returns a 32-bit value:

- `0x00000000` when both the prerequisite check and the selected callback return zero.
- `0xFFFFFFFF` on every other normally returning branch.

The failure result is specified as a bit pattern. Interpreting it as signed negative one is a consumer convention, not an additional distinction in the function’s output.

### Status output

On every normal completion, F-d52e9ba0fbc2:

1. Calls F-4af36158f863 with no explicit arguments.
2. Treats the returned value as the address of a writable 32-bit integer.
3. Stores the selected status at that address.
4. Returns the previously selected success or failure result.

There is no visible null check, alignment check, or validity check for the returned address. A writable destination suitable for the 32-bit store is therefore a precondition for normal completion.

The destination’s scope is unknown. It could designate shared, thread-specific, or otherwise managed storage; none of those interpretations is established by the supplied evidence.

## Behavior

### Selector acquisition occurs first

F-d52e9ba0fbc2 begins by calling F-2533eb5977bc with no explicit arguments. Its result is used as a 32-bit signed selector where subsequent processing requires one.

This call occurs before the second input is tested. Consequently, supplying a zero second input does not bypass selector acquisition or any effects that dependency may have.

The selector is obtained once. The visible body does not reacquire it after another dependency runs.

### Zero second input

If the second input equals zero:

- The selected status is decimal 26, represented as `0x0000001A`.
- The selected return value is `0xFFFFFFFF`.
- F-eb5289d344f1 is not called.
- No callback-table lookup or callback invocation is performed by the visible body.
- The status destination is subsequently obtained, and 26 is written to it before return.

This branch does not depend on the selector’s value after selector acquisition has returned.

The evidence establishes the numeric status 26 but does not establish a symbolic error name or application-level explanation for it.

### Nonzero second input: prerequisite check

If the second input is nonzero, F-d52e9ba0fbc2 calls F-eb5289d344f1 with the previously obtained selector as its sole explicit argument.

The dependency’s 32-bit integer result controls further processing:

- A zero result permits callback dispatch.
- Any nonzero result prevents callback dispatch.

If the result is nonzero, that exact 32-bit result becomes the published status, and the function selects `0xFFFFFFFF` as its return value. The result is not replaced with 26, converted to a boolean, or otherwise normalized for the status write.

Both positive and negative nonzero results select this failure branch.

### Callback selection and layout

A callback is selected only when the second input is nonzero and F-eb5289d344f1 returns zero.

The visible layout consists of:

- A global value supplying the table base.
- Entries spaced 20 bytes apart.
- A callback-pointer field beginning 8 bytes from the start of the selected entry.

The selected field is located at a displacement of 20 times the selector, plus 8 bytes, relative to the table base. The field is interpreted as an indirect call target in the supplied 32-bit representation.

Only this field and the entry stride are established. The meanings, types, and occupancy of the other bytes in each entry are unknown. A 20-byte stride does not establish five independently meaningful 32-bit fields.

The table-base access and callback-pointer fetch occur after the prerequisite check has returned zero. The visible body does not snapshot the table base before that call. Thus, the contract must not assume that dispatch uses the table state that existed before F-eb5289d344f1 ran.

The selector used for dispatch remains the value returned by the initial selector acquisition.

### Callback invocation and result

The selected callback receives the original second input as its sole explicit argument.

Its 32-bit integer result becomes the published status:

- Zero selects return value `0x00000000`.
- Any nonzero result selects return value `0xFFFFFFFF`.

The function does not visibly retry the callback, select another entry after a callback failure, or invoke the prerequisite check again.

A callback result of 26 is preserved as 26, just as any other nonzero result is preserved. The published status alone therefore does not necessarily identify which branch produced it.

### Final publication

After a normal branch selects its status and return value, F-4af36158f863 is called to obtain the destination for the final status store.

This ordering applies to all branches:

- Zero input: selector acquisition, destination acquisition, status store.
- Rejected nonzero input: selector acquisition, prerequisite check, destination acquisition, status store.
- Dispatched nonzero input: selector acquisition, prerequisite check, table lookup, callback, destination acquisition, status store.

The selected status is not passed as an explicit argument to F-4af36158f863. It is written through the returned address afterward.

The visible body performs one final 32-bit status store on normal completion. That statement does not restrict writes performed inside dependencies or callbacks.

## Side effects

The directly established effects are dependency calls, a conditional indirect call, conditional reads used to select that call, and a final 32-bit write.

No allocation, deallocation, explicit resource release, or direct write through either input appears in the visible body. This does not establish that the complete invocation is allocation-free or ownership-neutral: any called function may have additional effects that are not supplied here.

No ownership transfer is explicitly expressed by the wrapper. In particular:

- Passing the second input to the callback does not establish whether the callback retains or releases anything it denotes.
- Obtaining a status address does not establish ownership of that storage.
- The absence of an explicit release does not establish that returned storage must be released by the caller.

No rollback or compensation is visible. Effects performed during selector acquisition, the prerequisite check, or callback execution are not visibly undone if a later stage fails.

There is no visible synchronization protecting the table access or status write. Atomicity, thread safety, and visibility to other threads are undocumented.

A failed indirect call, invalid memory access, dependency that does not return, or failure during the final store has no visible recovery path. The normal return guarantees apply only when all operations required by the selected branch complete normally.

## Dependencies

| Opaque ID or role | Visible contract at this boundary |
|---|---|
| F-2533eb5977bc | Called once, first, with no explicit arguments. Supplies the selector. |
| F-eb5289d344f1 | Called once only for nonzero second input. Receives the selector. Zero permits dispatch; a nonzero result becomes the final status. |
| Table-selected callback | Invoked once only after a zero prerequisite result. Receives the second input unchanged. Its result becomes the final status. Its opaque ID is not provided. |
| F-4af36158f863 | Called once after status selection on every normally completing branch. Supplies the address used for the final 32-bit store. |

These are boundary roles, not complete dependency specifications. Their internal reads, writes, ownership behavior, and failure mechanisms are unknown. No dependency is assumed to preserve global state.

The callback table’s global base has no supplied opaque function ID because it is data, not an identified function.

## Acceptance examples

These examples are conditional static acceptance cases, not reports of execution.

### Zero input still acquires a selector

Given that selector acquisition and destination acquisition return normally, and the destination is writable, a zero second input must result in:

- Selector acquisition occurring first.
- No prerequisite-check call or callback dispatch.
- A 32-bit store of 26.
- Return value `0xFFFFFFFF`.

Changing only the unused first input must not change this function’s visible branch selection.

### Prerequisite rejection preserves its status

Given a nonzero second input and a prerequisite result of 7, normal completion must publish 7 and return `0xFFFFFFFF`, without performing the callback-table lookup or invoking a callback.

A prerequisite result represented by `0xFFFFFFFE` must likewise be preserved exactly as the status and treated as nonzero.

### Successful dispatch

Given selector 3, a zero prerequisite result, a valid table and callback target, and a callback result of zero:

- The callback field begins 68 bytes from the table base: three 20-byte strides plus the 8-byte field offset.
- The callback receives the original nonzero second input.
- The final status is zero.
- The return value is `0x00000000`.

### Callback failure preserves its status

Given that dispatch is reached and the callback returns 26, normal completion must publish 26 and return `0xFFFFFFFF`.

This status matches the zero-input branch numerically but results from a different call sequence.

### Updated table state may affect dispatch

If the prerequisite dependency changes the global table base and then returns zero, the subsequent lookup uses the base read after that dependency returns. The contract does not require selection from the earlier table state.

### Invalid dispatch storage has no defined recovery result here

If dispatch is reached but the selected callback field is unreadable or does not identify a callable target, the evidence does not establish a normal error return or guaranteed final status write. Such a case is outside the normal-completion preconditions, not an additional specified failure branch.

## Uncertainties

The following are **documented_only**:

- The application-level purpose of the selector and second input.
- The symbolic meaning of status 26.
- The selector’s valid range and the table’s entry count.
- Whether negative selectors can intentionally address valid storage.
- The other contents of each 20-byte entry.
- The callback’s identity, calling-convention details beyond the visible explicit argument, and ownership contract.
- The status destination’s lifetime, alignment guarantees, sharing model, and ownership.
- Dependency effects on globals, memory, resources, or control flow.
- Endianness and the bytewise representation of the final store.
- Concurrent modification behavior and synchronization requirements.

The supplied representation uses 32-bit integers and a 32-bit dispatch context. It does not justify evaluating the table-address calculation with unbounded arithmetic or treating the selector as an unsigned, bounds-checked index.

Multiplication by 20 and subsequent address additions can exceed a 32-bit range for sufficiently large selector magnitudes. The evidence does not establish portable signed-overflow behavior, usable wraparound semantics, or a safe address for such selectors. Acceptance of an arbitrary selector therefore requires separate evidence that the resulting field access is valid.

## Evidence

**Established static evidence:** the supplied function representation shows unconditional selector acquisition; a zero-input branch assigning status 26; a nonzero-input prerequisite call; conditional indirect dispatch through a 20-byte-stride table at field offset 8; zero-versus-nonzero result normalization; and destination acquisition followed by a 32-bit status store.

**Inferred intent:** the structure is consistent with a gated dispatcher that publishes an operation status separately from its normalized return value. This interpretation does not establish a particular subsystem, resource type, or error vocabulary.

**Verification limit:** no runtime tests have been performed. The acceptance examples specify consequences of the visible control flow under their stated preconditions; they are not observed execution results.
