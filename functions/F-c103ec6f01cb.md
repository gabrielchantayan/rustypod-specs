<!-- Trusted provenance: F-c103ec6f01cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c103ec6f01cb

## Purpose

F-c103ec6f01cb conditionally obtains a record, increments its first 32-bit field, and returns its second 32-bit field. Two calls to F-890414d1fd99 surround the descriptor checks, lookup, and increment. The returned field is read after the second call.

The function derives a signed selection value from its input. A descriptor-field nonzero test and a strict signed upper-bound comparison determine whether lookup occurs. If lookup is skipped or returns null, the function returns zero after the closing notification, assuming execution reaches that point normally.

The descriptor access has an important distinction: the function first reads the shared descriptor location to retain a base for a possible later lookup-argument read. It then references the shared location again to obtain the base for the initial descriptor-field gate. The supplied body does not establish that these two source-level reads must produce the same address.

Interpreting the record increment as reference acquisition, the returned field as a handle, or the notifications as synchronization is inferred intent only. None of those interpretations is required by the established behavior.

## Interface

### Input and selection value

The function accepts one signed 32-bit integer.

The selection value is:

- Zero when the input is zero.
- The negation of the input plus one when the input is nonzero, with the addition performed before negation.

For representable arithmetic:

| Input | Selection value |
|---:|---:|
| 0 | 0 |
| −1 | 0 |
| −2 | 1 |
| −3 | 2 |
| 1 | −2 |
| 2 | −3 |
| −2,147,483,648 | 2,147,483,647 |

For input 2,147,483,647, the intermediate addition exceeds the signed 32-bit range. Exact behavior is **documented_only**; this contract does not replace the displayed arithmetic with wrapping arithmetic or an algebraically equivalent expression.

There is no visible rejection of negative selection values.

### Output

The result is one signed 32-bit integer:

- Zero if no non-null record pointer was obtained.
- Otherwise, the record’s 32-bit field at byte offset 4, read after the closing notification returns.

Zero is ambiguous: it can indicate a skipped lookup, a null lookup result, or a successful lookup whose returned field is zero.

### Descriptor access and layout

A shared location supplies a descriptor base address. The function makes two source-level reads of that location after the opening notification:

| Read | Retained or immediate use |
|---|---|
| First | Retains a descriptor base for the later lookup-argument field read |
| Second | Supplies the base for the initial offset-4 nonzero gate |

These are distinct source-level accesses. They need not be treated as observations of different values, but equality is not an established invariant either. Treating them as the same base requires a stability assumption.

At either descriptor base, the relevant layout is:

| Byte offset | Width | Use |
|---:|---:|---|
| 4 | 32 bits | Nonzero gate when accessed through the gate base; first lookup argument when accessed through the retained base |

The gate reads this field as an integer and compares it with zero. The lookup-argument read supplies its 32-bit value to F-6bbddce1b930. Its meaning as an address, identifier, count, or other quantity is unknown.

The retained base is not necessarily dereferenced. Its offset-4 field is read only if the gate and upper-bound comparison both pass.

### Record layout

A non-null result from F-6bbddce1b930 is used as a record address:

| Byte offset | Width | Access |
|---:|---:|---|
| 0 | Signed 32 bits | Read, incremented by one, and written back |
| 4 | Signed 32 bits | Read after the closing notification and returned |

No record-size, alignment, address-validity, or lifetime check is visible.

### Preconditions for defined completion

The following conditions apply only when the corresponding operation is reached:

- The shared descriptor location must be readable for both source-level accesses.
- The gate base must permit the 32-bit read at offset 4.
- If both gates pass, the retained base must permit its separate 32-bit read at offset 4.
- A non-null lookup result must permit a read and write at record offset 0.
- The same record pointer must remain suitable for a read at offset 4 after the closing notification.
- Reached arithmetic must have defined semantics.
- Dependencies must return normally for execution to continue.

The function does not enforce these conditions. In particular, it checks neither descriptor base for null before the applicable field read.

## Behavior

### Selection and opening notification

The selection value is computed before any dependency call. The candidate record pointer is initially null.

F-890414d1fd99 is then called with four ordered arguments:

**9, 29, 0, 0**

Its result is unused.

Both shared-descriptor reads occur after this call returns. Consequently, changes made by the opening notification can affect the descriptor addresses subsequently observed.

### Retained descriptor base and gate base

After the opening notification:

1. The function reads the shared descriptor location and retains the resulting base.
2. It references that shared location again to obtain the base for the initial offset-4 field read.
3. It compares the gate field with zero.

There is no intervening explicit dependency call between the two shared-location reads. This does not establish a general guarantee of global stability, supported concurrent mutation, or two distinct machine-level memory transactions. The established distinction is in the supplied source-level access sequence.

If the gate field is zero:

- F-10b760faca36 is skipped.
- F-6bbddce1b930 is skipped.
- The retained descriptor’s field is not read.
- No record field is accessed.
- Execution proceeds to the closing notification.

A null gate base is not equivalent to a valid descriptor with a zero field: the field read occurs without a preceding null check.

### Signed upper-bound gate

If the gate field is nonzero, the function calls F-10b760faca36 without explicit arguments.

Its result is used as a signed 32-bit upper bound. Lookup is permitted only when the selection value is strictly less than that bound.

Consequences include:

- Equality skips lookup.
- A negative selection value can pass.
- A negative upper bound can also permit lookup if the selection value is still smaller.
- The descriptor field is not interpreted as the upper bound.

If the comparison fails, the candidate remains null and execution proceeds to the closing notification.

### Lookup-argument read

If the comparison succeeds, the function reads the 32-bit field at offset 4 from the retained descriptor base—the base obtained by the first shared-location read.

This is not necessarily the base used by the earlier nonzero gate.

The function does not reload the shared descriptor location at this stage. Therefore:

- Replacing only the shared descriptor address during F-10b760faca36 does not redirect this read.
- Changing the retained descriptor’s offset-4 field can change the lookup argument.
- Invalidating the retained descriptor can make this read invalid; no local lifetime protection is established.
- The value supplied to lookup need not equal the earlier gate value.
- The supplied value is not tested for nonzero again.

F-6bbddce1b930 receives, in order:

1. The freshly read 32-bit field from the retained descriptor.
2. The previously computed signed selection value.

Its result becomes the candidate record pointer.

### Record increment

A null lookup result causes no record access.

For a non-null result, the function reads the signed 32-bit field at record offset 0, adds one, and writes the result back to that field. This occurs before the closing notification.

The visible update is an ordinary read-modify-write. Atomicity, synchronization, and protection against concurrent modification are not established.

An initial field value of 2,147,483,647 makes the increment exceed the signed 32-bit range. Exact behavior is **documented_only**; wraparound is not guaranteed by this contract.

### Closing notification and result read

Every path that completes the preceding operations normally reaches F-890414d1fd99 with:

**10, 29, 0, 0**

This includes a zero gate field, a failed upper-bound comparison, a null lookup result, and a successful increment. The notification’s result is unused.

After that call returns:

- A null candidate produces zero.
- A non-null candidate causes a read of the 32-bit field at record offset 4, which becomes the return value.

The candidate pointer is retained across the closing notification. There is no repeated lookup or pointer refresh.

If the notification changes the second record field while preserving valid access, the function returns the changed value. If it invalidates the record, no visible check protects the subsequent read.

## Side effects

### Direct reads, writes, and calls

On a path reaching normal return, the source-level order is:

- Compute the selection value.
- Call the opening notification.
- Read the shared descriptor location to retain a lookup base.
- Read the shared descriptor location again for the gate base.
- Read the gate base’s offset-4 field.
- Conditionally call the upper-bound dependency.
- If the signed comparison passes, read the retained base’s offset-4 field and call lookup.
- For a non-null lookup result, read and increment the first record field.
- Call the closing notification.
- For a non-null candidate, read the second record field and return it; otherwise return zero.

There are two shared-location reads, not one. There are one or two descriptor-field reads, potentially through different bases.

The only explicit data write is the record’s first-field increment. No direct allocation, deallocation, descriptor replacement, or release operation appears in this function.

### Ownership and lifetime

The lookup result is used directly without a visible ownership-establishing operation other than the field increment, whose meaning is unknown.

The evidence does not establish:

- Whether lookup returns borrowed or owned storage.
- Whether the first field is a reference count.
- Whether either notification acquires or releases resources.
- Whether dependencies preserve the descriptor, record, or shared globals.
- Whether the caller has any release obligation.

Dependency effects must not be inferred from their absence in this function’s body.

### Errors and abnormal completion

There is no explicit error status, exception handler, rollback, or recovery path.

A null lookup result is handled locally by returning zero after the closing notification. Invalid memory, signed arithmetic overflow, and abnormal dependency termination have no locally established recovery behavior.

The closing notification is not guaranteed cleanup after failure. If an earlier operation does not complete normally, the supplied body establishes no mechanism that forces the closing call.

## Dependencies

| Opaque ID | Observed role | Invocation and result use |
|---|---|---|
| F-890414d1fd99 | Opening and closing notification | Called with `(9, 29, 0, 0)` before descriptor reads and `(10, 29, 0, 0)` before the result-field read. Results unused. |
| F-10b760faca36 | Upper-bound supplier | Called without explicit arguments only when the gate field is nonzero. Result participates in a strict signed comparison. |
| F-6bbddce1b930 | Record lookup | Called only after both gates pass. Receives the retained descriptor’s freshly read offset-4 field and the selection value. Its result is checked for null. |

These role labels describe local use, not complete dependency semantics. No dependency is assumed to preserve globals, avoid callbacks, retain or release storage, or return normally.

## Acceptance examples

These are conditional behavioral examples, not runtime results. Unless stated otherwise, memory accesses are valid, arithmetic is representable, and dependencies return normally.

### Zero gate field

If the second shared-location read supplies a base whose offset-4 field is zero, both conditional dependencies are skipped. The closing notification occurs, and the result is zero.

The retained base’s offset-4 field is not read. Its field value does not determine this gate.

### Stable shared descriptor

Assume both shared-location reads return the same base and its gate field is nonzero.

Inputs 0 and −1 both select zero. If the upper bound is 1, lookup returns a record whose first field is 7, and the closing notification leaves the second field at 42, the first field becomes 8 before the closing notification and the return value is 42.

The coincident bases in this example are an explicit assumption, not a general guarantee.

### Distinct retained and gate bases

Conditionally, suppose the first shared-location read supplies one base and the second supplies another. The retained base’s offset-4 field is zero; the gate base’s offset-4 field is nonzero.

If the upper bound permits lookup and neither relevant field changes, lookup receives zero as its first argument. The successful gate did not test the retained descriptor.

Conversely, a zero field at the gate base skips lookup even if the retained descriptor’s field is nonzero.

These examples specify the consequences of differing observations; they do not establish that concurrent mutation is supported or explain how such observations arise.

### Strict boundary and signed comparison

Input −4 selects 3. With a nonzero gate field and upper bound 3, lookup is skipped, the closing notification occurs, and the result is zero.

Input 1 selects −2. With a nonzero gate field and upper bound zero, lookup is attempted.

Input 2 selects −3. With a nonzero gate field and upper bound −2, lookup is also attempted.

### Mutation during the bound call

Suppose the gate succeeds and F-10b760faca36 changes the retained descriptor’s offset-4 field to zero while returning a permissive bound. Lookup receives zero; no second nonzero test occurs.

If the dependency instead changes only the shared descriptor address, lookup still reads from the retained base, provided that base remains valid. This remains true whether or not the earlier gate used that same base.

### Null lookup result

If both gates pass but lookup returns null, no record field is accessed. The closing notification occurs, and the result is zero.

### Closing notification affects the result

Suppose lookup returns a record with first field 5 and second field 11. The first field becomes 6 before the closing notification. If that notification changes the second field to 19 and preserves its validity, the result is 19.

If the second field is zero after the notification, the result is zero despite a successful lookup and increment.

## Uncertainties

The following remain **documented_only**:

- The intended input encoding and descriptor-field meanings.
- Whether the two shared-location reads always observe a stable address in the intended environment.
- The mapping of source-level accesses to machine-level loads.
- Pointer representation, target byte order, alignment requirements, and address-arithmetic behavior outside valid accesses.
- Overflow behavior for the maximum positive input and maximum signed first record field.
- The semantic purpose of either notification.
- Dependency effects on globals, callbacks, ownership, and object lifetime.
- Concurrent or reentrant use guarantees.
- Behavior following invalid access or abnormal dependency termination.

Paired notifications and a counter-like update suggest synchronization or lifetime accounting, but that is inferred intent rather than an established contract.

## Evidence

The supplied static body establishes the input transformation, notification arguments, two ordered source-level shared-descriptor reads, initial gate through the second observed base, later lookup-argument read through the retained first base, strict signed bound comparison, record-field widths and offsets, increment before the closing notification, and result-field read afterward.

No dependency implementations, caller contracts, allocation rules, or synchronization guarantees were supplied. Equality of the two observed descriptor bases is not established by the body.

This specification is based on static evidence only. No runtime tests have been performed.
