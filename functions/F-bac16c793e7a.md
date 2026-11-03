<!-- Trusted provenance: F-bac16c793e7a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bac16c793e7a

## Purpose

F-bac16c793e7a obtains an object through F-17fab3ca59b5, queries a 32-bit unsigned quantity from that object, and dispatches an update to the same object. The update receives a quantity derived from the query result and twice the second input, together with a fixed argument of 1.

The established numerical behavior is an unsigned, zero-clamped subtraction after the doubling operation has been reduced to 32 bits. This is not necessarily equivalent to subtracting twice a signed mathematical integer and clamping the result.

The apparent intent is to reduce an object-associated quantity by an amount determined by the second input. The meanings of the queried quantity, the update operation, and the fixed argument are not established. In particular, the supplied evidence does not establish that these quantities represent bytes, elements, capacity, position, or any other application-specific unit.

## Interface

### Inputs

The function has two explicit inputs:

- **First input:** a 32-bit value. The visible body does not inspect it, dereference it, forward it, or use it to select behavior.
- **Second input:** an input represented as a signed 32-bit integer. Its bit pattern participates in 32-bit arithmetic, and the doubled result is compared as an unsigned quantity.

No pointer, buffer, or caller-owned storage is explicitly accepted through either input. The first input therefore has no visible validity requirement beyond satisfying the calling convention. A pointer-shaped value supplied there is not dereferenced by this function.

### Output

There is no defined explicit return value. Normal completion consists of issuing the final indirect operation and returning without exposing its result.

The function provides no visible status code, error value, or direct result containing the computed quantity. That quantity is observable as an argument to the final indirect operation.

### Required object layout

F-17fab3ca59b5 supplies an object reference. The visible accesses require:

- A readable dispatch-table reference at byte offset 0 of the object.
- A callable entry at byte offset `0x5c` in the table used for the query.
- A callable entry at byte offset `0x54` in the table used for the update.

These are byte offsets, not method indices. The recovered representation is a 32-bit object-and-dispatch-table layout. The supplied evidence does not establish a portable ABI, object type, inheritance model, or dispatch-table extent beyond the accessed entries.

The query is invoked with the object reference as its explicit receiver argument. The update is invoked with that same receiver value, the computed 32-bit quantity, and the value 1.

## Behavior

### Acquisition and query

The first externally observable operation in the visible body is one call to F-17fab3ca59b5. Neither explicit input is forwarded to this dependency.

The returned value is used as an object reference without a null check or other visible validation. The function reads the object's dispatch-table reference, obtains the entry at byte offset `0x5c`, and invokes that entry with the object as receiver.

The query result is treated as a 32-bit unsigned quantity. No signed-range check or other validation is applied.

The receiver value obtained from F-17fab3ca59b5 is retained for the final operation. There is no second acquisition call, and the query does not provide a replacement receiver.

### Numerical transformation

Let:

- **Q** denote the unsigned 32-bit query result.
- **D** denote twice the second input, reduced modulo \(2^{32}\).
- **R** denote the quantity supplied to the final operation.

The established 32-bit transformation is:

\[
D = (2 \times \text{second input}) \bmod 2^{32}
\]

\[
R =
\begin{cases}
Q-D, & D<Q,\\
0, & Q\leq D.
\end{cases}
\]

The comparison is unsigned. When subtraction is selected, \(D<Q\), so the resulting difference is positive and representable in 32 bits.

The visible arithmetic also forms a negative-double term. At 32-bit width, adding that term to Q gives the same bit pattern as subtracting D. This supports the expression above without introducing a wider intermediate or signed comparison.

Consequences include:

- Equality produces zero.
- A doubled value greater than the query result produces zero.
- A doubled value smaller than the query result produces their exact unsigned difference.
- Doubling wraps before comparison; it does not saturate.
- Negative second inputs are not rejected.
- A zero query result always produces zero.
- The result is always between zero and Q, inclusive.

The computation uses the query result already returned. It does not query the object again before dispatching the update. If the underlying state changes in the meantime, the visible body does not reconcile the computed result with that change.

### Final dispatch

After determining R, the function reads the dispatch-table reference from the retained object again and obtains the entry at byte offset `0x54`. It invokes that entry with:

1. The retained object reference.
2. R as a 32-bit quantity.
3. The fixed value 1.

This second table lookup matters. The function does not visibly reuse the dispatch-table reference used for the query. If the query changes the object's table reference and leaves a valid object behind, the final dispatch uses the later table reference.

The final operation is attempted even when R is zero. There is no visible no-op shortcut for zero, equality, or an unchanged quantity.

On the reconstructed normal path, the function then returns. The supplied evidence contains a control-flow recovery warning at the final indirect transfer, so the exact call-versus-tail-transfer mechanism is not established. The receiver and argument values supplied to that transfer are established.

## Side effects

### Direct accesses and delegated changes

The visible body performs reads needed to resolve the two indirect operations. It contains no explicit store to object fields, caller storage, or global storage.

That does not establish that the overall operation is read-only. F-17fab3ca59b5, the query, and the update may have side effects. Their implementations are not supplied.

The observable dependency order is:

1. Call F-17fab3ca59b5.
2. Read the returned object's table reference and query entry.
3. Invoke the query and obtain Q.
4. Compute R from Q and the second input.
5. Read the retained object's current table reference and update entry.
6. Transfer control to the update with R and 1.

No explicit synchronization, rollback, allocation, deallocation, logging, or error-reporting operation appears in the visible body.

### Ownership and lifetime

No explicit ownership transfer, reference-count adjustment, or release appears. The evidence does not establish whether the acquisition dependency returns a borrowed reference, an owned reference, or a reference governed by another lifetime convention.

Successful execution requires the returned object and the dispatch data used at each access to remain valid. In particular, the query must leave the retained receiver usable for the later lookup and invocation. This is a precondition for successful execution, not a proven property of the query.

The function does not visibly release the object after the update. That absence must not be interpreted as proof that no release is required by the surrounding system.

### Failure behavior

There are no visible checks for an invalid acquisition result, unreadable table, absent entry, or incompatible target. There is also no visible local error-handling or recovery path.

If a dependency faults, fails to return, raises an exception, or otherwise transfers control away, the supplied body establishes no cleanup or compensation. It does not convert such behavior into a defined return status.

## Dependencies

### F-17fab3ca59b5

Called once before either indirect operation, with no explicit arguments shown. Its result supplies the receiver used throughout the remainder of the function.

Its selection policy, side effects, failure modes, and ownership convention are unknown. It is not established to return a singleton, fresh allocation, thread-local object, or global object.

### Unidentified indirect query dependency

Selected through byte offset `0x5c` in the receiver's dispatch table. It receives the receiver and supplies the 32-bit quantity interpreted as Q.

No opaque function ID is supplied for this target. Its semantic meaning and effects on object or global state are unknown; it must not be assumed pure.

### Unidentified indirect update dependency

Selected through byte offset `0x54` in the receiver's table as read after the query. It receives the retained receiver, R, and 1.

No opaque function ID is supplied for this target. The evidence establishes an invocation with these arguments, not that a particular field is assigned R or that a requested state change succeeds.

## Acceptance examples

These are conditional contract examples, not runtime observations. They assume valid dependency targets, a returning query, and a receiver that remains usable through the final dispatch.

| Second input | Query result Q | Doubled value D | Required update quantity R |
|---:|---:|---:|---:|
| 3 | 20 | 6 | 14 |
| 3 | 6 | 6 | 0 |
| 3 | 5 | 6 | 0 |
| 0 | 20 | 0 | 20 |
| 7 | 0 | 14 | 0 |
| −1 | 100 | 4,294,967,294 | 0 |
| −1 | 4,294,967,295 | 4,294,967,294 | 1 |
| −2,147,483,648 | 20 | 0 | 20 |
| 2,147,483,647 | 4,294,967,295 | 4,294,967,294 | 1 |

The minimum signed input illustrates wraparound: doubling its 32-bit representation produces zero. Treating it as an unbounded negative integer would produce a different result and would not satisfy this contract.

Additional conditional acceptance criteria:

- Changing only the first input must not change the visible body's dependency arguments or arithmetic.
- Each normally progressing invocation must perform one acquisition and one query before the update.
- A zero result must still be supplied to the final indirect operation.
- If the query replaces the receiver's table reference with another valid table, the update target must come from that later table.
- If an update target returns a value normally, that value must not become a defined explicit result of this function.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings and units of Q and R.
- Meaning of the fixed argument 1.
- Valid caller input ranges beyond the recovered 32-bit representations.
- Ownership and lifetime guarantees of the acquisition dependency.
- Side effects and failure behavior of all dependency targets.
- Whether the update changes state, rejects its arguments, or performs another operation.
- Thread-safety, reentrancy, and synchronization requirements.
- Exact ABI details and the final indirect transfer's call-versus-tail-transfer form.

The arithmetic contract describes the recovered 32-bit behavior. It does not claim that signed-overflow expressions in a higher-level implementation language are portable or defined. An implementation must preserve the specified bit-width behavior rather than rely on unspecified overflow assumptions.

## Evidence

**Established static evidence:** the supplied body shows one acquisition call, two dispatch-table lookups at the stated byte offsets, an unsigned query result, 32-bit doubled-input arithmetic, unsigned comparisons, the resulting update quantity, and the fixed final argument 1. The first explicit input is unused.

**Inferred intent:** the operation appears to reduce an object-associated quantity with a lower bound of zero. No stronger application-level interpretation is supported.

**Unknowns:** dependency implementations, ownership rules, runtime validity guarantees, and exact final-transfer mechanics are not established.

No runtime tests have been performed. The acceptance examples are derived from the static arithmetic and call ordering.
