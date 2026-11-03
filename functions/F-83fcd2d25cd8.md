<!-- Trusted provenance: F-83fcd2d25cd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-83fcd2d25cd8

## Purpose

F-83fcd2d25cd8 resolves an unsigned numeric identifier through one of two routes:

- A fixed-stride table for identifiers from **0 through 649 inclusive**.
- An auxiliary lookup dependency for identifiers from **650 through 4,294,967,295 inclusive**.

The result is a 32-bit value that is used as an address on the direct-table route. The auxiliary route returns a 32-bit field obtained from a lookup result. The supplied evidence does not establish that both routes return objects of the same type.

Some unsuccessful resolutions invoke a reporting dependency before returning zero. An unavailable auxiliary lookup context instead produces zero without that report.

This contract describes established static behavior. Meanings such as “identifier,” “table,” “lookup,” and “reporting” are descriptive interpretations, not independently verified domain semantics.

## Interface

### Input

One unsigned 32-bit argument, here called the **identifier**.

Its full representable range is accepted by the routing logic. No signed-negative case exists within this interface. Behavior for a caller using an incompatible calling convention or argument width is outside the established contract.

### Output

One 32-bit result, represented as a signed integer in the supplied interface.

The function performs no signed comparison on a successful result. Consequently, a nonzero result with its most significant bit set remains a successful returned bit pattern; it is not rejected as negative.

Possible results are:

- The computed address of a direct-table entry.
- The 32-bit field at byte offset 4 of an auxiliary lookup result.
- Zero.

A zero return is not a unique indication that reporting occurred. It can result from an unavailable auxiliary context, a reported resolution failure, or a successful auxiliary lookup whose returned field is itself zero.

### Externally supplied state

The function depends on two global address-bearing values:

- A **direct-table base**.
- An **auxiliary-state base**, from which a 32-bit context value is read at byte offset 4.

The direct table uses a **24-byte stride**. For nonzero direct-table identifiers, a 32-bit field at byte offset 8 of the selected entry determines whether that entry is returned.

The auxiliary lookup result must support a 32-bit read at byte offset 4 whenever the lookup returns a nonzero result.

### Preconditions and memory validity

For defined normal execution:

- Required global address-bearing values must be readable.
- Each address that the function actually dereferences must designate readable storage of sufficient extent.
- Any alignment requirements of the target execution environment must be satisfied.
- Dependencies must satisfy the calling conventions expected by the caller.
- The auxiliary dependency must return a value usable for the subsequent offset-4 read whenever its result is nonzero.

The function does not validate these conditions. In particular, checking that a lookup result is nonzero does not establish that its storage is valid.

No ownership transfer, allocation requirement, or release obligation is established for the returned value.

## Behavior

### 1. Route selection and arithmetic

The identifier is routed to the direct table when either:

- It is less than 512; or
- It is at least 512 and its unsigned difference from 512 is less than 138.

For a 32-bit unsigned input, this is exactly the interval **0–649 inclusive**. Every larger identifier uses the auxiliary route.

The subtraction does not create an unintended low-identifier case: identifiers below 512 already satisfy the first condition. Identifier 650 is the first auxiliary-route input.

For direct-table identifiers, the byte displacement is the identifier multiplied by 24. Across that route, the displacement ranges from 0 to **15,576 bytes**. This multiplication does not overflow 32 bits.

The displacement is added to the direct-table base. The evidence provides no bounds check on the resulting address and no check for address-space wraparound. The range of the displacement alone does not prove the resulting address is valid.

### 2. Direct-table route

The function obtains the direct-table base and computes the selected entry address before testing the special case for identifier zero.

For **identifier zero**, it immediately returns the computed address:

- It does not inspect the entry’s offset-8 field.
- It does not invoke the auxiliary dependency.
- It does not invoke the reporting dependency.

Thus, zero is a special identifier, not an automatic failure value. If the direct-table base itself has a zero bit pattern, identifier zero returns zero without reporting.

For **identifiers 1–649**, the function reads one 32-bit field at byte offset 8 of the selected entry.

- If the field is nonzero, the function returns the selected entry address.
- If the field is zero, it invokes the reporting dependency with the fixed arguments described below, then returns zero if that dependency returns normally.

The field is tested only for zero versus nonzero. No narrower Boolean encoding, sign restriction, or relationship between that field and the returned address is established.

A rejected direct-table entry does **not** fall back to auxiliary lookup.

### 3. Auxiliary-context availability

For identifiers of at least 650, the function reads the context field at byte offset 4 from the auxiliary-state base.

If that field is zero, it immediately returns zero:

- No auxiliary lookup occurs.
- No reporting call occurs.
- No direct-table entry is inspected.

If the field is nonzero, the function prepares local lookup state and proceeds to the auxiliary dependency.

The context supplied to that dependency is obtained through another evaluation of the auxiliary-state base and its offset-4 field. The supplied expression therefore contains an availability read followed by a later context read for the call. It does not explicitly retain the first context value for reuse.

Whether both evaluations become distinct physical memory loads is not established here. If external mutation is possible, the evidence supplies no synchronization or guarantee that the values agree. In particular, it does not establish a second nonzero check on the context ultimately passed to the dependency.

### 4. Auxiliary lookup preparation

Before calling F-932a35189ea0, the function prepares local state containing:

- An 8-byte scratch region.
- A 32-bit copy of the original identifier.
- A 32-bit word set to **3**.
- A pointer to the scratch region.

No initialization of the scratch bytes is visible. No initialization of other potentially relevant surrounding storage is established.

The dependency receives:

1. The auxiliary context value.
2. The address of the local word containing 3.

The supplied evidence does not independently define a complete request structure. In particular, the identifier copy and scratch pointer are prepared locally, but neither is a separate explicit argument. Their precise accessibility and meaning depend on the request layout expected by F-932a35189ea0.

It would therefore exceed the evidence to specify a fully initialized descriptor, a particular scratch-buffer interpretation, or an exact ABI for every surrounding field.

### 5. Auxiliary lookup result

If F-932a35189ea0 returns a **nonzero** result, F-83fcd2d25cd8 reads the 32-bit field at byte offset 4 of that result and returns the field.

There is no subsequent validation of the field:

- A zero field is returned as zero without reporting.
- A nonzero field is returned unchanged as a 32-bit value.
- No direct-table eligibility check is applied.
- No visible release operation is performed on the lookup result.

If F-932a35189ea0 returns **zero**, the function invokes the reporting dependency and then returns zero if that dependency returns normally.

### 6. Reporting path

The reporting dependency is F-4fc89ec41a6d. It receives five fixed integer arguments, in order:

**8, 103, 101, 0, 0.**

This call occurs only on the following visible paths:

- A nonzero direct-table identifier selects an entry whose offset-8 field is zero.
- An auxiliary lookup is attempted and returns zero.

The dependency’s return value is ignored. If it returns normally, F-83fcd2d25cd8 returns zero.

The evidence does not establish whether the call logs, changes error state, raises an exception, terminates execution, or performs some other operation. A normal zero return from the enclosing function is conditional on the dependency returning control.

## Side effects

### Direct effects established in this function

The visible body:

- Reads global address-bearing state and selected memory fields.
- Initializes local lookup-related state on the auxiliary lookup path.
- Calls dependencies on the paths described above.

It contains no explicit write to the direct table, auxiliary-state storage, or lookup-result storage.

There is no visible allocation, deallocation, reference-count adjustment, or ownership handoff.

### Effects mediated by dependencies

The auxiliary dependency receives access to local storage through the address passed to it. Whether its expected layout also exposes the scratch region and identifier copy is not fully established.

Neither dependency is known to preserve globals, leave memory unchanged, or return normally. Their effects must not be narrowed to read-only behavior based solely on this caller.

The scratch region and other local objects have invocation-local lifetime. The caller does not establish that retaining their addresses after return is safe. There is no explicit contract here authorizing asynchronous use or retention.

### Ordering constraints

The following orders are established:

- Direct-table address computation precedes the identifier-zero early return.
- For nonzero direct-table identifiers, the offset-8 read precedes any reporting call.
- Auxiliary-context availability is checked before lookup preparation and lookup invocation.
- Lookup invocation completes before its nonzero result is dereferenced at offset 4.
- A reporting call precedes the final zero return on reported-failure paths.

No global state is explicitly restored after a dependency call.

## Dependencies

| Opaque ID | Invocation condition | Inputs | Result handling |
|---|---|---|---|
| F-932a35189ea0 | Identifier is at least 650 and the initial context check is nonzero | Context value and address of the local word initialized to 3 | Zero leads to reporting; nonzero is used for a 32-bit read at byte offset 4 |
| F-4fc89ec41a6d | Rejected nonzero direct-table entry, or zero auxiliary lookup result | Five fixed integers: 8, 103, 101, 0, 0 | Return value ignored; enclosing function returns zero if control returns |

No additional dependency behavior is established. In particular, this contract does not assign ownership of lookup-result storage to either party.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, all accessed storage is valid and dependencies return normally.

1. **Identifier zero bypasses entry validation.**  
   Given a direct-table base of address A, identifier 0 returns A. The offset-8 field is not read, even if its stored value would be zero.

2. **Lowest nonzero direct identifier.**  
   Identifier 1 selects the entry at A plus 24 bytes. If its offset-8 field is nonzero, the function returns that entry address without calling either dependency.

3. **Rejected direct entry does not trigger fallback.**  
   Identifier 511 selects the entry at A plus 12,264 bytes. If its offset-8 field is zero, the function calls F-4fc89ec41a6d with the fixed five arguments and returns zero. It does not call F-932a35189ea0.

4. **The intermediate arithmetic boundary remains direct.**  
   Identifier 512 selects the entry at A plus 12,288 bytes. It follows the same offset-8 validation rule as other nonzero direct identifiers.

5. **Last direct identifier.**  
   Identifier 649 selects the entry at A plus 15,576 bytes. A nonzero offset-8 field causes that address to be returned.

6. **First auxiliary identifier with unavailable context.**  
   For identifier 650, if the initial auxiliary context field is zero, the function returns zero without calling either dependency.

7. **Successful auxiliary result.**  
   For identifier 650 with an available context, suppose F-932a35189ea0 returns address R and the 32-bit field at R plus 4 bytes contains V. The function returns V without reporting.

8. **Successful lookup can still return zero.**  
   Under the preceding conditions, if V is zero, the function returns zero without calling F-4fc89ec41a6d.

9. **Failed auxiliary lookup.**  
   For identifier 1,000 with an available context, a zero result from F-932a35189ea0 causes the fixed reporting call, followed by a zero return.

10. **Maximum unsigned identifier remains an auxiliary input.**  
    Identifier 4,294,967,295 uses the auxiliary route. The local identifier copy retains that 32-bit value; no truncation to a smaller identifier width is visible.

11. **Nonzero signed-negative field is accepted.**  
    A direct entry’s offset-8 field with bit pattern `0x80000000` passes the nonzero check. Its sign bit does not cause rejection.

## Uncertainties

The following are **documented_only**:

- The application-level meaning of the identifiers and returned values.
- The complete direct-entry layout beyond the 24-byte stride and tested offset-8 field.
- The complete auxiliary-state and lookup-result layouts.
- The complete request layout expected by F-932a35189ea0, including how it accesses the prepared identifier and scratch pointer.
- Whether the value 3 denotes a type, operation, count, or another concept.
- Whether uninitialized scratch bytes are read by the dependency.
- Ownership, validity duration, and mutability of returned storage.
- Dependency side effects, failure mechanisms, and normal-return guarantees.
- Concurrency rules, synchronization, volatility, and the physical load behavior of repeated context expressions.
- Behavior after invalid memory access or calling-convention violations.

Inferred intent is that the function combines direct indexing with an auxiliary lookup and reports selected misses. That interpretation does not add fallback behavior, ownership rules, or error semantics absent from the established operations.

## Evidence

The supplied static body establishes the routing predicates, constants, 24-byte stride, memory-field offsets, local preparation, dependency arguments, early returns, and final result handling described above.

The direct-route interval and maximum displacement are arithmetic derivations from those predicates and constants. Dependency purposes and domain terminology remain inferred where explicitly identified.

No implementation execution, runtime inspection, or tests have been performed. No acceptance example above is presented as an observed runtime outcome.
