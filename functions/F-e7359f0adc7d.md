<!-- Trusted provenance: F-e7359f0adc7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7359f0adc7d

## Purpose

F-e7359f0adc7d updates two caller-supplied signed integer cells using an object-held quantity and two bounds reached through that object.

The established behavior is asymmetric:

- An initial first value of −1 selects derivation of the first value from the second.
- Any other initial first value selects derivation of the second value from the first.
- After that derivation, the first value is conditionally raised to a lower bound.
- The second value is then conditionally lowered to an upper bound.
- A value of −1 is exempt from the corresponding bound adjustment.

The arithmetic is consistent with conversion between inclusive interval endpoints using a span. That interpretation is **inferred intent**, not an established semantic meaning of the fields. In particular, the function does not preserve the original span after applying bounds, enforce endpoint ordering, or validate a positive span.

A dependency is called before any of this function’s explicit data reads. Its effects are unknown and can affect the values subsequently used.

## Interface

The function has three inputs and no explicit return value:

| Input | Role |
|---|---|
| Object address | Locates an object-held signed quantity and a pointer to bound-bearing storage. |
| First-cell address | Identifies a writable signed integer cell used as the first value and branch selector. |
| Second-cell address | Identifies a writable signed integer cell used as the second value. |

For this contract, the object-held quantity is called the **span**, and the two indirect fields are called the **lower bound** and **upper bound**. These names describe their use here, not independently verified domain meanings.

### Widths and layout

The supplied representation uses signed `int` accesses and a 32-bit-style address layout. Under that representation:

- Each value cell occupies four bytes and holds a signed 32-bit integer.
- The span is a signed 32-bit field at byte offset 0x20 from the object address.
- A four-byte address-valued field is read at byte offset 0x04 from the object address.
- The lower bound is a signed 32-bit field at byte offset 0x38 from the address obtained through that field.
- The upper bound is a signed 32-bit field at byte offset 0x3C from the address obtained through that field.

The underlying ABI, byte order, alignment requirements, and exact machine behavior for exceptional arithmetic are not independently supplied. The four-byte layout is the static interpretation of the provided representation, not a separately verified platform declaration.

### Preconditions

For an ordinary, defined execution:

- The object and all storage actually reached through it must support the required reads.
- Both value-cell addresses must support the reads and writes selected by the execution.
- Required accesses must satisfy the platform’s alignment and memory-access rules.
- Arithmetic must remain within the applicable signed integer range unless the actual implementation’s overflow behavior is separately established.
- The preliminary dependency must return normally for the described local behavior to occur.

No non-aliasing requirement is established. The two value-cell addresses may identify the same cell, or overlap object-related storage, but such cases must be evaluated using the actual access sequence. They cannot safely be modeled as independent input values followed by simultaneous output assignment.

No ownership transfer, allocation, release, or retention of either value-cell address is visible in the local body. The dependency’s ownership behavior is unknown.

## Behavior

### Preliminary dependency call

F-e7359f0adc7d first calls F-166b2cfe94fd without explicit arguments.

This occurs before the function explicitly reads either value cell, the span, or the bound-bearing address field. Consequently, initial values for the remaining behavior are the values present **after that call returns**, not necessarily the values present at entry.

No returned value from the dependency is used. This does not establish that the dependency has no return value or side effects.

### Selecting and applying the derivation

The function reads the first cell and compares it with signed −1.

If the first value equals −1:

- The second value and span are read.
- The first cell receives the second value minus the span, followed by addition of one.
- The second cell is not directly assigned in this phase.

Otherwise:

- The first value and span are read.
- The second cell receives the first value plus the span, followed by subtraction of one.
- The first cell is not directly assigned in this phase.

For distinct cells and stable object fields, the mathematical results are:

- Sentinel-selected branch: first result = second input − span + 1.
- Other branch: second result = first input + span − 1.

These formulas apply only where their signed arithmetic is defined. The grouping matters: subtraction precedes addition of one in the first branch; addition precedes subtraction of one in the other branch. A mathematically representable final result does not establish safe execution if an intermediate result exceeds the signed range.

The second input’s original value is irrelevant to the derivation when the first input is not −1, except where aliasing or external effects make it relevant to some other access.

### Applying the lower bound

After the derivation write, the function reads the object’s address-valued field and obtains the lower bound from offset 0x38 of the reached storage.

It then compares the current first value with that bound using signed ordering.

- If the first value is not below the bound, no lower-bound write occurs.
- If it is below the bound, the function checks whether the first value is −1.
- Only a value that is both below the bound and different from −1 is replaced by the bound.

The lower-bound value used for the comparison is also the value assigned if replacement occurs.

A computed first value of −1 therefore remains −1 even if it lies below the lower bound. Sentinel treatment is based on the current numeric value, not on whether that value originated as a sentinel.

### Applying the upper bound

After lower-bound processing, the function reads the object’s address-valued field again and obtains the upper bound from offset 0x3C of the storage reached by that later read.

It compares the upper bound with the current second value using signed ordering.

- If the second value is not above the bound, no upper-bound write occurs.
- If it is above the bound, the function checks whether the second value is −1.
- Only a value that is both above the bound and different from −1 is replaced by the bound.

The function then returns without an explicit result.

The second lookup of the address-valued field is significant. The function does not establish a single immutable bounds-object snapshot. A preceding aliased write can change the address used for the upper-bound access.

### Consequences and limits

There is no recomputation after either bound adjustment.

With distinct value cells, raising the first value does not cause the second value to be recalculated. Lowering the second value does not cause the first value to be recalculated.

The function does not establish any of the following:

- First value no greater than second value.
- Positive or nonzero span.
- Lower bound no greater than upper bound.
- Preservation of an inclusive interval length.
- Clamping both values into the same closed interval.
- Rejection or normalization of negative values other than the special treatment of −1.

Only the first value is checked against the lower bound. Only the second value is checked against the upper bound.

## Side effects

### Local writes

Excluding the dependency, writes are limited to the two caller-supplied value-cell addresses:

1. One derivation write always occurs after the branch is selected.
2. A first-cell lower-bound write occurs conditionally.
3. A second-cell upper-bound write occurs conditionally.

There is no visible allocation, deallocation, logging, explicit error reporting, or direct global write in the local body.

### Observable access order

The established phase order is:

1. Dependency call.
2. First-cell sentinel check.
3. Derivation reads and selected destination write.
4. Lower-bound lookup.
5. Lower comparison, optional sentinel check, and optional first-cell write.
6. Fresh bound-storage address lookup and upper-bound read.
7. Upper comparison, optional sentinel check, and optional second-cell write.
8. Return.

The sentinel checks inside the bound conditions are conditional reads: they occur only if the preceding strict comparison succeeds. Equality with a bound causes no bound write.

The representation does not establish an ordering between independent operand reads within an arithmetic expression. Nor does it establish atomicity, volatile semantics, or a concurrent-access protocol.

### Aliasing

If both value addresses identify the same integer cell, derivation changes the value later used by both bound phases. The function then effectively applies the lower-bound phase and upper-bound phase to the same evolving cell.

If a destination overlaps the span, the address-valued field, or bound storage, earlier writes can change later reads. In particular:

- A derivation write can change the subsequent lower-bound lookup.
- A lower-bound write can change the address or value used for the upper-bound lookup.
- A changed address-valued field can cause the upper-bound access to reach different storage.

There is no alias detection or recovery path.

## Dependencies

### F-166b2cfe94fd

Established call contract at this call site:

- Called exactly once before the local processing, if execution reaches the call.
- Receives no explicit arguments.
- Has no returned value consumed here.

The following are **documented_only**:

- Its purpose.
- Whether it reads or changes globals.
- Whether it accesses the object or value cells through other references.
- Whether it allocates, frees, retains, or invalidates memory.
- Whether it can fail, terminate execution, or decline to return.
- Any synchronization or initialization guarantee it might provide.

The local function neither checks a status from this dependency nor performs visible recovery around it.

## Acceptance examples

Unless stated otherwise, these examples assume distinct value cells, stable disjoint object storage, a normally returning dependency that does not alter the relevant values, and representable signed arithmetic. Values shown are those present after the dependency returns.

| Case | Span | Lower | Upper | First input | Second input | Final first | Final second |
|---|---:|---:|---:|---:|---:|---:|---:|
| Derive first without clipping | 10 | 3 | 20 | −1 | 14 | 5 | 14 |
| Derive second without clipping | 10 | 3 | 20 | 5 | 999 | 5 | 14 |
| Raise first without recomputation | 10 | 3 | 20 | 2 | 999 | 3 | 11 |
| Lower second without recomputation | 10 | 3 | 20 | 15 | 999 | 15 | 20 |
| Preserve computed sentinel | 10 | 3 | 20 | −1 | 8 | −1 | 8 |
| Allow reversed endpoints | 10 | 30 | 20 | 5 | 999 | 30 | 14 |
| Accept zero span arithmetically | 0 | 0 | 20 | 8 | 999 | 8 | 7 |
| Do not cap first at upper bound | 2 | 0 | 20 | 30 | 999 | 30 | 20 |

Additional conditional examples:

- **Upper sentinel exemption:** With span 0, first input 0, lower bound −10, and upper bound −2, derivation produces second value −1. Although −1 is above −2, it is not replaced. Final values are 0 and −1.
- **Exact bound equality:** With span 10, lower bound 5, upper bound 14, and first input 5, derivation writes second value 14. Neither bound phase writes, because both comparisons are strict.
- **Same-cell alias:** If both value addresses identify one cell initially holding 5, with span 10 and bounds 3 and 12, derivation writes 14 into that cell. Lower processing leaves it unchanged; upper processing replaces it with 12. Both addresses then observe 12.
- **Intermediate overflow:** With signed 32-bit arithmetic, first input 2,147,483,647 and span 1 cause the addition in second-value derivation to exceed the signed maximum before subtracting one. No wrapped result is an acceptance requirement.
- **Invalid reached storage:** A null, stale, misaligned, or otherwise unreadable reached address has no specified graceful outcome. Returning an error or leaving both cells unchanged is not established behavior.

## Uncertainties

The following remain **documented_only**:

- The business meaning of the span and endpoint-like values.
- Whether −1 denotes “unspecified,” “unbounded,” or another state outside this function.
- The actual ABI and instruction-level overflow behavior.
- Byte order and alignment guarantees.
- The validity and intended lifetime of object-related storage.
- Whether callers prohibit aliasing or concurrent mutation.
- The dependency’s complete effects and failure behavior.
- The larger system’s handling of memory-access faults or arithmetic exceptions.

The visible body supplies no validation that resolves these uncertainties. They must not be replaced by assumed invariants.

## Evidence

This contract is based solely on static inspection of the supplied function representation and its dependency-ID mapping.

Established static evidence includes the preliminary dependency call, the signed −1 branch, the two derivation expressions, the field offsets, the lower-then-upper processing order, repeated lookup of the bound-storage address, conditional destination writes, and absence of an explicit returned result.

The interval and span terminology is inferred from the arithmetic pattern, not independently established domain documentation.

No runtime tests have been performed. The acceptance examples are conditional deductions from the visible operations, not observed executions.
