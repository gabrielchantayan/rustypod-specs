<!-- Trusted provenance: F-5b103afaa2d2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b103afaa2d2

## Purpose

F-5b103afaa2d2 resolves a 32-bit unsigned identifier to a 32-bit result using one of two mechanisms:

- Identifiers from 0 through 649 use a fixed-stride table.
- Identifiers from 650 through 4,294,967,295 use a lookup dependency, provided a configured lookup context is nonzero.

The function distinguishes certain unsuccessful resolutions by whether it calls an error-reporting dependency. An unavailable lookup context produces a zero result without that call. An invalid direct-table entry or an unsuccessful delegated lookup invokes the error-reporting dependency before returning zero, if that dependency returns normally.

A zero result is not, by itself, proof of failure: either successful resolution mechanism can also supply a stored zero value.

These are static behavioral findings. The semantic meaning of the identifiers, stored results, table validity field, and lookup context is not established.

## Interface

**Input**

One unsigned 32-bit identifier. Its full range is 0 through 4,294,967,295. No signed interpretation or negative-identifier behavior is established.

**Output**

One 32-bit result word. Depending on the selected path, this is:

- A word read from a direct-table entry.
- A word read through a nonzero lookup result.
- Zero when the lookup context is unavailable.
- Zero after error reporting, assuming the reporting dependency returns normally.

The result is not established to be a pointer, handle, integer status, or enumeration. Consumers must not infer a narrower semantic type from the supplied evidence.

**Required external state**

The function relies on two externally supplied bases:

1. A direct-table base.
2. A lookup-state base whose field at byte offset 4 supplies the lookup context.

For direct-table operation, each selected entry has a 24-byte stride and includes:

| Byte offset within entry | Width | Observed use |
|---|---:|---|
| 4 | 32 bits | Returned result |
| 8 | 32 bits | Nonzero eligibility condition, except for identifier zero |

The meaning of the remaining entry bytes is unknown. This function does not visibly inspect them.

For delegated operation, a successful lookup result is interpreted as an object containing a 32-bit address-bearing field at byte offset 4. The function follows that address and reads the 32-bit result at byte offset 4 of the referenced object.

**Preconditions**

The supplied source contains no explicit checks for null, inaccessible, misaligned, or corrupted addresses used in these reads. Normal completion therefore requires that each address actually used on the selected path support the indicated access under the target execution environment.

No requirement for the direct table to be valid is established for an invocation that uses only the delegated path. Conversely, a direct-table invocation does not visibly require a valid lookup-state object.

## Behavior

### 1. Select the resolution mechanism

The selection is based on unsigned 32-bit arithmetic.

The direct-table condition consists of two short-circuited alternatives: the identifier is below 512, or its unsigned difference from 512 is below 138. Together, these select precisely identifiers 0 through 649.

Consequently:

- 511 uses the direct table.
- 512 also uses the direct table.
- 649 is the last direct-table identifier.
- 650 is the first delegated identifier.
- The largest unsigned 32-bit value uses delegated lookup.

Unsigned subtraction does not bring a very large identifier back into the direct-table range. For identifiers below 512, the first alternative already selects the table path.

### 2. Resolve a direct-table identifier

The function obtains the direct-table base and selects the entry at a byte displacement equal to 24 times the identifier.

Within the selected identifier range, this displacement is at most 15,576 bytes. The multiplication itself therefore does not overflow a 32-bit unsigned value. This does not establish that adding the displacement to the configured base produces a valid address.

Identifier zero is special:

- The entry’s field at offset 8 is not read for the eligibility decision.
- The function reads and returns the 32-bit word at offset 4.

For identifiers 1 through 649:

1. The function reads the 32-bit field at entry offset 8.
2. Any nonzero bit pattern permits the result read.
3. If permitted, the function reads and returns the 32-bit word at entry offset 4.
4. If the eligibility field is zero, the function proceeds to error reporting instead.

The eligibility test is zero versus nonzero, not a test for positivity or equality to one. For example, a field with all bits set satisfies it.

A successful direct-table result is returned unchanged. There is no visible rejection of a zero result and no subsequent delegated lookup.

An ineligible direct-table entry does not fall back to delegated lookup. It goes directly to the common error-reporting path.

### 3. Check the delegated-lookup context

For identifiers of 650 or greater, the function reads a 32-bit field at offset 4 from the lookup-state base.

If that field is zero, the function immediately returns zero. On this path it does not visibly:

- Access a direct-table entry.
- Invoke the lookup dependency.
- Invoke the error-reporting dependency.

If the field is nonzero, the function prepares temporary request state and invokes F-932a35189ea0. The first argument is obtained from the lookup-state field at offset 4; the second identifies the temporary descriptor.

The supplied source expresses the context test and the context argument as separate field accesses. It does not establish a synchronized snapshot between them. In particular, the contract must not assume that externally mutable state is stable merely because the earlier test succeeded. Nor does the source establish volatile or atomic access semantics.

### 4. Prepare and submit the lookup request

Before the lookup call, the function visibly initializes:

- A 32-bit descriptor discriminator to 3.
- A descriptor pointer to an eight-byte temporary scratch region.
- A separate 32-bit temporary value to the original identifier.

The eight-byte scratch region is not visibly initialized before the call.

The recovered temporary placement suggests a descriptor consisting of the discriminator followed by a pointer, and an identifier immediately following the eight-byte scratch region. That physical relationship is relevant to a possible lookup-key layout, but it is not an independently established interface declaration. The uncertainty is recorded below rather than treating all implied stack adjacency as a guaranteed public structure.

F-932a35189ea0 receives access to temporary storage that exists during this invocation. The supplied body does not establish whether the dependency reads, writes, retains, or otherwise interprets that storage.

### 5. Interpret the lookup result

The lookup dependency’s result is tested only for zero versus nonzero.

If it is zero, the function proceeds to error reporting.

If it is nonzero, the function:

1. Reads the 32-bit address-bearing field at byte offset 4 of the returned object.
2. Uses that field as the base of another object.
3. Reads and returns the 32-bit word at byte offset 4 of that second object.

There is no visible null or validity check on the intermediate address. A nonzero lookup result alone is therefore insufficient to establish that the final result read is safe.

No additional validation of the final word is performed. A final word of zero is returned as a successful-path value, without error reporting.

### 6. Report an unsuccessful resolution

Two conditions reach the common reporting path:

- A direct-table identifier other than zero selects an entry whose field at offset 8 is zero.
- A delegated lookup returns zero after the nonzero-context path is taken.

The function invokes F-4fc89ec41a6d with these five positional values:

| Argument position | Value |
|---|---:|
| 1 | 8 |
| 2 | 102 |
| 3 | 101 |
| 4 | 0 |
| 5 | 0 |

If the dependency returns normally, F-5b103afaa2d2 returns zero.

The supplied evidence does not establish that F-4fc89ec41a6d always returns. Termination, exceptional control transfer, or other behavior inside that dependency remains outside this function’s established contract.

## Side effects

The function visibly writes only its own temporary request storage. It contains no explicit assignment to a global field or table entry.

Its external reads are path-dependent:

- Direct-table success reads the selected result field, preceded by the eligibility field for nonzero identifiers.
- Direct-table failure reads the eligibility field but does not read the result field.
- Delegated operation reads the lookup-context field and may subsequently read two fields through the lookup result.
- An unavailable lookup context avoids both dependency calls.

Calls to F-932a35189ea0 and F-4fc89ec41a6d may have additional side effects. Their implementations are not supplied, so this contract does not assume that either preserves globals, avoids allocation, releases memory, or leaves caller-visible state unchanged.

No allocation, release, ownership transfer, or result copying beyond the returned 32-bit word is explicit in this function. No release of a nonzero lookup result is visible before return. This absence does not establish that such a release is required or that a leak exists.

Temporary request storage ceases to be valid when the invocation ends. Whether the lookup dependency is permitted to retain its address is unknown; safe retention beyond that lifetime is not established.

There is no visible locking, synchronization, retry, or rollback behavior.

## Dependencies

| Opaque function ID | Invocation condition | Established interaction | Unknowns |
|---|---|---|---|
| F-932a35189ea0 | Delegated identifier and a nonzero context test | Receives the context field value and a pointer to a temporary descriptor; a zero result triggers reporting, while a nonzero result is dereferenced | Request interpretation, mutation, ownership, retention, global side effects, and complete result-object guarantees |
| F-4fc89ec41a6d | Ineligible direct-table entry or zero lookup result | Receives the five fixed values listed above; the caller returns zero if the dependency returns | Reporting destination, global effects, argument meanings, and whether it returns normally |

Neither dependency is called on a successful direct-table path. Neither is called when the delegated context field is zero.

## Acceptance examples

These are conditional static acceptance cases, not reports of executed tests. Each assumes valid memory for all accesses taken and normally returning dependencies unless stated otherwise.

1. **Identifier zero bypasses eligibility.**  
   Given identifier 0 and a result word of 37 at offset 4 of entry zero, the function returns 37 even if that entry’s field at offset 8 is zero. Neither dependency is called.

2. **A nonzero eligibility value is sufficient.**  
   Given identifier 1, an eligibility field of `0xFFFFFFFF`, and a stored result of 92, the function returns 92. The eligibility value is not rejected because its signed interpretation would be negative.

3. **The direct-table upper boundary is inclusive.**  
   Given identifier 649 and a nonzero eligibility field, the function returns that entry’s result. It does not consult the lookup context.

4. **An ineligible table entry does not trigger fallback lookup.**  
   Given identifier 649 and a zero eligibility field, the function calls F-4fc89ec41a6d with 8, 102, 101, 0, 0. If that call returns, the result is zero. F-932a35189ea0 is not called.

5. **The next identifier uses delegated lookup.**  
   Given identifier 650 and an unavailable lookup context, the function returns zero without either dependency call. Table entry 650 is not used.

6. **Delegated success follows both result fields.**  
   Given identifier 700, an available context, and a nonzero lookup result whose field at offset 4 refers to an object containing 1234 at offset 4, the function returns 1234 without reporting an error.

7. **A successful-path zero remains zero.**  
   If the final result field in example 6 contains zero, the function returns zero without invoking F-4fc89ec41a6d. The return value alone cannot distinguish this case from unavailable context or a normally reported failure.

8. **Lookup miss reports an error.**  
   Given an available context and a zero result from F-932a35189ea0, the function calls F-4fc89ec41a6d with the fixed arguments and then returns zero if reporting returns normally.

9. **The maximum input does not select a table entry.**  
   Identifier 4,294,967,295 follows the delegated path. It is not narrowed to a smaller identifier before request preparation.

10. **Nonzero lookup result is not complete pointer validation.**  
    If the lookup result is nonzero but its field at offset 4 is invalid, normal completion is not guaranteed. No defined recovery result is established for that case.

## Uncertainties

**documented_only — Domain meaning.** The apparent intent is identifier-to-value resolution with a direct range and an extended lookup range. The underlying domain and result type are unknown.

**documented_only — Temporary request layout.** The recovered placement suggests a 32-bit discriminator at descriptor offset 0, a pointer at offset 4, and a 32-bit identifier eight bytes after the pointed-to scratch start. The supplied representation does not independently prove a declared aggregate layout or the dependency’s access rules. Only eight scratch bytes are explicitly represented, and those bytes are not visibly initialized.

**documented_only — Memory representation.** The evidence uses 32-bit words and address-bearing fields. Endianness, alignment guarantees, and behavior on invalid memory are not specified here.

**documented_only — Context stability.** There is no established concurrency model or guarantee that separate context accesses observe the same value.

**documented_only — Ownership and dependency effects.** Neither dependency’s memory-management responsibilities or global-state behavior is known. No preservation or ownership guarantee is inferred from the caller’s lack of explicit cleanup.

**documented_only — Error semantics.** The fixed reporting arguments are established; their meanings and the reporter’s control-flow guarantees are not.

## Evidence

The findings derive solely from static inspection of the supplied function representation: unsigned range selection, a 24-byte table stride, accesses at byte offsets 4 and 8, conditional calls to the two mapped dependencies, temporary request initialization, and explicit return paths.

The combined 0–649 table range and maximum 15,576-byte entry displacement are arithmetic deductions from those operations. The proposed purpose and temporary aggregate interpretation are inferences, not independently verified interfaces.

No runtime tests, dependency implementations, external type declarations, or execution traces were available. The acceptance examples specify conditional behavior supported by the visible control flow; they do not claim observed execution.
